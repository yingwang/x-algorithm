# 5.4 Mini Phoenix 模型规格

随仓库一起发布的预训练模型是一个精心缩减过的版本。线上真正在生产环境中运行的 Phoenix 模型层数更深、隐藏维度更宽、动作维度更多，本节所列的所有配置都是相对线上版本而言的缩减结果。

需要事先澄清一处文档不一致的地方：仓库根目录下的 `README.md` 顶部声明的是"256 维、4 头、2 层"，而 `phoenix/README.md` 以及代码 `recsys_model.py` 中的默认配置实际上是"128 维、4 层"。两者不一致时以 `phoenix/README.md` 以及随模型一同下载的 `config.json` 为准。

## 完整配置

下面这张表完整地列出了 Mini Phoenix 模型的所有关键超参，连同每一项的含义说明。

| 参数 | Mini 默认值 | 含义 |
|---|---|---|
| `emb_size` | 128 | Transformer 的 hidden 维度 |
| `model.num_layers` | 4 | Transformer 层数 |
| `model.num_heads` | 4 | 每一层的注意力头数 |
| `model.key_size` | 32 | 每一个头的 key 与 query 维度 |
| `model.widening_factor` | 2 | FFN 中间层的放大倍数 |
| `history_seq_len` | 127 | 用户历史的最大长度 |
| `candidate_seq_len` | 64 | 一次打分最多能容纳的候选数 |
| `product_surface_vocab_size` | 16 | 产品面（Home、Profile、Search 等）数量 |
| `post_age_granularity_mins` | 60 | 帖龄分桶的粒度（分钟） |
| `post_age_vocab_size` | 4800 / 60 + 2 = 82 | 帖龄桶数，含 missing 与 overflow |
| `num_actions` | 19 | 离散动作类型数 |
| `num_continuous_actions` | 8 | 连续动作维度，包含停留时长等 |
| `continuous_action_hidden_dim` | 64 | 连续值投影到 D 的中间层维度 |
| `hash_config.num_user_hashes` | 2 | 每一个 user_id 所使用的哈希函数数量 |
| `hash_config.num_item_hashes` | 2 | item 同上 |
| `hash_config.num_author_hashes` | 2 | author 同上 |
| `hash_config.num_ip_hashes` | 0 | IP 默认禁用，线上版本中启用 |
| `vocab_size`（user、item、author 各一张） | 各 1,000,000 | 每张哈希表的桶数 |
| `fprop_dtype` | bfloat16 | 前向计算使用 bf16，权重以 fp32 存储 |
| `right_anchored_rope` | 可选 | 开启之后让最新一条 history 拿到固定位置编码 |
| `mask_neg_feedback_on_negatives` | True | 负反馈 loss 只在确实出现负反馈标签时才计算 |

## 内存与磁盘

模型文件本身体积并不大，但是嵌入表占用了相当可观的磁盘空间。下面这张表列出了所有组件的体积情况。

| 部件 | 大小 |
|---|---|
| Retrieval `model_params.npz` | 约 3 MB |
| Retrieval `embedding_tables.npz` | 约 1.4 GB，由三张 1M × 128 fp32 表加上 offset 组成 |
| Ranker `model_params.npz` | 约 3 MB |
| Ranker `embedding_tables.npz` | 约 1.4 GB |
| `sports_corpus.npz` | 53.7 万行候选，加上预先算好的形状为 [537K, 128] 的 candidate embedding |
| `example_sequence.json` | 一份 demo 用户互动序列，包含三条历史（NFL、NBA、NHL） |
| 合计 zip | 约 3 GB，通过 Git LFS 分发 |

模型权重之所以看起来"那么小"（仅有几 MB），是因为绝大部分参数都集中在 embedding 表里。每一张表的规模都是 1M × 128 × 4 字节，单张就有 512 MB，三张合起来便达到约 1.5 GB。相比之下 Transformer 主干本身只有百万级别的参数。

## 与线上模型的差异

仓库根目录的 `README.md` 与 `phoenix/README.md` 都明确说明了 Mini 版本与线上版本之间的几处主要差异。

第一处是规模。线上模型的层数更深、隐藏维度更宽、注意力头数也更多。具体的线上规模并未公开，但 Mini 版本明显是一份压缩过的"蒸馏"或者"轻量替身"。

第二处是训练状态。线上 Phoenix 是处于连续训练之中的模型，每隔一段时间就会产出新的 checkpoint；而仓库里发布的版本是某一时刻的冻结快照，并不会随时间更新。这一区别意味着读者在本地跑出的结果只能反映那一刻的模型状态，与生产环境会有时间差。

第三处是语料规模。`sports_corpus.npz` 文件中所包含的，仅仅是过去六小时内被打上 "Sports" 标签的约 53.7 万条帖子，规模上完全无法与线上数亿级别的全平台语料相比。线上推理时所使用的语料还会通过 ANN 索引服务来加速检索，而 ANN 服务本身并不在开源范围之内。

## 跑通它需要多大的机器

把模型在本地跑起来对硬件的要求并不算高。

内存方面，模型占用约 3 GB，加上 JAX 与 NumPy 的中间张量大约还要再多 1 到 2 GB，因此 8 GB 内存起步会比较稳。

算力方面，纯 CPU 也能跑通 Demo 用例（batch 大小为 1 的情形），但是首次 JIT 编译需要等几十秒。如果机器上安装了 CUDA 版本的 JAX，会自动启用 GPU 加速，编译完之后再次调用就会进入毫秒级延迟。

## 把配置调小做实验

如果想要用更小的模型快速做迭代实验，下面这张表给出了几条常见的调整方向。

| 想做的事 | 改哪一项 |
|---|---|
| 缩短历史窗口 | `history_seq_len = 64` |
| 一次打分覆盖更少的候选 | `candidate_seq_len = 32` |
| 让单头窄一点 | `key_size = 16`，`num_heads = 2` |
| 减小词表 | `vocab_size = 200_000`，但是需要重新训练，否则原有 hash 无法对应到新表 |

embedding 表的大小直接决定了模型能够容纳多少独立 ID 而不发生冲突。修改 `vocab_size` 之后必须重新训练，不能简单地对已有 checkpoint 做 reshape。
