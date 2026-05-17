# 4. 端到端流水线阶段

这一章把 `PhoenixCandidatePipeline.execute` 里的七个核心阶段连同前后两个收尾阶段按顺序拆解开来，并结合实际代码逐一回答四个问题：每一阶段的输入是什么、做了哪些事情、输出是什么、为什么要这样设计。读完整章之后，对于一次 For You 请求从进入服务到离开服务所经历的全部过程，应当能够形成一份非常具体的认识。

本章的目标读者是想要把整套数据流彻底搞明白的工程师，因此在每一阶段都会同时给出技术性的实现细节与设计意图。如果只是想快速浏览结构，可以先看每一阶段开头的"输入加做什么加输出"那一段，再按需深入。

---

## 阶段一 · Query Hydration（查询注水，并行）

**输入：** `ScoredPostsQuery { user_id, surface, params, decider, ... }`。请求刚进来时这一对象的字段非常少，几乎只有 `user_id` 与上下文相关的几个 ID。

这里的 "surface" 指的是产品面，也就是这一次请求所对应的展示位置（例如 Home、Profile、Search 等）。"params" 包含本次请求所属的所有 feature switch 配置，"decider" 是 A/B 灰度判定器。在请求最初进入时，这些字段就已经被框架填好，而 query hydrator 要做的就是在它们的基础上进一步补充用户上下文。

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

这里有一处容易被忽略的细节需要特别澄清：召回阶段使用的动作序列与排序阶段使用的动作序列并不是同一份。召回阶段的序列被双塔结构里的用户塔编码成一个固定维度的向量；排序阶段的序列则要进入排序 Transformer，里面除了正向动作之外，还包含负反馈以及停留时长这一类连续值。两份序列的窗口长度、采样策略、负样本占比都不一样，因此分成两个 hydrator 各自拉取。具体到工程上："窗口长度"决定了模型看多远以前的历史；"采样策略"决定了在保留多少条历史的同时如何取舍；"负样本占比"则影响模型对正负反馈的学习平衡。

**输出：** 同一份 `ScoredPostsQuery`，但是所有字段都已经填好。某些字段如果拉取失败会保留为空，但不会让整个请求失败。这种"允许部分字段缺失"的设计在分布式系统里非常重要：每一个 hydrator 背后都是一次或多次远程调用，难免会出现超时或者抖动；如果让任何一次失败就让整个请求失败，那么 P99 延迟与可用性都会非常难看。

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

每一路 source 都按 fail-fast 的策略独立执行：超时或者发生异常时直接返回一份空列表，绝不阻塞主路径。"fail-fast" 是错误处理中的一种策略，指的是一旦检测到问题就立刻退出当前操作，不去尝试恢复或重试，把决策权交给上层。

**为什么这样做：** 多路召回的核心价值在于覆盖度。任何一路单独都不可能挑出全部合适的候选，多路并行汇总之后再交给后续阶段统一处理，无论从相关性还是从工程鲁棒性角度看都更稳。下面分几个层面来理解。

从相关性角度看，不同源覆盖的内容维度不同。`thunder_source` 覆盖用户主动关注的作者，符合"用户最直接的兴趣"；`phoenix_source` 通过模型相似度找出用户没关注但可能喜欢的内容；`phoenix_topics_source` 按主题维度补充，避免某些用户感兴趣的主题被前两路漏掉；`phoenix_moe_source` 走的是混合专家路径，与普通双塔召回形成互补。这种"多视角召回"是工业级推荐系统提升上限的常规手段。

从鲁棒性角度看，任何单一源都可能出现故障。`thunder_source` 依赖 Thunder 服务，`phoenix_source` 依赖 Phoenix 服务，每一个外部依赖都有自己的运维风险。多路并行召回意味着即便某一路完全失败，其它路仍然能够提供候选，整个 Feed 不会因此变成空白屏。

`cached_posts_source` 的设计同样重要：当用户在短时间内连续刷新时，这一缓存源能立刻给出一批可用的候选，避免每一次刷新都重新做一次完整的 Phoenix Retrieval。从用户体验的角度看，这种缓存能够让"连续下拉刷新"这一动作变得流畅自然。

**输出：** 一份 `Vec<PostCandidate>`，规模通常在几千条之内。各路源之间会出现大量重复（同一篇帖子可能被多路同时召回），后续的 `DropDuplicatesFilter` 会把这些重复合并掉。这种"先并集再去重"的做法在大数据处理里很常见，比"在召回阶段就保证互斥"要简单也更鲁棒。

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

这里的几个后端服务都简要介绍一下。"TES"（Tweet Entity Service）是 Twitter 系内部专门用来查询帖子详情的服务。"Gizmoduck"是用户档案服务。"SocialGraph"是社交图查询服务，承载关注、屏蔽、静音等关系的查询。"Strato"是图与数据查询服务。这些服务各自承担不同领域的数据，对外提供统一的查询接口。

**为什么这样做：** 不同字段分别来自不同的后端服务，例如 TES、Strato、Gizmoduck、SocialGraph 等。如果按串行方式逐一去拉，端到端延迟会等于所有服务延迟之和。改为并发拉取之后，整个阶段的延迟基本上只取决于最慢的那一个服务。

举一个具体的数字例子。假设 TES 的延迟是 30 毫秒，Gizmoduck 是 20 毫秒，SocialGraph 是 25 毫秒，Strato 是 35 毫秒。如果串行调用，总延迟约为 110 毫秒；如果并发调用，总延迟约为 35 毫秒（即最慢的那一路）。在用户感知层面，这一差距非常显著。这种"按字段切并发"的设计是大规模在线服务中的常见做法，本质上是用 IO 等待时间来换 CPU 等待时间。

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

这种"先廉价后昂贵"的过滤顺序在数据库查询优化里被称作"谓词下推"或者"早过滤"。其核心思想很简单：处理 1000 条数据所花费的代价远小于处理 100 万条数据所花费的代价，因此应当让候选数尽快变小，再去做那些需要更精细判断的工作。

**为什么先过滤再打分：** 排序模型的推理是单次请求中最昂贵的一步，每减少一条候选都直接节省 Phoenix gRPC 的算力。把过滤放在打分之前可以显著降低整个请求的成本。这一点对于一个每天承载数亿次请求的系统来说，成本节省是非常可观的。

**输出：** 一份干净的候选列表，加上一份"被剔除掉的"候选列表。被剔除掉的列表仍然保留下来，供 side effect 阶段写入训练日志使用。在机器学习里，这一类被剔除的候选可以作为"困难负样本"出现在训练数据中，帮助模型学到更精细的判别能力。

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

这一组动作覆盖了用户与帖子可能发生互动的几乎所有形式，从最显式的点赞、转推、回复，到比较隐式的点击详情、放大图片、停留时长，再到负向的不感兴趣、拉黑、举报。让模型对每一种动作都单独预测概率，这就是所谓的"多任务学习"（multi-task learning）。多任务学习的好处在于：所有任务共享同一份底层表示，让模型可以从更丰富的监督信号中受益；同时每一种动作的概率单独可见，便于线上调权、监控与解释。

与此同时，`EgressPhoenixPredictionClient`（属于实验路径）也会被异步触发，相关逻辑见 `PhoenixExperimentsSideEffect`。这一路调用的结果不会影响当次请求所返回给用户的内容，但会被写入 Kafka，供离线分析作为对照。这种"在线影子流量"机制让新版模型可以在不打扰用户的前提下接受真实流量的检验。

### 5.2 RankingScorer

`RankingScorer` 本身并不是单一的打分器，而是由三个子 scorer 顺序组合而成，逐一应用到候选上。这种"组合 scorer"的写法让原本可能彼此影响的多种打分逻辑被清晰地拆分到独立模块，每一个模块只关心自己的职责。

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

这里有一个特殊的处理：VQV（视频质量观看）权重存在门槛，只有视频时长大于 `MIN_VIDEO_DURATION_MS` 时这一权重才会生效。这样做是为了避免某些以"伪视频卡片"刷分的行为。比方说，一些极短的视频卡片在用户毫无意识地划过时就会触发"视频观看"信号，如果不加约束地把这种信号计入分数，会导致这一类候选被异常推高。

更为关键的一点是：负向动作（包括 not_interested、block、mute、report 等）所对应的权重都是负数。这种设计让模型预测出来的"高厌恶概率"会直接拖低 final score，而不是被简单忽略。这一点与传统单一相关性打分有根本性区别：传统做法只问"用户多大概率喜欢"，本仓库的做法同时问"用户多大概率讨厌"，并把答案体现到最终排序里。

#### 5.2.2 AuthorDiversityScorer（同作者衰减）

如果不加约束，同一位作者所发的帖子可能因为相关性都很高而集中出现在 Feed 顶部，最终让用户看到的内容缺乏多样性。`AuthorDiversityScorer` 的作用就是对同一作者的多次出现引入衰减。

```rust
multiplier(position) = (1 - floor) * decay^position + floor
adjusted = weighted_score * multiplier(同作者第几次出现)
```

第一次出现的同作者帖子 multiplier 等于 1，第二次衰减为 `floor + (1 - floor) * decay`，依此类推。`floor` 这一参数的存在是为了防止把同一作者完全压死：即便是第十条同作者帖子，它的分数也不会被砍到零。

这种"指数衰减加下限"的曲线在工程上有几个好处：第一，曲线是平滑的，不会在某一个 position 上出现陡降；第二，前几次出现的衰减相对剧烈，让"作者级多样性"很快生效；第三，下限的存在让那些真正优质的作者多产时仍然能被推出来，不会被简单的硬规则一刀切。

算法的另一个细节同样值得展开：

```rust
let mut ordered: Vec<(usize, &PostCandidate)> = candidates.iter().enumerate().collect();
ordered.sort_by(|(_, a), (_, b)| b.weighted_score.partial_cmp(&a.weighted_score)...);
for (original_idx, candidate) in ordered { ... }
```

这一段代码先把候选按当前分数倒序排好，再按这一顺序进行编号。这意味着每一位作者得分最高的那一条 multiplier 等于 1，根本不会被衰减；只有分数较低的几条才会被打折。这一处理方式背后的取向是"保留好的、压制重复的"。

#### 5.2.3 OONScorer（关注网外打折）

```rust
if c.in_network == Some(false):
    score *= OON_WEIGHT_FACTOR
```

`OONScorer` 的目的是在"全部相关"的前提下让关注网内的候选稍微占优一些。原因在于：默认希望让用户看到更多自己关注的人所发出的内容，但又不希望走到"完全只看关注的人"那么极端。这一系数可以通过 feature switch 在线调节。

这种"温和打折"的做法是一种产品策略选择：既保留了 OON 内容进入 Feed 的可能性（高质量 OON 内容仍然可以通过 Phoenix 拿到很高的相关性分数最终上榜），又让 In-Network 内容在相关性近似的情况下占优。比"按比例固定填充"的做法要灵活得多。

### 5.3 VMRanker

整套打分阶段的最后一步是 `VMRanker`，它通过远程调用一个独立的服务完成"价值最大化重排"。"价值最大化"在这里是一个产品层面的术语，意指除了相关性之外，还要兼顾业务目标，例如某些类型帖子的曝光配额、商业化目标等。这一层与 Phoenix 的纯相关性视角是正交的，两者结合起来给出最终分数。

VMRanker 本身在开源代码中只暴露了 client，模型逻辑并不公开。这是一种常见的安排：与商业化高度耦合的算法往往不会被一并开源，因为它们包含特定平台的商业秘密。

**输出：** 每一条候选都获得一个 `score: f64` 字段，下一阶段直接按这一字段排序。`f64` 是 Rust 中的 64 位浮点数类型，相当于 C 中的 double。

---

## 阶段六 · Selection（选择）

`TopKScoreSelector` 把候选按 `score` 倒序排列，截取前 K 条（K 等于 `RESULT_SIZE`）。返回值是一对 `(selected, non_selected)`，后者仍然保留，供 side effect 阶段使用。

这一阶段表面上简单，但它是整套流程中"漏斗"明显收紧的位置。前面经过了召回、注水、过滤、打分四个阶段精心准备出来的几千条候选，到这里被压缩到最终的几十条。也正是因为压缩比例如此之大，前面阶段的每一步设计都直接影响到最终呈现给用户的内容质量。这种"漏斗式"的处理是推荐系统的标准架构，每一层漏斗从粗到精，把候选规模逐步压缩到可以人眼直接看的量级。

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

举一个具体的延迟代价例子。假设 VF 服务对每一条候选的查询延迟是 5 毫秒，那么如果对 5000 条候选都查一次，总延迟会变成 5000 × 5 = 25000 毫秒，显然完全不可接受（即便用并发也很可能压不住）；如果只对入选的 50 条查询，总延迟就是 50 × 5 = 250 毫秒，完全可承受。当然，实际实现中也会做批量与并发，数字会更优；这里只是用来说明"延迟昂贵注水"的必要性。

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

`DedupConversationFilter` 处理的是 X 平台特有的一种重复问题：原帖会衍生大量回复链，如果同一根对话树上的多条回复都被排在前面，用户看到的体验会像看了五份克隆。这一过滤器按 `root_id`（对话根 ID）做归类，每一根对话最多只保留一两条，从而保证 Feed 在话题层面的多样性。

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

这种"主响应快速返回、副作用后台处理"的模式在大规模在线服务里被反复证明是行之有效的设计。它的本质是把"必须立刻完成才能给用户回话"的工作与"完成了对用户没有直接影响但有长期价值"的工作分开处理，前者放在关键路径里追求最低延迟，后者放在后台路径里追求最终一致性。如果某一个副作用因为下游故障而失败，主响应不会受到任何影响；下次健康检查发现后，运维同学可以单独修复副作用通路。
