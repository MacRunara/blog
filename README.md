# P5 工程博客 —— 内部索引(不发布)

作战手册 P5「工程博客:把集群故事讲成技术品牌」的四篇起草稿。**每篇英文为 canonical 版,中文为掘金/知乎分发版。**

| # | 选题 | 英文 canonical | 中文版 | 主投渠道 |
|---|---|---|---|---|
| ① | 我们用 Mac mini 集群(M4 与 M6)跑 GitHub CI:架构、虚拟化与调度 | `01-m4-cluster-architecture/index.en.md` | `01-m4-cluster-architecture/index.zh-CN.md` | HN + dev.to / 掘金 |
| ② | Xcode 构建缓存的一致性难题与我们的解法 | `02-xcode-build-cache-consistency/index.en.md` | `02-xcode-build-cache-consistency/index.zh-CN.md` | dev.to / 掘金 + iOS 周报投稿 |
| ③ | 在 ephemeral runner 上安全地做 iOS 代码签名 | `03-ephemeral-runner-ios-signing/index.en.md` | `03-ephemeral-runner-ios-signing/index.zh-CN.md` | dev.to / 知乎 |
| ④ | macOS CI 成本横评:官方 / 托管 / 自建的 TCO 账单 | `04-macos-ci-tco-comparison/index.en.md` | `04-macos-ci-tco-comparison/index.zh-CN.md` | dev.to + r/iosprogramming / 知乎 |

## 发布流程(对照手册 P5 分发策略)

1. **canonical 落点**:英文版先提交到 `macrunara/docs` 仓库的 `blog/` 目录 —— 这是所有分发的 canonical URL;
2. **跨平台分发**:dev.to / Hashnode / Medium 发布时在平台设置里填 **canonical URL** 链回 GitHub;掘金/知乎发中文版,文首注明"原文"链接;
3. **互相引流**:文尾统一挂 awesome-macos-ci 列表(P4)与对应示例仓库(P1)/benchmark 仓库(P2)链接 —— ④ 挂 benchmark,③ 挂 ios-swift-example 和 macos-notarize-example;
4. **节奏**:双周一篇,按 ①→②→③→④ 顺序;④ 依赖 P2 benchmark 首跑数据,没有实测数据前不发。

## 发布前检查单(每篇)

- [ ] 所有 `TODO[eng]` 占位(集群规模、启动秒数、价格等)已替换为工程团队核实的真实数字
- [ ] 数字类主张与 benchmark 仓库(P2)公开数据一致,不一致宁可删
- [ ] 代码块在真实环境跑通过(① 的 API 示例、③ 的 keychain 脚本)
- [ ] 中文版与英文版内容同步
- [ ] canonical URL 已填写;文尾互链已加
