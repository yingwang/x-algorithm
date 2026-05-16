# 5.4 Mini Phoenix 模型规格

随仓库一起发布的预训练模型是**缩水版**——线上模型层数更深、维度更宽、动作数更多。

> 注意：仓库根 `README.md` 顶部说 "256-dim · 4 head · 2 layer"，而 `phoenix/README.md` 和代码 `recsys_model.py` 默认配置是 **"128-dim · 4 layer"**。以 `phoenix/README.md` 和实际下载的 `config.json` 为准。

## 完整配置

| 参数 | Mini 默认 | 含义 |
|---|---|---|
| `emb_size` | 128 | Transformer hidden dim |
| `model.num_layers` | 4 | Transformer 层数 |
| `model.num_heads` | 4 | 每层 attention 头数 |
| `model.key_size` | 32 | 每个头的 key/query 维度 |
| `model.widening_factor` | 2 | FFN 中间层放大倍数 |
| `history_seq_len` | 127 | 用户历史最大长度 |
| `candidate_seq_len` | 64 | 一次最多打多少候选 |
| `product_surface_vocab_size` | 16 | 产品面（Home / Profile / Search 等）数量 |
| `post_age_granularity_mins` | 60 | 帖龄分桶粒度（分钟） |
| `post_age_vocab_size` | `4800 / 60 + 2 = 82` | 帖龄桶数（含 missing 与 overflow） |
| `num_actions` | 19 | 离散动作类型数 |
| `num_continuous_actions` | 8 | 连续动作维度（含 dwell time 等） |
| `continuous_action_hidden_dim` | 64 | 连续值投影到 D 的中间层 |
| `hash_config.num_user_hashes` | 2 | 每个 user_id 用几个 hash |
| `hash_config.num_item_hashes` | 2 | item 同上 |
| `hash_config.num_author_hashes` | 2 | author 同上 |
| `hash_config.num_ip_hashes` | 0 | IP 默认禁用（线上启用） |
| `vocab_size` (user / item / author) | 1,000,000 各 | 哈希桶数 |
| `fprop_dtype` | bfloat16 | 前向用 bf16，权重存 fp32 |
| `right_anchored_rope` | 可选 | 让最新 history 拿固定位置编码 |
| `mask_neg_feedback_on_negatives` | True | 负反馈 loss 只在确实有负反馈标签时算 |

## 内存与磁盘

| 部件 | 大小 |
|---|---|
| Retrieval `model_params.npz` | ~3 MB |
| Retrieval `embedding_tables.npz` | ~1.4 GB（3 张 1M × 128 fp32 表 + offset） |
| Ranker `model_params.npz` | ~3 MB |
| Ranker `embedding_tables.npz` | ~1.4 GB |
| `sports_corpus.npz` | 537K 行候选 + 预计算的 [537K, 128] candidate embedding |
| `example_sequence.json` | 一条 demo 用户互动序列（NFL / NBA / NHL 三条） |
| 合计 zip | ~3 GB（通过 Git LFS 分发） |

模型权重之所以"那么小"（几 MB），是因为参数主要在 embedding 表里（每张 1M × 128 × 4 bytes = 512 MB，三张就 1.5 GB）。Transformer 主干本身就 100 万级别参数。

## 与线上模型的差异

仓库 `README.md` 和 `phoenix/README.md` 都明确说：

- **更小**：线上模型层数更深、维度更宽。
- **冻结快照**：线上 Phoenix 是**连续训练**的，每隔一段时间产出新 checkpoint；这里发布的是某一刻的冻结版。
- **示例语料**：`sports_corpus.npz` 是过去 6 小时内被打上 "Sports" 标签的 ~537K 条帖。线上语料是数亿规模，并按 ANN 索引服务（不在开源范围）。

## 跑通它要多大机器

- 内存：3 GB 模型 + 1~2 GB jax/numpy 中间张量 → **8 GB RAM** 起步比较稳。
- 算力：纯 CPU 也能跑（demo 用例 batch=1），但首次 jit 编译要等几十秒。
- GPU 加速：有 CUDA jax 装上自动用，编译完再调一次就毫秒级。

## 把配置调小做实验

如果想用更小的模型快速迭代，可改：

| 想做的事 | 改哪儿 |
|---|---|
| 缩短历史窗口 | `history_seq_len = 64` |
| 一次打更少候选 | `candidate_seq_len = 32` |
| 单头窄一点 | `key_size = 16`，`num_heads = 2` |
| 减小词表 | `vocab_size = 200_000` —— 但需要**重新训练**，否则 hash 不上 |

embedding 表的大小决定了能容纳多少独立 ID 不冲突；改了之后必须重训，不能简单 reshape 已有 checkpoint。
