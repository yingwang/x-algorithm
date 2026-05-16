# 3.5 Grox · 内容理解服务（Python）

**目录：** [`grox/`](https://github.com/yingwang/x-algorithm/tree/main/grox) — 2026-05-15 新增。

为推荐与安全策略提供统一的**内容理解**能力，避免在多个下游各自重复造轮子。

## 内部结构

```
grox/
  engine.py             调度引擎入口
  dispatcher.py         按 plan/schedule 把任务发到对应 task
  main.py               进程入口
  classifiers/content/  各类内容分类器（spam / 品类 / PTOS 策略）
  embedder/             把内容向量化
  summarizer/
    summarizer.py
    post_embedding_summarizer.py
    eapi_summarizer.py
  generators/           生成类（标题、推荐理由等）
  tasks/                所有"原子任务"的实现
  plans/                把任务组合成"业务计划"
  schedules/            什么时候触发哪个 plan
  data_loaders/         训练 / 推理数据装载
```

## tasks/ —— 干活的最小单元

`grox/tasks/` 下每个 `task_*.py` 是一种**原子能力**：

| 文件 | 做的事 |
|---|---|
| `task_spam_detection.py` | 垃圾信息识别 |
| `task_safety_ptos_policy.py` · `task_safety_ptos_category.py` | PTOS（Platform Terms of Service）策略检查、品类判定 |
| `task_post_safety_screen_deluxe.py` | 综合安全筛查 |
| `task_write_safety_post_annotations_result_sink.py` | 安全标签写回 sink |
| `task_grok_upa_action_with_labels.py` | 调 Grok 模型给帖子打 UPA 动作标签 |
| `task_multimodal_post_embedding.py` · `task_write_mm_embedding_sink.py` | 多模态 embedding + 写入 |
| `task_summarizer_for_post_embedding.py` · `task_load_post_with_summary.py` | 让帖子先过摘要再 embedding，省 token / 抗噪声 |
| `task_post_embedding_pub.py` 等 `task_*_pub.py` | 计算完写 Pub / 下游 sink |
| `task_load_post_with_not_found_retry.py` | 帖子加载 + not-found 重试 |
| `task_load_user_recent_posts.py` | 加载用户近期帖（如给"banger 屏"用） |
| `task_banger_screen.py` · `task_initial_banger`（plan 侧） | 高互动帖筛选 |
| `task_rank_replies.py` | 回复排序 |
| `task_rate_limit.py` | 速率限制 |
| `task_asr.py` | 自动语音识别（视频转文字） |
| `task_media.py` | 媒体相关 |
| `task_filters.py` · `disable_rules.py` | 任务级过滤、disable 规则 |
| `task_pub.py` | 通用 publish 工具 |
| `task_embedding_pub.py` | embedding publish 工具 |

`task.py` 是 base class；`task_filters.py` 提供组合 / 短路的工具方法。

## plans/ —— 把 tasks 串成业务流

| Plan | 用途 |
|---|---|
| `plan_master.py` | 总入口 / 顶层 plan 路由 |
| `plan_post_safety.py` · `plan_safety_ptos.py` · `plan_spam_comment.py` | 安全 / 合规相关 |
| `plan_initial_banger.py` | 选"高互动帖" |
| `plan_reply_ranking.py` | 回复排序 |
| `plan_post_embedding_v5.py` · `plan_post_embedding_v5_for_reply.py` | v5 帖子 embedding |
| `plan_post_embedding_with_summary.py` · `..._for_reply.py` | 摘要后再 embedding 的变体 |

> 把"内容理解"切成 task + plan 这种粒度，相比"一个 monolith 服务"的好处：每个 task 单独可测、可重启、可单独 A/B；plan 改动不必动 task 实现。

## engine.py + dispatcher.py

`engine.py` 注册 task / plan / schedule，`dispatcher.py` 接到事件后查询配置把请求路由到对应 task。这一对是"插件框架"，约束所有 task 走同一个生命周期（init → run → emit → ack）。

## 在 For You 链路里的角色

Grox 是**独立服务**，不被 home-mixer 同步调用。它做的事更像离线 / 准实时 enrichment：

- 帖子被发布 → Grox 异步给它打安全标签、品类、embedding。
- home-mixer 读这些"已经被打过标签"的字段（通过 `VFCandidateHydrator`、`FilteredTopicsHydrator`、`AdsBrandSafetyHydrator` 等 hydrator 从下游服务拉）。

也就是说，Grox 与 home-mixer 是**生产者—消费者**关系，靠 Strato / VF / Safety Label Store 等数据层解耦。

## 与 Grok 模型的关系

`task_grok_upa_action_with_labels.py` 直接调 Grok 模型给帖子打标签（"User Performed Action" 类标签）。这是 xAI 的多用：Grok-1 风格的 Transformer 既被 Phoenix 用来做相关性排序，也被 Grox 用来做内容理解。两个用途都受益于同一份基础架构（注意力、RoPE、词表）。
