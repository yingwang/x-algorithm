# 6. 过滤规则

整条推荐流水线里的过滤逻辑被分成了两段，分别位于打分前与选择后这两个位置。打分前的过滤主要承担"节省算力"的职责：把那些注定不应当被推给用户的候选尽早剔除，避免它们消耗掉昂贵的 Phoenix 推理算力。选择后的过滤则承担"最后一道把关"的职责：在已经排好序的少量入选候选上做更精细但相对昂贵的合规检查。这种"两段式过滤"的设计是大规模推荐系统中常见的工程取舍。

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

第 1 到第 5 这一组 filter 全部依赖于候选自身已经携带的字段，几乎没有任何额外开销。把它们放在最前面执行可以用极低的代价先剔除掉一大批显然不合适的候选。例如同一篇帖子被两路召回同时召出，去重 filter 一眼就能识别；当前用户自己发的帖子，对比 user_id 与 author_id 即可判断。

第 6 到第 9 这一组 filter 需要查询 Redis 或 Bloom Filter，单条候选的开销虽然不大，但是涉及网络往返。把它们放在中间执行，前面已经被剔除掉的候选自然就不会再被无谓地查询。这里需要解释一下 Bloom Filter 的工作原理：它是一种用很少空间就能回答"某项是否在集合中"的概率数据结构，通过把每一项分别映射到位数组中的若干位来记录其存在；查询时检查这些位是否都为 1，全为 1 则报告"可能存在"，否则报告"一定不存在"。它的特点是不会出现假阴性（说"不在"就一定不在），但是允许小概率的假阳性（说"在"实际可能不在）。在去重曝光的场景下，这种特性是可以接受的。

第 10 到第 14 这一组 filter 需要查询更多的边表（屏蔽词列表、SocialGraph 数据、话题边表等），代价相对最高，因此放在最后。每一步执行完之后，剩下需要被下一个 filter 处理的候选数量都会变小，整体计算总量也随之降低。

这种"按代价排序的串行过滤"在数据库与搜索引擎的查询计划里有一个对应的术语叫做"早终止"（early termination）：让能够被廉价规则排除的对象尽早离开处理流水线，把昂贵的资源留给真正需要细致判断的对象。

### Bloom Filter 的双重保险

第 7 与第 8 两个 filter 都是"已展示去重"，主 Bloom Filter 失效时由备份 Bloom Filter 顶上。Bloom Filter 是一种概率数据结构，存在小概率的假阳性，会让某些"用户其实没有看过"的帖子被错误地过滤掉。双源做"或"运算可以缓解这一问题，让单一数据源故障时整体系统仍然能够给出合理的结果。

这一份冗余设计反映的是产品层面"宁可少给几条，也不要把同一条重复展示给同一用户"的取向。"用户已经看过的内容再次出现在 Feed 中"是 X 这一类社交产品的标志性糟糕体验，比"少出现几条新内容"严重得多。因此即便代价是多出一份 Bloom Filter 服务的维护成本，整套系统仍然愿意承担。

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

如果把 VF（可见性过滤）放到打分之前来做，会带来一个严重的延迟问题。VF 服务的延迟通常在 100 到 300 毫秒区间，如果对所有几千条候选都拉一次，整个请求的首字节时间会被 VF 直接卡住。"首字节时间"（Time To First Byte，简称 TTFB）是衡量请求响应速度的关键指标，它直接影响用户的主观感知。

放到 Top-K 选择之后再做就解决了这一问题。这一阶段只需要对入选的几十条候选做 VF 检查，延迟可控；即便 VF 把其中的 5 条剔除掉，剩下的 25 条仍然足以填满用户屏幕。

代价是显而易见的：那些最终被 VF 剔除掉的候选，实际上已经白白消耗了一次 Phoenix 打分算力。这是一种延迟与算力之间的工程取舍。Phoenix 的算力相对充裕，而用户感知到的延迟则更稀缺，因此整体取向倾向于"宁愿浪费一点算力，也要把延迟压住"。

这种"昂贵但必要的过滤推迟到选择后"的做法在工程上有个对应的概念叫做"延迟绑定"（late binding）：把那些代价昂贵但只在最终决定时才必要的判断推迟到最后一刻，而不是在每一个中间环节都强求一次。这种策略让系统的整体效率得到显著提升。

### DedupConversationFilter 的意义

X 平台上一条原帖往往会衍生出大量的回复链。如果某一次返回的 Top-K 同时包含了同一对话根的 5 条回复，用户看到的体验会像是看了五份克隆。`DedupConversationFilter` 通过对话的 root_id 做归类，每一根对话只保留一到两条，从而避免这一种局部信息高度重复的局面。这种"对话级去重"是 X 类社交产品特有的需求，区别于"帖子级去重"：后者关注的是同一篇帖子不要重复出现，前者关注的是同一话题的多个变体不要全部出现。

## 哪些地方做了"软"抑制（而非硬过滤）

除了硬性的 filter 之外，整套系统中还有几处属于"广义过滤"的逻辑。它们的工作方式不是把候选完全剔除掉，而是通过降低分数让它们退后。下面这张表把这一类机制汇总了起来。

| 机制 | 在哪里 | 效果 |
|---|---|---|
| `AuthorDiversityScorer` | scorer 阶段 | 同作者的第 N 条乘以一个衰减系数 |
| `OONScorer` | scorer 阶段 | 关注网外的候选乘以 `OON_WEIGHT_FACTOR` |
| 负向动作权重 | `WeightedScorer` | block、mute、report 等动作的概率高时，组合分变为负数，经 offset 后排名靠后 |
| `MIN_VIDEO_DURATION_MS` 门槛 | `WeightedScorer::vqv_weight_eligibility` | 过短的视频不给 VQV 权重 |

软硬结合是整套过滤体系的核心思想。硬过滤负责把"绝对不应当展示给用户"的内容拒之门外，软抑制负责让"可以展示但不应当优先"的内容自然退后。两层配合起来，既保证了合规性，又不会因为过于严格而让 Feed 失去多样性。这种"硬规则加软调节"的两层结构在工业级推荐系统中是相当典型的设计。

## Filter 框架自带的指标

每一个 filter 都基于 `xai_candidate_pipeline::filter::Filter` 这一 trait 实现，框架在执行时会自动记录下面这些指标。

```
kept_count
removed_count
filter_rate = removed / (kept + removed)
removed_per_filter = "DropDuplicatesFilter=12, AgeFilter=3, ..."
```

这一组指标对运维诊断有着直接的价值。线上同学只需要观察 `removed_per_filter` 这一项，就能立刻判断哪一个 filter 出现了异常活跃的状况。举一个具体的例子：某一次发版之后，`MutedKeywordFilter` 的过滤率从平时的 1% 一下子跳到了 30%，这就意味着关键词词典本身可能出了问题，需要立刻回滚或者排查。

这种"按组件粒度记录指标"的可观测性设计在大型系统里非常重要。如果只记录一个总的过滤率指标，那么任何一个 filter 出问题都只能看到"总数变高了"这种模糊信号，无法快速定位到具体的故障点。把指标按组件粒度切开之后，每一次异常都能精准对应到某一个具体的 filter 上，排查效率因此大大提升。
