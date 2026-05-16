# 5.2 排序 Transformer 与候选隔离

**代码：** [`phoenix/recsys_model.py`](https://github.com/yingwang/x-algorithm/blob/main/phoenix/recsys_model.py) 与 [`phoenix/grok.py`](https://github.com/yingwang/x-algorithm/blob/main/phoenix/grok.py)

排序阶段在整套推荐系统中承担着最关键的相关性判断。它接收的是召回阶段截出来的几百到几千条候选，需要对每一条候选预测出用户可能采取的多种动作的概率，并把这些概率按照预设权重组合成一个最终的排序分。这一阶段所使用的模型，是从 Grok-1 移植过来的 Transformer 主干，再配合上一种非常关键的注意力掩码设计，也就是本节标题中所说的"候选隔离"。

## 输入序列长这样

排序 Transformer 的输入并不是普通的文本序列，而是由用户 token、历史 token 与候选 token 三段共同拼接出来的复合序列。下面这张图直观地呈现了拼接方式。

```
位置:  0           1            ...   S            S+1            ...    S+C
      ┌─────┐ ┌─────┬─────┬─────┬─────┐ ┌─────┬─────┬─────┬─────┬─────┐
      │User │ │Hist1│Hist2│ ... │HistS│ │Cand1│Cand2│Cand3│ ... │CandC│
      └─────┘ └─────┴─────┴─────┴─────┘ └─────┴─────┴─────┴─────┴─────┘
       1 个   S 个（默认 127）            C 个（默认 64）
```

`build_inputs` 函数把这三段拼接成一条完整的序列，整段一并送入 Transformer。模型对这三段如何相互"看见"的控制，则通过精心设计的注意力掩码来实现。

## 每个 token 的内容

每一个 token 并不是单一来源的 embedding，而是若干个 embedding 经过拼接再做一次线性投影压缩到 D 维所形成的向量。下面这张表给出了三段 token 各自的组成成分。

| 段 | 由谁组成（拼接后线性投影到 D 维） |
|---|---|
| User token | 多哈希 user embedding，加上可选的 IP embedding |
| History token | 帖子 hash、作者 hash、动作 embedding、产品面 embedding、停留时长 embedding |
| Candidate token | 帖子 hash、作者 hash、产品面 embedding、帖龄桶 embedding |

具体的拼接实现可以在 `recsys_model.py` 中的 `block_user_reduce`、`block_history_reduce` 与 `block_candidate_reduce` 三个函数里找到。

值得特别强调的一点是：history token 携带着"用户对这一条历史帖做了什么"这样的动作信号，因此多动作信息与停留时长都包含在 history 内部。Candidate token 则只包含"帖子自身"的信息，并不携带任何动作，因为对这一条候选用户尚未发生任何互动。

## 注意力掩码：Candidate Isolation

整套设计的核心创新就藏在这一份注意力掩码里。掩码的构造逻辑写在 `phoenix/grok.py` 文件中的 `make_recsys_attn_mask` 函数。

```python
def make_recsys_attn_mask(seq_len, candidate_start_offset, dtype):
    causal_mask = jnp.tril(jnp.ones((1, 1, seq_len, seq_len), dtype=dtype))
    # 把"候选块对候选块"的整片区域置零
    attn_mask = causal_mask.at[:, :, candidate_start_offset:, candidate_start_offset:].set(0)
    # 再把对角线置一（让每一个候选能看到自己）
    candidate_indices = jnp.arange(candidate_start_offset, seq_len)
    attn_mask = attn_mask.at[:, :, candidate_indices, candidate_indices].set(1)
    return attn_mask
```

这一段代码所产生的掩码可以可视化成下面这张表。

```
        Keys → User | History (S) | Candidates (C)
        ────────────────────────────────────────────
Queries:
  User      ✓  | ✓ ✓ … ✓     | ✗ ✗ ✗ … ✗
  History   ✓  | ✓ ✓ … ✓     | ✗ ✗ ✗ … ✗      ← History 之间是 causal 关系（下三角）
  Cand 1    ✓  | ✓ ✓ … ✓     | ✓ ✗ ✗ … ✗      ← 只能看到自己加上 User 与 History
  Cand 2    ✓  | ✓ ✓ … ✓     | ✗ ✓ ✗ … ✗
  ...
  Cand C    ✓  | ✓ ✓ … ✓     | ✗ ✗ ✗ … ✓
```

把上述掩码翻译成精确语义可以这样表述：

第一，User 段对自己可见，对其它段不可见。第二，History 段之间采用因果掩码（即标准的下三角结构），最早的 history 看不到后面的 history，最新的 history 可以看到全部前面的 history，并且可以看到 User 段。第三，Candidate 段对 User 段与 History 段全部可见。第四，Candidate 段之间彼此不可见，只能看到对角线上自己这一个位置。

## 为什么要候选隔离

把候选之间的注意力切断这一设计，所带来的工程价值非常显著，可以从三个角度来看。

### 第一项收益：得分稳定

经过候选隔离之后，候选 A 的分数只取决于 User、History 与候选 A 自身这三段内容，与同批中其它候选完全无关。在 batch 里换走候选 B、加入候选 D，候选 A 的分数都不会发生任何变化。这一性质让下面三种工程做法天然合理。

第一种做法是请求级缓存。形如 (user_hash, post_id) → score 的键值对可以直接缓存使用，不会出现"批组合"所带来的爆炸性键空间。第二种做法是分批打分。当候选数量过多时可以分多个 batch 完成，所得分数完全一致，不会像 listwise 模型那样受到 batch 切分方式的影响。第三种做法是 A/B 实验的可比性。实验组与对照组之间只更换模型权重，不更换批组合，得到的分数可以直接比较；如果模型内部存在候选之间的相互影响，对照差会被"两组的 batch 组成不同"这一因素污染。

### 第二项收益：天然支持并行与流水线

经过候选隔离之后，每一个候选位置上的注意力计算都只依赖前两段（User 加 History）的 KV cache 与候选自身。这意味着两件重要的事情。一是可以预先计算出 (User, History) 的 KV cache，针对来自不同请求的候选进行复用。二是候选之间没有任何依赖，可以把不同的候选分配到不同 GPU 上并行计算。这两点合起来让排序服务能够在工程上获得显著的吞吐与延迟优势。

### 第三项收益：可解释

如果某一条候选的分数发生了明显变化，原因只可能落在两件事情上：一是用户的 history 发生了变化（出现了新的互动）；二是候选自身的特征发生了变化（例如帖龄从 0 到 60 分钟的桶跳到了 60 到 120 分钟的桶）。绝不可能是"恰好和别的高分候选凑在一批"这样难以排查的原因导致的偏移。这种可解释性让线上问题的定位变得简单很多。

## 为什么 candidate 仍然要能看见自己

到这里有一个常见的疑问需要专门解释一下：如果完全把 candidate 行整行清零（让它什么都看不见），那么 candidate 位置经过 attention 之后将会失去自身的信号。然而候选自身的 embedding 才是判断这一条候选是否合适的核心特征，不可能让模型把它丢掉。

保留对角线上的 1 这一选择，本质上等价于"让 self-attention 退化为恒等映射（输出等于自身的 value），同时再额外吸收来自 User 段与 History 段的上下文"。这样一来，模型既知道"这一条候选是什么"，又知道"用户最近喜欢什么"，但是不知道"同一批里还有谁"。这恰恰就是我们想要的"用户与候选的交叉特征"形态。

## 输出与读取

Transformer 计算结束之后，紧接着是输出层的处理。

```python
out_embeddings = layer_norm(out_embeddings)
candidate_embeddings = out_embeddings[:, candidate_start_offset:, :]   # 形状 [B, C, D]
logits = candidate_embeddings @ unembedding   # 形状 [B, C, num_actions]
continuous_preds = sigmoid(candidate_embeddings @ continuous_head)   # 形状 [B, C, num_continuous]
```

输出层只取候选段的输出送入 unembedding，User 段与 History 段的输出在排序阶段并不参与最终的预测（它们的输出在召回阶段会通过 mean-pool 形成 user_repr）。

`unembedding` 是一个 `[D, num_actions]` 的矩阵，等价于"每一个动作对应一个分类头"。`continuous_head` 与之类似，但它的输出经过 sigmoid 之后变成连续值（例如归一化后的停留时长）。

## 训练目标（从配置反推）

仓库并未开源训练代码，但是从 `ContinuousActionConfig`、`mask_neg_feedback_on_negatives` 以及多动作 logits 这几处配置可以大致反推出训练目标的形式。

离散动作头采用的是每一个动作独立 BCE 损失的多任务组合，每一种动作可以有不同的权重。连续动作头采用 MAE 损失或者 Tweedie 损失，其中 Tweedie 的 power 参数 1.5 是开源版本的默认值。负反馈相关的部分则有一项特殊处理：`mask_neg_feedback_on_negatives` 让模型只在确认出现负反馈时计算负反馈 loss，避免把"用户根本没有标签"误当作"用户讨厌"这一种语义上完全不同的信号。

## 完整一次排序请求

把所有阶段串起来回顾一下，一次完整的排序请求大致经历下面这些步骤。

```
1. 用户 user_id 与 scoring_sequence（共 S 条）           ── Home Mixer 通过 query hydrator 准备好
2. 候选 post_id（共 C 条，来自召回阶段）                  ── Home Mixer 通过 source 与 filter 收齐
3. 全部 ID 经过哈希，从统一 embedding 表里 gather，形成 RecsysBatch 与 RecsysEmbeddings
4. 调用 PhoenixModel(batch, embeddings)，得到 logits [1, C, num_actions] 与 continuous [1, C, num_cont]
5. 通过 sigmoid 把 logits 转成概率，写回 Rust 侧的 PhoenixScores 结构
6. Home Mixer 中的 WeightedScorer 把多动作概率按权重聚合成单一分数
```

到这一步 Phoenix 模型的工作就完成了。接下来的[多动作预测与加权打分](phoenix-multiaction.md)一节会详细讲解上面第六步的具体实现。
