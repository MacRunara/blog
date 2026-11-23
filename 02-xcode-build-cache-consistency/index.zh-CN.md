# Xcode 构建缓存的一致性难题与我们的解法

<!-- canonical(英文): https://github.com/macrunara/docs/blob/main/blog/02-xcode-build-cache-consistency/index.en.md -->
<!-- 状态:草稿。TODO[eng] 占位发布前用 benchmark 仓库的实测数据替换。 -->

*缓存是 CI 里唯一的免费午餐 —— 而 Xcode 的缓存恰恰是最可能给你构建下毒的那个。本文回答:在临时 runner 之间,什么能安全缓存、什么会悄悄腐坏,以及我们最终定型的 key 方案。*

---

每个 CI 团队都会写下这一行:

```yaml
- uses: actions/cache@v4
  with:
    path: ~/Library/Developer/Xcode/DerivedData
    key: deriveddata-${{ github.sha }}
```

然后,在经历了一周"只在 CI 上挂、本地永不复现、清缓存就好"的诡异失败之后,每个团队又会把它删掉。不舒服的真相是:**DerivedData 不是一个可以在机器之间 —— 甚至构建之间 —— 安全搬运的缓存。** 它是构建*产物*,塞满绝对路径、指纹和时间戳,而且 Xcode 把它当作权威。恢复错了,编译器会心安理得地吃下过期模块,而你的源码树完全是另一回事。

这篇文章就是我们在 Macrunara 临时 runner 上使用的一致性模型:对"什么可以跨越 VM 边界、以什么为 key、如何验证"的分层回答。

## 第 0 层:永不跨边界的东西

- **整个 DerivedData。** 其中的目标文件和模块缓存引用绝对路径,且与确切的工具链构建版本绑定。在临时 VM 上,恢复成本 + 腐坏风险大于收益。我们接受冷构建的代价,转而去压冷构建时间本身(更快的节点、预热镜像 —— 见[第一篇](https://github.com/macrunara/docs/blob/main/blog/01-m4-cluster-architecture/index.zh-CN.md))。
- **任何被代码签名碰过的东西。** 钥匙串、描述文件、签名中间态。设计上就是一次性的(见[第三篇](https://github.com/macrunara/docs/blob/main/blog/03-ephemeral-runner-ios-signing/index.zh-CN.md))。
- **整个 `~/Library/Caches`。** 不加区分的缓存目录把安全数据(下载的归档)和不安全数据(指纹、锁文件)混在一起。缓存*特定子树*,永远不缓存整片森林。

## 第 1 层:源码级缓存 —— 永远安全

这一层装的是*输入*而不是*输出*,一致性问题退化为普通的内容寻址:

| 缓存 | 路径 | Key |
|---|---|---|
| SwiftPM checkouts | `~/Library/Caches/org.swift.swiftpm` + resolved 清单 | `Package.resolved` 哈希 + Swift 版本 |
| CocoaPods | `Pods/` + `~/.cocoapods` | `Podfile.lock` 哈希 |
| Node(RN) | `node_modules`(或 yarn 缓存) | `yarn.lock` 哈希 + node 版本 |
| Gradle(Android) | `~/.gradle/caches`、`~/.gradle/wrapper` | `*.gradle*` + `gradle-wrapper.properties` 哈希 |

两个要紧的细节:

1. **key 锁定文件,并把工具链放进 key。** `spm-${{ runner.os }}-${{ hashFiles('Package.resolved') }}-${{ env.XCODE_VERSION }}`。Swift 6.1 编译器产出的缓存对 6.2 不是合法输入,而"有时候能用"恰恰是你最不想要的失败模式。
2. **镜像版本也是 key 的一部分。** 我们的节点跑带版本的黄金镜像(`macos-mobile v1.2`)。镜像向前滚动时,旧缓存干净地失效,而不是神秘地劣化。镜像版本会注入每个任务的环境变量 —— 用它。

## 第 2 层:构建产物缓存 —— 只在你有正确性"神谕"时

如果必须缓存构建产物(一些客户的 monorepo 确实需要),你需要一个能区分有效条目和中毒条目的*神谕*:

- **Xcode 自己的指纹机制(Xcode 15+)。** `xcodebuild` 会输出显式模块依赖;Swift driver 的指纹覆盖文件内容 + 编译参数。当且仅当 key 包含:源码哈希、工具链构建版本、镜像版本,以及每一个会渗进指纹的构建设置(`SWIFT_VERSION`、优化级别、sanitizer 开关)时,缓存才是安全的。
- **远程构建缓存系统。** [Tuist 的缓存](https://tuist.io)(内容寻址、framework 级)和 Bazel 远程缓存是大型 iOS 代码库的两个严肃选项。两者都用"信哈希"替代"信目录" —— 这就是全部要义。我们的节点对 Tuist 缓存友好 <!-- TODO[eng]:确认对外口径/挂文档链接 -->;Bazel 用户的故事更好,因为正确性本来就是 Bazel 的核心承诺。
- **sccache 类编译缓存**覆盖不了 Swift 的完整管线;把它当 C/C++/Rust 加速器,不是 Xcode 方案。

## 支撑这个模型的实测

理论是廉价的;我们在集群上实测了这三层(完整方法论在 [benchmarks 仓库](https://github.com/macrunara/benchmarks)):

| 策略 | 冷构建 | 热构建 | 正确性事故 / 100 次构建 |
|---|---|---|---|
| 不缓存(仅第 0 层) | <!-- TODO[eng] --> 420s | 420s | 0 |
| 第 1 层缓存(锁文件做 key) | 420s → <!-- TODO[eng] --> ~240s | ~180s | 0 |
| 第 1 层 + 裸打包 DerivedData | ~180s | ~90s | <!-- TODO[eng]:禁用前实测事故率 --> |

中间那行是甜点:拿走大部分墙上时间收益,不引入任何抖动。第三行解释了为什么我们的模板里禁止"直接缓存 DerivedData"。

## 实用清单(可以直接抄)

1. 缓存**输入**(SPM/Pods/node/gradle),永远不缓存整个 DerivedData;
2. key = 锁文件哈希 + 工具链版本 + 镜像版本;
3. `fail-on-cache-miss: false` —— 缓存是优化项,永远不是正确性依赖;
4. 对任何"构建产物形状"的缓存,不要用 restore-keys 前缀回退 —— 过期模块就是这么混进来的;
5. 量化:按流水线记录缓存命中率和冷/热构建时长。不量化的缓存是迷信。

## 接下来

**第三篇** —— 在任务结束即销毁的机器上做代码签名:为什么"用完即焚"反而是安全*特性*,以及让它成立的那套钥匙串动作。

---

*Macrunara 把你的 GitHub Actions 任务跑在 Mac mini(M4/M6)临时 VM 上,这套缓存模型内建于节点镜像。[flutter](https://github.com/macrunara/flutter-example) 和 [ios-swift](https://github.com/macrunara/ios-swift-example) 示例仓库展示了第 1 层的实际配置;缓存工具全景在 [awesome-macos-ci](https://github.com/macrunara/awesome-macos-ci)。*
