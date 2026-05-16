# 3.1 Home Mixer · 编排层（Rust）

**目录：** [`home-mixer/`](https://github.com/yingwang/x-algorithm/tree/main/home-mixer)

## 它是什么

For You 的**入口与编排服务**。本身不做模型预测、也不存数据，职责只有一件：把"取候选 → 注水 → 过滤 → 打分 → 选 Top-K → 复滤 → 副作用"串起来，并以 gRPC 暴露给上游。

## 进程入口

```
home-mixer/main.rs        ─▶  启动 tokio 运行时
home-mixer/server.rs      ─▶  注册 ForYouService + ScoredPostsService
home-mixer/for_you_server.rs       ─▶  绑定 ForYouCandidatePipeline
home-mixer/scored_posts_server.rs  ─▶  绑定 PhoenixCandidatePipeline
```

两个 gRPC 服务都是 **`tonic`** 实现，复用同一个 tokio runtime。

## 两条 Pipeline 的拼装

`home-mixer/candidate_pipeline/` 目录下两个文件 = 两条 Pipeline 的"装配清单"：

```
candidate_pipeline/
  phoenix_candidate_pipeline.rs   核心：召回 + 注水 + 过滤 + 打分 + 复滤
  for_you_candidate_pipeline.rs   外层：把核心结果 + 广告 + WTF + Prompts 混排
```

每条 Pipeline 都实现了 [`xai_candidate_pipeline::CandidatePipeline`](components-pipeline.md) trait，告诉框架"我有哪些 query hydrators / sources / hydrators / filters / scorers / selector / side_effects"。框架 (`xai_candidate_pipeline`) 负责怎么并发、怎么记 trace、怎么算 filter rate、怎么 spawn 副作用 —— 业务代码不用关心。

## 子目录速览

| 子目录 | 数量 | 角色 |
|---|---|---|
| `query_hydrators/` | 15 | 拉取**用户级**上下文：动作序列、关注列表、屏蔽列表、Bloom Filter、IP、互关图、Grok 话题、Starter Pack、人口学、推断性别等 |
| `sources/` | 10 | 候选源（前文已列） |
| `candidate_hydrators/` | 10 | **候选级**注水：核心元数据、引用帖展开、视频时长、是否带媒体、订阅、Gizmoduck 作者、被屏蔽信号、品类过滤、语言 |
| `filters/` | 14 | 见 [§6 过滤规则](filters.md) |
| `scorers/` | 4 | `phoenix_scorer` · `ranking_scorer`（含 `weighted_scorer` / `author_diversity_scorer` / `oon_scorer`）· `vm_ranker` |
| `selectors/` | 2 | `TopKScoreSelector`（核心用）· `BlenderSelector`（外层广告混排用） |
| `ads/` | 3 | `safe_gap_blender.rs` · `partition_organic_blender.rs` · `util.rs` —— 广告插入位置、品牌安全间隔 |
| `side_effects/` | ~8 | 写 Kafka、更新 Redis 缓存、更新已展示历史、上报指标 |
| `models/` | 6 | `query.rs` · `candidate.rs` · `user_features.rs` · `brand_safety.rs` · `in_network_reply.rs` · `candidate_features.rs` |

## Query Hydrators 列表

```rust
let query_hydrators = vec![
    ScoringSequenceQueryHydrator,        // 拉取用户最近动作序列（给排序模型）
    RetrievalSequenceQueryHydrator,      // 拉取另一份动作序列（给召回模型）
    BlockedUserIdsQueryHydrator,         // 被该用户 block 的人
    MutedUserIdsQueryHydrator,           // 被静音的人
    FollowedUserIdsQueryHydrator,        // 关注列表
    SubscribedUserIdsQueryHydrator,      // 付费订阅关系
    CachedPostsQueryHydrator,            // 上一次请求的候选缓存
    MutualFollowQueryHydrator,           // 互关图
    UserDemographicsQueryHydrator,       // 年龄 / 地区 / 语言
    FollowedGrokTopicsQueryHydrator,     // 关注的 Grok 话题
    FollowedStarterPacksQueryHydrator,   // 加入的 Starter Pack
    InferredGrokTopicsQueryHydrator,     // 推断兴趣话题
    ImpressionBloomFilterQueryHydrator,  // 已曝光帖 Bloom Filter
    IpQueryHydrator,                     // IP → 地理
    UserInferredGenderQueryHydrator,     // 推断性别
];
```

15 个 hydrator **全部并行执行**（`futures::future::join_all`），任何一个失败不会让请求失败 —— 失败时该 hydrator 不写值，下游 scorer / filter 当成"没有该信号"处理。

## Candidate Hydrators

候选级注水也是并行：

```rust
let hydrators = vec![
    InNetworkCandidateHydrator,           // 标记是否来自关注网内
    CoreDataCandidateHydrator,            // 帖子正文 / 媒体 / 时间戳（必拿）
    QuoteHydrator,                        // 引用帖展开
    VideoDurationCandidateHydrator,       // 视频时长（用于 VQV 权重门槛）
    HasMediaHydrator,                     // 是否含媒体
    SubscriptionHydrator,                 // 是否需要订阅
    GizmoduckCandidateHydrator,           // 作者档案（认证、粉丝数）
    BlockedByHydrator,                    // 作者是否屏蔽了当前用户
    FilteredTopicsHydrator,               // 帖子的话题标签
    LanguageCodeHydrator,                 // 帖子语言
];
```

> 拿不到核心元数据的候选会被紧随其后的 `CoreDataHydrationFilter` 直接剔掉。这是"先尽量并发拉、再按结果筛"的典型 fan-out / fan-in。

## 跟踪与监控

`candidate-pipeline` 框架在每个阶段都套了 `#[tracing::instrument(...)]`，自动记录：

- `total_count` / `enabled_count` / `disabled`（哪些组件被 feature switch 关掉了）
- `latency_ms`
- 过滤阶段额外记录 `kept_count` / `removed_count` / `filter_rate` / 每个 filter 各自的 `removed`

这是后续 A/B 实验和容量评估的主要数据源。

## Side Effects

副作用都是 `tokio::spawn` **不阻塞主链路**地发出去：

| Side Effect | 用途 |
|---|---|
| `PhoenixRequestCacheSideEffect` | 把 Phoenix 请求 / 响应缓到 Redis，下次同用户短期内不用重算 |
| `RedisPostCandidateCacheSideEffect` | 把入选 + 落选的候选缓到 Redis，给 `CachedPostsSource` 用 |
| `PhoenixExperimentsSideEffect` | 给 A/B 实验对照组也跑一次 Phoenix，写 Kafka 比对 |
| `RerankingKafkaSideEffect` | 写 Kafka 供离线训练再排模型 |
| `ScoredStatsSideEffect` / `MutualFollowStatsSideEffect` | 上报打分分布、互关命中率 |
| `UpdateServedHistorySideEffect` / `TruncateServedHistorySideEffect` | 维护"已展示历史"，供下次请求去重 |
| `PublishSeenIdsToKafkaSideEffect` | 写 Kafka 供 Bloom Filter 服务更新曝光过滤 |
| `ClientEventsKafkaSideEffect` | 把客户端事件转发到 Kafka |

注意 `PhoenixExperimentsSideEffect` 接了两个 Phoenix client：一个常规，一个 `EgressPhoenixPredictionClient`（走 egress sidecar）。这是为了**新版模型在生产流量下做对照**而不影响主返回结果。
