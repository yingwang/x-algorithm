# 3.2 Thunder · 实时帖子库（Rust）

**目录：** [`thunder/`](https://github.com/yingwang/x-algorithm/tree/main/thunder)

## 角色

承担 **In-Network**（你关注的人发的帖子）那条候选源。设计目标是**亚毫秒级查询**，因此完全跑在内存里、不打数据库。

## 数据流

```
Kafka (tweet_events) ──▶ TweetEventsListener (v1 + v2)
                                │
                                ▼
                          反序列化 → LightPost
                                │
                                ▼
                          PostStore（in-memory）
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
   original_posts_by_user  secondary_posts  video_posts_by_user
   (原帖)                 (回复 / 转推)     (视频)
              ▲                 ▲                 ▲
              └─────────────────┴─────────────────┘
                                │
                  ThunderService::get_posts_by_users
                                │
                          按"被请求用户的关注列表"返回
```

## 核心数据结构

`thunder/posts/post_store.rs`:

```rust
pub struct PostStore {
    posts: Arc<DashMap<i64, LightPost>>,                     // 全部帖子（按 post_id）
    original_posts_by_user: Arc<DashMap<i64, VecDeque<TinyPost>>>,
    secondary_posts_by_user: Arc<DashMap<i64, VecDeque<TinyPost>>>,
    video_posts_by_user: Arc<DashMap<i64, VecDeque<TinyPost>>>,
    deleted_posts: Arc<DashMap<i64, bool>>,
    retention_seconds: u64,
    request_timeout: Duration,
}
```

- **`DashMap`**：并发哈希表，多线程读写无锁。
- **`TinyPost`**：用户索引里只放 `{post_id, created_at}`，正文存在 `posts` 主表里。这样维护 N 个分类索引也不会重复存储 LightPost。
- **`VecDeque`**：按时间老化时只需 `pop_front`，O(1)。

每个用户分三类索引：

| 索引 | 上限 | 用途 |
|---|---|---|
| `original_posts_by_user` | `MAX_ORIGINAL_POSTS_PER_AUTHOR` | 原帖 |
| `secondary_posts_by_user` | `MAX_REPLY_POSTS_PER_AUTHOR` | 回复 + 转推 |
| `video_posts_by_user` | `MAX_VIDEO_POSTS_PER_AUTHOR` | 视频帖 |

> 拆三类索引是为了让"只要视频"、"不要回复"这种过滤变成 O(1) 的索引选择，而不是扫完再过滤。

## 写入路径

```rust
pub fn insert_posts(&self, mut posts: Vec<LightPost>) {
    let current_time = ...;
    // 1) 把"未来时间戳"和"超出 retention 的旧帖"先扔掉
    posts.retain(|p| {
        p.created_at < current_time
            && current_time - p.created_at <= retention_seconds as i64
    });
    // 2) 按时间排序后 batch 插入
    posts.sort_unstable_by_key(|p| p.created_at);
    Self::insert_posts_internal(self, posts);
}
```

- 来源是 Kafka 的 `tweet_events`（`thunder/kafka/tweet_events_listener.rs` / `..._v2.rs`）。同时维护两个 listener 是为了平滑切换 v1 → v2 协议。
- 删除事件单独走 `mark_as_deleted`：从 `posts` 主表移除，写 `deleted_posts`，下次读取时跳过。

## 读取路径

`ThunderService::get_posts_by_users(user_ids)` 接收"当前用户的关注列表"，遍历这些作者拿到 `TinyPost` 列表，再回查 `posts` 主表组装。中间有一个 **`request_timeout`** 保护：扫到一半超时就提前返回，绝不阻塞 home-mixer 主路径。

```rust
// 指标埋点（节选）
POST_STORE_REQUESTS, POST_STORE_REQUEST_TIMEOUTS,
POST_STORE_POSTS_RETURNED, POST_STORE_POSTS_RETURNED_RATIO,
POST_STORE_DELETED_POSTS_FILTERED,
```

也就是说，框架已经做好了**"按超时退化为不完整结果"**的准备 —— 这是为了让 P99 延迟可控（而不是因为单一作者的索引大就拖慢整体）。

## 为什么这么设计

| 选择 | 原因 |
|---|---|
| 全内存 + DashMap | 关注流是高 QPS 路径，数据库会成瓶颈；内存大几十 GB 远比每请求几百次远程读划算 |
| 三类索引分桶 | 让"按内容类型过滤"在索引层就完成 |
| TinyPost + 主表 | 多索引共享同一份正文，节省内存 |
| 超时优先返回 | P99 重要性 > 完整性；少几条候选不致命，但拖慢主路径会全平台体验差 |
| 自动按 retention 老化 | 否则单机内存会随时间爬升，最终 OOM |

## Thunder 不做的事

- **不做排序**：Thunder 只返回原始候选，相关性留给 Phoenix。
- **不做可见性过滤**：被删的帖会跳过，但屏蔽 / Block / 关键词屏蔽都在 home-mixer 的 filters 阶段做。
- **不做跨用户去重**：去重在 home-mixer 的 `DropDuplicatesFilter` / `RetweetDeduplicationFilter`。

Thunder 就一个职责：**"给我用户 X 关注的最近发文"** —— 又快又准就够了。
