<a id="top"></a>

**语言 / Language:** [中文](#x-for-you-推荐算法中文详解) · [English](#x-for-you-feed-algorithm)

---

# X For You 推荐算法（中文详解）

本仓库公开了 X（前 Twitter）"For You" 信息流的核心推荐系统：把**关注内容（in-network）**与**通过机器学习检索到的非关注内容（out-of-network）**统一打分排序，最终输出给用户的个性化时间线。整套排序模型基于 **Grok 风格的 Transformer**，从 [xAI Grok-1 开源版本](https://github.com/xai-org/grok-1)移植后改造为推荐场景。

> 这一节是对仓库的详细中文解读，覆盖整体架构、各组件职责、流水线阶段、Phoenix 模型内部结构、注意力掩码（Candidate Isolation）原理、运行方式与设计权衡。如果你只想看官方英文说明，请直接跳到 [English Version](#x-for-you-feed-algorithm)。

## 中文目录

- [1. 项目背景与整体定位](#1-项目背景与整体定位)
- [2. 顶层架构总览](#2-顶层架构总览)
- [3. 五大组件详解](#3-五大组件详解)
  - [3.1 Home Mixer（编排层 · Rust）](#31-home-mixer编排层--rust)
  - [3.2 Thunder（实时帖子库 · Rust）](#32-thunder实时帖子库--rust)
  - [3.3 Phoenix（检索 + 排序模型 · Python/JAX）](#33-phoenix检索--排序模型--pythonjax)
  - [3.4 Candidate Pipeline（流水线框架 · Rust）](#34-candidate-pipeline流水线框架--rust)
  - [3.5 Grox（内容理解服务 · Python）](#35-grox内容理解服务--python)
- [4. 端到端流水线阶段](#4-端到端流水线阶段)
- [5. Phoenix 模型深入](#5-phoenix-模型深入)
  - [5.1 双塔检索模型](#51-双塔检索模型)
  - [5.2 排序 Transformer 与候选隔离](#52-排序-transformer-与候选隔离)
  - [5.3 多动作预测与加权打分](#53-多动作预测与加权打分)
  - [5.4 Mini Phoenix 模型规格](#54-mini-phoenix-模型规格)
- [6. 过滤规则](#6-过滤规则)
- [7. 关键设计决策（含原因）](#7-关键设计决策含原因)
- [8. 快速上手：本地运行 Phoenix 推理](#8-快速上手本地运行-phoenix-推理)
- [9. 与 X 官方仓库 the-algorithm 的关系](#9-与-x-官方仓库-the-algorithm-的关系)
- [10. 术语对照表](#10-术语对照表)
- [11. 2026-05-15 更新明细](#11-2026-05-15-更新明细)

---

## 1. 项目背景与整体定位

For You 信息流要在用户每次刷新时，从**全平台数亿条帖子**中挑出几十条最可能让该用户产生有意义互动的内容。这个问题被拆成两个相对独立的子问题：

1. **召回（Retrieval）**：从全量语料里按相关性快速截取几百到几千条候选。代价必须低，因此采用近似最近邻（ANN）的双塔架构。
2. **精排（Ranking）**：对召回的候选用更重的 Transformer 打分，预测多种用户动作的概率，加权后排序输出。

本仓库的特点：

- 已经**几乎完全去掉了人工特征工程**：Phoenix Transformer 直接从用户**动作序列**学习相关性。
- 排序阶段使用一种特殊的注意力掩码——**候选隔离（Candidate Isolation）**：候选之间互相看不见，从而让单条候选的得分只与"用户上下文"有关，可以被独立缓存、跨批稳定。
- 服务层用 **Rust** 重写，模型层用 **Python + JAX/Haiku**；候选 Pipeline 抽象成可复用的 trait 框架。
- 内置了一份**预训练 Mini Phoenix 模型 + 一个体育语料库**，可以本地一键跑通"召回 → 排序"的端到端推理。

---

## 2. 顶层架构总览

```
请求 ──▶ Home Mixer ──┬─▶ Query Hydrators（拉取用户上下文）
                      │
                      ├─▶ Sources（Thunder + Phoenix 检索 + 广告 + WTF + Phoenix MoE/Topics …）
                      │
                      ├─▶ Candidate Hydrators（核心元数据 / 作者 / 媒体 / 互动数 / 品牌安全 …）
                      │
                      ├─▶ Pre-Scoring Filters（去重 / 太旧 / 自帖 / 黑名单 / 关键词屏蔽 …）
                      │
                      ├─▶ Scorers（Phoenix 打分 → 加权 → 作者多样性 → OON 修正）
                      │
                      ├─▶ Selector（Top-K 排序选择）
                      │
                      ├─▶ Post-Selection Filters（可见性 / 会话去重）
                      │
                      └─▶ Side Effects（缓存 / 监控）   ──▶ 返回排序后的 Feed
```

> 完整 ASCII 架构图见下方英文章节，结构与上图一致。

---

## 3. 五大组件详解

### 3.1 Home Mixer（编排层 · Rust）

**目录：** [`home-mixer/`](home-mixer/)

For You 的入口与编排服务。它本身不做模型预测，而是把"取候选 → 注水 → 过滤 → 打分 → 选 Top-K → 复滤 → 副作用"串起来。

- 通过 **gRPC** 暴露 `ScoredPostsService`，给上游接口返回打分后的帖子列表。
- 利用 [`candidate-pipeline`](#34-candidate-pipeline流水线框架--rust) 框架声明式地组合各阶段。
- 子目录：
  - `query_hydrators/`：拉取用户级上下文（关注列表、关注的话题、Starter Pack、曝光 Bloom Filter、IP、互关图、已展示历史等）。
  - `sources/`：候选源，包含 Thunder、Phoenix 检索、Phoenix MoE、Phoenix Topics、Who-To-Follow、广告、Prompts 等。
  - `candidate_hydrators/`：候选级注水（互动计数、品牌安全信号、语言、媒体类型、引用帖展开、互关分等）。
  - `filters/`：见 [§6 过滤规则](#6-过滤规则)。
  - `scorers/`：Phoenix 打分、加权聚合、作者多样性衰减、OON 调整。
  - `selectors/`：按最终分排序、截断到 Top-K。
  - `ads/`：本次新增的**广告插入与品牌安全**模块，负责广告插入位置控制并尊重敏感内容边界。
  - `side_effects/`：缓存请求信息以备后续使用、记录指标等。

### 3.2 Thunder（实时帖子库 · Rust）

**目录：** [`thunder/`](thunder/)

承担 **In-Network**（你关注的人发的帖子）这条候选源。设计目标是**亚毫秒级查询**，因此完全跑在内存里。

- 从 **Kafka** 实时消费帖子的 create / delete 事件（`thunder/kafka/`、`kafka_utils.rs`）。
- 为每个用户维护多个分仓的索引（`thunder/posts/`）：
  - 原始帖
  - 回复 / 转推
  - 视频帖
- 收到请求时按"被请求用户的关注关系"返回候选。
- 自动按保留时长滚动淘汰旧帖，避免内存膨胀。

这种"按用户分桶 + 内存常驻"的做法，避免了高 QPS 下打到外部数据库的瓶颈。

### 3.3 Phoenix（检索 + 排序模型 · Python/JAX）

**目录：** [`phoenix/`](phoenix/) — 详细子文档见 [`phoenix/README.md`](phoenix/README.md)。

Phoenix 是整套系统的"大脑"，由两个 JAX/Haiku 模型组成：

| 模型 | 职责 | 输出 |
|---|---|---|
| **Retrieval（双塔）** | 从全量语料里按相似度召回 Top-K | 候选 ID 列表 |
| **Ranking（带掩码的 Transformer）** | 对每条候选预测多种动作的发生概率 | `[B, num_candidates, num_actions]` 的 logits |

实现细节：

- 用户塔（user tower）和 item 塔结构对称，但共享底层 Transformer 设计。
- ID 类特征采用**多哈希查表（Hash-Based Embedding）**：每个用户/作者/帖子 ID 用多个 hash 函数索引到 100 万规模的 embedding 表，再求和；这样不需要为每个 ID 维护单独条目，能容纳几乎无限词表。
- 训练数据为线上**实时互动流**（喜欢、回复、转推、停留时长、视频质量观看等）。
- 关键文件：
  - `grok.py`：Grok-1 移植过来的 Transformer 主干。
  - `recsys_model.py` / `recsys_retrieval_model.py`：把 Grok 改造成召回/排序两个模型。
  - `runners.py`：统一加载 checkpoint、embedding 表与配置的工具。
  - `run_pipeline.py`：本次新增的"召回 → 排序"端到端入口（替换了原 `run_ranker.py` + `run_retrieval.py`）。
  - `artifacts/`：通过 **Git LFS** 分发的 ~3 GB 预训练 Mini 模型 + 体育语料压缩包。

### 3.4 Candidate Pipeline（流水线框架 · Rust）

**目录：** [`candidate-pipeline/`](candidate-pipeline/)

把"组装一个推荐流"抽象成 6 类 trait，业务方只需实现 trait，框架负责并发、错误隔离、监控：

| Trait | 职责 |
|---|---|
| `Source` | 拉取一类候选 |
| `Hydrator` | 给候选补充字段 |
| `Filter` | 把不合规的候选剔掉 |
| `Scorer` | 对候选打分 |
| `Selector` | 排序 + 截断 |
| `SideEffect` | 异步副作用（缓存、日志、埋点） |

特点：

- 独立 Source / Hydrator **并行执行**，整体延迟取决于最慢一个，而不是串行总和。
- 单个阶段失败可降级，不会拖垮整条流水线。
- 业务逻辑与"如何执行 + 如何监控"完全解耦——加一种新候选源或新打分器，几乎不用碰核心调度代码。

### 3.5 Grox（内容理解服务 · Python）

**目录：** [`grox/`](grox/) — 本次新增。

为推荐与安全策略提供统一的**内容理解**能力，避免在多个下游各自重复造轮子。

- `classifiers/content/`：内容分类器，例如垃圾信息识别、帖子品类、PTOS（Platform Terms of Service）策略检查。
- `embedder/`：把内容向量化，供检索 / 相似度判定 / 下游模型使用。
- `generators/`、`summarizer/`：生成与摘要类任务。
- `engine.py` + `dispatcher.py` + `tasks/` + `plans/` + `schedules/`：任务调度引擎，按 plan/schedule 派发到具体的 task 执行器。
- `data_loaders/`：训练 / 推理用的数据装载器。

---

## 4. 端到端流水线阶段

一次 For You 请求会按下面 7 步执行（与英文版顺序一致，下面把每步**为什么这么做**也说清楚）：

1. **Query Hydration（查询注水）** — 拉用户最近的互动序列、关注列表、被屏蔽列表、已曝光 Bloom Filter、IP、互关图等。这些是后续模型与过滤器的"用户上下文"。
2. **Candidate Sourcing（候选召回）** — 并行从 Thunder（关注网内）和 Phoenix Retrieval（关注网外）以及广告、WTF、Phoenix MoE/Topics 等源拉候选。
3. **Candidate Hydration（候选注水）** — 给候选补充正文、媒体、作者认证状态、视频时长、订阅信息、互动数、品牌安全标签、语言等。
4. **Pre-Scoring Filters（打分前过滤）** — 把"不该参与排序"的候选先剔掉（重复、太旧、自帖、黑名单、关键词屏蔽、已看过 / 已发过、付费墙不可见…）。先过滤后打分能显著省算力。
5. **Scoring（打分）** — 顺序执行多个 Scorer：
   - **Phoenix Scorer**：调用 Phoenix Transformer，得到每条候选 × 每个动作的概率。
   - **Weighted Scorer**：用配置好的权重把多动作概率聚合为单一分数。
   - **Author Diversity Scorer**：同一作者的多条候选会被逐次衰减，避免 Feed 被同一人刷屏。
   - **OON Scorer**：对来自关注网外的候选做调整，平衡 in/out 比例。
6. **Selection（选择）** — 按最终分排序，取 Top-K。
7. **Post-Selection Filters（选择后过滤）** — 在用户真正看到之前再做最后一道把关：可见性（被删除、暴力、垃圾等）和会话级去重（避免同一对话的多个分支同时出现）。

---

## 5. Phoenix 模型深入

### 5.1 双塔检索模型

```
 用户特征 + 历史动作序列 ──▶ User Tower  ──▶ user_emb   ┐
                                                       │  dot(·)  ──▶ Top-K
 全量语料（每条帖子+作者特征）──▶ Candidate Tower ──▶ item_emb  ┘
```

- 两塔的 embedding 都做了 L2 归一化，所以"点积相似度"等价于余弦。
- Candidate Tower 在**离线**为整个语料预计算好 `[N, D]` 的索引矩阵；线上只需算一次 user embedding，再做一次稠密点积取 Top-K（语料规模到达 ANN 量级时改为近似搜索）。
- Demo 用的体育语料 `sports_corpus.npz` 来自 6 小时窗口内被打上 "Sports" 标签的 ~537K 条帖子。

### 5.2 排序 Transformer 与候选隔离

排序模型把序列拼成：

```
 [ User token | History tokens (S 个) | Candidate tokens (C 个) ]
```

然后用**特殊的注意力掩码**控制谁能看见谁（详细可视化见 `phoenix/README.md`）：

| Query \ Key | User | History | Candidates |
|---|---|---|---|
| **User**       | ✓ | ✓ | ✗ |
| **History**    | ✓ | ✓ | ✗ |
| **Candidates** | ✓ | ✓ | **仅自身（对角线）** |

也就是说：

- 候选**可以看见**用户和历史，从而获得个性化上下文。
- 候选**看不见彼此**——这就是 Candidate Isolation。

为什么这么做？

1. **得分稳定**：一条候选的分数不依赖同批里有哪些其它候选，便于跨请求缓存与分批打分。
2. **可并行**：不同候选之间无依赖，可以 batch / 流水线并发。
3. **可解释**：得分变化只能归因于"用户上下文"或"候选自身特征"。

### 5.3 多动作预测与加权打分

模型最后一层把每个候选位置投影到 `num_actions` 个 logit，对应动作（节选）：

```
P(favorite)  P(reply)        P(repost)   P(quote)
P(click)     P(profile_click) P(video_view) P(photo_expand)
P(share)     P(dwell)        P(follow_author)
P(not_interested) P(block_author) P(mute_author) P(report)
```

加权聚合公式：

```
Final Score = Σ_i  weight_i × P(action_i)
```

正向动作（喜欢、转推、分享…）权重为正；负向动作（屏蔽、静音、举报…）权重为负，会**主动压低**用户大概率讨厌的内容。这种"多任务 + 显式负权"比"单一相关性分数"更能体现真实偏好。

### 5.4 Mini Phoenix 模型规格

本次随仓库一同发布的预训练模型是缩水版（线上模型层数更深、维度更宽）：

| 参数 | 取值 |
|---|---|
| Embedding 维度 | 128 |
| Transformer 层数 | 4 |
| Attention 头数 | 4 |
| Key size | 32 |
| Widening factor | 2 |
| 历史序列长度 | 127 |
| 候选序列长度 | 64 |
| 用户 / 物品 / 作者词表 | 各 1,000,000 |
| 每个实体的 hash 数 | 2 |
| 动作类型数 | 19 |

> README 顶部声称 "256-dim · 4 head · 2 layer"；`phoenix/README.md` 的 Mini 配置实际为 "128-dim · 4 layer"。两份文档之一可能尚未同步，以代码 `recsys_model.py` 与下载的 `config.json` 为准。

---

## 6. 过滤规则

**打分前（Pre-Scoring）：**

| 过滤器 | 作用 |
|---|---|
| `DropDuplicatesFilter` | 去掉同一 post id 的重复 |
| `CoreDataHydrationFilter` | 去掉核心元数据注水失败的帖 |
| `AgeFilter` | 去掉超过时间阈值的旧帖 |
| `SelfpostFilter` | 不给用户看自己发的帖 |
| `RepostDeduplicationFilter` | 同一原帖的多次转发只留一份 |
| `IneligibleSubscriptionFilter` | 屏蔽用户无订阅权限的付费内容 |
| `PreviouslySeenPostsFilter` | 屏蔽已展示过的帖（基于曝光 Bloom Filter） |
| `PreviouslyServedPostsFilter` | 屏蔽本次会话已经返回过的帖 |
| `MutedKeywordFilter` | 命中用户屏蔽词 |
| `AuthorSocialgraphFilter` | 屏蔽被 block / mute 的作者 |

**选择后（Post-Selection）：**

| 过滤器 | 作用 |
|---|---|
| `VFFilter` | 可见性过滤：删除态、垃圾、暴力、血腥等 |
| `DedupConversationFilter` | 同一对话树的多条分支去重 |

---

## 7. 关键设计决策（含原因）

1. **零手工特征**：让 Grok Transformer 从用户动作序列自学相关性，砍掉了人工特征工程的整条链路（特征仓库、特征同步、训练-服务一致性问题）。
2. **候选隔离的注意力掩码**：保证候选打分**与同批其它候选无关**，从而可缓存、可分批，也方便跨实验对比。
3. **Hash-Based Embedding**：用多哈希查表替代"每个 ID 一行 embedding"。词表"无限大"也只占固定显存，新 ID（新用户/新帖子）天然有 embedding，无冷启动 OOV 问题。
4. **多动作预测**：多任务比单标量更能拟合真实偏好，并允许显式给负向动作上负权。
5. **可组合的 Pipeline 框架**：业务（什么源、什么过滤）与执行（怎么并发、怎么降级、怎么监控）解耦，加新策略基本不动调度器。

---

## 8. 快速上手：本地运行 Phoenix 推理

### 8.1 安装依赖

推荐 [uv](https://docs.astral.sh/uv/getting-started/installation/)：

```bash
cd phoenix
uv sync
```

或者：

```bash
pip install jax jaxlib dm-haiku numpy
```

### 8.2 下载并解压模型 artifacts

`phoenix/artifacts/oss-phoenix-artifacts.zip` 通过 Git LFS 分发，约 3 GB：

```bash
cd phoenix
unzip artifacts/oss-phoenix-artifacts.zip -d artifacts/
```

解压后目录：

```
oss-phoenix-artifacts/
  retrieval/        召回模型权重 + 哈希参数 + 配置
  ranker/           排序模型权重 + 哈希参数 + 配置
  sports_corpus.npz 537K 条体育帖（含已预计算的候选表示）
  example_sequence.json  示例用户互动序列（NFL / NBA / NHL 三条）
```

### 8.3 跑端到端推理

```bash
uv run run_pipeline.py --artifacts_dir artifacts/oss-phoenix-artifacts
```

它会：

1. 加载召回与排序模型
2. 读取示例用户历史
3. 在 537K 体育语料中召回 Top-200
4. 用排序模型对召回结果按 favorite / reply / repost / dwell / video view 等动作的概率打分
5. 打印最终 Top-N 排序及每个动作的预测概率

### 8.4 常用参数

- `--top_k_retrieval 500`：放宽召回数量（默认 200）。
- `--top_k_display 50`：多展示几条最终排序结果。
- 修改 `example_sequence.json` 可以换成你自己的"用户历史"，每条需要 `post_id`、`author_id`、`actions`（动作 index → 数值）；动作 index 与 proto 中的 `ActionName` 一致：`1`=favorite、`4`=reply、`5`=quote、`6`=repost、`11`=dwell、`13`=video quality view。

### 8.5 跑测试

```bash
uv run pytest test_recsys_model.py test_recsys_retrieval_model.py
```

---

## 9. 与 X 官方仓库 the-algorithm 的关系

本仓库在结构与命名上与 X 官方开源的 `twitter/the-algorithm` 高度同构（Home Mixer、候选源、双塔召回、Transformer 排序、可见性过滤），可以视为它的一个**演进版本快照**：

- **服务层用 Rust 重写**（官方版以 Scala/Java 为主），强调内存安全和低延迟。
- **排序模型替换为 Grok 风格 Transformer**，并显式声明从 Grok-1 开源版本移植。
- **附带可直接运行的 mini 模型 + 体育语料**——官方版基本不带可运行权重，本仓库可以本地真正跑通推理。
- 新增独立的 **Grox 内容理解服务**与广告 / 品牌安全模块。

实际部署时，本仓库的子模块仍然只是"参考实现"——线上模型更大、特征更丰富、并接到完整的实时流系统；这里发布的是结构与算法，而不是线上权重本身。

---

## 10. 术语对照表

| 英文 | 中文 | 含义 |
|---|---|---|
| In-Network | 关注网内 | 来自你关注的账号 |
| Out-of-Network (OON) | 关注网外 | 通过模型从全量语料检索到的候选 |
| Hydration / Hydrator | 注水 / 注水器 | 给候选补全字段 |
| Candidate Isolation | 候选隔离 | 排序时候选互相不可见的注意力掩码 |
| Two-Tower | 双塔 | 用户塔 + 物品塔，分别编码后做点积 |
| Hash-Based Embedding | 哈希 Embedding | 用多哈希查表代替每 ID 一条 embedding |
| Dwell | 停留 | 用户在帖子上停留的时长信号 |
| VF Filter | Visibility Filter | 选择后的可见性过滤 |
| Side Effect | 副作用 | 异步执行的缓存 / 日志 / 埋点 |
| MoE | Mixture of Experts | 混合专家（Phoenix MoE 是一类候选源） |
| WTF | Who To Follow | "推荐你关注"候选源 |
| PTOS | Platform Terms of Service | 平台政策合规检查 |

---

## 11. 2026-05-15 更新明细

1. **端到端推理脚本** [`phoenix/run_pipeline.py`](phoenix/run_pipeline.py)：合并了原先分开的 `run_ranker.py` / `run_retrieval.py`，一条命令完成"召回 → 排序"。
2. **预训练 Mini 模型 artifact**：通过 Git LFS 分发（~3 GB），开箱即用，不需要先自己训练。
3. **Grox 内容理解服务** [`grox/`](grox/)：分类器、嵌入器、任务调度引擎，覆盖垃圾识别、品类分类、PTOS 策略等。
4. **广告插入与品牌安全** [`home-mixer/ads/`](home-mixer/ads/)：广告插入、位置控制、敏感内容边界保护。
5. **Query Hydrators 扩充**：关注的话题、Starter Pack、曝光 Bloom Filter、IP、互关图、已展示历史。
6. **Candidate Hydrators 扩充**：互动计数、品牌安全信号、语言、媒体、引用帖展开、互关分等。
7. **新增候选源**：广告、Who To Follow、Phoenix MoE、Phoenix Topics、Prompts，并刷新 Thunder / Phoenix 原有源。

---

[⬆ 回到顶部](#top)

---

# X For You Feed Algorithm

> English version below. 中文版见 [上方](#x-for-you-推荐算法中文详解).

This repository contains the core recommendation system powering the "For You" feed on X. It combines in-network content (from accounts you follow) with out-of-network content (discovered through ML-based retrieval) and ranks everything using a Grok-based transformer model.

> **Note:** The transformer implementation is ported from the [Grok-1 open source release](https://github.com/xai-org/grok-1) by xAI, adapted for recommendation system use cases.

## Table of Contents

- [Updates — May 15th, 2026](#updates--may-15th-2026)
- [Overview](#overview)
- [System Architecture](#system-architecture)
- [Components](#components)
  - [Home Mixer](#home-mixer)
  - [Thunder](#thunder)
  - [Phoenix](#phoenix)
  - [Candidate Pipeline](#candidate-pipeline)
- [How It Works](#how-it-works)
  - [Pipeline Stages](#pipeline-stages)
  - [Scoring and Ranking](#scoring-and-ranking)
  - [Filtering](#filtering)
- [Key Design Decisions](#key-design-decisions)
- [License](#license)

---

## Updates — May 15th, 2026

This release updates the For You algorithm code, including a runnable end-to-end inference pipeline alongside new components for content understanding, ads, and candidate sourcing.

1. **End-to-end inference pipeline:** A new [`phoenix/run_pipeline.py`](phoenix/run_pipeline.py) replaces the separate `run_ranker.py` and `run_retrieval.py` scripts with a single entry point that runs **retrieval → ranking** from exported checkpoints, mirroring how the two stages are composed in production.

2. **Pre-trained model artifacts:** A pre-trained mini Phoenix model (256-dim embeddings, 4 attention heads, 2 transformer layers) is now packaged as a ~3 GB archive distributed via Git LFS, enabling out-of-the-box inference without training your own model first.

3. **Grox content-understanding pipeline:** A new [`grox/`](grox/) service is included, providing classifiers, embedders, and a task-execution engine for content understanding workloads such as spam detection, post-category classification, and PTOS policy enforcement.

4. **Ads blending system:** Includes a new [`home-mixer/ads/`](home-mixer/ads/) module that handles ad injection and positioning within the feed, including brand-safety tracking that respects sensitive content boundaries.

5. **Query hydrators:** Home mixer now hydrates user context including followed topics, starter packs, impression bloom filters, IP, mutual follow graphs, and served history.

6. **Candidate hydrators:** Additional hydrators for engagement counts, brand safety signals, language codes, media detection, quote post expansion, mutual follow scores, and more.

7. **Candidate sources:** Adds sources for ads, who to follow, Phoenix MoE, Phoenix topics, prompts, and updates Thunder/Phoenix ones.

---

## Overview

The For You feed algorithm retrieves, ranks, and filters posts from two sources:

1. **In-Network (Thunder)**: Posts from accounts you follow
2. **Out-of-Network (Phoenix Retrieval)**: Posts discovered from a global corpus

Both sources are combined and ranked together using **Phoenix**, a Grok-based transformer model that predicts engagement probabilities for each post. The final score is a weighted combination of these predicted engagements.

We have eliminated every single hand-engineered feature and most heuristics from the system. The Grok-based transformer does all the heavy lifting by understanding your engagement history (what you liked, replied to, shared, etc.) and using that to determine what content is relevant to you.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    FOR YOU FEED REQUEST                                     │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                         HOME MIXER                                          │
│                                    (Orchestration Layer)                                    │
├─────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                   QUERY HYDRATION                                   │   │
│   │  ┌──────────────────────────┐    ┌──────────────────────────────────────────────┐   │   │
│   │  │ User Action Sequence     │    │ User Features                                │   │   │
│   │  │ (engagement history)     │    │ (following list, preferences, etc.)          │   │   │
│   │  └──────────────────────────┘    └──────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                  CANDIDATE SOURCES                                  │   │
│   │         ┌─────────────────────────────┐    ┌────────────────────────────────┐       │   │
│   │         │        THUNDER              │    │     PHOENIX RETRIEVAL          │       │   │
│   │         │    (In-Network Posts)       │    │   (Out-of-Network Posts)       │       │   │
│   │         │                             │    │                                │       │   │
│   │         │  Posts from accounts        │    │  ML-based similarity search    │       │   │
│   │         │  you follow                 │    │  across global corpus          │       │   │
│   │         └─────────────────────────────┘    └────────────────────────────────┘       │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                      HYDRATION                                      │   │
│   │  Fetch additional data: core post metadata, author info, media entities, etc.       │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                      FILTERING                                      │   │
│   │  Remove: duplicates, old posts, self-posts, blocked authors, muted keywords, etc.   │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                       SCORING                                       │   │
│   │  ┌──────────────────────────┐                                                       │   │
│   │  │  Phoenix Scorer          │    Grok-based Transformer predicts:                   │   │
│   │  │  (ML Predictions)        │    P(like), P(reply), P(repost), P(click)...          │   │
│   │  └──────────────────────────┘                                                       │   │
│   │               │                                                                     │   │
│   │               ▼                                                                     │   │
│   │  ┌──────────────────────────┐                                                       │   │
│   │  │  Weighted Scorer         │    Weighted Score = Σ (weight × P(action))            │   │
│   │  │  (Combine predictions)   │                                                       │   │
│   │  └──────────────────────────┘                                                       │   │
│   │               │                                                                     │   │
│   │               ▼                                                                     │   │
│   │  ┌──────────────────────────┐                                                       │   │
│   │  │  Author Diversity        │    Attenuate repeated author scores                   │   │
│   │  │  Scorer                  │    to ensure feed diversity                           │   │
│   │  └──────────────────────────┘                                                       │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                      SELECTION                                      │   │
│   │                    Sort by final score, select top K candidates                     │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                              │
│                                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                              FILTERING (Post-Selection)                             │   │
│   │                 Visibility filtering (deleted/spam/violence/gore etc)               │   │
│   └─────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                     RANKED FEED RESPONSE                                    │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Components

### Home Mixer

**Location:** [`home-mixer/`](home-mixer/)

The orchestration layer that assembles the For You feed. It leverages the `CandidatePipeline` framework with the following stages:

| Stage | Description |
|-------|-------------|
| Query Hydrators | Fetch user context (engagement history, following list) |
| Sources | Retrieve candidates from Thunder and Phoenix |
| Hydrators | Enrich candidates with additional data |
| Filters | Remove ineligible candidates |
| Scorers | Predict engagement and compute final scores |
| Selector | Sort by score and select top K |
| Post-Selection Filters | Final visibility and dedup checks |
| Side Effects | Cache request info for future use |

The server exposes a gRPC endpoint (`ScoredPostsService`) that returns ranked posts for a given user.

---

### Thunder

**Location:** [`thunder/`](thunder/)

An in-memory post store and realtime ingestion pipeline that tracks recent posts from all users. It:

- Consumes post create/delete events from Kafka
- Maintains per-user stores for original posts, replies/reposts, and video posts
- Serves "in-network" post candidates from accounts the requesting user follows
- Automatically trims posts older than the retention period

Thunder enables sub-millisecond lookups for in-network content without hitting an external database.

---

### Phoenix

**Location:** [`phoenix/`](phoenix/)

The ML component with two main functions:

#### 1. Retrieval (Two-Tower Model)
Finds relevant out-of-network posts:
- **User Tower**: Encodes user features and engagement history into an embedding
- **Candidate Tower**: Encodes all posts into embeddings
- **Similarity Search**: Retrieves top-K posts via dot product similarity

#### 2. Ranking (Transformer with Candidate Isolation)
Predicts engagement probabilities for each candidate:
- Takes user context (engagement history) and candidate posts as input
- Uses special attention masking so candidates cannot attend to each other
- Outputs probabilities for each action type (like, reply, repost, click, etc.)

See [`phoenix/README.md`](phoenix/README.md) for detailed architecture documentation.

---

### Candidate Pipeline

**Location:** [`candidate-pipeline/`](candidate-pipeline/)

A reusable framework for building recommendation pipelines. Defines traits for:

| Trait | Purpose |
|-------|---------|
| `Source` | Fetch candidates from a data source |
| `Hydrator` | Enrich candidates with additional features |
| `Filter` | Remove candidates that shouldn't be shown |
| `Scorer` | Compute scores for ranking |
| `Selector` | Sort and select top candidates |
| `SideEffect` | Run async side effects (caching, logging) |

The framework runs sources and hydrators in parallel where possible, with configurable error handling and logging.

---

## How It Works

### Pipeline Stages

1. **Query Hydration**: Fetch the user's recent engagements history and metadata (eg. following list)

2. **Candidate Sourcing**: Retrieve candidates from:
   - **Thunder**: Recent posts from followed accounts (in-network)
   - **Phoenix Retrieval**: ML-discovered posts from the global corpus (out-of-network)

3. **Candidate Hydration**: Enrich candidates with:
   - Core post data (text, media, etc.)
   - Author information (username, verification status)
   - Video duration (for video posts)
   - Subscription status

4. **Pre-Scoring Filters**: Remove posts that are:
   - Duplicates
   - Too old
   - From the viewer themselves
   - From blocked/muted accounts
   - Containing muted keywords
   - Previously seen or recently served
   - Ineligible subscription content

5. **Scoring**: Apply multiple scorers sequentially:
   - **Phoenix Scorer**: Get ML predictions from the Phoenix transformer model
   - **Weighted Scorer**: Combine predictions into a final relevance score
   - **Author Diversity Scorer**: Attenuate repeated author scores for diversity
   - **OON Scorer**: Adjust scores for out-of-network content

6. **Selection**: Sort by score and select the top K candidates

7. **Post-Selection Processing**: Final validation of post candidates to be served

---

### Scoring and Ranking

The Phoenix Grok-based transformer model predicts probabilities for multiple engagement types:

```
Predictions:
├── P(favorite)
├── P(reply)
├── P(repost)
├── P(quote)
├── P(click)
├── P(profile_click)
├── P(video_view)
├── P(photo_expand)
├── P(share)
├── P(dwell)
├── P(follow_author)
├── P(not_interested)
├── P(block_author)
├── P(mute_author)
└── P(report)
```

The **Weighted Scorer** combines these into a final score:

```
Final Score = Σ (weight_i × P(action_i))
```

Positive actions (like, repost, share) have positive weights. Negative actions (block, mute, report) have negative weights, pushing down content the user would likely dislike.

---

### Filtering

Filters run at two stages:

**Pre-Scoring Filters:**
| Filter | Purpose |
|--------|---------|
| `DropDuplicatesFilter` | Remove duplicate post IDs |
| `CoreDataHydrationFilter` | Remove posts that failed to hydrate core metadata |
| `AgeFilter` | Remove posts older than threshold |
| `SelfpostFilter` | Remove user's own posts |
| `RepostDeduplicationFilter` | Dedupe reposts of same content |
| `IneligibleSubscriptionFilter` | Remove paywalled content user can't access |
| `PreviouslySeenPostsFilter` | Remove posts user has already seen |
| `PreviouslyServedPostsFilter` | Remove posts already served in session |
| `MutedKeywordFilter` | Remove posts with user's muted keywords |
| `AuthorSocialgraphFilter` | Remove posts from blocked/muted authors |

**Post-Selection Filters:**
| Filter | Purpose |
|--------|---------|
| `VFFilter` | Remove posts that are deleted/spam/violence/gore etc. |
| `DedupConversationFilter` | Deduplicate multiple branches of the same conversation thread |

---

## Key Design Decisions

### 1. No Hand-Engineered Features
The system relies entirely on the Grok-based transformer to learn relevance from user engagement sequences. No manual feature engineering for content relevance. This significantly reduces the complexity in our data pipelines and serving infrastructure.

### 2. Candidate Isolation in Ranking
During transformer inference, candidates cannot attend to each other—only to the user context. This ensures the score for a post doesn't depend on which other posts are in the batch, making scores consistent and cacheable.

### 3. Hash-Based Embeddings
Both retrieval and ranking use multiple hash functions for embedding lookup

### 4. Multi-Action Prediction
Rather than predicting a single "relevance" score, the model predicts probabilities for many actions.

### 5. Composable Pipeline Architecture
The `candidate-pipeline` crate provides a flexible framework for building recommendation pipelines with:
- Separation of pipeline execution and monitoring from business logic
- Parallel execution of independent stages and graceful error handling
- Easy addition of new sources, hydrations, filters, and scorers

---

## License

This project is licensed under the Apache License 2.0. See [LICENSE](LICENSE) for details.
