# 4. 端到端流水线阶段

这一章把 `PhoenixCandidatePipeline.execute` 里的七个核心阶段连同前后两个收尾阶段按顺序拆解开来，并结合实际代码逐一回答四个问题：每一阶段的输入是什么、做了哪些事情、输出是什么、为什么要这样设计。读完整章之后，对于一次 For You 请求从进入服务到离开服务所经历的全部过程，应当能够形成一份非常具体的认识。

---

## 阶段一 · Query Hydration（查询注水，并行）

**输入：** `ScoredPostsQuery { user_id, surface, params, decider, ... }`。请求刚进来时这一对象的字段非常少，几乎只有 `user_id` 与上下文相关的几个 ID。

**做什么：** 并行执行十五个 `QueryHydrator`，把用户的上下文信息一次性填充进 query 中，供后续所有阶段共用。

| Hydrator | 拉什么 |
|---|---|
| `ScoringSequenceQueryHydrator` | 用户最近的互动序列，专供排序模型使用 |
| `RetrievalSequenceQueryHydrator` | 用户最近的互动序列，专供召回模型使用（可能采用不同的窗口与采样策略） |
| `Blocked`、`Muted`、`Followed`、`Subscribed` 一系列 `UserIdsQueryHydrator` | 黑名单、静音、关注、订阅 |
| `CachedPostsQueryHydrator` | 从 Redis 取上一次请求的候选缓存 |
| `MutualFollowQueryHydrator` | 互关图，供后续 Jaccard 相似度打分使用 |
| `UserDemographicsQueryHydrator` 与 `UserInferredGenderQueryHydrator` | 人口学信号 |
| `FollowedGrokTopicsQueryHydrator`、`FollowedStarterPacksQueryHydrator`、`InferredGrokTopicsQueryHydrator` | 兴趣话题与 Starter Pack |
| `ImpressionBloomFilterQueryHydrator` | 已展示过帖子的 Bloom Filter |
| `IpQueryHydrator` | IP 到地理位置以及风险信号 |

**为什么这样做：** Phoenix 模型几乎不依赖任何手工特征，但它仍然需要"用户最近做了什么"这一段输入序列。这些 hydrator 的存在就是为了在请求级别上一次性把序列拉好，后续的 scorer 不需要再回头取一次。

这里有一处容易被忽略的细节需要特别澄清：召回阶段使用的动作序列与排序阶段使用的动作序列并不是同一份。召回阶段的序列被双塔结构里的用户塔编码成一个固定维度的向量；排序阶段的序列则要进入排序 Transformer，里面除了正向动作之外，还包含负反馈以及停留时长这一类连续值。两份序列的窗口长度、采样策略、负样本占比都不一样，因此分成两个 hydrator 各自拉取。

**输出：** 同一份 `ScoredPostsQuery`，但是所有字段都已经填好。某些字段如果拉取失败会保留为空，但不会让整个请求失败。

---

## 阶段二 · Candidate Sourcing（候选召回，并行）

**输入：** 已经完成 hydration 的 query。

**做什么：** 并行执行六路候选源，每一路独立返回一批候选。下面这段代码体现了召回阶段的实例化顺序。

```rust
let sources = vec![
    thunder_source,        // 关注网内，亚毫秒级内存查询
    tweet_mixer_source,    // 来自 TweetMixer 的混合候选
    phoenix_source,        // Phoenix 双塔召回，关注网外的主要候选源
    phoenix_topics_source, // Phoenix 按"话题"做的召回
    phoenix_moe_source,    // Phoenix 混合专家路径的召回
    cached_posts_source,   // 上一次请求所缓存的候选
];
```

每一路 source 都按 fail-fast 的策略独立执行：超时或者发生异常时直接返回一份空列表，绝不阻塞主路径。

**为什么这样做：** 多路召回的核心价值在于覆盖度。任何一路单独都不可能挑出全部合适的候选，多路并行汇总之后再交给后续阶段统一处理，无论从相关性还是从工程鲁棒性角度看都更稳。`cached_posts_source` 的设计同样重要：当用户在短时间内连续刷新时，这一缓存源能立刻给出一批可用的候选，避免每一次刷新都重新做一次完整的 Phoenix Retrieval。

**输出：** 一份 `Vec<PostCandidate>`，规模通常在几千条之内。各路源之间会出现大量重复（同一篇帖子可能被多路同时召回），后续的 `DropDuplicatesFilter` 会把这些重复合并掉。

---

## 阶段三 · Candidate Hydration（候选注水，并行）

**输入：** 上一步召回出来的所有候选。

**做什么：** 并行执行十个 hydrator，给每一条候选补全各项字段。

| Hydrator | 字段 | 来源 |
|---|---|---|
| `InNetworkCandidateHydrator` | `in_network: bool` | 查询关注列表 |
| `CoreDataCandidateHydrator` | 正文、媒体、时间戳 | TES（Tweet Entity Service） |
| `QuoteHydrator` | 引用帖展开 | TES 与 SocialGraph |
| `VideoDurationCandidateHydrator` | 视频时长 | TES |
| `HasMediaHydrator` | 是否含有媒体 | TES |
| `SubscriptionHydrator` | 付费墙信息 | TES |
| `GizmoduckCandidateHydrator` | 作者档案 | Gizmoduck |
| `BlockedByHydrator` | 作者是否屏蔽了当前用户 | SocialGraph |
| `FilteredTopicsHydrator` | 帖子的话题标签 | Strato |
| `LanguageCodeHydrator` | 帖子的语言代码 | TES |

**为什么这样做：** 不同字段分别来自不同的后端服务，例如 TES、Strato、Gizmoduck、SocialGraph 等。如果按串行方式逐一去拉，端到端延迟会等于所有服务延迟之和。改为并发拉取之后，整个阶段的延迟基本上只取决于最慢的那一个服务。这种"按字段切并发"的设计是大规模在线服务中的常见做法。

**输出：** 字段已经被补全的候选列表。注水失败的字段会被保留为空，紧随其后的过滤阶段会把那些核心字段缺失的候选剔除掉。

---

## 阶段四 · Pre-Scoring Filters（打分前过滤，串行）

**输入：** 注水之后的候选。

**做什么：** 串行执行十四个过滤器，逐一剔除不合规的候选。完整的过滤器清单见[过滤规则](filters.md)一章。

```rust
let filters = vec![
    DropDuplicatesFilter,           // 去重
    CoreDataHydrationFilter,        // 核心元数据缺失的扔掉
    AgeFilter(MAX_POST_AGE),        // 时间过老的扔掉
    SelfTweetFilter,                // 自己发的帖子
    RetweetDeduplicationFilter,     // 同一原帖的转推只保留一份
    IneligibleSubscriptionFilter,   // 用户无权查看的付费内容
    PreviouslySeenPostsFilter,      // 已曝光过的，由 Bloom Filter 判断
    PreviouslySeenPostsBackupFilter,// 备份的 Bloom Filter
    PreviouslyServedPostsFilter,    // 本会话已经返回过的
    MutedKeywordFilter,             // 命中屏蔽关键词的
    AuthorSocialgraphFilter,        // 作者被屏蔽或被静音的
    VideoFilter,                    // 视频可见性
    TopicIdsFilter,                 // 话题黑名单
    NewUserTopicIdsFilter,          // 新用户额外的话题策略
];
```

**为什么要串行：** 这一段串行执行是设计意图，而不是偶然的实现选择。后面的 filter 通常更"贵"，例如需要计算 Bloom Filter 哈希、读 Redis 等。先用便宜的 filter 把候选数压下来，再去做贵的，是经典的"过滤优化"思路。框架内部的 `run_filters` 函数不会跳过任何一个 filter，每一条候选都必须依次经过全部 filter 才能进入下一阶段。

**为什么先过滤再打分：** 排序模型的推理是单次请求中最昂贵的一步，每减少一条候选都直接节省 Phoenix gRPC 的算力。把过滤放在打分之前可以显著降低整个请求的成本。

**输出：** 一份干净的候选列表，加上一份"被剔除掉的"候选列表。被剔除掉的列表仍然保留下来，供 side effect 阶段写入训练日志使用。

---

## 阶段五 · Scoring（打分，串行）

**输入：** 过滤之后的候选。

**做什么：** 串行执行三个 scorer。

### 5.1 PhoenixScorer

`PhoenixScorer` 通过 `PhoenixPredictionClient` 远程调用 Phoenix gRPC 服务。它把用户的 `scoring_sequence` 与全部候选打包发送过去，模型返回每一条候选在每一种动作维度上的概率，全部写入 `candidate.phoenix_scores` 字段中。下面这一段代码列出了所有被预测的动作类型。

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

与此同时，`EgressPhoenixPredictionClient`（属于实验路径）也会被异步触发，相关逻辑见 `PhoenixExperimentsSideEffect`。这一路调用的结果不会影响当次请求所返回给用户的内容，但会被写入 Kafka，供离线分析作为对照。

### 5.2 RankingScorer

`RankingScorer` 本身并不是单一的打分器，而是由三个子 scorer 顺序组合而成，逐一应用到候选上。

#### 5.2.1 WeightedScorer（多动作加权求和）

`WeightedScorer` 的核心逻辑是把多个动作的概率按预设权重组合成一个标量分数。

```rust
combined = Σ_i  weight_i × phoenix_scores.action_i
// 然后做 offset 归一化
if combined < 0:
    combined = (combined + NEGATIVE_WEIGHTS_SUM) / WEIGHTS_SUM * NEGATIVE_SCORES_OFFSET
else:
    combined = combined + NEGATIVE_SCORES_OFFSET
```

这里有一个特殊的处理：VQV（视频质量观看）权重存在门槛，只有视频时长大于 `MIN_VIDEO_DURATION_MS` 时这一权重才会生效。这样做是为了避免某些以"伪视频卡片"刷分的行为。

更为关键的一点是：负向动作（包括 not_interested、block、mute、report 等）所对应的权重都是负数。这种设计让模型预测出来的"高厌恶概率"会直接拖低 final score，而不是被简单忽略。这一点与传统单一相关性打分有根本性区别。

#### 5.2.2 AuthorDiversityScorer（同作者衰减）

如果不加约束，同一位作者所发的帖子可能因为相关性都很高而集中出现在 Feed 顶部，最终让用户看到的内容缺乏多样性。`AuthorDiversityScorer` 的作用就是对同一作者的多次出现引入衰减。

```rust
multiplier(position) = (1 - floor) * decay^position + floor
adjusted = weighted_score * multiplier(同作者第几次出现)
```

第一次出现的同作者帖子 multiplier 等于 1，第二次衰减为 `floor + (1 - floor) * decay`，依此类推。`floor` 这一参数的存在是为了防止把同一作者完全压死：即便是第十条同作者帖子，它的分数也不会被砍到零。

算法的另一个细节同样值得展开：在做衰减之前，会先把候选按当前分数倒序排序，然后按这一顺序逐一统计同作者出现次数。这意味着同一位作者得分最高的那一篇永远保留全权重，得分越低的越会被衰减。

#### 5.2.3 OONScorer（关注网外打折）

```rust
if c.in_network == Some(false):
    score *= OON_WEIGHT_FACTOR    // 通常小于 1
```

`OONScorer` 的目的是在"全部相关"的前提下让关注网内的候选稍微占优一些。原因在于：默认希望让用户看到更多自己关注的人所发出的内容，但又不希望走到"完全只看关注的人"那么极端。这一系数可以通过 feature switch 在线调节。

### 5.3 VMRanker

整套打分阶段的最后一步是 `VMRanker`，它通过远程调用一个独立的服务完成"价值最大化重排"。这一层的主要价值不在于相关性本身，而在于承担产品层面的目标，例如某些类型帖子的曝光配额、商业化目标等。它与 Phoenix 的纯相关性视角是正交的，两者结合起来给出最终分数。

**输出：** 每一条候选都获得一个 `score: f64` 字段，下一阶段直接按这一字段排序。

---

## 阶段六 · Selection（选择）

`TopKScoreSelector` 把候选按 `score` 倒序排列，截取前 K 条（K 等于 `RESULT_SIZE`）。返回值是一对 `(selected, non_selected)`，后者仍然保留，供 side effect 阶段使用。

这一阶段表面上简单，但它是整套流程中"漏斗"明显收紧的位置。前面经过了召回、注水、过滤、打分四个阶段精心准备出来的几千条候选，到这里被压缩到最终的几十条。也正是因为压缩比例如此之大，前面阶段的每一步设计都直接影响到最终呈现给用户的内容质量。

---

## 阶段七 · Post-Selection Hydration（选择后注水，并行）

这一阶段只对已经入选的候选做进一步的"昂贵注水"。下面这张表把主要的几个 hydrator 列了出来。

| Hydrator | 字段 |
|---|---|
| `VFCandidateHydrator` | 可见性标签（被删除、仇恨内容、暴力内容等） |
| `AdsBrandSafetyHydrator` 与 `AdsBrandSafetyVfHydrator` | 品牌安全标签，供广告间隔判定使用 |
| `TweetTypeMetricsHydrator` | 帖类型相关指标 |
| `FollowingRepliedUsersHydrator` | "你回复过的人"信号 |
| `MutualFollowJaccardHydrator` | 互关 Jaccard 系数 |

之所以要把这一组 hydrator 放到选择阶段之后，原因在于它们的计算代价相对较高。对那些没有入选的候选拉这些字段是一种纯粹的浪费。"选完再补可见性"这样一种延迟补全策略，是大规模推荐系统里反复出现的优化手法。

---

## 阶段八 · Post-Selection Filters（选择后过滤，串行）

经过选择与昂贵注水之后，还要再做最后一道把关。

```rust
let post_selection_filters = vec![
    VFFilter,                     // 可见性
    AncillaryVFFilter,            // 辅助可见性
    DedupConversationFilter,      // 同一对话树多分支去重
];
```

**为什么留到最后才做：** 如果在打分之前过滤可见性，万一可见性服务卡上 100 毫秒，整个排序流程也跟着卡 100 毫秒；放到选择之后，只对几十条 Top-K 做这层过滤，几乎不会带来额外延迟代价。这种"昂贵但必要的过滤推迟到最后"的做法，与上一阶段的"昂贵注水推迟到最后"在思想上是一致的。

---

## 阶段九 · Truncate、Finalize 与 Side Effects（异步）

最后一段是整个流程的收尾。

```rust
let truncated = final_candidates.split_off(result_size().min(final_candidates.len()));
non_selected_candidates.extend(truncated);

self.finalize(query, &mut final_candidates);   // 钩子函数
self.stat_result_size(&final_candidates);      // 空结果计数器

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

整个 side effect 一旦通过 `tokio::spawn` 被发起出去，主请求会立刻返回。换言之，用户看到的 Feed 完全不需要等待 Kafka、Redis、监控指标这些后续写入操作完成。这种把"副作用与主响应解耦"的工程做法让用户感知到的延迟仅取决于核心计算路径，与下游写入操作的稳定性完全无关。
