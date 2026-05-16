# 3.1 Home Mixer · 编排层（Rust）

**目录：** [`home-mixer/`](https://github.com/yingwang/x-algorithm/tree/main/home-mixer)

## 它是什么

Home Mixer 是整个 For You 推荐链路的入口与编排服务。这一层有一个值得特别强调的特点：它本身既不做任何模型预测，也不持久化任何数据。它的全部职责，就是把"取候选、注水、过滤、打分、选 Top-K、复滤、副作用"这一长串阶段按正确的顺序和并发模式串起来，并以 gRPC 接口暴露给上游调用方。

把编排层与计算层、存储层严格分离开来，是大规模推荐系统里一种相当常见的工程做法。它的好处在于让单一进程的职责变得非常清晰：编排服务关心的是流程，模型服务关心的是预测，存储服务关心的是检索。每一层都可以独立地横向扩展、独立地发布升级、独立地排查问题，互不牵连。

## 进程入口

整个服务的启动顺序非常直接，可以从代码的目录结构里看得很清楚。

```
home-mixer/main.rs        ─▶  启动 tokio 运行时
home-mixer/server.rs      ─▶  注册 ForYouService 与 ScoredPostsService
home-mixer/for_you_server.rs       ─▶  绑定 ForYouCandidatePipeline
home-mixer/scored_posts_server.rs  ─▶  绑定 PhoenixCandidatePipeline
```

两个 gRPC 服务都基于 `tonic` 框架实现，共享同一个 tokio 运行时。共享运行时这一点决定了所有候选源、注水、过滤、打分阶段都在同一个 async 调度池里执行，跨阶段的协作与背压管理变得很容易处理。

## 两条 Pipeline 的拼装

`home-mixer/candidate_pipeline/` 目录下放着两个文件，分别对应两条 Pipeline 的"装配清单"。

```
candidate_pipeline/
  phoenix_candidate_pipeline.rs   核心：召回 + 注水 + 过滤 + 打分 + 复滤
  for_you_candidate_pipeline.rs   外层：把核心结果与广告、WTF、Prompts 一起混排
```

两条 Pipeline 都实现了 [`xai_candidate_pipeline::CandidatePipeline`](components-pipeline.md) 这一 trait。该 trait 要求实现者声明自己有哪些 query hydrators、哪些 sources、哪些 hydrators、哪些 filters、哪些 scorers、用什么 selector、跟随哪些 side effects。底层框架 `xai_candidate_pipeline` 负责把这些声明翻译成实际的并发执行计划：哪些阶段并行、哪些阶段串行、如何记录 trace、如何统计过滤率、如何把副作用 spawn 出去。业务代码因此不需要直接关心这些细节，只需要专心地写好每一个阶段的具体逻辑。

这种"装配清单 + 框架"的写法在 Rust 生态里并不少见。它让一份 Pipeline 的定义读起来更像是一份配置文件而非一段程序，新增或者替换某一个阶段只需要改装配清单本身，整套调度逻辑保持不变。

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

这十五个 hydrator 在执行时是全部并行的，由 `futures::future::join_all` 一次性发起。这里有一个值得留意的实现细节：任何一个 hydrator 失败都不会让整次请求失败，框架会把这个 hydrator 对应的字段当作"没有取到"留空，下游的 scorer 与 filter 会把它视作"该信号缺失"来处理。这种宽容失败的策略对于一个由众多远程依赖共同支撑起来的请求来说至关重要，因为任何单点抖动都可能拖垮整次刷新，必须从一开始就在框架层面把这一类失败局部化。

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

那些拉不到核心元数据的候选会被紧随其后的 `CoreDataHydrationFilter` 直接剔除掉。这一种"先尽量并发地拉、再按结果筛选"的执行模式，是一种典型的 fan-out 与 fan-in 配合：先把所有可能成功的请求一次性发出去，等结果回来之后再判断哪些候选具备继续往后走的条件。

## 跟踪与监控

`candidate-pipeline` 框架在每一个阶段都套上了 `#[tracing::instrument(...)]` 宏，自动记录下面这些指标：

- `total_count` 与 `enabled_count` 以及 `disabled`，用以追踪哪些组件被 feature switch 关掉了
- `latency_ms`，记录每一个阶段的实际耗时
- 过滤阶段额外记录 `kept_count`、`removed_count`、`filter_rate`，并对每一个 filter 单独记录它过滤掉的候选数量

这一整套自动埋点构成了后续 A/B 实验设计与容量评估的主要数据源。换言之，框架本身不仅负责调度，也负责可观测性，业务代码只要按约定写好阶段，就能自动获得一套完整的观测能力。

## Side Effects

副作用阶段在整套 Pipeline 中比较特别。它的所有动作都是通过 `tokio::spawn` 异步发出的，不会阻塞主响应链路。这一点对于用户感知到的延迟非常关键：所有写 Kafka、写 Redis、上报指标的耗时都不会被加到这一次请求的总延迟上。下面这张表列出了主要的几个副作用。

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

值得特别留意的是 `PhoenixExperimentsSideEffect` 这一项。它实际上挂载了两个 Phoenix 客户端：一个是常规的客户端，另一个是 `EgressPhoenixPredictionClient`，后者会走 egress sidecar 路径。这种设计是为了在生产流量上对新版本的 Phoenix 模型做对照实验，又不会影响当次请求实际返回给用户的结果。模型迭代过程中这一类"影子流量"机制对于安全地评估新版本的影响有着极其重要的价值。
