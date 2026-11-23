# 在 ephemeral runner 上安全地做 iOS 代码签名:一份防证书泄漏实战指南

<!-- canonical(英文): https://github.com/macrunara/docs/blob/main/blog/03-ephemeral-runner-ios-signing/index.en.md -->
<!-- 状态:草稿。发布前对照示例仓库逐条验证命令。 -->

*代码签名把你最危险的凭证集中到了执行不可信代码的机器上。在临时化基础设施上,这可以变成特性而非缺陷 —— 前提是搭对了。本文给出威胁模型和我们推荐的完整配置。*

---

想想 iOS 发布流水线要碰什么:**分发证书私钥**(可在全球每一部 iPhone 上冒充你的公司)、**App Store Connect 凭证**(提交应用、改元数据,最坏情况下架构建)、macOS 的**公证凭证**。再想想它跑在哪:一台五分钟前刚执行过你从未审计的依赖里 `pod install` 脚本的 CI 机器。

行业的答案已经收敛到三个理念,而临时 runner 让这三条都变得更强:

1. **API Key 优于密码。** App Store Connect API Key 有权限范围、可吊销,且不需要绕开 2FA。
2. **把 secret 约束在最小的时间与空间里。** secret 在磁盘上存在的时间越短越好,边界越小越好。
3. **让机器可丢弃。** 最彻底的清理,是对整台计算机执行 `rm -rf`。

## 威胁模型(简版)

| 威胁 | 对策 |
|---|---|
| 恶意依赖在构建中途外泄签名私钥 | secret 只在签名步骤进入钥匙串,而非贯穿整个任务;私钥永不落明文 |
| 同一机器上的后续任务翻找残留 | 一任务一 VM;VM —— 连同钥匙串 —— 即毁 |
| 长期凭证被偷后在攻击者机器上复用 | App Store Connect API Key(一键吊销,不牵出密码重置连环套) |
| 令牌权限过大 | ASC Key 用最小角色;发布任务挂 GitHub 环境 + 审批人 |
| CI 日志泄漏 secret | 密钥材料走 stdin/env,绝不走命令行参数;注意 `set +x` 卫生 |

## 黄金法则:一切用 App Store Connect API Key

现代苹果发布流水线里的所有环节 —— TestFlight 上传、公证、用 fastlane 申请证书/描述文件 —— 都可以用 **App Store Connect API Key** 鉴权。一种 key 替代掉:Apple ID + 密码 + App 专用密码 + 2FA 短信杂技。

```
App Store Connect → 用户和访问 → 集成 → App Store Connect API
→ 团队密钥 → (+) → 角色:App Manager(签名 + TestFlight 足够)
```

仓库里存三个 secret:`KEY_ID`、`ISSUER_ID`、`.p8` 内容(base64)。Key ID 和 Issuer ID 不是高熵 secret;`.p8` 是 —— 注意 **Apple 只允许下载一次 `.p8`**,妥善保管。定期轮换(每季度一次是便宜的保险),有泄漏嫌疑立即轮换。

## 临时钥匙串七步舞(macOS / Developer ID 流程)

当必须导入 `.p12` 时(macOS 分发的 Developer ID 流程),在一次性钥匙串、一次性 VM 里做:

```bash
set -euo pipefail
# secret 通过环境变量传入,绝不走 argv:
#   MACOS_CERTIFICATE_P12_BASE64、MACOS_CERTIFICATE_PASSWORD、KEYCHAIN_PASSWORD
echo "$MACOS_CERTIFICATE_P12_BASE64" | base64 --decode > certificate.p12

security create-keychain -p "$KEYCHAIN_PASSWORD" ci.keychain-db
security default-keychain -s ci.keychain-db
security unlock-keychain -p "$KEYCHAIN_PASSWORD" ci.keychain-db

# -T 只允许 codesign 非交互访问这一个钥匙串
security import certificate.p12 -k ci.keychain-db \
  -P "$MACOS_CERTIFICATE_PASSWORD" -T /usr/bin/codesign

# 少了这一行,codesign 会在构建中途弹出钥匙串密码框,
# 你的无头任务永远卡住
security set-key-partition-list -S apple-tool:,apple:,codesign: \
  -s -k "$KEYCHAIN_PASSWORD" ci.keychain-db

shred -u certificate.p12 2>/dev/null || rm -f certificate.p12
```

签名时用显式身份并加**安全时间戳**(`notarytool` 拒绝没有时间戳的签名):

```bash
xcodebuild -scheme NotarizeExample -configuration Release \
  CODE_SIGN_STYLE=Manual CODE_SIGN_IDENTITY="Developer ID Application" \
  DEVELOPMENT_TEAM="$MACOS_TEAM_ID" OTHER_CODE_SIGN_FLAGS="--timestamp" \
  ENABLE_HARDENED_RUNTIME=YES build
```

公证同样走 API Key —— 全程没有 Apple ID:

```bash
xcrun notarytool submit app.dmg \
  --key AuthKey.p8 --key-id "$KEY_ID" --issuer "$ISSUER_ID" \
  --wait --timeout 30m
xcrun stapler staple app.dmg
```

这条流水线的完整可运行版本在 [macos-notarize-example](https://github.com/macrunara/macos-notarize-example) 仓库 —— fork、配 7 个 secret、推 tag 即可。

## iOS / TestFlight:cert + sigh 走 API Key,不需要 match 仓库

fastlane 的 `match` 是团队共享签名的经典答案,但它给你的威胁面多加了一个证书 git 仓库。纯 CI 分发场景下,用 API Key 驱动 `cert` + `sigh` 更精简:

```ruby
api_key = app_store_connect_api_key(
  key_id: ENV["APP_STORE_CONNECT_API_KEY_ID"],
  issuer_id: ENV["APP_STORE_CONNECT_API_ISSUER_ID"],
  key_content: ENV["APP_STORE_CONNECT_API_KEY_CONTENT"],
  is_key_content_base64: true
)
cert(api_key: api_key)
sigh(api_key: api_key, app_identifier: "com.example.app", force: true)
build_app(scheme: "App", export_method: "app-store")
upload_to_testflight(api_key: api_key)
```

完整 lane 在 [ios-swift-example](https://github.com/macrunara/ios-swift-example)。两个注意点:

- `cert` 需要用同一把 API Key 登录 Developer 门户,**App Manager** 角色够用。如果你们对门户角色拆分得很细,改用 `match` + 只读 CI deploy key 也完全没问题;
- 工程的 Release 配置应该是 `CODE_SIGN_STYLE = Manual`;Debug 保持 Automatic,本地开发永不接触这套机制。

## 为什么"用完即焚"本身是安全特性

上面每条对策都有一个失败模式:某个东西被留下了 —— 一个钥匙串文件、一个未锁定的会话、一个被人 echo 进构建日志还躺在磁盘上的 `.p8`。在长期存活的自托管 runner 上,这些残留会跨任务、跨客户永远累积。

在 Macrunara 上,清理是结构性的:任务结束,**运行它的 VM 即被销毁** —— 钥匙串、登录会话、DerivedData、shell 历史,一切。下一个任务从黄金镜像克隆全新 VM。"我们记得删 secret 了吗?"不再是个要紧的问题,因为根本没有东西可删。

配合 GitHub 侧的卫生 —— 发布 workflow 挂环境 + 审批人、secret 按环境隔离、生产环境关闭 `ACTIONS_STEP_DEBUG` —— 你得到的流水线的最坏暴露窗口是:*单个任务的时长、单台 VM 之内、一把可吊销的 key*。

## 检查单

- [ ] TestFlight + 公证 + cert/sigh 全部走 ASC API Key;CI 里不存在 Apple ID 密码
- [ ] secret 存仓库/环境 secrets,发布任务挂审批人
- [ ] 每任务一次性钥匙串(`create-keychain` → `set-key-partition-list` → 签名)
- [ ] 密钥材料走 env/stdin 不走 argv;`.p12` 在导入它的同一步骤里删除
- [ ] 送往 notarytool 的一切带 `OTHER_CODE_SIGN_FLAGS="--timestamp"`
- [ ] 签名路径上只有临时 runner,没有温热的有状态机器
- [ ] 日历上有 key 轮换提醒(每季度)

---

*本系列第三篇。第一篇讲[集群架构](https://github.com/macrunara/docs/blob/main/blog/01-m4-cluster-architecture/index.zh-CN.md),第二篇讲[构建缓存一致性模型](https://github.com/macrunara/docs/blob/main/blog/02-xcode-build-cache-consistency/index.zh-CN.md)。下一篇:TCO 账单。工具与延伸阅读见 [awesome-macos-ci](https://github.com/macrunara/awesome-macos-ci)。*
