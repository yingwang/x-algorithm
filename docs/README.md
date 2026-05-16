# X For You 推荐算法 · 中文详解

> 本站点是对 [`yingwang/x-algorithm`](https://github.com/yingwang/x-algorithm) 仓库的详细中文解读，覆盖整体架构、各组件职责、流水线阶段、Phoenix 模型内部结构、注意力掩码（Candidate Isolation）原理、运行方式以及关键设计权衡。
>
> 仓库本身公开了 X（前 Twitter）"For You"信息流的核心推荐系统：把关注内容（in-network）与通过机器学习检索到的非关注内容（out-of-network）统一打分排序，最终输出给用户的个性化时间线。整套排序模型基于 Grok 风格的 Transformer，从 [xAI Grok-1 开源版本](https://github.com/xai-org/grok-1)移植到推荐场景之后再做改造而来。

---

## 一句话概括

> 关注网内（Thunder）加上关注网外（Phoenix 召回），统一交给 Phoenix Transformer 打分，再经过 Top-K 选择与可见性复滤，最终返回 Feed。
>
> 与传统推荐系统不同的地方在于：本仓库几乎没有手工特征工程，模型直接从用户的动作序列中学习相关性；排序使用带候选隔离掩码的注意力机制，让单条候选的得分只取决于"用户上下文"，从而可以缓存、可以并行、也更容易解释。

---

## 仓库结构速览

下面这张表列出了仓库内主要目录的语言与角色。

| 目录 | 语言 | 角色 |
|---|---|---|
| [`home-mixer/`](https://github.com/yingwang/x-algorithm/tree/main/home-mixer) | Rust | 编排服务，把候选源、注水、过滤、打分、选择、副作用整套链路串起来 |
| [`thunder/`](https://github.com/yingwang/x-algorithm/tree/main/thunder) | Rust | 关注网内帖子的内存索引，亚毫秒级查询 |
| [`phoenix/`](https://github.com/yingwang/x-algorithm/tree/main/phoenix) | Python 与 JAX | 双塔召回加上 Grok 风格的排序 Transformer |
| [`candidate-pipeline/`](https://github.com/yingwang/x-algorithm/tree/main/candidate-pipeline) | Rust | 推荐流的可复用 trait 框架 |
| [`grox/`](https://github.com/yingwang/x-algorithm/tree/main/grox) | Python | 统一的内容理解服务，包含分类、嵌入、调度等能力 |

---

## 阅读顺序建议

如果想要最快理解整体面貌，建议按下面这一顺序逐章阅读：先看[概览](overview.md)，再看[架构](architecture.md)，然后是[流水线阶段](pipeline-stages.md)，最后看 [Phoenix 候选隔离](phoenix-ranking.md)。

如果首要目标是跑通 Demo，可以直接跳到[本地运行 Phoenix 推理](running.md)这一章。

如果希望与官方版本做对比，可以参见[与 the-algorithm 的关系](relation-to-official.md)一章。

---

> 左侧目录列出了全部章节。每一页底部都有上一节与下一节的翻页链接。顶栏右上角是搜索框。
