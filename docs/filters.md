# 6. 过滤规则

整条推荐流水线里的过滤逻辑被分成了两段，分别位于打分前与选择后这两个位置。打分前的过滤主要承担"节省算力"的职责：把那些注定不应当被推给用户的候选尽早剔除，避免它们消耗掉昂贵的 Phoenix 推理算力。选择后的过滤则承担"最后一道把关"的职责：在已经排好序的少量入选候选上做更精细但相对昂贵的合规检查。

## 打分前过滤（共 14 个，串行执行）

打分前过滤器的具体执行顺序写在 `home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs` 文件中。下面这张表完整地列出了每一个过滤器的位置、作用与所依赖的信号来源。

| 顺序 | 过滤器 | 文件 | 作用 | 数据源与信号 |
|---|---|---|---|---|
| 1 | `DropDuplicatesFilter` | `drop_duplicates_filter.rs` | 去掉同一 `post_id` 的重复候选，多路召回时常见 | 候选自身 |
| 2 | `CoreDataHydrationFilter` | `core_data_hydration_filter.rs` | 注水失败的候选（连正文都没拉到）直接扔掉 | `CoreDataCandidateHydrator` 写入 |
| 3 | `AgeFilter` | `age_filter.rs` | 时间超过 `MAX_POST_AGE` 的旧帖扔掉 | 帖子时间戳 |
| 4 | `SelfTweetFilter` | `self_tweet_filter.rs` | 当前用户自己发的帖子 | 当前 `user_id` 与 `author_id` 对比 |
| 5 | `RetweetDeduplicationFilter` | `retweet_deduplication_filter.rs` | 同一原帖的多次转发只保留一份 | 转推链 |
| 6 | `IneligibleSubscriptionFilter` | `ineligible_subscription_filter.rs` | 用户不具备订阅权限的付费内容 | `SubscriptionHydrator` |
| 7 | `PreviouslySeenPostsFilter` | `previously_seen_posts_filter.rs` | 已展示过的（Bloom Filter 命中） | `ImpressionBloomFilterQueryHydrator` |
| 8 | `PreviouslySeenPostsBackupFilter` | `previously_seen_posts_backup_filter.rs` | 备份 Bloom Filter，在主源失效时兜底 | 备用 Bloom Filter |
| 9 | `PreviouslyServedPostsFilter` | `previously_served_posts_filter.rs` | 本次会话已经返回过的候选 | `ServedHistoryQueryHydrator` |
| 10 | `MutedKeywordFilter` | `muted_keyword_filter.rs` | 命中用户屏蔽词的 | 用户的 muted keywords 列表 |
| 11 | `AuthorSocialgraphFilter` | `author_socialgraph_filter.rs` | 作者被当前用户 block 或 mute 的 | `Blocked/Muted/Followed UserIdsQueryHydrator` |
| 12 | `VideoFilter` | `video_filter.rs` | 视频可见性、编码兼容性等 | 视频元信息 |
| 13 | `TopicIdsFilter` | `topic_ids_filter.rs` | 话题黑名单 | `FilteredTopicsHydrator` |
| 14 | `NewUserTopicIdsFilter` | `new_user_topic_ids_filter.rs` | 新用户额外的话题策略 | 用户冷启信号加上话题 |

### 顺序背后的逻辑

整组 filter 的排序遵循"便宜的先做、贵的后做"这一经典原则。

第 1 到第 5 这一组 filter 全部依赖于候选自身已经携带的字段，几乎没有任何额外开销。把它们放在最前面执行可以用极低的代价先剔除掉一大批显然不合适的候选。

第 6 到第 9 这一组 filter 需要查询 Redis 或 Bloom Filter，单条候选的开销虽然不大，但是涉及网络往返。把它们放在中间执行，前面已经被剔除掉的候选自然就不会再被无谓地查询。

第 10 到第 14 这一组 filter 需要查询更多的边表（屏蔽词列表、SocialGraph 数据、话题边表等），代价相对最高，因此放在最后。每一步执行完之后，剩下需要被下一个 filter 处理的候选数量都会变小，整体计算总量也随之降低。

### Bloom Filter 的双重保险

第 7 与第 8 两个 filter 都是"已展示去重"，主 Bloom Filter 失效时由备份 Bloom Filter 顶上。Bloom Filter 是一种概率数据结构，存在小概率的假阳性，会让某些"用户其实没有看过"的帖子被错误地过滤掉。双源做"或"运算可以缓解这一问题，让单一数据源故障时整体系统仍然能够给出合理的结果。这一份冗余设计反映的是产品层面"宁可少给几条，也不要把同一条重复展示给同一用户"的取向。

## 选择后过滤（共 3 个，串行执行）

选择后过滤的实例化代码如下。

```rust
let post_selection_filters = vec![
    Box::new(VFFilter),
    Box::new(AncillaryVFFilter),
    Box::new(DedupConversationFilter),
];
```

每一个过滤器的作用列在下表中。

| 过滤器 | 文件 | 作用 |
|---|---|---|
| `VFFilter` | `vf_filter.rs` | 可见性过滤：剔除被删除、垃圾、暴力、血腥、仇恨等不合规候选 |
| `AncillaryVFFilter` | `ancillary_vf_filter.rs` | 辅助可见性，依赖次要安全标签做判定 |
| `DedupConversationFilter` | `dedup_conversation_filter.rs` | 同一对话树的多条分支只保留一支 |

### 为什么这一组过滤器留到最后才做

如果把 VF（可见性过滤）放到打分之前来做，会带来一个严重的延迟问题。VF 服务的延迟通常在 100 到 300 毫秒区间，如果对所有几千条候选都拉一次，整个请求的首字节时间会被 VF 直接卡住。

放到 Top-K 选择之后再做就解决了这一问题。这一阶段只需要对入选的几十条候选做 VF 检查，延迟可控；即便 VF 把其中的 5 条剔除掉，剩下的 25 条仍然足以填满用户屏幕。

代价是显而易见的：那些最终被 VF 剔除掉的候选，实际上已经白白消耗了一次 Phoenix 打分算力。这是一种延迟与算力之间的工程取舍。Phoenix 的算力相对充裕，而用户感知到的延迟则更稀缺，因此整体取向倾向于"宁愿浪费一点算力，也要把延迟压住"。

### DedupConversationFilter 的意义

X 平台上一条原帖往往会衍生出大量的回复链。如果某一次返回的 Top-K 同时包含了同一对话根的 5 条回复，用户看到的体验会像是看了五份克隆。`DedupConversationFilter` 通过对话的 root_id 做归类，每一根对话只保留一到两条，从而避免这一种局部信息高度重复的局面。

## 哪些地方做了"软"抑制（而非硬过滤）

除了硬性的 filter 之外，整套系统中还有几处属于"广义过滤"的逻辑。它们的工作方式不是把候选完全剔除掉，而是通过降低分数让它们退后。下面这张表把这一类机制汇总了起来。

| 机制 | 在哪里 | 效果 |
|---|---|---|
| `AuthorDiversityScorer` | scorer 阶段 | 同作者的第 N 条乘以一个衰减系数 |
| `OONScorer` | scorer 阶段 | 关注网外的候选乘以 `OON_WEIGHT_FACTOR` |
| 负向动作权重 | `WeightedScorer` | block、mute、report 等动作的概率高时，组合分变为负数，经 offset 后排名靠后 |
| `MIN_VIDEO_DURATION_MS` 门槛 | `WeightedScorer::vqv_weight_eligibility` | 过短的视频不给 VQV 权重 |

软硬结合是整套过滤体系的核心思想。硬过滤负责把"绝对不应当展示给用户"的内容拒之门外，软抑制负责让"可以展示但不应当优先"的内容自然退后。两层配合起来，既保证了合规性，又不会因为过于严格而让 Feed 失去多样性。

## Filter 框架自带的指标

每一个 filter 都基于 `xai_candidate_pipeline::filter::Filter` 这一 trait 实现，框架在执行时会自动记录下面这些指标。

```
kept_count
removed_count
filter_rate = removed / (kept + removed)
removed_per_filter = "DropDuplicatesFilter=12, AgeFilter=3, ..."
```

这一组指标对运维诊断有着直接的价值。线上同学只需要观察 `removed_per_filter` 这一项，就能立刻判断哪一个 filter 出现了异常活跃的状况。举一个具体的例子：某一次发版之后，`MutedKeywordFilter` 的过滤率从平时的 1% 一下子跳到了 30%，这就意味着关键词词典本身可能出了问题，需要立刻回滚或者排查。
