# 3.3 Phoenix · 模型层（Python / JAX）

**目录：** [`phoenix/`](https://github.com/yingwang/x-algorithm/tree/main/phoenix) — 详细子文档见 [`phoenix/README.md`](https://github.com/yingwang/x-algorithm/blob/main/phoenix/README.md)。

> Phoenix 是整套系统的"大脑"。本节给一个**总览**，深入细节请看 [Phoenix 模型深入](phoenix-retrieval.md) 系列章节。

## 由两个模型组成

| 模型 | 文件 | 职责 | 输出 |
|---|---|---|---|
| **Retrieval（双塔）** | `recsys_retrieval_model.py` | 从全量语料里按相似度召回 Top-K | 候选 ID 列表 + 相似度分 |
| **Ranking（带候选隔离掩码的 Transformer）** | `recsys_model.py` | 对每条候选预测多种动作的发生概率 | `[B, num_candidates, num_actions]` 的 logits + 连续值预测 |

两者共用同一份 **Grok 风格 Transformer 主干**（`grok.py`，从 [xAI Grok-1](https://github.com/xai-org/grok-1) 移植）。区别只在头部 + 注意力掩码 + 损失。

## 关键文件

```
phoenix/
  grok.py                       Transformer 主干（含特殊的候选隔离掩码工厂）
  recsys_model.py               Ranking：PhoenixModel + RecsysBatch / RecsysEmbeddings
  recsys_retrieval_model.py     Retrieval：双塔（User Tower 复用 Transformer + Candidate Tower 是 2 层 MLP）
  runners.py                    加载 checkpoint / embedding 表 / 配置的工具
  run_pipeline.py               端到端入口：召回 → 排序，本次新增（合并旧的 run_retrieval.py + run_ranker.py）
  run_retrieval.py              旧：只跑召回
  run_ranker.py                 旧：只跑排序
  test_recsys_model.py          排序模型单元测试
  test_recsys_retrieval_model.py 召回模型单元测试
  pyproject.toml                uv 依赖（jax / jaxlib / dm-haiku / numpy）
  artifacts/oss-phoenix-artifacts.zip   预训练 Mini 模型 + 体育语料（~3 GB · Git LFS）
```

## 特征架构（怎么把"动作序列"喂进 Transformer）

排序模型的输入序列拼成：

```
位置 0:                      User token (用户特征 + 可选 IP)
位置 1 ... S:                History tokens (S=127 个，最近的用户互动)
位置 S+1 ... S+C:            Candidate tokens (C=64 个待打分的候选)
```

每个 token 是多种 embedding 的**拼接 + 线性投影**（见 `recsys_model.py:block_*_reduce`）：

| Token 类型 | 组成成分（concat 后投影到 D 维） |
|---|---|
| **User** | 多哈希 user embedding（求和）+ 可选 IP 多哈希 embedding |
| **History** | post embedding · author embedding · 动作 embedding · 产品面 embedding · 停留时长 embedding |
| **Candidate** | post embedding · author embedding · 产品面 embedding · 帖龄桶 embedding |

特别值得注意的几个设计：

1. **动作不是 one-hot**：是 `[B, S, num_actions]` 的多热，通过 `actions_signed = (2*actions-1)` 变成 ±1，再投影到 D 维。一个 history token 可以同时携带"喜欢 + 转推 + 停留 8 秒"。
2. **帖龄分桶**：`compute_post_age_bucket` 把 `impr_ts - post_creation_ts` 按 `granularity_mins` 切桶，桶 0 留给"缺失/异常"，最后一桶留给 overflow。
3. **停留时长是连续值**：单独走 `_project_continuous_value_to_embedding`（1 → hidden → D 的两层 MLP）。归一化方式是 `normalize_continuous_value`（线性 clip 或 log1p）。
4. **RoPE 右锚定**：可选启用 `right_anchored_rope`，让"最新一条 history"永远拿到固定位置编码。这样不同用户的 history 长度不同，但"最新动作的相对位置"对模型来说一致 —— 训练 / 服务长度不一致时仍然稳。

## 输出头

排序模型的输出层在 `recsys_model.py:__call__`：

```python
out_embeddings = layer_norm(out_embeddings)
candidate_embeddings = out_embeddings[:, candidate_start_offset:, :]

# 离散头：num_actions 个 logit
logits = jnp.dot(candidate_embeddings, unembeddings)   # [B, C, num_actions]

# 连续头：num_continuous_actions 个 sigmoid（如停留时长）
continuous_preds = sigmoid(dot(candidate_embeddings, continuous_mat))
```

两个头并行预测，离散动作走 BCE / 多任务，连续动作走 MAE 或 Tweedie（`ContinuousActionConfig.loss_type`）。

## 多哈希 Embedding（容纳无限 ID）

ID 类特征（user / item / author）采用 **Hash-Based Embedding**，由 `runners.py` + `run_pipeline.py:_hash_ids` 实现：

```python
raw = (id * scale_j + bias_j) % modulus
bucket = (raw % (vocab_size - 1)) + 1   # 0 留给 padding
```

- 每个实体用 `num_*_hashes`（默认 2）个独立 hash 函数索引到 `vocab_size`（默认 1,000,000）的 embedding 表，结果**拼接后线性投影**。
- ID 为 0 永远 hash 到 bucket 0 —— 模型把 bucket 0 当 padding 屏蔽掉。
- 三个表（user / item / author）合并到一张"统一 embedding 表"里，每张占 `vocab + 65 padding offset` 行。`build_unified_emb_table` 负责按 offset 拼成一张大表，供 jax 在线上一次 gather。

> 好处：词表"无限大"也只占固定显存；新用户 / 新帖天然有 embedding（hash 到某桶即可）；不需要维护 ID → row 的映射服务。代价：不同 ID 会偶尔哈希到同一行，但 2 个独立 hash 函数让冲突概率指数级下降。

## 服务侧（Rust 这边怎么用）

排序模型不在 home-mixer 进程里跑，而是通过 **`PhoenixPredictionClient`**（gRPC）远程调用：

```rust
// home-mixer/scorers/phoenix_scorer.rs（节选）
struct PhoenixScorer {
    phoenix_client: Arc<dyn PhoenixPredictionClient>,      // 主路径
    egress_client: Arc<dyn PhoenixPredictionClient>,       // 实验 / 对照路径（egress sidecar）
}
```

召回侧类似，走 `PhoenixRetrievalClient`：

```rust
ProdPhoenixRetrievalClient::new(Some((
    PhoenixRetrievalCluster::Experiment1Fou,
    PhoenixRetrievalCluster::Experiment1Lap7,
)))
```

> 两个 cluster 同时拿，体现的是"线上几乎永远有 ≥2 个版本在做 A/B"。`PhoenixExperimentsSideEffect` 在 home-mixer 侧把对照组结果异步写 Kafka 做离线比对。

## Mini 模型规格

随仓库一起发布的预训练模型是缩水版，详见 [§5.4 Mini Phoenix 模型规格](phoenix-mini-specs.md)。
