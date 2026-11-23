# macOS CI 的真实成本:官方 / 托管 / 自建 TCO 账单

<!-- canonical(英文): https://github.com/macrunara/docs/blob/main/blog/04-macos-ci-tco-comparison/index.en.md -->
<!-- 状态:草稿。所有费率均为 TODO[pricing] 占位 —— 发布当天逐一核对厂商最新报价
     (价格会变),并保持模型诚实:这篇文章的可信度完全取决于数字是否可查。 -->

*大多数"macOS CI 贵"的讨论只比了标价就停笔。诚实的对比有四行:算力、排队时间、维护人力,以及"慢"本身的成本。本文给出完整模型 —— 表格逻辑全部摊开,你可以代入自己的数字。*

---

## 工作负载模型(换成你团队的数字)

下面所有数字都由四个输入推出。示例用一个中型 iOS 团队:

| 输入 | 示例值 |
|---|---|
| 提交代码的工程师 | 10 人 |
| 每个工作日的 CI 任务 | 60 个(push + PR +  nightly) |
| 平均任务时长(M1 档机器) | 12 分钟 |
| 每月工作日 | 21 天 |

也就是 **每月约 756 个构建小时**(M1 档)。下面每一条价格都是这个数字的函数 —— 外加人人都会忘掉的两行。

## 第一行:算力(标价)

| 方案 | 计费模式 | 756 构建小时的月算力成本 |
|---|---|---|
| GitHub 托管 `macos-14`(M1) | **公开仓库免费。** 私有仓库按分钟计费,macOS 分钟 = Linux 的 10 倍 <!-- TODO[pricing]:当前每分钟费率 → 约 $X/min --> | 公开:**$0** · 私有:约 $<!-- TODO[pricing] -->/月 |
| GitHub larger runner(M 系) | 按分钟,约为标准 macOS 费率的 2 倍 <!-- TODO[pricing] --> | 约 $<!-- TODO[pricing] -->/月 |
| MacStadium / MacWeb(专属机) | 按机器包月,每台 Mac mini 约 $<!-- TODO[pricing] --> | 按 p95 并发配 N 台:约 $<!-- TODO[pricing] -->/月 |
| 自建(自购 Mac mini) | 每台 $<!-- TODO[pricing] --> 按 3 年摊销 + 机房/电力 | 约 $<!-- TODO[pricing] -->/月 + 你的时间(第三行) |
| **Macrunara** | 每台节点包月(M4/M6) <!-- TODO[pricing]:刊例价 --> | 按 p95 并发配节点:约 $<!-- TODO[pricing] -->/月 |

标价里还藏着一个微妙点:**计费粒度 × 任务形状**。按分钟计费惩罚"大量短任务"(每个任务各自支付启动开销);按节点包月惩罚深夜和周末的空转。你的任务直方图决定哪种更疼 —— 先画出来再选型。

## 第二行:排队时间(没人列出来的价格)

CI 的一分钟和*工程师的*一分钟不是同一种货币。一个 PR 等 runner 排了 25 分钟,作者就会切走上下文;关于任务切换的研究把重新进入状态的代价定在 10-23 分钟。<!-- TODO[eng]:引用具体研究,或删掉数字只保留定性主张 -->

放进模型:

```
月度排队成本 =(平均排队 + 每任务启动开销)
            × 每月任务数
            × 上下文切换惩罚系数
            × 工程师综合时薪(约 $<!-- TODO[pricing] -->/h)
```

对示例团队,惩罚系数保守取 25%,**平均 8 分钟的排队+启动开销 ≈ 每月 $<!-- TODO[eng] --> 的注意力损失** —— 常常比整张算力账单还大。对私有仓库团队,"免费"托管 runner 最贵的地方就在这一行;而按节点包月(容量内零排队)的价值也在这里。

## 第三行:维护人力(自建方案的冰山)

拥有机器就拥有了一切:全机群的 OS 和 Xcode 升级、runner 程序更新、证书续期、磁盘磨损(SSD 在 CI 负载下是消耗品)、网络/PDU/散热,以及凌晨两点的"runner 掉线了"告警。小型自建 macOS 机群的行业经验值:**0.2-0.5 个 FTE**。按综合薪酬算,每月 $<!-- TODO[pricing] -->-$<!-- TODO[pricing] --> —— 通常比硬件本身还贵。

这也是虚拟化纪律兑现价值的地方:黄金镜像 + 临时 VM(见[第一篇](https://github.com/macrunara/docs/blob/main/blog/01-m4-cluster-architecture/index.zh-CN.md))把"给机群打补丁"从放牧雪花服务器变成重建一次镜像。选择自建,就把这项预算做进去,否则你会以事故的形式悄悄支付它。

## 第四行:"慢"的成本

更快的机器不只是省计费分钟 —— 它压缩第二行,还会改变开发者行为。CI 12 分钟时,人们推完代码愿意等;40 分钟时,他们开始攒改动,PR 越变越大,评审质量下降,发版节奏被拉长。这一行拒绝精确度量,但任何诚实的 TCO 都得承认:**构建时长是团队速度的复利输入,不只是发票上的一行。**(我们自己测的 M4 对 M1 差值公开在 [benchmarks 仓库](https://github.com/macrunara/benchmarks) —— 用我们的,或者用你自己的实测。)

## 汇总对比

| 成本行 / 月(示例团队) | GitHub 托管 | larger runner | 专属机(MacStadium) | 自建 | Macrunara |
|---|---|---|---|---|---|
| 算力 | $<!-- TODO --> | $<!-- TODO --> | $<!-- TODO --> | $<!-- TODO --> | $<!-- TODO --> |
| 排队时间(注意力损失) | $<!-- TODO --> | $<!-- TODO --> | 容量内 ~$0 | 容量内 ~$0 | 容量内 ~$0 |
| 维护(0.2-0.5 FTE) | $0 | $0 | $0 | $<!-- TODO --> | $0 |
| **合计** | **$<!-- TODO -->** | **$<!-- TODO -->** | **$<!-- TODO -->** | **$<!-- TODO -->** | **$<!-- TODO -->** |

我们刻意把模型摊开而不是甩一句"我们最便宜",因为诚实的答案是有条件的:

- **公开开源仓库**:GitHub 托管免费且够用。(所以我们的 OSS 计划拼的是*速度*,不是价格。)
- **小规模私有团队、低用量**:按分钟计费通常占优,直到排队痛感在第二行显现出来。
- **持续高用量或对延迟敏感的团队**:当排队行和维护行开始主导,按节点包月(专属机或 Macrunara)胜出 —— 在我们的模型里,拐点大约在 <!-- TODO[eng]:盈亏平衡构建小时数 --> 构建小时/月。
- **自建**只在两种情况下赢:你*享受*运营硬件,或有合规约束逼你这么做。决定之前,先诚实给 FTE 定价。

## 给你团队的复现步骤

1. 拉最近 90 天的 Actions 用量(Settings → Billing 或用量 API),拿到真实的任务数/天和分钟/任务;
2. 用 Actions 指标里真实的 p50 排队时长算第二行;
3. 把你的薪酬带和厂商报价代入四行;
4. 如果包月那一列赢了,[拿一个仓库试我们](https://github.com/macrunara/quickstart) —— fork 即跑的示例三分钟上手,[benchmarks](https://github.com/macrunara/benchmarks) 里有可复现的实测方法。

---

*本系列第四篇(完)。系列目录:[集群架构](https://github.com/macrunara/docs/blob/main/blog/01-m4-cluster-architecture/index.zh-CN.md) · [构建缓存一致性](https://github.com/macrunara/docs/blob/main/blog/02-xcode-build-cache-consistency/index.zh-CN.md) · [临时 runner 上的签名安全](https://github.com/macrunara/docs/blob/main/blog/03-ephemeral-runner-ios-signing/index.zh-CN.md)。赛道全景:[awesome-macos-ci](https://github.com/macrunara/awesome-macos-ci)。*
