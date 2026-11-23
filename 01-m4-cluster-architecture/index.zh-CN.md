# 我们用 Mac mini 集群(M4 与 M6)跑 GitHub CI:架构、虚拟化与调度

<!-- canonical(英文): https://github.com/macrunara/docs/blob/main/blog/01-m4-cluster-architecture/index.en.md -->
<!-- 状态:草稿。所有 TODO[eng] 占位发布前由 infra 团队替换为核实数字。 -->

*我们如何把一机柜 Mac mini 变成"一任务一虚拟机"的 GitHub Actions 临时 runner —— 以及让这一切变得"无聊"(褒义)的工程决策。*

---

给 iOS 做云端 CI 有一个绕不开的底层约束:**macOS 只能跑在苹果硬件上**。不存在 `docker run macos`。你用过的每一个"macOS runner"—— 托管的也好、自建的也好 —— 最终都落在某台实体苹果机器上。问题从来不是"有没有一台 Mac",而是"你如何诚实、高效地切分它"。

这篇文章讲清楚 Macrunara 集群的工作方式:硬件布局、为什么用 Virtualization.framework 做虚拟化而不是裸机跑任务,以及一个任务如何从你 workflow 文件里的 `runs-on: [macos, arm64, macrunara]` 一路落到机柜里某台 Mac mini(M4 或 M6)上的一台全新虚拟机。

## 硬件:刻意追求无聊

我们的计算单元是 **Mac mini M4 与 M6** —— <!-- TODO[eng]:两代各自的具体配置(芯片型号/内存/硬盘) --> 我们做 CI 以来找到的单位功耗、单位机架空间性能最高的组合。两代跑同一个节点镜像、同属一个池,调度器不区分它们,你的 workflow 也不用区分。每个机架层放 <!-- TODO[eng]:N --> 台 mini,共享电源层和 10GbE 交换网络 <!-- TODO[eng]:确认交换机型号 -->,每台机器挂在可管理 PDU 上,控制面可以远程硬重启卡死的节点。

两条用教训换来的物理运维经验:

- **散热是风道问题,不是温度问题。** Mac mini 从底部边缘进风。平放叠架、不留风道,持续 `xcodebuild` 负载下就会热节流;竖放加密封风道,就能一直满血睿频。我们 benchmark 仓库里的冷构建数据([benchmarks](https://github.com/macrunara/benchmarks))能成立,前提就是这个。
- **SSD 是有寿命的。** CI 是写密集型负载 —— 每台临时 VM 每个任务写入再删除几十 GB。我们按节点监控 TBW(总写入量),在达到厂商阈值前主动轮换。<!-- TODO[eng]:当前磨损数据/轮换策略 -->

## 为什么要虚拟化?

直接在宿主机上跑任务更简单,大多数自建方案也确实这么干。但这条路在安全和可复现性上是死胡同:第 N 次构建留下的钥匙串、DerivedData、Pods 缓存、环境变量,会成为第 N+1 次构建的坑 —— 或者被它偷走。CI runner 执行的是*别人 PR 里的任意代码*。

所以 Macrunara 上每个任务都跑在一台**全新的临时 VM** 里:

```
任务被认领 → 从黄金镜像克隆 VM → 启动(约 TODO[eng]:X 秒)
          → 向 GitHub 注册 JIT runner → 执行任务
          → 销毁 VM → 磁盘回到资源池
```

Apple Silicon 上的 macOS 有一等公民虚拟化 API —— **Virtualization.framework**,我们在其上用 tart 风格的工具链管理 VM <!-- TODO[eng]:确认对外口径;如果基于 tart 构建,在此致谢 -->。我们依赖的关键特性:

- **写时复制克隆。** 启动 VM 不需要复制黄金镜像;APFS 克隆只存增量。从启动到 runner 就绪是秒级,不是分钟级。
- **硬件级隔离。** 每台 VM 有独立的内核、钥匙串和安全隔区视图。任务结束,VM —— 连同它碰过的所有 secret —— 不复存在。(这对代码签名为什么重要,见本系列[第三篇](https://github.com/macrunara/docs/blob/main/blog/03-ephemeral-runner-ios-signing/index.zh-CN.md)。)
- **一任务一 VM,没有例外。** 不存在"基本干净"的温热机器池。干净不是我们维护出来的状态,而是每次*重建*出来的状态。

## 黄金镜像

每台 VM 的基础是我们的 `macos-mobile` 镜像(当前 v1.2):macOS 26 + Xcode 26.x + Flutter 3.47 + Android SDK 35 + JDK 17 + Node 20 + CocoaPods,证书协议预接受、模拟器运行时预热。完整清单在 [quickstart 仓库](https://github.com/macrunara/quickstart)公开。

两个值得借鉴的设计决策:

1. **预装,而不是现装。** 任务里装工具链的每一分钟,都是计费的一分钟、也是引入网络抖动的一分钟。超过 20% 任务用到的依赖,就应该进镜像;其余的才是任务自己的事。
2. **像管理软件一样管理镜像版本。** 镜像是有 semver 标签和 changelog 的不可变产物。任务通过标签隐式锁定环境;我们按镜像版本滚动升级机群,坏镜像是回滚,不是事故。当"我在哪个环境上?"有一个单词的答案时,`flutter: command not found` 这类故障就消失了。

## 调度:从 `runs-on:` 到一台启动的 VM

GitHub Actions 自托管 runner 是拉取模型,和临时 VM 天然契合。控制回路长这样:

```
                ┌──────────────┐
   workflow ──▶ │ GitHub       │ 排队任务,带标签
                │ Actions API  │ [macos, arm64, macrunara]
                └──────┬───────┘
                       │ 轮询 / webhook
                ┌──────▼───────┐
                │ 编排器        │ 容量检查,选节点
                └──────┬───────┘
                       │ 克隆黄金镜像,启动 VM
                ┌──────▼───────┐
                │ JIT runner   │ 通过 API 拿一次性注册令牌,
                │ 注册          │ 随 VM 过期
                └──────┬───────┘
                       │ 任务在 VM 内执行
                ┌──────▼───────┐
                │ VM 销毁       │ 增量丢弃,节点回收磁盘
                └──────────────┘
```

实践中真正要紧的细节:

- **JIT runner,而不是常驻注册。** 每个任务我们都通过 GitHub API 申请一次性(just-in-time)runner 注册。VM 里的 runner 程序向外连接、认领恰好一个任务,注册随即作废 —— 任何磁盘上都不存在长期有效的 runner 令牌。
- **标签是契约,不是建议。** `[macos, arm64, macrunara]` 是编排器匹配的唯一精确三元组。写 `self-hosted` 或 `macos-latest` 的任务永远不会被拾起 —— 它会*按设计*永远排队,这也是为什么我们文档 FAQ 第一条就是讲标签。
- **容量按突发配,不按均值配。** CI 流量是尖峰型的(周一早上的合并潮、发版 tag)。我们为 p95 排队深度保留余量 <!-- TODO[eng]:当前余量/目标拾取延迟 -->,编排器最重要的指标是*排队等待时长*,不是 CPU 利用率。
- **出口走内部代理。** VM 没有直连外网的路由;包仓库、Actions API、产物上传都走一层缓存代理。所以你第一次 CocoaPods CDN 拉取会比预期慢 —— 第二次就不了。

## 数字长什么样

来自我们公开的 [benchmarks](https://github.com/macrunara/benchmarks)(方法论和原始 CSV 都在仓库里):一个中型 Swift 应用在 M4 节点上冷构建约 7 分钟,GitHub 托管 M1 约 12 分钟;VM 启动只给任务增加 <!-- TODO[eng]:实测 VM 启动开销 --> 秒。M6 节点的数字随下一期月度基准发布 —— <!-- TODO[eng]:M6 实测后回填 -->。这些数字我们会每月持续公布 —— 这个系列的全部意义就在于:结论是可验证的。

## 接下来

- **第二篇** —— 构建缓存的一致性难题:临时 runner 之间什么能缓存、什么不能,以及我们如何保证增量构建的正确性。
- **第三篇** —— 在"用完即焚"的基础设施上,安全地做 iOS 代码签名。
- **第四篇** —— TCO 账单:托管 / 托管服务 / 自建 macOS CI 的真实成本横评。

*Macrunara 是面向 GitHub Actions 的 Mac mini M4/M6 专用 runner 服务。想亲手戳一戳这套架构,[quickstart](https://github.com/macrunara/quickstart) 三分钟能把任务跑上我们的集群;[awesome-macos-ci](https://github.com/macrunara/awesome-macos-ci) 列表里有整个赛道的地图,包括我们的竞品。*
