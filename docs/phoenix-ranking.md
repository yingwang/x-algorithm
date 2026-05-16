# 5.2 排序 Transformer 与候选隔离

**代码：** [`phoenix/recsys_model.py`](https://github.com/yingwang/x-algorithm/blob/main/phoenix/recsys_model.py) + [`phoenix/grok.py`](https://github.com/yingwang/x-algorithm/blob/main/phoenix/grok.py)

## 输入序列长这样

```
位置:  0           1            ...   S            S+1            ...    S+C
      ┌─────┐ ┌─────┬─────┬─────┬─────┐ ┌─────┬─────┬─────┬─────┬─────┐
      │User │ │Hist1│Hist2│ ... │HistS│ │Cand1│Cand2│Cand3│ ... │CandC│
      └─────┘ └─────┴─────┴─────┴─────┘ └─────┴─────┴─────┴─────┴─────┘
       1 个   S 个（默认 127）           C 个（默认 64）
```

`build_inputs` 把三段拼成一条序列，**整体**送入 Transformer，配合特殊掩码控制谁能看见谁。

## 每个 token 的内容

| 段 | 由谁组成（concat 后线性投影到 D 维） |
|---|---|
| **User token** | 多哈希 user embedding · (可选) IP embedding |
| **History token** | post hash · author hash · 动作 embedding · 产品面 embedding · dwell 时长 embedding |
| **Candidate token** | post hash · author hash · 产品面 embedding · 帖龄桶 embedding |

具体见 `recsys_model.py:block_user_reduce` / `block_history_reduce` / `block_candidate_reduce`。

> **细节**：history token 携带"用户对那条帖做了什么"，所以多动作和 dwell 都在 history 里；candidate token 只有"帖子自身"，不带动作（因为还没发生）。

## 注意力掩码：Candidate Isolation

代码在 `phoenix/grok.py:make_recsys_attn_mask`：

```python
def make_recsys_attn_mask(seq_len, candidate_start_offset, dtype):
    causal_mask = jnp.tril(jnp.ones((1, 1, seq_len, seq_len), dtype=dtype))
    # 把"候选块对候选块"的整片置 0
    attn_mask = causal_mask.at[:, :, candidate_start_offset:, candidate_start_offset:].set(0)
    # 再把对角线置 1（让每个候选能看自己）
    candidate_indices = jnp.arange(candidate_start_offset, seq_len)
    attn_mask = attn_mask.at[:, :, candidate_indices, candidate_indices].set(1)
    return attn_mask
```

写成可视化：

```
        Keys → User | History (S) | Candidates (C)
        ────────────────────────────────────────────
Queries:
  User      ✓  | ✓ ✓ … ✓     | ✗ ✗ ✗ … ✗
  History   ✓  | ✓ ✓ … ✓     | ✗ ✗ ✗ … ✗      ← History 之间是 causal（下三角）
  Cand 1    ✓  | ✓ ✓ … ✓     | ✓ ✗ ✗ … ✗      ← 只能看自己 + User/History
  Cand 2    ✓  | ✓ ✓ … ✓     | ✗ ✓ ✗ … ✗
  ...
  Cand C    ✓  | ✓ ✓ … ✓     | ✗ ✗ ✗ … ✓
```

精确语义：

1. **User → User**：✓（自己）
2. **History → User/History**：causal（下三角，最早 history 看不到后面的 history；最新 history 能看全部前面的）
3. **Candidate → User/History**：✓ 全部
4. **Candidate → Candidate**：仅对角线（只看自己）

## 为什么要候选隔离

### 收益 1：得分稳定

候选 A 的分数**只**取决于 User + History 与候选 A 自身。在 batch 里换走候选 B、加入候选 D，都不会改变 A 的分。这让以下行为天然合理：

- **请求级缓存**：(user_hash, post_id) → score 可以直接缓存，没有"批组合"的爆炸性键空间。
- **分批打分**：候选太多时分多个 batch 打，分数完全一致，不像 listwise 模型会受 batch 切分影响。
- **A/B 实验可比**：实验组与对照组只换权重不换批，scores 直接可比；如果有候选间交互，对照差会被"batch 组成不同"污染。

### 收益 2：并行 / 可流水线

每个候选位置的注意力都只依赖前两段（User+History）的 KV cache 与自己。线上服务可以：

- 预计算 (User, History) 的 KV cache，对来自不同请求的候选**复用**。
- 候选间无依赖 → 不同 GPU 并行打不同候选。

### 收益 3：可解释

如果候选 A 的分突然变化，原因只可能是：

- 用户 history 变了（新的互动进来）
- 候选 A 自身特征变了（如帖龄从 0~60 分桶跳到 60~120 分桶）

而不可能是"刚好和别的高分候选一批"导致的偏移。这让排查问题简单很多。

## 为什么 candidate 还能看自己（对角线 = 1）

如果完全把 candidate 行整行清零（看不见任何东西），那么 candidate 位置经过 attention 后会丢掉自身信号 —— 但它本身的 embedding 才是候选的核心特征。

保留对角线 = 1 等于："self-attention 退化为恒等（输出 = 自身 value）" + "再额外摄入 User/History 的上下文"。这样模型既知道"这条候选是什么"，又知道"用户最近喜欢什么"，但不知道"同批里还有谁" —— 正是我们想要的"用户-候选交叉特征"形态。

## 输出与读取

Transformer 结束后：

```python
out_embeddings = layer_norm(out_embeddings)
candidate_embeddings = out_embeddings[:, candidate_start_offset:, :]   # [B, C, D]
logits = candidate_embeddings @ unembedding   # [B, C, num_actions]
continuous_preds = sigmoid(candidate_embeddings @ continuous_head)   # [B, C, num_continuous]
```

**只**取候选段的输出送进 unembedding —— User 和 History 段的输出在排序时没用（在召回里用 mean-pool 形成 user_repr）。

`unembedding` 是个 `[D, num_actions]` 的矩阵，相当于"每个动作一个分类头"。`continuous_head` 类似，但走 sigmoid 输出连续值（比如归一化后的 dwell 时长）。

## 训练目标（隐含）

仓库无训练代码，但从 `ContinuousActionConfig` / `mask_neg_feedback_on_negatives` / 多动作 logits 能反推：

- **离散动作头**：每个动作独立 BCE（多任务），权重可调。
- **连续动作头**：MAE 或 Tweedie（power 1.5 是开源版默认值）。
- **负反馈掩码**：`mask_neg_feedback_on_negatives` 让"模型只在确认负反馈出现时计算负反馈 loss"，避免把"没标签"误当成"用户讨厌"。

## 完整一次排序请求

```
1. 用户 user_id + scoring_sequence (S 条)            ── home-mixer 通过 query hydrator 准备好
2. 候选 post_id × C （来自召回阶段）                  ── home-mixer 通过 source + filter 收齐
3. 全部哈希 → 在统一 embedding 表里 gather → RecsysBatch + RecsysEmbeddings
4. PhoenixModel(batch, embeddings) → logits [1, C, num_actions] + continuous [1, C, num_cont]
5. 用 sigmoid 把 logits 转成概率，写回 Rust 侧 PhoenixScores
6. home-mixer 的 WeightedScorer 把多动作概率按权重聚合成单一分
```

到此 Phoenix 模型的活就干完了，下面 [§5.3 多动作预测与加权打分](phoenix-multiaction.md) 详解 6 步里的最后一步。
