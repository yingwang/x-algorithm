# 9. 与 X 官方仓库 the-algorithm 的关系

本仓库（`yingwang/x-algorithm`）在结构与命名上与 X 官方开源的 [`twitter/the-algorithm`](https://github.com/twitter/the-algorithm) 高度同构，可以视为它的一个**演进版本快照**。下面把主要异同列清。

> 注：上游 `xai-org/x-algorithm`（本 fork 的源）和老的 `twitter/the-algorithm` 之间的演进，是从 Twitter → X → xAI 推荐栈代换的一部分。这里只比较"系统结构"层面的差异，不深入代码 diff。

## 结构同构的部分

| 概念 | the-algorithm | 本仓库 |
|---|---|---|
| 入口编排服务 | Home Mixer | `home-mixer/` |
| 关注网内候选 | Tweet Service / EarlyBird | `thunder/` |
| 关注网外候选 | SimClusters / TwHIN / RealGraph | `phoenix/`（统一） |
| 排序 | MaskNet / Heavy Ranker | `phoenix/recsys_model.py` |
| 可见性过滤 | VF service | `VFFilter` / `AncillaryVFFilter` |
| 候选流水线框架 | product-mixer | `candidate-pipeline/` |
| 多动作预测 | 13 个动作 | 19 个动作 + 8 个连续值 |
| 候选源混排 | timeline-mixer | `BlenderSelector` + `home-mixer/ads/` |

可以说"骨架几乎一样"。

## 显著差异

### 1. 服务层语言

| | 官方 | 本仓库 |
|---|---|---|
| 编排 / 候选服务 | Scala（部分 Java）+ JVM | **Rust + Tokio** |
| 模型 | Tensorflow / Torch | **JAX + Haiku** |
| 候选库 | EarlyBird（Lucene 改造） | **Thunder（纯内存 + DashMap）** |

Rust 化的动机：可控延迟、低尾延、无 GC 暂停、内存安全。

### 2. 排序模型架构

| | 官方 | 本仓库 |
|---|---|---|
| 模型族 | MaskNet / DLRM 类多 MLP + 大量手工特征 | **Grok-1 风格 Transformer** |
| 输入 | 手工特征向量 | **用户动作序列** |
| 注意力 | 无（MLP） | **Transformer + 候选隔离掩码** |
| 候选 ID 表示 | 大 embedding 表（每 ID 一行） | **多哈希 embedding** |
| 多动作 | 13 个 | **19 个 + 8 个连续值** |

这是最大区别：**砍掉手工特征，依靠 Transformer 学习序列**。

### 3. Candidate Isolation 注意力

官方 the-algorithm 的 MaskNet 是单候选打分（无候选间交互），但实现上是 pointwise；候选并发打分时并不显式声明"不能 listwise"。

本仓库的排序 Transformer 把所有候选拼在同一条序列里**统一过 attention**，要靠 Candidate Isolation 掩码才回到 pointwise 形态 —— 是"把 listwise 架构按 pointwise 用"。

这种做法保留了 Transformer 形状的好处（KV cache 共享、并行）同时拿到 pointwise 的可缓存性。详见 [§5.2 排序 Transformer 与候选隔离](phoenix-ranking.md)。

### 4. 可运行的 Demo

| | 官方 | 本仓库 |
|---|---|---|
| 是否带预训练权重 | 基本不带 | **带 Mini Phoenix 模型 + 537K 体育语料**（~3 GB Git LFS） |
| 端到端推理 | 没有可一键跑通的脚本 | `phoenix/run_pipeline.py` 一行命令搞定 |

这是本仓库相对官方版的最大"教学价值"。你能真的看到一次召回 + 排序的结果是什么样。

### 5. 新增 Grox 内容理解服务

官方版没有独立的"内容理解"服务这一概念，安全标签、品类等是散在不同任务里。

本仓库把这些抽出来变成独立的 `grox/`，统一调度，对 home-mixer 是异步生产者—消费者关系。详见 [§3.5 Grox](components-grox.md)。

### 6. 广告 / 品牌安全模块独立化

本仓库把广告插入抽到外层 Pipeline（`ForYouCandidatePipeline`）和 `home-mixer/ads/`：

- `safe_gap_blender.rs`：广告与敏感内容的安全间隔
- `partition_organic_blender.rs`：有机帖 + 广告的混排策略
- `util.rs`：插入位置工具

这种"内核做相关性、外层做版面与商业化"的分层比官方 the-algorithm 的耦合实现更清晰。

### 7. Query Hydrator 扩充

新版多了不少：`followed_grok_topics` / `followed_starter_packs` / `inferred_grok_topics` / `impression_bloom_filter` / `ip` / `mutual_follow` / `user_inferred_gender` / `user_demographics` / `served_history` …

这些都是"用户画像 / 上下文"补充，让 Phoenix 的 user representation 更准。

### 8. Candidate Hydrator 扩充

新版多了 `gizmoduck` / `blocked_by` / `subscription` / `filtered_topics` / `quote` / `has_media` / `language_code` / `video_duration` 等。这些字段一部分是 filter 用（quote、subscription），一部分是 scorer 用（video_duration 决定 VQV 权重门槛）。

## 部署 / 运行的本质差异

| | 官方 | 本仓库 |
|---|---|---|
| 是不是"可以直接部署的代码" | **不是**：依赖大量 Twitter 内部服务（Manhattan、Strato、Gizmoduck、TES 等） | **也不是**：依赖 X / xAI 内部服务 |
| 是不是"参考实现" | 是 | 是 |
| 是不是"真实生产代码" | 是 | 是（但模型是 Mini 快照） |

两个仓库都是"参考实现 + 真实代码结构"，让你能看到大公司怎么组织推荐栈，但**不能 docker-compose up 一下就跑起来**。要跑通就只剩 [`phoenix/run_pipeline.py`](running.md) 这一条 Demo。

## 一句话定位

> **官方 the-algorithm 是"Twitter 时代的快照"，本仓库是"xAI/Grok 时代的快照"——保留了同一套架构骨架，把核心排序换成了 Transformer，把语言换成了 Rust，并第一次让人真的能下载权重跑通。**
