# X For You 推荐算法 · 中文详解

> 本站点是对 [`yingwang/x-algorithm`](https://github.com/yingwang/x-algorithm) 仓库的详细中文解读，覆盖整体架构、各组件职责、流水线阶段、Phoenix 模型内部结构、注意力掩码（Candidate Isolation）原理、运行方式与设计权衡。
>
> 仓库本身公开了 X（前 Twitter）"For You" 信息流的核心推荐系统：把**关注内容（in-network）**与**通过机器学习检索到的非关注内容（out-of-network）**统一打分排序，最终输出给用户的个性化时间线。整套排序模型基于 **Grok 风格的 Transformer**，从 [xAI Grok-1 开源版本](https://github.com/xai-org/grok-1)移植后改造为推荐场景。

---

## 一句话概括

> **关注网内（Thunder）+ 关注网外（Phoenix 召回）→ Phoenix Transformer 打分 → Top-K 选择 → 可见性复滤 → 返回 Feed。**
>
> 与传统推荐系统不同：**几乎没有手工特征**，模型直接从用户**动作序列**学相关性；排序使用**候选隔离的注意力掩码**，让单条候选的得分只取决于"用户上下文"，从而可缓存、可并行、可解释。

---

## 仓库结构速览

| 目录 | 语言 | 角色 |
|---|---|---|
| [`home-mixer/`](https://github.com/yingwang/x-algorithm/tree/main/home-mixer) | Rust | 编排服务，把候选源→注水→过滤→打分→选择→副作用串起来 |
| [`thunder/`](https://github.com/yingwang/x-algorithm/tree/main/thunder) | Rust | 关注网内帖子的内存索引，亚毫秒级查询 |
| [`phoenix/`](https://github.com/yingwang/x-algorithm/tree/main/phoenix) | Python / JAX | 双塔召回 + Grok 风格排序 Transformer |
| [`candidate-pipeline/`](https://github.com/yingwang/x-algorithm/tree/main/candidate-pipeline) | Rust | 推荐流的可复用 trait 框架 |
| [`grox/`](https://github.com/yingwang/x-algorithm/tree/main/grox) | Python | 统一的内容理解服务（分类、嵌入、调度） |

---

## 阅读顺序建议

如果你想**最快理解整体**：依次读 [概览](overview.md) → [架构](architecture.md) → [流水线阶段](pipeline-stages.md) → [Phoenix 候选隔离](phoenix-ranking.md)。

如果你想**跑通 Demo**：直接跳到 [本地运行 Phoenix 推理](running.md)。

如果你想**对比官方版**：见 [与 the-algorithm 的关系](relation-to-official.md)。

---

> 左侧目录列出全部章节。每页底部有上一节 / 下一节翻页。顶栏右上角是搜索框。
