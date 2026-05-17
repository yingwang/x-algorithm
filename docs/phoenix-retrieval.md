# 5.1 双塔检索模型（Phoenix Retrieval）

**代码：** [`phoenix/recsys_retrieval_model.py`](https://github.com/yingwang/x-algorithm/blob/main/phoenix/recsys_retrieval_model.py)

## 总体形状

召回阶段在整个推荐链路里所处的位置非常特殊。它面对的是平台上的全量语料，规模可能达到数亿条之多，每一次刷新都需要在毫秒级别的时间内从中挑出几百到几千条最有希望与当前用户相关的候选。这一规模与延迟的双重压力直接决定了召回阶段不可能像排序阶段那样使用厚重的 Transformer 给每一条语料逐一打分，只能采用一种参数效率极高、可以离线预计算并且能与近似最近邻搜索后端配合的结构。Phoenix Retrieval 采用的正是这种思路下的标准方案：双塔架构。

双塔架构（two-tower architecture）的核心思想可以用一句话概括："把用户表示与候选表示分开来编码"。具体来说，模型由两个相互独立的子网络组成：一座塔接收用户信息，输出一个用户向量；另一座塔接收候选信息，输出一个候选向量；两个向量通过点积来度量用户与候选之间的相关性。这一架构看起来简单，但是它带来的工程价值极为巨大：因为两座塔互相独立，所有候选的向量都可以离线一次性计算好并写入索引，线上只需要把用户实时编码一次，再到索引里做一次近似最近邻搜索即可拿到 Top-K 候选。

下面这张图直观地展示了 Phoenix Retrieval 的整体形状。

```
 用户特征加上历史动作序列 ──▶ User Tower（Phoenix Transformer）
                                     │
                                     ▼
                              user_embedding [B, D]   ← L2 归一化
                                                     ┐
 全量语料（post 与 author）  ──▶ Candidate Tower（两层 MLP）
                                     │
                                     ▼
                            corpus_embedding [N, D]   ← L2 归一化  ┘
                                     │
                                     ▼
                     scores = user_embedding · corpus_embedding.T   [B, N]
                                     │
                                     ▼
                            jax.lax.top_k(scores, K)
                                     │
                                     ▼
                              Top-K 候选索引加上对应的相似度分
```

把"用户表示"与"候选表示"完全独立开来分别编码，是双塔架构最核心的工程价值所在。如果让用户与候选在某一层互相做注意力，那么每一条候选都得在每一次请求时重新计算它的表示，亿级候选根本不可能在毫秒内做完。双塔架构通过"独立编码加上点积匹配"这一巧妙的拆分，把原本"亿级语料乘以每次请求逐一打分"的不可行问题，转化成了"亿级语料离线一次性编码加上每次请求一次 ANN 查询"这一在工程上完全可行的方案。

下面分别介绍两座塔的具体结构。

## User Tower：复用 Phoenix Transformer

User Tower 并不是一份独立的网络结构。它直接复用了排序模型那一份 Phoenix Transformer 主干，仅在使用方式上做了相应的调整。具体定义在 `PhoenixRetrievalModel.build_user_representation` 函数中。

```python
embeddings = jnp.concatenate([user_embeddings, history_embeddings], axis=1)
padding_mask = jnp.concatenate([user_padding_mask, history_padding_mask], axis=1)

model_output = self.model(embeddings, padding_mask, candidate_start_offset=None)
# candidate_start_offset=None 意味着没有"候选段"，纯做用户编码

# 在有效位置上做 mean-pool
mask_float = padding_mask.astype(jnp.float32)[:, :, None]
user_embedding_sum = jnp.sum(user_outputs * mask_float, axis=1)
mask_sum = jnp.sum(mask_float, axis=1)
user_representation = user_embedding_sum / jnp.maximum(mask_sum, 1.0)

# L2 归一化
user_representation /= ||user_representation||₂
```

这一段实现里有三个值得展开来谈的细节。

第一处是序列里并没有"候选段"。召回阶段的 query 只包含用户本身的信息，因此 Transformer 的输入只有 `[User, History]` 两部分。`candidate_start_offset=None` 这一参数告诉模型"不要再去做候选与用户的交叉计算"，整段计算纯粹用来编码用户。这一点与排序阶段形成对比：排序阶段会把候选作为输入序列的一部分送进 Transformer，让模型对每一条候选做细致打分；召回阶段则只需要一个稳定的用户表示，候选侧的工作交给候选塔。

第二处是采用 mean-pool 而不是直接取最后一个 token 作为用户表示。"mean-pool"（平均池化）指的是对序列中所有有效位置上的输出向量求平均，得到一个固定维度的句子级表示。"取最后一个 token"则是另一种常见的池化方式，类似于 GPT 类语言模型的做法。这两种方式各有优劣，本仓库选择 mean-pool 背后的考量是：用户偶尔的误点不应当完全主导整个召回方向。如果取最后一个 token，模型对"最新一次动作"的敏感程度会过高；mean-pool 则把整个 history 里的有效信息平均化处理，使得用户表示更稳健。

第三处是 L2 归一化。"L2 归一化"指的是把一个向量除以它自身的 L2 范数（也就是各分量平方和的平方根），使得最终向量的长度变为 1。这一步看似简单，却让点积运算等价于余弦相似度。"余弦相似度"是衡量两个向量方向是否相近的常用指标，取值范围在 -1 到 1 之间。这一等价性极为重要，因为它让 User Tower 输出的向量可以直接交给主流的 ANN 后端（例如 FAISS 或 ScaNN）使用，无需再做额外的转换。

## Candidate Tower：参数高效的 MLP

候选塔的设计哲学与用户塔截然不同。`CandidateTower` 提供了两种模式，由参数 `enable_linear_proj` 切换。

### 模式 A：`enable_linear_proj=True`（默认）

```python
# 输入 [B, C, num_hashes, D]，先展平为 [B, C, num_hashes*D]
hidden = silu(post_author_embedding @ proj_1)   # 结果形状 [B, C, 2D]
candidate = hidden @ proj_2                      # 结果形状 [B, C, D]
candidate /= ||candidate||₂
```

这是一种很简洁的两层 MLP 加 SiLU 激活的设计。"SiLU"（Sigmoid Linear Unit）也叫做 Swish，是一种常用的激活函数，公式为 `x * sigmoid(x)`，相比传统 ReLU 在很多任务上表现更佳。与 User Tower 使用的多层 Transformer 相比，Candidate Tower 故意做得非常轻量。这种轻量化背后有两层考量。

第一层是空间考量：候选语料的规模可能是几千万到几亿条，每一条都需要预先计算出 embedding 并存入索引，参数越少则存储与带宽成本越低。第二层是时间考量：候选 embedding 是离线一次性计算的，只在索引重建时才会重新计算，并不像 user embedding 那样每次请求都需要算一遍。

### 模式 B：`enable_linear_proj=False`

```python
candidate_representation = jnp.mean(post_author_embedding, axis=-2)  # 在 hash 维度上做 mean
candidate /= ||candidate||₂
```

这是一种更加极端的参数效率方案：直接对多哈希 embedding 做平均，不再附加任何可学习的权重。它适合用在一些特殊场景中，例如早期实验阶段、词表规模特别大、或者训练数据相对不足的情形。在这些情形下，引入太多可学习参数反而可能因为数据不足而导致过拟合，简单的平均反而能给出更稳健的表示。

## 检索的实现

把用户塔与候选塔各自的输出准备好之后，真正的"检索"动作只剩下一次大规模的矩阵乘加上一次 top_k 选取。

```python
def _retrieve_top_k(user_representation, corpus_embeddings, top_k, corpus_mask):
    scores = jnp.matmul(user_representation, corpus_embeddings.T)   # 结果 [B, N]
    if corpus_mask is not None:
        scores = jnp.where(corpus_mask[None, :], scores, -INF)      # 屏蔽无效语料
    top_k_scores, top_k_indices = jax.lax.top_k(scores, top_k)
    return top_k_indices, top_k_scores
```

这里有几处实现细节同样值得说明。

仓库里提供的 Demo 走的是精确稠密点积。"精确稠密点积"指的是把用户向量与全部候选向量两两都计算一遍点积，不做任何加速。这种做法在 53.7 万条规模的语料上仍然是可承受的，但在线上数亿级别的语料上就完全不可行，必须切换到 ANN 后端。

近似最近邻搜索（Approximate Nearest Neighbor，简称 ANN）的核心思想是放弃"找出真正最近的 K 个"这一严格目标，转而追求"以远低于精确搜索的代价，找出大概率与最近 K 个高度重合的 K 个"。常见的 ANN 算法家族包括基于树的方法（例如 KD-Tree、Annoy）、基于哈希的方法（例如 LSH，Locality Sensitive Hashing）、基于量化的方法（例如 PQ，Product Quantization）、以及基于图的方法（例如 HNSW，Hierarchical Navigable Small World）。工业界常用的开源 ANN 库包括 FAISS（Meta）、ScaNN（Google）、Annoy（Spotify）等，它们大多支持上述多种算法。

仓库本身并没有把 ANN 索引服务一同开源，但是 `PhoenixRetrievalClient` 在设计上就是一个 gRPC 抽象，允许底层替换为任意一种 ANN 实现。

`corpus_mask` 这一参数用于在线屏蔽某些不应当被召回的语料。它的工作方式是把那些需要屏蔽的位置上的分数直接设置为负无穷，从而保证它们永远不会被 top_k 选中。这种做法用一行掩码运算就实现了"按用户级黑名单屏蔽"的能力。

## 训练目标（从权重反推）

仓库里并没有提供训练相关的代码，但是从模型的结构本身能够大致反推出训练目标的形式。

正样本来自用户的真实正向互动，例如喜欢、长停留、转推等。负样本来自随机负采样以及 in-batch negatives 这两种常见做法。"in-batch negatives" 是双塔类模型训练时非常流行的一种负采样方式：把同一个 batch 中其它用户的正样本当作当前用户的负样本。这种做法不需要额外存储负样本，效率高，又能让模型学到"区分用户偏好"的能力。

损失函数大概率是基于相似度的 softmax 交叉熵，或者类似的 sampled InfoNCE 形式。"InfoNCE" 是对比学习领域常用的一种损失，本质是让正样本对的相似度比负样本对的相似度尽可能大；"sampled InfoNCE" 是它在样本数量很大时的一种实用变体，通过采样而非穷举来近似归一化项。两塔的 embedding 一起被优化，使得"正样本对的点积分数比随机对要高"。

值得强调的一点是：召回模型不直接预测任何具体的动作。它学习到的是"哪些 item 对这个用户来说是 plausible 候选"，至于这一候选具体是否会被喜欢、是否会被讨厌，那是排序模型阶段的工作。这种"召回只管粗筛、排序负责细判"的分工是大规模推荐系统的标准做法，可以让两个阶段各自采用最适合自己的模型结构与训练目标。

## 一次完整的召回（沿用 Demo）

仓库内 `phoenix/run_pipeline.py` 中的 `run_retrieval` 函数走的就是上述完整流程，可以一步步拆开来看。

```
1. 读取 example_sequence.json，里面记录了用户的三条历史互动（依次是 NFL、NBA、NHL）
2. 通过 hash_user、hash_item、hash_author 把 ID 哈希到 [0, vocab) 区间
3. 在统一 embedding 表里 gather 出对应的向量
4. 把这些向量拼成 RecsysBatch，调用 PhoenixRetrievalModel
5. 得到 user_representation，形状为 [1, D=128]
6. 加载 sports_corpus.npz，里面包含 candidate_post_id、candidate_author_id 以及预先算好的 candidate_emb，形状为 [537K, 128]
7. 做矩阵乘以及 top_k 取 200
8. 把这 200 个 post_id 送入下一阶段（排序）
```

值得特别强调的一点是：`sports_corpus.npz` 里的 candidate_emb 是离线一次性计算好的。线上服务不需要每一次请求都重新过一遍 Candidate Tower，这正是双塔架构在工程层面带来的核心收益所在。

## 设计思想小结

把整套双塔召回模型背后的工程取舍并列起来，可以看到一条清晰的设计主线。

| 选择 | 原因 |
|---|---|
| 双塔结构（用户塔与候选塔独立编码） | 让候选表示可以离线预计算，再借助 ANN 加速线上查询 |
| 用户塔使用 Transformer | 用户序列长，需要捕捉顺序信息以及动作组合的语义 |
| 候选塔使用 MLP | 候选数量极其庞大，必须把每一条候选的编码成本压到最低 |
| L2 归一化转化为余弦相似度 | 与主流 ANN 库（例如 FAISS、ScaNN）天然兼容 |
| 对 history 做 mean-pool | 不让"最近一次动作"对用户表示产生过度影响 |
| 多哈希 embedding 共享 | 不存在 OOV，显存占用固定，详见[Mini 模型规格](phoenix-mini-specs.md)一节 |

这些选择合在一起形成了 Phoenix Retrieval 的整体设计哲学：在保证模型表达能力足够的前提下，把工程效率拉到最高，让亿级语料的实时召回成为可能。
