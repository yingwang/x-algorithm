# 6. 过滤规则

过滤分两段：**打分前（Pre-Scoring）** 和 **选择后（Post-Selection）**。前者帮助省算力，后者负责最后一道把关。

## Pre-Scoring（14 个 · 串行）

代码顺序见 `home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs`：

| 顺序 | 过滤器 | 文件 | 作用 | 数据源 / 信号 |
|---|---|---|---|---|
| 1 | `DropDuplicatesFilter` | `drop_duplicates_filter.rs` | 去掉同一 `post_id` 的重复（多源召回时常见） | 候选自身 |
| 2 | `CoreDataHydrationFilter` | `core_data_hydration_filter.rs` | 注水失败的（连正文都没拿到的）扔掉 | `CoreDataCandidateHydrator` 写入 |
| 3 | `AgeFilter` | `age_filter.rs` | 超过 `MAX_POST_AGE` 的旧帖 | 帖子时间戳 |
| 4 | `SelfTweetFilter` | `self_tweet_filter.rs` | 自己发的帖 | 当前 user_id vs author_id |
| 5 | `RetweetDeduplicationFilter` | `retweet_deduplication_filter.rs` | 同一原帖的多次转发只留一份 | 转推链 |
| 6 | `IneligibleSubscriptionFilter` | `ineligible_subscription_filter.rs` | 用户无订阅权限的付费内容 | `SubscriptionHydrator` |
| 7 | `PreviouslySeenPostsFilter` | `previously_seen_posts_filter.rs` | 已展示（Bloom Filter 命中） | `ImpressionBloomFilterQueryHydrator` |
| 8 | `PreviouslySeenPostsBackupFilter` | `previously_seen_posts_backup_filter.rs` | 备份 Bloom Filter（主源失效时兜底） | 备用 Bloom Filter |
| 9 | `PreviouslyServedPostsFilter` | `previously_served_posts_filter.rs` | 本次会话已返回过 | `ServedHistoryQueryHydrator` |
| 10 | `MutedKeywordFilter` | `muted_keyword_filter.rs` | 命中用户屏蔽词 | 用户的 muted keywords |
| 11 | `AuthorSocialgraphFilter` | `author_socialgraph_filter.rs` | 被 block / mute 的作者 | `Blocked/Muted/Followed UserIdsQueryHydrator` |
| 12 | `VideoFilter` | `video_filter.rs` | 视频可见性 / 编码兼容性等 | 视频元信息 |
| 13 | `TopicIdsFilter` | `topic_ids_filter.rs` | 话题黑名单 | `FilteredTopicsHydrator` |
| 14 | `NewUserTopicIdsFilter` | `new_user_topic_ids_filter.rs` | 新用户额外的话题策略 | 用户冷启信号 + 话题 |

### 顺序逻辑

> "便宜的先做，贵的后做"。

- 1-5 都在候选自带的字段上做判定，几乎没有开销。
- 6-9 需要查 Redis / Bloom Filter，开销小但每候选一次。
- 10-14 需要查更多边表（屏蔽词、SocialGraph、话题边表），所以放最后。

每步执行完，剩余候选数就少一截，后续 filter 处理的样本量更小。

### Bloom Filter 双重保险

7 + 8 都是"已展示去重"，主 Bloom Filter 失效时备份顶上。Bloom Filter 是概率数据结构，假阳性会让"用户其实没看过"的帖被错误过滤；双源做或可以缓解这点。这部分体现的是"宁可少给几条、不要重复出现给同一用户"的产品取向。

## Post-Selection（3 个 · 串行）

代码：

```rust
let post_selection_filters = vec![
    Box::new(VFFilter),
    Box::new(AncillaryVFFilter),
    Box::new(DedupConversationFilter),
];
```

| 过滤器 | 文件 | 作用 |
|---|---|---|
| `VFFilter` | `vf_filter.rs` | **可见性过滤**：删除态 / 垃圾 / 暴力 / 血腥 / 仇恨等 |
| `AncillaryVFFilter` | `ancillary_vf_filter.rs` | 辅助可见性：依赖于次要安全标签的判定 |
| `DedupConversationFilter` | `dedup_conversation_filter.rs` | 同一对话树的多条分支只留一支 |

### 为什么留到最后

打分前如果做 VF：
- VF 服务延迟 100~300ms 是常态。如果对所有 N 千个候选都拉一次，整个请求 TTFB 直接被 VF 卡住。

放到 Top-K 之后：
- 只对入选的几十条做，延迟可控。
- 即使 VF 把 5 条剔掉，剩下 25 条照样能填满屏幕。

代价：被 VF 剔掉的候选**本来浪费了打分算力**。这是延迟与算力之间的取舍 —— Phoenix 算力相对充裕，延迟更稀缺。

### `DedupConversationFilter` 的意义

X 上一条原帖会衍生很多回复链。如果 Top-K 同时包含同一根对话的 5 条回复，用户体验上像看了 5 个克隆。这条 filter 用对话 root_id 做归类，每根只留 1~2 条。

## 在哪些地方做了**软**抑制（不是硬过滤）

除了硬性 filter，下列地方也算"广义过滤"，但不剔掉候选只压分：

| 机制 | 在哪 | 效果 |
|---|---|---|
| `AuthorDiversityScorer` | scorer | 同作者第 N 条乘以衰减系数 |
| `OONScorer` | scorer | 关注网外候选乘以 OON_WEIGHT_FACTOR |
| 负向动作权重 | `WeightedScorer` | block / mute / report 概率高 → 分变负 → offset 后排名靠后 |
| `MIN_VIDEO_DURATION_MS` 门槛 | `WeightedScorer::vqv_weight_eligibility` | 短视频不给 VQV 权重 |

软硬结合：硬过滤把"绝不能给"的拒之门外，软抑制让"可以给但不优先"的退后。

## Filter 框架自带的指标

每个 filter 走 `xai_candidate_pipeline::filter::Filter`，框架自动 record：

```
kept_count
removed_count
filter_rate = removed / (kept + removed)
removed_per_filter = "DropDuplicatesFilter=12, AgeFilter=3, ..."
```

线上看 `removed_per_filter` 就能判断哪条 filter 异常活跃 —— 比如某次发版后 `MutedKeywordFilter` 突然从 1% 跳到 30%，那就是关键词词典出问题。
