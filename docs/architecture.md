# 2. 顶层架构总览

## 一图看完

```
┌────────────────────────────────────────────────────────────────────────────┐
│                            Home Mixer (Rust)                               │
│                                                                            │
│  请求 ─▶ For You Pipeline ─────────┐                                       │
│                                    │  调用                                 │
│                                    ▼                                       │
│                          Scored Posts Pipeline (核心)                      │
│                          ┌───────────────────────────────────┐             │
│                          │ ① Query Hydrators (15 个 · 并行)  │             │
│                          ├───────────────────────────────────┤             │
│                          │ ② Sources (6 路 · 并行)           │             │
│                          │    Thunder | Phoenix Retr.        │             │
│                          │    | Phoenix Topics | MoE         │             │
│                          │    | TweetMixer | CachedPosts     │             │
│                          ├───────────────────────────────────┤             │
│                          │ ③ Candidate Hydrators (10 个并行) │             │
│                          ├───────────────────────────────────┤             │
│                          │ ④ Pre-Scoring Filters (14 个串行) │             │
│                          ├───────────────────────────────────┤             │
│                          │ ⑤ Scorers (3 个 串行)             │             │
│                          │    Phoenix → Ranking(组合分)      │             │
│                          │    → VMRanker                     │             │
│                          ├───────────────────────────────────┤             │
│                          │ ⑥ Selector  TopKScoreSelector     │             │
│                          ├───────────────────────────────────┤             │
│                          │ ⑦ Post-Selection Hydrators (6 个) │             │
│                          ├───────────────────────────────────┤             │
│                          │ ⑧ Post-Selection Filters (3 个)   │             │
│                          ├───────────────────────────────────┤             │
│                          │ ⑨ Side Effects (异步)             │             │
│                          └───────────────────────────────────┘             │
│                                    │                                       │
│                                    ▼                                       │
│       ForYou 再加 ads / WTF / prompts / push-to-home, 走 BlenderSelector   │
│                                    │                                       │
└────────────────────────────────────┼───────────────────────────────────────┘
                                     ▼
                          gRPC 返回 FeedItem 列表
```

> 上图来自代码 `home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs` 与 `for_you_candidate_pipeline.rs` 的实际实例化顺序，并非示意。

## 两条嵌套的 Pipeline

读源码会发现一个容易绕的细节：仓库里其实有**两条** Pipeline，外层的 `ForYouCandidatePipeline` 把核心的 `PhoenixCandidatePipeline` 当作一个 Source 来调用。

| 层 | 文件 | 干什么 |
|---|---|---|
| **外层 (For You)** | `home-mixer/candidate_pipeline/for_you_candidate_pipeline.rs` | 把"已经排好序的有机帖"和**广告 / Who-To-Follow / Prompts / Push-to-Home** 一起塞进 `BlenderSelector` 排版（控制广告插入位置、品牌安全间隔等） |
| **内层 (Phoenix 核心)** | `home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs` | 真正干推荐的活：召回、注水、过滤、打分、选 Top-K、复滤 |

这种"外层负责版面、内层负责相关性"的拆分是为了让广告团队可以独立改插入策略，而不动核心排序逻辑。

## 两个 gRPC 入口

| 服务 | 路径 | 用途 |
|---|---|---|
| `ScoredPostsService` | `home-mixer/scored_posts_server.rs` | 暴露**有机帖**打分能力；供 For You 内部调用，也可供其他面（Search Home、Profile 推荐等）复用 |
| `ForYouService` | `home-mixer/for_you_server.rs` | 暴露**最终 Feed**（有机 + 广告 + WTF + Prompts 混合后） |

## Sources（候选源 · 并行）

| Source | 文件 | 类型 |
|---|---|---|
| `ThunderSource` | `sources/thunder_source.rs` | 关注网内（内存索引）|
| `PhoenixSource` | `sources/phoenix_source.rs` | Phoenix 双塔召回（关注网外）|
| `PhoenixTopicsSource` | `sources/phoenix_topics_source.rs` | Phoenix 主题维度召回 |
| `PhoenixMOESource` | `sources/phoenix_moe_source.rs` | Phoenix 混合专家召回 |
| `TweetMixerSource` | `sources/tweet_mixer_source.rs` | 复用 Tweet Mixer 服务的候选 |
| `CachedPostsSource` | `sources/cached_posts_source.rs` | 复用上一次请求缓存的候选，平滑刷新体验 |
| `AdsSource` | `sources/ads_source.rs` | 广告（**外层** Pipeline）|
| `WhoToFollowSource` | `sources/who_to_follow_source.rs` | 推荐你关注（外层）|
| `PromptsSource` | `sources/prompts_source.rs` | 顶部提示卡（外层）|
| `PushToHomeSource` | `sources/push_to_home_source.rs` | 推送-到-Home 的特殊候选（外层）|

## Scorers（打分器 · 串行）

```rust
// from phoenix_candidate_pipeline.rs
let scorers = vec![phoenix_scorer, ranking_scorer, vm_ranker];
```

| 顺序 | Scorer | 作用 |
|---|---|---|
| 1 | `PhoenixScorer` | 远程调用 Phoenix gRPC，得到**每个动作的概率**，写入 `candidate.phoenix_scores` |
| 2 | `RankingScorer` | 内部组合：先 `WeightedScorer`（多动作加权 → 单一分），再 `AuthorDiversityScorer`（同作者衰减），再 `OONScorer`（关注网外打折）|
| 3 | `VMRanker` | 接 VMRanker 服务做最后一层"价值最大化重排"（线上 A/B 测） |

> Scorer 是**串行**执行的（见 `candidate-pipeline/candidate_pipeline.rs:394` 的 `score()`），因为后一个 scorer 往往依赖前一个写入的字段。

## 选择 → 复滤 → 返回

- **Selector**：`TopKScoreSelector` 按最终分排序，截到 `RESULT_SIZE`。
- **Post-Selection Hydrators**：再补一层"只对入选的候选才值得拉"的字段（如可见性、品牌安全、互动指标、互关 Jaccard）。
- **Post-Selection Filters**：`VFFilter`（可见性）→ `AncillaryVFFilter` → `DedupConversationFilter`。
- **Side Effects**：缓存 Phoenix 请求快照、写 Kafka、更新已展示历史、上报指标。这一步是 **`tokio::spawn` 异步**的，不阻塞返回。

完整流程见 [§4 端到端流水线阶段](pipeline-stages.md)。
