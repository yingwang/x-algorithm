# 3.1 Home Mixer · 编排层（Rust）

**目录：** [`home-mixer/`](https://github.com/yingwang/x-algorithm/tree/main/home-mixer)

## 它是什么

Home Mixer 是整个 For You 推荐链路的入口与编排服务。这一层有一个值得特别强调的特点：它本身既不做任何模型预测，也不持久化任何数据。它的全部职责，就是把"取候选、注水、过滤、打分、选 Top-K、复滤、副作用"这一长串阶段按正确的顺序和并发模式串起来，并以 gRPC 接口暴露给上游调用方。

把编排层与计算层、存储层严格分离开来，是大规模推荐系统里一种相当常见的工程做法。它的好处在于让单一进程的职责变得非常清晰：编排服务关心的是"流程"，模型服务关心的是"预测"，存储服务关心的是"检索"。每一层都可以独立地横向扩展、独立地发布升级、独立地排查问题，互不牵连。这种"按职责分层"的做法在分布式系统设计里被反复证明是行之有效的。

## 进程入口

整个服务的启动顺序非常直接，可以从代码的目录结构里看得很清楚。

```
home-mixer/main.rs        ─▶  启动 tokio 运行时
home-mixer/server.rs      ─▶  注册 ForYouService 与 ScoredPostsService
home-mixer/for_you_server.rs       ─▶  绑定 ForYouCandidatePipeline
home-mixer/scored_posts_server.rs  ─▶  绑定 PhoenixCandidatePipeline
```

两个 gRPC 服务都基于 `tonic` 框架实现。`tonic` 是 Rust 生态里最主流的 gRPC 实现，它把 protobuf 接口定义自动转换成对应的 Rust 类型与 trait，让开发者只需要专注于业务逻辑。两个服务共享同一个 tokio 运行时（tokio 是 Rust 生态里最广泛使用的异步运行时框架），意味着所有候选源、注水、过滤、打分阶段都在同一个 async 调度池里执行，跨阶段的协作与背压管理变得很容易处理。

这里有几个名词需要简单介绍。"异步运行时"指的是一个可以同时调度大量协程（轻量级线程）的执行器，让代码可以在等待 IO 的时候自动让出 CPU 给其它任务，从而用很少的操作系统线程支撑非常高的并发量。"背压"是指系统在某一处出现处理速度跟不上输入速度时，如何把这一压力传递回上游，让上游主动减速以避免内存爆炸的机制。

## 两条 Pipeline 的拼装

`home-mixer/candidate_pipeline/` 目录下放着两个文件，分别对应两条 Pipeline 的"装配清单"。

```
candidate_pipeline/
  phoenix_candidate_pipeline.rs   核心：召回 + 注水 + 过滤 + 打分 + 复滤
  for_you_candidate_pipeline.rs   外层：把核心结果与广告、WTF、Prompts 一起混排
```

两条 Pipeline 都实现了 [`xai_candidate_pipeline::CandidatePipeline`](components-pipeline.md) 这一 trait。"trait" 是 Rust 中类似于其它语言"接口"的概念，定义一组方法签名，要求实现类型必须提供这些方法的具体实现。这一 trait 要求实现者声明自己有哪些 query hydrators、哪些 sources、哪些 hydrators、哪些 filters、哪些 scorers、用什么 selector、跟随哪些 side effects。底层框架 `xai_candidate_pipeline` 负责把这些声明翻译成实际的并发执行计划：哪些阶段并行、哪些阶段串行、如何记录 trace、如何统计过滤率、如何把副作用 spawn 出去。业务代码因此不需要直接关心这些细节，只需要专心地写好每一个阶段的具体逻辑。

这种"装配清单加上框架"的写法在 Rust 生态里并不少见。它让一份 Pipeline 的定义读起来更像是一份配置文件而不是一段程序，新增或者替换某一个阶段只需要改装配清单本身，整套调度逻辑保持不变。这种写法在工程上类似于依赖注入（dependency injection）：组件之间的依赖关系不写死在代码里，而是通过外部装配清单声明。

## 子目录速览

Home Mixer 内部按职能划分成若干个子目录，每一个都对应着 Pipeline 中的一个阶段。下面这张表把每一类组件的数量与角色列了出来。

| 子目录 | 数量 | 角色 |
|---|---|---|
| `query_hydrators/` | 15 | 拉取用户级别的上下文信息：动作序列、关注列表、屏蔽列表、Bloom Filter、IP、互关图、Grok 话题、Starter Pack、人口学属性、推断性别等等 |
| `sources/` | 10 | 全部候选源，分布在内外两层 Pipeline 中 |
| `candidate_hydrators/` | 10 | 候选级别的注水：核心元数据、引用帖展开、视频时长、是否带媒体、订阅信息、Gizmoduck 作者档案、被屏蔽信号、品类、语言 |
| `filters/` | 14 | 见[过滤规则](filters.md)一节 |
| `scorers/` | 4 | `phoenix_scorer`、`ranking_scorer`（其内部由 `weighted_scorer`、`author_diversity_scorer`、`oon_scorer` 三者组合而成）、`vm_ranker` |
| `selectors/` | 2 | `TopKScoreSelector` 供核心 Pipeline 使用；`BlenderSelector` 供外层广告混排使用 |
| `ads/` | 3 | `safe_gap_blender.rs`、`partition_organic_blender.rs`、`util.rs`，共同决定广告插入位置与品牌安全间隔 |
| `side_effects/` | 约 8 | 写 Kafka、更新 Redis 缓存、维护已展示历史、上报监控指标 |
| `models/` | 6 | `query.rs`、`candidate.rs`、`user_features.rs`、`brand_safety.rs`、`in_network_reply.rs`、`candidate_features.rs` |

表格中出现了几个推荐系统特有的概念，下面简要解释一下。"动作序列"指的是用户最近发生过的互动按时间顺序排列后的列表，例如"用户在某条帖子上点了赞、在另一条帖子上停留了三秒、点开了第三条帖子的作者主页"，这一序列是 Phoenix 排序模型的核心输入。"Bloom Filter"是一种概率数据结构，用很少的空间就能回答"某一项是否在集合中"这种查询，但允许小概率的假阳性（也就是误认为"在集合中"而实际上不在）；推荐系统中常用 Bloom Filter 记录"曾经展示给用户看过的帖子集合"，从而做去重。"互关图"指的是用户之间双向关注的关系图，用来度量用户之间的社交亲密程度。"Starter Pack"是 X 平台上的一种"作者合集"概念，用户可以一键关注一组相关账号。"人口学属性"包括年龄、性别、地区、语言等基础信息。"Gizmoduck"是 Twitter 系内部的用户档案服务。

## Query Hydrators 列表

Query Hydrator 这一阶段负责为每一次请求准备好用户侧的全部上下文。它的输出会被后续的 source、filter、scorer 阶段反复使用。下面这段代码直接展示了所有十五个 hydrator 的实例化顺序。

```rust
let query_hydrators = vec![
    ScoringSequenceQueryHydrator,        // 拉取用户最近动作序列，供排序模型使用
    RetrievalSequenceQueryHydrator,      // 拉取另一份动作序列，供召回模型使用
    BlockedUserIdsQueryHydrator,         // 该用户屏蔽过的所有作者
    MutedUserIdsQueryHydrator,           // 该用户静音过的所有作者
    FollowedUserIdsQueryHydrator,        // 关注列表
    SubscribedUserIdsQueryHydrator,      // 付费订阅关系
    CachedPostsQueryHydrator,            // 上一次请求的候选缓存
    MutualFollowQueryHydrator,           // 互关图
    UserDemographicsQueryHydrator,       // 年龄、地区、语言等人口学属性
    FollowedGrokTopicsQueryHydrator,     // 用户关注的 Grok 话题
    FollowedStarterPacksQueryHydrator,   // 用户加入的 Starter Pack
    InferredGrokTopicsQueryHydrator,     // 根据行为推断的兴趣话题
    ImpressionBloomFilterQueryHydrator,  // 已曝光帖的 Bloom Filter
    IpQueryHydrator,                     // IP 到地理位置
    UserInferredGenderQueryHydrator,     // 推断的性别属性
];
```

这十五个 hydrator 在执行时是全部并行的，由 `futures::future::join_all` 一次性发起。`futures::future::join_all` 是 Rust 异步生态里的一个工具，它接收一组 future（异步任务），同时发起、并行等待，直到全部完成为止。

这里有一个值得留意的实现细节：任何一个 hydrator 失败都不会让整次请求失败，框架会把这个 hydrator 对应的字段当作"没有取到"留空，下游的 scorer 与 filter 会把它视作"该信号缺失"来处理。这种宽容失败的策略对于一个由众多远程依赖共同支撑起来的请求来说至关重要，因为任何单点抖动都可能拖垮整次刷新，必须从一开始就在框架层面把这一类失败局部化。这种设计哲学在分布式系统里被称为"优雅降级"（graceful degradation），即在部分依赖失效时仍然能够提供有意义的服务，而不是完全失败。

## Candidate Hydrators

候选级别的注水同样并行执行，与 Query Hydrator 不同的是，它的输入是已经召回出来的一批候选。下面是这十个候选注水器的实例化顺序。

```rust
let hydrators = vec![
    InNetworkCandidateHydrator,           // 标记候选是否来自关注网内
    CoreDataCandidateHydrator,            // 拉取帖子正文、媒体、时间戳，这是必备字段
    QuoteHydrator,                        // 引用帖展开
    VideoDurationCandidateHydrator,       // 视频时长，供 VQV 权重门槛使用
    HasMediaHydrator,                     // 是否含有媒体
    SubscriptionHydrator,                 // 是否需要订阅才能查看
    GizmoduckCandidateHydrator,           // 作者档案，包括认证状态与粉丝数
    BlockedByHydrator,                    // 作者是否屏蔽了当前用户
    FilteredTopicsHydrator,               // 帖子被打上的话题标签
    LanguageCodeHydrator,                 // 帖子的语言标识
];
```

那些拉不到核心元数据的候选会被紧随其后的 `CoreDataHydrationFilter` 直接剔除掉。这一种"先尽量并发地拉、再按结果筛选"的执行模式，是一种典型的 fan-out 与 fan-in 配合。所谓 fan-out，是指把一个请求扇出成多个并发的子请求；fan-in 则是把多个子请求的结果收集汇聚回来。在这个具体场景里，先把所有可能成功的请求一次性发出去，等结果回来之后再判断哪些候选具备继续往后走的条件，这种做法可以让最差情况下的延迟接近"最慢的那一路下游服务的延迟"，而不是各路服务延迟之和。

## 跟踪与监控

`candidate-pipeline` 框架在每一个阶段都套上了 `#[tracing::instrument(...)]` 这一宏。"宏"在 Rust 中是一种代码生成机制，可以在编译时自动展开成更复杂的代码。`tracing::instrument` 这一具体的宏会在被注解的函数前后自动添加日志埋点，自动记录下面这些指标：

- `total_count` 与 `enabled_count` 以及 `disabled`，用以追踪哪些组件被 feature switch 关掉了
- `latency_ms`，记录每一个阶段的实际耗时
- 过滤阶段额外记录 `kept_count`、`removed_count`、`filter_rate`，并对每一个 filter 单独记录它过滤掉的候选数量

这一整套自动埋点构成了后续 A/B 实验设计与容量评估的主要数据源。"feature switch"是一种线上灵活控制的机制，允许工程师在不发版的前提下通过配置文件远程打开或关闭某一个组件，方便快速止损或者灰度发布。换言之，框架本身不仅负责调度，也负责可观测性，业务代码只要按约定写好阶段，就能自动获得一套完整的观测能力。

## Side Effects

副作用阶段在整套 Pipeline 中比较特别。它的所有动作都是通过 `tokio::spawn` 异步发出的，不会阻塞主响应链路。`tokio::spawn` 这一函数会把一段异步任务交给 tokio 运行时去后台执行，调用者无需等待任务完成。这一点对于用户感知到的延迟非常关键：所有写 Kafka、写 Redis、上报指标的耗时都不会被加到这一次请求的总延迟上。下面这张表列出了主要的几个副作用。

| Side Effect | 用途 |
|---|---|
| `PhoenixRequestCacheSideEffect` | 把 Phoenix 请求与响应缓存到 Redis，使得同一用户短期内再次请求时不需要重算 |
| `RedisPostCandidateCacheSideEffect` | 把入选与落选的候选一并缓存到 Redis，供下次请求中的 `CachedPostsSource` 使用 |
| `PhoenixExperimentsSideEffect` | 给 A/B 实验中的对照组也跑一次 Phoenix，把结果写入 Kafka 以便后续比对 |
| `RerankingKafkaSideEffect` | 把当前请求的打分快照写入 Kafka，供离线训练再排模型 |
| `ScoredStatsSideEffect` 与 `MutualFollowStatsSideEffect` | 上报打分分布以及互关命中率 |
| `UpdateServedHistorySideEffect` 与 `TruncateServedHistorySideEffect` | 维护"已展示历史"，供下一次请求做去重 |
| `PublishSeenIdsToKafkaSideEffect` | 把已曝光的 ID 写到 Kafka，供 Bloom Filter 服务更新它的曝光过滤数据 |
| `ClientEventsKafkaSideEffect` | 把客户端事件转发到 Kafka |

值得特别留意的是 `PhoenixExperimentsSideEffect` 这一项。它实际上挂载了两个 Phoenix 客户端：一个是常规的客户端，另一个是 `EgressPhoenixPredictionClient`，后者会走 egress sidecar 路径。这里的 "egress sidecar" 是一种网络架构模式：在每一个服务实例旁边部署一个轻量级代理（sidecar），所有出向流量都通过这个代理转发，从而把网络相关的能力（路由、TLS、监控等）从业务代码里剥离出来。这种设计是为了在生产流量上对新版本的 Phoenix 模型做对照实验，又不会影响当次请求实际返回给用户的结果。模型迭代过程中这一类"影子流量"机制对于安全地评估新版本的影响有着极其重要的价值，因为它让工程师可以在真实流量下观察新模型的表现，而无需让任何用户真正接触到尚未完全验证过的输出。
