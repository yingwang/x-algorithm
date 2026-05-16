# 4. 端到端流水线阶段

下面把 `PhoenixCandidatePipeline.execute` 的 7 + 2 个阶段按顺序拆开，结合实际代码说**输入是什么、做了什么、输出是什么、为什么这样做**。

---

## 阶段 ① Query Hydration（查询注水 · 并行）

**输入：** `ScoredPostsQuery { user_id, surface, params, decider, ... }` —— 几乎只有 user_id 和上下文 ID。

**做什么：** 并行执行 15 个 `QueryHydrator`，把用户上下文填充进 query：

| Hydrator | 拉什么 |
|---|---|
| `ScoringSequenceQueryHydrator` | 用户最近的互动序列（喂给排序模型） |
| `RetrievalSequenceQueryHydrator` | 用户最近的互动序列（喂给召回模型，可能用不同窗口/采样） |
| `Blocked` / `Muted` / `Followed` / `Subscribed` `UserIdsQueryHydrator` | 黑名单 / 静音 / 关注 / 订阅 |
| `CachedPostsQueryHydrator` | 从 Redis 拿上一次请求的候选缓存 |
| `MutualFollowQueryHydrator` | 互关图（用于 Jaccard 相似度后续打分） |
| `UserDemographicsQueryHydrator` / `UserInferredGenderQueryHydrator` | 人口学信号 |
| `FollowedGrokTopicsQueryHydrator` · `FollowedStarterPacksQueryHydrator` · `InferredGrokTopicsQueryHydrator` | 兴趣话题 / Starter Pack |
| `ImpressionBloomFilterQueryHydrator` | 已展示过帖子的 Bloom Filter |
| `IpQueryHydrator` | IP → 地理 / 风险 |

**为什么这样：** Phoenix 模型几乎不接手工特征，但仍然需要"用户最近做了什么"这样的**输入序列**。这些 hydrator 就是在请求级别一次性把序列拉好，后续 scorer 不再回头取。

> ⚠️ 注意区分**两条 sequence**：用于召回的（双塔 User Tower 编码）和用于排序的（含负反馈、连续值）。两者的窗口长度、采样策略、负样本占比都不一样，所以分两个 hydrator。

**输出：** 同一个 `ScoredPostsQuery`，但所有字段已填好（拉失败的字段为空，不报错）。

---

## 阶段 ② Candidate Sourcing（候选召回 · 并行）

**输入：** hydrated query。

**做什么：** 并行执行 6 路 source：

```rust
let sources = vec![
    thunder_source,        // 关注网内（亚毫秒级内存查询）
    tweet_mixer_source,    // 来自 TweetMixer 的混合候选
    phoenix_source,        // Phoenix 双塔召回（核心 OON 源）
    phoenix_topics_source, // Phoenix 按"话题"召回
    phoenix_moe_source,    // Phoenix 混合专家召回
    cached_posts_source,   // 上一次请求缓存
];
```

每路独立 fail-fast：超时或异常都返回空列表，主链路继续。

**为什么这样：** 多路召回带来覆盖度，单路失败时其他路兜底。`cached_posts_source` 的存在让"用户刷新太快时"也能立刻出结果，而不必每次重做 Phoenix Retrieval。

**输出：** `Vec<PostCandidate>`，可能数千条，互相之间会有大量重复（同一 post 被多源召回） —— 后面 `DropDuplicatesFilter` 来收。

---

## 阶段 ③ Candidate Hydration（候选注水 · 并行）

**输入：** 上一步的所有候选。

**做什么：** 并行 10 个 hydrator 给候选补字段：

| Hydrator | 字段 | 来源 |
|---|---|---|
| `InNetworkCandidateHydrator` | `in_network: bool` | 查关注列表 |
| `CoreDataCandidateHydrator` | 正文、媒体、时间戳 | TES（Tweet Entity Service） |
| `QuoteHydrator` | 引用帖展开 | TES + SocialGraph |
| `VideoDurationCandidateHydrator` | 视频时长 | TES |
| `HasMediaHydrator` | 是否含媒体 | TES |
| `SubscriptionHydrator` | 付费墙信息 | TES |
| `GizmoduckCandidateHydrator` | 作者档案 | Gizmoduck |
| `BlockedByHydrator` | 作者是否屏蔽了当前用户 | SocialGraph |
| `FilteredTopicsHydrator` | 帖子话题标签 | Strato |
| `LanguageCodeHydrator` | 帖子语言 | TES |

**为什么这样：** 不同字段来自不同后端服务（TES / Strato / Gizmoduck / SocialGraph），并发拉的延迟 ≈ 最慢的那个，而不是各服务延迟之和。

**输出：** 字段补全的候选列表。注水失败的字段为空，下一步过滤会把不完整的剔掉。

---

## 阶段 ④ Pre-Scoring Filters（打分前过滤 · 串行）

**输入：** 注水后的候选。

**做什么：** 14 个 filter 按顺序剔候选。完整列表见 [§6 过滤规则](filters.md)。

```rust
let filters = vec![
    DropDuplicatesFilter,           // 去重
    CoreDataHydrationFilter,        // 正文都没拉到的扔掉
    AgeFilter(MAX_POST_AGE),        // 太老的扔掉
    SelfTweetFilter,                // 自己的帖
    RetweetDeduplicationFilter,     // 同原帖的转推只留一份
    IneligibleSubscriptionFilter,   // 用户无权看的付费内容
    PreviouslySeenPostsFilter,      // 已展示（Bloom Filter）
    PreviouslySeenPostsBackupFilter,// 备份 Bloom Filter
    PreviouslyServedPostsFilter,    // 本会话已返回
    MutedKeywordFilter,             // 屏蔽词命中
    AuthorSocialgraphFilter,        // 被 block / mute 的作者
    VideoFilter,                    // 视频可见性
    TopicIdsFilter,                 // 话题黑名单
    NewUserTopicIdsFilter,          // 新用户额外的话题策略
];
```

**为什么串行：** 后面的 filter 通常更贵（要打 Bloom Filter 哈希、读 Redis 等）。先用便宜的 filter 把候选数压下来再做贵的，是经典的"过滤优化"思路。框架的 `run_filters` 不会跳过 —— 串行就是设计意图。

**为什么先过滤再打分：** 排序模型推理是单请求最贵的一步（Phoenix gRPC）。每减少 1 条候选都直接省 Phoenix 的算力。

**输出：** 干净的候选列表 + 一份"被剔掉的"（保留供 side effect 写训练日志）。

---

## 阶段 ⑤ Scoring（打分 · 串行）

**输入：** 过滤后的候选。

**做什么：** 3 个 scorer 按顺序：

### 5.1 `PhoenixScorer`

远程调用 Phoenix gRPC（`PhoenixPredictionClient`），把用户的 `scoring_sequence` + 候选打包发过去，模型返回每个候选 × 每个动作的概率，写入 `candidate.phoenix_scores`：

```rust
struct PhoenixScores {
    favorite_score, reply_score, retweet_score, quote_score,
    click_score, profile_click_score,
    photo_expand_score, video_view_score (vqv),
    share_score, share_via_dm_score, share_via_copy_link_score,
    dwell_score, dwell_time,                    // dwell_time 是连续值
    quoted_click_score,
    follow_author_score,
    // 负向动作
    not_interested_score, block_author_score, mute_author_score, report_score,
}
```

同时 `EgressPhoenixPredictionClient`（实验路径）也会被异步触发 —— 见 `PhoenixExperimentsSideEffect`。

### 5.2 `RankingScorer`

内部组合了三个子 scorer，逐一应用：

#### 5.2.1 `WeightedScorer`（多动作加权 → 单一分）

```rust
combined = Σ_i  weight_i × phoenix_scores.action_i
// 然后做 offset 归一化
if combined < 0:
    combined = (combined + NEGATIVE_WEIGHTS_SUM) / WEIGHTS_SUM * NEGATIVE_SCORES_OFFSET
else:
    combined = combined + NEGATIVE_SCORES_OFFSET
```

特殊：**VQV（视频质量观看）权重门槛** —— 只有视频时长 > `MIN_VIDEO_DURATION_MS` 才生效（避免短视频卡片刷分）。

负向动作（not_interested / block / mute / report）权重是负数，让模型预测的"高厌恶概率"直接拖低 final score。

#### 5.2.2 `AuthorDiversityScorer`（同作者衰减）

```rust
multiplier(position) = (1 - floor) * decay^position + floor
adjusted = weighted_score * multiplier(同作者第几次出现)
```

第一次出现的同作者帖子 multiplier = 1，第二次 = `floor + (1-floor)*decay`，依此衰减。**`floor`** 防止彻底封禁——即使是第 10 条同作者帖，分数也不会被砍到 0。

注意算法的细节：先把候选按当前分数倒序排，**按这个顺序计数**。这意味着同一作者得分最高的那条永远保留全权重，分越低的越被衰减。

#### 5.2.3 `OONScorer`（关注网外打折）

```rust
if c.in_network == Some(false):
    score *= OON_WEIGHT_FACTOR    // 通常 < 1
```

为什么：默认想让关注网内多一些，但又不要"只看关注的人"那么极端。系数可以通过 feature switch 调。

### 5.3 `VMRanker`

最后一层"价值最大化重排"，调远程 `VMRanker` 服务。这一层主要承担产品层目标（比如某些类型帖子的曝光配额、商业化目标），与 Phoenix 纯相关性正交。

**输出：** 每条候选都有 `score: f64`，下一步直接按它排序。

---

## 阶段 ⑥ Selection

`TopKScoreSelector` 按 `score` 倒序，取前 K（K = `RESULT_SIZE`）。返回 `(selected, non_selected)`，后者也保留供 side effect。

---

## 阶段 ⑦ Post-Selection Hydration（并行）

只对入选的候选才做的"贵注水"：

| Hydrator | 字段 |
|---|---|
| `VFCandidateHydrator` | 可见性标签（被删 / 仇恨 / 暴力等） |
| `AdsBrandSafetyHydrator` · `AdsBrandSafetyVfHydrator` | 品牌安全标签（供广告间隔判定） |
| `TweetTypeMetricsHydrator` | 帖类型 metric |
| `FollowingRepliedUsersHydrator` | "你回复过的人"信号 |
| `MutualFollowJaccardHydrator` | 互关 Jaccard 系数 |

为什么放到选择**之后**：这些字段计算贵，对没入选的候选拉了也白拉。"选完再补可见性"是经典的延迟补全策略。

---

## 阶段 ⑧ Post-Selection Filters（串行）

最后把关：

```rust
let post_selection_filters = vec![
    VFFilter,                     // 可见性
    AncillaryVFFilter,            // 辅助可见性
    DedupConversationFilter,      // 同一对话树多分支去重
];
```

**为什么留到最后**：如果在打分前过滤可见性，万一可见性服务卡 100ms，就让整个排序都卡 100ms；放到选择后，只对 Top-K 几十条做，几乎无延迟代价。

---

## 阶段 ⑨ Truncate + Finalize + Side Effects（异步）

```rust
let truncated = final_candidates.split_off(result_size().min(final_candidates.len()));
non_selected_candidates.extend(truncated);

self.finalize(query, &mut final_candidates);   // hook
self.stat_result_size(&final_candidates);      // 0-result counter

tokio::spawn(async move {                       // 副作用全部异步
    PhoenixExperimentsSideEffect ...
    RerankingKafkaSideEffect ...
    RedisPostCandidateCacheSideEffect ...
    PhoenixRequestCacheSideEffect ...
    ScoredStatsSideEffect ...
    MutualFollowStatsSideEffect ...
    UpdateServedHistorySideEffect ...
    TruncateServedHistorySideEffect ...
});
```

副作用一旦 spawn 出去，主请求**立刻**返回 —— 用户看到的 Feed 与"是否写完 Kafka"无关。
