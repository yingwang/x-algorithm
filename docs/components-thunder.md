# 3.2 Thunder · 实时帖子库（Rust）

**目录：** [`thunder/`](https://github.com/yingwang/x-algorithm/tree/main/thunder)

## 角色

Thunder 承担的是 In-Network 那条候选源。所谓 In-Network，指的就是用户所关注的那些作者发出的帖子，也就是社交关系图里距离当前用户最近的那一圈人。这一类候选在整个 For You 推荐体系里的位置非常特别：它不需要任何机器学习模型来挑选，只需要把"用户所关注的作者最近发了什么"这件事情查清楚就够了，但同时它对查询延迟的要求又极为苛刻。Thunder 的设计目标因此被定在了亚毫秒级查询这一档上，整个服务完全跑在内存里，不去触碰任何数据库。

为什么这一类候选不需要机器学习模型来挑选？原因在于它已经具备了非常强的先验信号：用户主动选择关注的作者所发的内容，本身就有比较高的相关性下限。剩下的工作只是把这些内容按时间顺序检索出来，交给后续的精排阶段进一步排序即可。换句话说，Thunder 是"基于关系的召回"，而 Phoenix Retrieval 是"基于内容相似度的召回"，两者覆盖不同维度的候选。

## 数据流

Thunder 内部的数据流可以用下面这张图概括。从 Kafka 那一端流进来的写事件，与从 Home Mixer 那一端发起的读请求，在中间通过几张内存索引交汇。

```
Kafka (tweet_events) ──▶ TweetEventsListener (v1 + v2)
                                │
                                ▼
                          反序列化为 LightPost
                                │
                                ▼
                          PostStore（内存中）
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
   original_posts_by_user  secondary_posts  video_posts_by_user
   （原帖）               （回复与转推）   （视频帖）
              ▲                 ▲                 ▲
              └─────────────────┴─────────────────┘
                                │
                  ThunderService::get_posts_by_users
                                │
                  按"被请求用户的关注列表"返回结果
```

简单解释一下图中的几个名词。Kafka 是分布式消息队列，前面已经介绍过。"序列化"指的是把内存中的对象编码成字节流，方便在网络上传输或者存储；"反序列化"是相反的过程，把字节流解码回内存对象。`LightPost` 是 Thunder 内部使用的一种"瘦身版"帖子结构，只包含 Thunder 需要的字段，省去那些与排序无关的部分。

## 核心数据结构

Thunder 最关键的一份数据结构定义在 `thunder/posts/post_store.rs` 文件中。

```rust
pub struct PostStore {
    posts: Arc<DashMap<i64, LightPost>>,                     // 全部帖子，键为 post_id
    original_posts_by_user: Arc<DashMap<i64, VecDeque<TinyPost>>>,
    secondary_posts_by_user: Arc<DashMap<i64, VecDeque<TinyPost>>>,
    video_posts_by_user: Arc<DashMap<i64, VecDeque<TinyPost>>>,
    deleted_posts: Arc<DashMap<i64, bool>>,
    retention_seconds: u64,
    request_timeout: Duration,
}
```

这一段定义里有几个工程选择值得逐个解释。

第一是 `DashMap`，它是一种支持并发读写的哈希表，多线程同时读写时无需获取全局锁。"全局锁"是指多个线程在访问共享数据时通过同一把锁互斥，会导致线程之间相互等待，吞吐量受到严重限制。`DashMap` 内部把整张表分成多个段，每一段有独立的锁，让不同段上的操作可以并行进行；对于一个需要承载高 QPS 写入与查询的内存索引来说，避免全局锁几乎是必须的。`Arc` 是 Rust 中的"原子引用计数智能指针"（Atomic Reference Counted），允许多个线程共享同一份数据的所有权，引用计数变为零时数据自动释放。

第二是 `TinyPost` 与 `LightPost` 的两级存储。每一个用户对应的索引里只放一份很小的结构体 `TinyPost`，里面只包含 `post_id` 与 `created_at` 两个字段；真正的帖子正文则集中存放在主表 `posts` 中。这种分层的好处在于：哪怕一篇帖子同时出现在多个分类索引里（既是原帖、又是视频帖），它的正文也只会被存储一份，不会重复占用内存。这种思路在数据库与缓存设计里有一个对应的概念叫做"规范化"，即把可能重复的数据提取出来单独存放，通过引用来共享。

第三是 `VecDeque`，每个用户的帖子按时间顺序排好后用一个双端队列保存。双端队列（double-ended queue）是一种数据结构，支持在队首与队尾都做常数时间的插入与删除。按时间老化时只需要从队头 `pop_front` 即可，操作复杂度是常数级。如果使用普通数组（Vec），从头部删除元素需要把后续所有元素向前挪一格，复杂度是线性，对高吞吐场景不友好。

每一个作者对应三类索引，分别对应不同类型的内容。

| 索引 | 上限 | 用途 |
|---|---|---|
| `original_posts_by_user` | `MAX_ORIGINAL_POSTS_PER_AUTHOR` | 原帖 |
| `secondary_posts_by_user` | `MAX_REPLY_POSTS_PER_AUTHOR` | 回复与转推 |
| `video_posts_by_user` | `MAX_VIDEO_POSTS_PER_AUTHOR` | 视频帖 |

把同一作者的内容拆到三张索引里，并不是为了存储上的好处，而是为了让"只要视频"或者"不要回复"这一类过滤变成 O(1) 的索引选择操作。这里出现的"O(1)"是计算机科学中描述时间复杂度的一种记号，表示"无论输入规模多大，操作所需时间是一个常数"。如果只有一张大索引，那么按内容类型过滤就要把作者的全部内容取出来再逐条判断类型，多做的工作纯属浪费。

## 写入路径

写入路径的入口在 Kafka，事件流由 `thunder/kafka/tweet_events_listener.rs` 以及 `..._v2.rs` 这两个监听器负责消费。同时维护 v1 与 v2 两个监听器，目的是为了在协议升级期间能够平滑切换，新旧两份事件流可以共存一段时间，避免在切换瞬间出现数据丢失。这种"新旧并行"的发布方式在工程上是处理协议演进的标准做法，比"硬切换"安全得多。

每一批新到的帖子在写入主表之前会经过两步预处理：先剔除掉那些时间戳明显异常或者已经超出保留期的帖子，再按时间排序后批量插入。下面这段代码体现了这一逻辑。

```rust
pub fn insert_posts(&self, mut posts: Vec<LightPost>) {
    let current_time = ...;
    // 第一步：把"未来时间戳"以及"超过 retention 的旧帖"剔除掉
    posts.retain(|p| {
        p.created_at < current_time
            && current_time - p.created_at <= retention_seconds as i64
    });
    // 第二步：按时间排序后批量插入
    posts.sort_unstable_by_key(|p| p.created_at);
    Self::insert_posts_internal(self, posts);
}
```

为什么需要剔除"未来时间戳"？这是一种防御性编程做法：在分布式系统中，时钟可能不完全同步，或者上游处理逻辑可能出现 bug 让事件的时间戳出现明显不合理的值。把这些异常事件直接丢弃，可以避免它们污染整个索引。"超出保留期"的剔除则是为了控制内存占用：Thunder 只关心"最近一段时间"内的帖子，远古的内容对推荐已经没有价值。

删除事件走的是另一条独立路径 `mark_as_deleted`。它会把对应的帖子从主表里移除，同时在 `deleted_posts` 里标记一下，使得下一次读取时即便残留索引指向这条已经被删除的帖子，也会在装配阶段被识别出来并跳过。这种"删除墓碑"的设计是分布式数据系统里非常常见的做法，因为索引的更新有可能存在延迟，墓碑可以在过渡期保证读取的正确性。

## 读取路径

读取路径的入口是 `ThunderService::get_posts_by_users(user_ids)` 这一方法。它接收的参数是当前用户的关注列表，遍历这些作者，从相应的索引里取出 `TinyPost` 列表，再回查主表 `posts` 把对应的正文拼装起来，最终返回给调用方。

整个查询过程被一个 `request_timeout` 保护起来。当扫描进行到一半发生超时时，服务会主动提前返回当时已经攒到的部分结果，绝不去阻塞 Home Mixer 的主调用路径。框架同时埋好了相应的指标。

```rust
// 指标埋点（节选）
POST_STORE_REQUESTS, POST_STORE_REQUEST_TIMEOUTS,
POST_STORE_POSTS_RETURNED, POST_STORE_POSTS_RETURNED_RATIO,
POST_STORE_DELETED_POSTS_FILTERED,
```

这一种"按超时退化为不完整结果"的策略背后有一个非常重要的考量：在大规模在线服务里，P99 延迟的稳定性往往比"返回结果的完整性"更加重要。"P99 延迟"指的是 99% 的请求所能完成的时间上限，是衡量在线服务延迟分布的关键指标，远比平均延迟更能反映用户的实际体验。少返回几条候选不会让推荐质量明显下降，但是单一作者索引膨胀拖慢主路径会直接影响平台上所有用户的体验。

## 为什么这么设计

把上面几处取舍并列对比，可以看出它们背后的一种统一的设计哲学。

| 选择 | 原因 |
|---|---|
| 全内存加 DashMap 的方案 | 关注流是高 QPS 路径，数据库会成为瓶颈；几十 GB 量级的内存远比每一次请求都做几百次远程读划算 |
| 按内容类型拆分三类索引 | 让按内容类型的过滤动作在索引选择层就完成，不必拉取出来再筛 |
| TinyPost 加主表的两级存储 | 多张索引共享同一份正文，节省内存 |
| 超时优先返回部分结果 | 在 P99 与完整性之间倾向 P99；少几条候选不致命，主路径被拖慢则会让全平台体验劣化 |
| 自动按 retention 老化 | 否则单机内存会随时间不断增长，最终引发 OOM |

"OOM"指的是 Out-Of-Memory，即进程因为内存耗尽被操作系统强制终止。对于一个长期运行的内存服务来说，OOM 是一种灾难性故障，必须从设计层面就严格避免。

## Thunder 不做的事

Thunder 这一服务有一个非常清晰的边界，理解这条边界对于读懂整个推荐链路同样重要。

第一，Thunder 不做排序。它的输出是按时间倒序的原始候选列表，相关性方面的判定全部交给 Phoenix 完成。这种"召回只管粗筛、排序负责细判"的分工是大规模推荐系统的标准做法。

第二，Thunder 不做可见性过滤。被删除的帖子会被它跳过，但是诸如屏蔽、Block、关键词屏蔽这一类涉及用户偏好的过滤动作，都被放在 Home Mixer 的 filters 阶段统一处理。这种"集中过滤"的设计让规则可以统一维护，避免在多个不同位置重复实现同样的逻辑。

第三，Thunder 不做跨用户的去重。去重相关的工作交给 Home Mixer 中的 `DropDuplicatesFilter` 以及 `RetweetDeduplicationFilter`。

整套设计的核心在于：让 Thunder 只承担一件事，把它做到又快又准；其它一切都交给上层 Home Mixer 去统一处理。这种职责清晰的拆分是整个系统在工程层面能够长期演进的关键。当某一类规则需要变化时，只需要去对应位置改一处即可，不会出现"一个改动需要改好几个地方"的难维护局面。
