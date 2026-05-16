# 5.1 双塔检索模型（Phoenix Retrieval）

**代码：** [`phoenix/recsys_retrieval_model.py`](https://github.com/yingwang/x-algorithm/blob/main/phoenix/recsys_retrieval_model.py)

## 总体形状

```
 用户特征 + 历史动作序列 ──▶ User Tower (Phoenix Transformer)
                                     │
                                     ▼
                              user_embedding [B, D]   ← L2 归一化
                                                     ┐
 全量语料 (post + author) ──▶ Candidate Tower (2 层 MLP)
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
                              Top-K 候选索引 + 相似度分
```

## User Tower：复用 Phoenix Transformer

User Tower 不是单独的网络，**直接复用排序模型的 Transformer 主干**（`PhoenixRetrievalModel.build_user_representation`）：

```python
embeddings = jnp.concatenate([user_embeddings, history_embeddings], axis=1)
padding_mask = jnp.concatenate([user_padding_mask, history_padding_mask], axis=1)

model_output = self.model(embeddings, padding_mask, candidate_start_offset=None)
# candidate_start_offset=None → 没有"候选段"，纯做用户编码

# mean-pool over valid positions
mask_float = padding_mask.astype(jnp.float32)[:, :, None]
user_embedding_sum = jnp.sum(user_outputs * mask_float, axis=1)
mask_sum = jnp.sum(mask_float, axis=1)
user_representation = user_embedding_sum / jnp.maximum(mask_sum, 1.0)

# L2 normalize
user_representation /= ||user_representation||₂
```

要点：

- **没有 candidate 段**：召回的 query 只是用户本身，所以 Transformer 输入只有 `[User, History]`。
- **mean-pool 而不是取最后一个 token**：避免对"最新一次动作"过度敏感（用户偶尔的误点不该完全主导召回方向）。
- **L2 归一化**：让点积等价于余弦相似度，便于离线索引和 ANN 后端。

## Candidate Tower：参数高效的 MLP

`CandidateTower` 有两种模式，由 `enable_linear_proj` 切换：

### 模式 A：`enable_linear_proj=True`（默认）

```python
# 输入 [B, C, num_hashes, D] → 展平 → [B, C, num_hashes*D]
hidden = silu(post_author_embedding @ proj_1)   # → [B, C, 2D]
candidate = hidden @ proj_2                      # → [B, C, D]
candidate /= ||candidate||₂
```

两层 MLP + SiLU 激活。比起 User Tower 的多层 Transformer，Candidate Tower 故意轻量化：

- 候选语料几千万到几亿条，每条都要预计算并存索引，参数越少越省存储/带宽。
- 离线一次性算好后只在重建索引时再算 —— 不像 user embedding 每请求都要算。

### 模式 B：`enable_linear_proj=False`

```python
candidate_representation = jnp.mean(post_author_embedding, axis=-2)  # hash 维 mean
candidate /= ||candidate||₂
```

更极端的参数效率：直接对多哈希 embedding 做平均，不加可学权重。适用场景：早期实验、词表特别大、训练数据不足。

## 检索的实现

```python
def _retrieve_top_k(user_representation, corpus_embeddings, top_k, corpus_mask):
    scores = jnp.matmul(user_representation, corpus_embeddings.T)   # [B, N]
    if corpus_mask is not None:
        scores = jnp.where(corpus_mask[None, :], scores, -INF)      # 屏蔽无效语料
    top_k_scores, top_k_indices = jax.lax.top_k(scores, top_k)
    return top_k_indices, top_k_scores
```

- Demo 是**精确稠密**点积（语料 537K 还撑得住）。
- 线上语料是数亿级别，会切换到 **ANN**（近似最近邻）后端（仓库没开源 ANN 索引服务，但 `PhoenixRetrievalClient` 是 gRPC 抽象，可换实现）。
- `corpus_mask` 用于在线屏蔽（例如某用户的语料黑名单）—— 屏蔽后这些位置 score 变 -INF，永远不会被 top_k 选中。

## 训练目标（隐含在权重里）

仓库没有训练代码，但从模型形状能反推训练目标：

- **正样本**：用户的真实正向互动（喜欢、长停留、转推）。
- **负样本**：随机负采样 + in-batch negatives。
- **损失**：常见的 **softmax cross-entropy on similarities** 或 **sampled InfoNCE**。
- 两塔的 embedding 同时优化，使"正样本对的点积比随机对高"。

> 召回模型不直接预测动作；它学的是"哪些 item 对这个用户来说是 plausible 候选"。剩下"是不是喜欢、是不是讨厌"由排序模型决定。

## 一次完整的召回（沿用 Demo）

`phoenix/run_pipeline.py` 里的 `run_retrieval` 流程：

```
1. 读 example_sequence.json（用户的 3 条历史：NFL / NBA / NHL）
2. 用 hash_user / hash_item / hash_author 把 ID 哈希到 [0, vocab)
3. 在统一 embedding 表里 gather
4. 拼成 RecsysBatch，调 PhoenixRetrievalModel
5. 得到 user_representation [1, D=128]
6. 加载 sports_corpus.npz：candidate_post_id + candidate_author_id + 预算好的 candidate_emb [537K, 128]
7. dot + top_k(200)
8. 返回 200 个 post_id 进入下一阶段（排序）
```

`sports_corpus.npz` 的 candidate_emb 是**离线一次性算好**的 —— 线上不需要每次再过 Candidate Tower，这是双塔架构的核心收益。

## 设计思想小结

| 选择 | 原因 |
|---|---|
| 双塔（User vs Candidate 独立编码） | 让 Candidate 表示可预计算 + ANN 加速 |
| User Tower 用 Transformer | 用户序列长，需要捕捉顺序与动作组合 |
| Candidate Tower 用 MLP | 候选数巨大，必须参数 / 算力廉价 |
| L2 归一化 → 余弦 | 与 ANN 库（如 FAISS、ScaNN）原生兼容 |
| mean-pool over history | 不对"最近一次动作"过敏感 |
| 多哈希 embedding 共享 | 无 OOV、固定显存（详见 [§5.4 Mini 模型规格](phoenix-mini-specs.md)） |
