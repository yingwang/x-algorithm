# 3.3 Phoenix · 模型层（Python / JAX）

**目录：** [`phoenix/`](https://github.com/yingwang/x-algorithm/tree/main/phoenix) — 详细子文档参见仓库内 [`phoenix/README.md`](https://github.com/yingwang/x-algorithm/blob/main/phoenix/README.md)。

Phoenix 是整套推荐系统的"大脑"。Home Mixer 那一层只负责把数据准备好、把流程串起来，真正承担相关性判断的工作都集中在 Phoenix 模型里。本节先给出一个完整的总览，更深入的细节请参见后面的 [Phoenix 模型深入](phoenix-retrieval.md) 系列章节。

## 由两个模型组成

Phoenix 内部其实包含两个相对独立、却又共享同一份主干的模型，分别负责召回与排序两个阶段。

| 模型 | 文件 | 职责 | 输出 |
|---|---|---|---|
| Retrieval（双塔） | `recsys_retrieval_model.py` | 从全量语料里按相似度召回 Top-K | 候选 ID 列表加上对应的相似度分数 |
| Ranking（带候选隔离掩码的 Transformer） | `recsys_model.py` | 对每一条候选预测多种用户动作的发生概率 | 形状为 `[B, num_candidates, num_actions]` 的 logits 以及一组连续值预测 |

两者共用同一份 Grok 风格的 Transformer 主干，定义在 `grok.py` 文件中。这一份主干是从 [xAI 公开的 Grok-1](https://github.com/xai-org/grok-1) 移植过来的，只在头部、注意力掩码以及损失函数三处做了与推荐场景相关的改造。其余架构层面的实现，包括分块多头注意力、混合专家路由、RoPE 位置编码等，都与原始 Grok 一致。

## 关键文件

整个 `phoenix/` 目录下文件并不多，下面这份清单把每一个文件的角色都标了出来。

```
phoenix/
  grok.py                       Transformer 主干，内含特殊的候选隔离掩码工厂
  recsys_model.py               排序模型，定义 PhoenixModel 与 RecsysBatch、RecsysEmbeddings
  recsys_retrieval_model.py     召回模型，定义双塔结构（用户塔复用 Transformer，物品塔为两层 MLP）
  runners.py                    加载 checkpoint、embedding 表与配置的工具
  run_pipeline.py               端到端入口，依次跑召回与排序，本次新增（合并自旧的 run_retrieval.py 与 run_ranker.py）
  run_retrieval.py              旧版：只跑召回
  run_ranker.py                 旧版：只跑排序
  test_recsys_model.py          排序模型单元测试
  test_recsys_retrieval_model.py 召回模型单元测试
  pyproject.toml                uv 包管理器的依赖清单，包括 jax、jaxlib、dm-haiku、numpy
  artifacts/oss-phoenix-artifacts.zip   预训练好的 Mini 模型与体育语料库，约 3 GB，通过 Git LFS 管理
```

## 特征架构：怎么把"动作序列"喂进 Transformer

排序模型的输入是一段精心拼装出来的序列。它不再像传统推荐模型那样依赖各种各样的手工特征，而是直接把用户的动作历史以 token 序列的形式送进 Transformer。

```
位置 0:                      User token（用户特征加上可选的 IP）
位置 1 至 S:                History tokens（S 取 127，对应用户最近的 127 次互动）
位置 S+1 至 S+C:            Candidate tokens（C 取 64，对应需要打分的候选）
```

每一个 token 都不是单一来源的 embedding，而是多种 embedding 通过拼接再线性投影的方式压缩到 D 维空间里。具体的拼接方式定义在 `recsys_model.py` 的 `block_*_reduce` 函数中。

| Token 类型 | 组成成分（拼接后投影到 D 维） |
|---|---|
| User | 多哈希用户 embedding（按哈希求和）加上可选的 IP 多哈希 embedding |
| History | 帖子 embedding、作者 embedding、动作 embedding、产品面 embedding、停留时长 embedding |
| Candidate | 帖子 embedding、作者 embedding、产品面 embedding、帖龄桶 embedding |

特征架构里有几处设计特别值得展开来谈一谈。

第一处是动作 embedding。这里的动作并不是 one-hot 编码，而是一种形状为 `[B, S, num_actions]` 的多热（multi-hot）编码。代码里把 `actions` 转换为 `actions_signed = (2*actions - 1)` 这样一个取值范围为正负一的张量，再投影到 D 维。这种做法的好处在于：一个 history token 可以同时携带多种动作信号，比如同时表示"喜欢加转推加停留八秒"。换句话说，模型看到的是一个用户在某一条帖子上所做出的整组反应，而不是被强行拆成几条独立动作。

第二处是帖龄分桶。代码中的 `compute_post_age_bucket` 函数把 `impr_ts - post_creation_ts`（也就是曝光时刻与帖子创建时刻之差）按照参数 `granularity_mins` 切分成若干个桶。其中第 0 号桶专门留给"缺失或异常"的情况，最后一个桶则用来承载所有超出预期范围的 overflow。这种处理方式既覆盖了正常时序情况，也兼容了脏数据。

第三处是停留时长。停留时长在所有信号中相对特殊，因为它本质上是一个连续值，不能像离散动作那样直接做 one-hot。代码中专门写了一个 `_project_continuous_value_to_embedding` 函数：先把单一数值经过一层 1 维到 hidden 维的 MLP，再经过一层 hidden 维到 D 维的 MLP。归一化部分则由 `normalize_continuous_value` 完成，可以选择线性 clip 或者 log1p 两种变换。

第四处是 RoPE 的右锚定。RoPE 是一种相对位置编码，模型可以通过参数 `right_anchored_rope` 来开启右锚定模式。开启之后，序列里"最新的一条 history"永远会拿到一个固定的位置编码。这样一来，不同用户的 history 长度即便不同，"最新动作"的相对位置在模型眼中始终是一致的。这一设计让模型在训练时与服务时的序列长度可以不一致而仍然保持稳定，是工程上的一个关键细节。

## 输出头

排序模型在最后的输出层做了一件相当聪明的事情：它同时输出两组并行的预测结果，一组对应离散动作，一组对应连续值动作。

```python
out_embeddings = layer_norm(out_embeddings)
candidate_embeddings = out_embeddings[:, candidate_start_offset:, :]

# 离散头：输出 num_actions 个 logit
logits = jnp.dot(candidate_embeddings, unembeddings)   # 形状 [B, C, num_actions]

# 连续头：输出 num_continuous_actions 个 sigmoid，例如停留时长
continuous_preds = sigmoid(dot(candidate_embeddings, continuous_mat))
```

两个头并行地完成预测，但在损失函数上各走各的路径。离散动作走的是 BCE 多任务损失；连续动作则可以选择 MAE 损失或者 Tweedie 损失，具体配置由 `ContinuousActionConfig.loss_type` 决定。这种"一个模型同时输出多种信号"的设计，让所有动作共享主干学到的语义信息，又能让每一种动作根据自己的特点采用合适的损失函数。

## 多哈希 Embedding：容纳无限 ID

推荐系统里 ID 类特征（用户、物品、作者）天然有一个棘手的问题：词表规模随着平台增长而无限膨胀。传统的"每一个 ID 一行 embedding"做法很快就会让显存爆炸，新出现的 ID 还会面临冷启动困境。Phoenix 采用了一种叫做 Hash-Based Embedding 的方案来规避这一问题，相关实现集中在 `runners.py` 与 `run_pipeline.py:_hash_ids` 中。

```python
raw = (id * scale_j + bias_j) % modulus
bucket = (raw % (vocab_size - 1)) + 1   # 第 0 号桶留给 padding
```

这套方案的核心想法可以这样描述。每一个实体并不固定地占用一行 embedding，而是经过 `num_*_hashes`（默认值为 2）个相互独立的哈希函数，分别映射到大小为 `vocab_size`（默认值为 1,000,000）的 embedding 表中的某些桶上，再把这些桶对应的向量拼接起来线性投影。ID 为 0 的实体永远会被映射到第 0 号桶，模型把第 0 号桶视为 padding 直接屏蔽。三张表（用户、物品、作者）最终会被合并到一张"统一 embedding 表"里，每一张原表占据 `vocab + 65 padding offset` 行，由 `build_unified_emb_table` 按照预先确定的 offset 拼接起来，供 JAX 在线上一次性 gather 出所需的所有向量。

这种方案最显著的好处在于：词表的"逻辑大小"可以无限大，但实际占用的显存是一个固定值；任何新出现的用户或者帖子天然就有 embedding 可用（哈希到某些桶上即可），不需要冷启动；同时也不需要单独维护一个"ID 到行号"的映射服务。代价是不同 ID 偶尔会被哈希到同一个桶，但是采用两个独立的哈希函数同时索引时，两次都恰好碰撞的概率会指数级降低，工程上完全可以接受。

## 服务侧：Rust 那边怎么用

排序模型并不在 Home Mixer 进程内部运行，而是被部署在另外一组进程中，通过 gRPC 接口提供服务。Home Mixer 那一侧通过 `PhoenixPredictionClient` 来发起远程调用。

```rust
// home-mixer/scorers/phoenix_scorer.rs（节选）
struct PhoenixScorer {
    phoenix_client: Arc<dyn PhoenixPredictionClient>,      // 主路径
    egress_client: Arc<dyn PhoenixPredictionClient>,       // 实验对照路径（走 egress sidecar）
}
```

召回侧的访问方式与排序侧基本一致，只是客户端换成了 `PhoenixRetrievalClient`。下面这一段代码体现了召回侧的实例化方式。

```rust
ProdPhoenixRetrievalClient::new(Some((
    PhoenixRetrievalCluster::Experiment1Fou,
    PhoenixRetrievalCluster::Experiment1Lap7,
)))
```

值得注意的是，召回侧的客户端同时持有两个 cluster 的句柄。这种设计反映了一个真实存在的工程现实：线上几乎永远有两个或更多版本的 Phoenix 同时在做 A/B 测试。`PhoenixExperimentsSideEffect` 那一段副作用代码会把对照组的结果异步写入 Kafka，供离线分析做版本比对。

## Mini 模型规格

随仓库一起发布的预训练模型并不是线上服务所使用的那一份完整模型，而是一份精心缩减过的"Mini 模型"。它的规格、训练数据规模与本地运行的注意事项请参见[第 5.4 节 Mini Phoenix 模型规格](phoenix-mini-specs.md)。
