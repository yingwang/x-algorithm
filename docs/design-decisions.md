# 7. 关键设计决策（含原因）

## 1) 零手工特征

**做法：** 让 Grok Transformer 从用户**动作序列**自学相关性，砍掉了人工特征工程的整条链路。

**原因：**

| 传统做法的代价 | 本仓库的回避方式 |
|---|---|
| 维护特征仓库（线上线下两套实现） | 不存在 —— 全靠模型 |
| 训练 / 服务一致性问题 | 输入只剩"动作序列 + ID 哈希"，几乎没分歧空间 |
| 加新特征要走完整发布流程 | 加新动作类型只需扩 `num_actions` 重训 |
| 特征工程师与算法工程师隔阂 | 一个团队对模型负责 |

代价：单个特征的"可解释性"变差（模型黑盒）。但仓库用**多动作输出**部分补偿了这一点 —— 至少能看到模型预测的"喜欢概率 / 转推概率 / 讨厌概率"分别是多少，归因仍可能。

## 2) 候选隔离的注意力掩码

**做法：** 排序 Transformer 里候选互相不可见（attention mask 对角线 + 看 user/history）。

**收益：**

- **得分稳定**：A 的分不依赖批里有没有 B、C、D。
- **可缓存**：(user, post) → score 可以直接缓存，没有"批组合"键空间爆炸。
- **可并行**：候选间无依赖，可流水线化。
- **可解释**：分变化的归因空间只剩"用户上下文"和"候选自身"。

**代价：** 失去了 listwise 信号（"候选间相对优劣"）。但 X 的产品要的是绝对相关性，不是 listwise rank。所以这个 trade-off 划算。

详见 [§5.2 排序 Transformer 与候选隔离](phoenix-ranking.md)。

## 3) Hash-Based Embedding

**做法：** ID（user / item / author）用多哈希查表替代"每 ID 一行 embedding"。

**收益：**

- **无 OOV**：新用户 / 新帖天然有 embedding —— 它们 hash 到某些桶，桶里已有训练好的向量。
- **固定显存**：词表"无限大"，但 embedding 表大小固定（默认 1M 行 × 128 维 × 3 类 ≈ 1.5 GB）。
- **不用维护 ID → row 映射服务**：哈希函数本身就是映射。
- **冲突指数级下降**：用 2 个独立 hash 函数 + 投影，单一冲突的概率从 1/V 变成 ≈ 1/V²。

**代价：** 冲突不可避免（理论上）。但实际中，2 个 hash + 1M 桶下，意外"两个完全不同的 ID 在两个表上都同桶"的概率小到可忽略。

更多见 [§5.4 Mini 模型规格](phoenix-mini-specs.md) 与 `phoenix/run_pipeline.py:_hash_ids`。

## 4) 多动作预测

**做法：** 排序头一次性输出 19 个动作的概率 + 8 个连续值，下游加权聚合。

**收益：**

- **真实偏好建模**：用户喜欢一条帖的原因可能是想点赞，也可能是想转推，也可能是觉得视频好看；单一"相关性"丢掉了这个层次。
- **负权显式表达"讨厌"**：not_interested / block / mute / report 直接用负权拉低，模型不需要"假装喜欢就好"。
- **业务目标可调**：想多曝光 reshares？提 RETWEET_WEIGHT 一行配置。想压短视频？降 VQV_WEIGHT。
- **A/B 解释力强**：实验对比时直接看哪个动作概率变了。

详见 [§5.3 多动作预测与加权打分](phoenix-multiaction.md)。

## 5) 可组合的 Pipeline 框架

**做法：** `xai_candidate_pipeline` 抽象出 Source / Hydrator / Filter / Scorer / Selector / SideEffect 7 类 trait，业务实现 trait，框架管并发。

**收益：**

- **加新候选源不动调度代码**：实现 `Source` + 加到 `vec![]` 即可。
- **单组件失败不传染**：框架对 Err 静默忽略，主链路继续。
- **监控开箱即用**：阶段级 latency / count / filter_rate 自动 record。
- **A/B 与 feature switch 一致接口**：每个组件有 `enable(&query)`，由 `Decider` / `Params` 决定。
- **副作用不阻塞主路径**：`tokio::spawn`。

**代价：** trait 抽象有一定学习曲线，新加组件要符合 trait 形状。

详见 [§3.4 Candidate Pipeline](components-pipeline.md)。

## 6) 服务层用 Rust 重写

**做法：** 编排 / 候选库 / 框架全 Rust，模型层 Python。

**原因：**

- **内存安全 + 无 GC 停顿**：Home Mixer 的 P99 延迟要求严格。
- **零成本异步**：Tokio + tonic 的吞吐 / 延迟 / 资源占比远优于 JVM 系。
- **DashMap / 无锁并发**：Thunder 的内存索引能在多核上线性 scale。
- **数据结构控制力强**：TinyPost / VecDeque / Bloom Filter 这些低层结构在 Rust 里能精确控制内存布局。

模型保留 Python 是因为：

- JAX / Haiku 训练栈太成熟，Python 是事实标准。
- 模型在独立服务里跑（`PhoenixPredictionClient` 远程调用），Rust 这边不用做 Python interop。

## 7) 多源召回 + 单一 Transformer 排序

**做法：** 召回端有 6 路（Thunder / Phoenix 多种 / TweetMixer / Cache），但排序只一个模型。

**原因：**

- **召回求覆盖**：不同源的优势互补，单源失败能兜底。
- **排序求一致**：所有候选都被同一个模型按同一标准打分，不会出现"in-network 用模型 A、out-of-network 用模型 B 导致不可比"。

这是经典推荐系统的"召回 vs 精排"分工，本仓库做到了相对纯粹。

## 8) 前过滤 + 后过滤分两段

**做法：** 14 个 filter 在打分前，3 个 VF/Conversation filter 在选择后。

**原因：** 见 [§6 过滤规则](filters.md) 末段 —— 平衡"打分算力"和"VF 服务延迟"。

## 9) 副作用全部异步

**做法：** Kafka 写入 / Redis 缓存 / 实验对照 / 已展示历史更新 全 `tokio::spawn`。

**原因：** 用户感知的"出 Feed 速度" 与 "副作用是否完成" 解耦。即使 Kafka 宕机，用户照样能看 Feed —— 只是后台分析会缺一段数据。

**代价：** Spawn 后失败没人接住（除了 metrics）。但这些 side effect 本身就是"尽力而为"性质，不影响线上正确性。

## 10) 把"广告插入"放到外层 Pipeline

**做法：** 核心 `PhoenixCandidatePipeline` 只管有机帖；外层 `ForYouCandidatePipeline` 把核心结果 + 广告 + WTF + Prompts 用 `BlenderSelector` 混排。

**原因：**

- **关注点分离**：广告 / 商业化策略可以独立改，不动核心排序逻辑。
- **品牌安全可控**：`SafeGapBlender` 在敏感内容附近不插广告（具体规则在 `home-mixer/ads/`）。
- **复用性**：`ScoredPostsService` 可供其它面（Search Home、Profile）复用，不带广告。

---

## 这些决策合起来想达成什么

如果用一句话概括：**让"相关性"成为一个可观测、可控、可演进的子系统，把所有非相关性的东西（广告、可见性、商业化）放到外面层叠**。

这与"端到端一个大模型搞定一切"是两条路 —— 这里选择的是"模型负责相关性，框架负责一切其他"。代价是框架本身复杂；收益是任何环节都能独立改、独立测、独立监控。
