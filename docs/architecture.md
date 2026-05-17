# 2. 顶层架构总览

本章把整套推荐系统从外部看起来的整体面貌呈现出来。读者在这一章里不需要深入到任何具体组件的内部实现，只需要建立起对"一次请求从哪里进入、经历哪些阶段、从哪里离开"的清晰认识即可。后续章节会逐一打开每一个阶段的盖子，把里面的细节讲清楚。

## 一图看完整套系统

在进入文字描述之前，先用一张完整的示意图把整套系统的骨架呈现出来。从一次外部请求进入服务起算，直到最终的 Feed 列表通过 gRPC 接口返回为止，整个数据流大致经历下面这些阶段。

```
┌────────────────────────────────────────────────────────────────────────────┐
│                            Home Mixer (Rust)                               │
│                                                                            │
│  请求 ──▶ For You Pipeline ─────────┐                                       │
│                                    │  调用                                 │
│                                    ▼                                       │
│                          Scored Posts Pipeline (核心)                      │
│                          ┌───────────────────────────────────┐             │
│                          │ ① Query Hydrators (15 个 · 并行)  │             │
│                          ├───────────────────────────────────┤             │
│                          │ ② Sources (6 路 · 并行)           │             │
│                          │    Thunder | Phoenix Retr.        │             │
│                          │    | Phoenix Topics | MoE         │             │
│                          │    | TweetMixer | CachedPosts     │             │
│                          ├───────────────────────────────────┤             │
│                          │ ③ Candidate Hydrators (10 个并行) │             │
│                          ├───────────────────────────────────┤             │
│                          │ ④ Pre-Scoring Filters (14 个串行) │             │
│                          ├───────────────────────────────────┤             │
│                          │ ⑤ Scorers (3 个 串行)             │             │
│                          │    Phoenix → Ranking(组合分)      │             │
│                          │    → VMRanker                     │             │
│                          ├───────────────────────────────────┤             │
│                          │ ⑥ Selector  TopKScoreSelector     │             │
│                          ├───────────────────────────────────┤             │
│                          │ ⑦ Post-Selection Hydrators (6 个) │             │
│                          ├───────────────────────────────────┤             │
│                          │ ⑧ Post-Selection Filters (3 个)   │             │
│                          ├───────────────────────────────────┤             │
│                          │ ⑨ Side Effects (异步)             │             │
│                          └───────────────────────────────────┘             │
│                                    │                                       │
│                                    ▼                                       │
│       ForYou 再加 ads / WTF / prompts / push-to-home, 走 BlenderSelector   │
│                                    │                                       │
└────────────────────────────────────┼───────────────────────────────────────┘
                                     ▼
                          gRPC 返回 FeedItem 列表
```

这张图不是示意图，而是直接对应代码里 `home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs` 与 `for_you_candidate_pipeline.rs` 两份文件里阶段的真实实例化顺序，并非美化处理。读到后面具体阶段的章节时可以随时翻回来对照。

## 一次请求的完整旅程

让我们沿着图中的箭头，把一次完整请求的旅程用文字描述一遍。

请求从外部到达 Home Mixer 服务，进入"For You Pipeline"这一外层流水线。外层流水线的工作并不是直接做推荐，而是先调用内层的"Scored Posts Pipeline"取出一批已经排好序的有机帖子，然后把广告、Who-To-Follow、顶部提示卡、推送到 Home 的内容一并合并进去，最终通过一个叫做 `BlenderSelector` 的混排器决定每一条目在最终列表中的位置。

内层"Scored Posts Pipeline"才是真正承担推荐工作的核心。它的执行顺序是：第一步先用 15 个并行的 Query Hydrator 给请求补充上下文信息，例如用户的关注列表、屏蔽列表、动作历史等等；第二步用 6 路并行的 Source 拉取候选；第三步用 10 个并行的 Candidate Hydrator 给每一条候选补全字段；第四步用 14 个串行的 Filter 把不合规的候选剔除掉；第五步用 3 个串行的 Scorer 给剩下的候选打分；第六步用 Top-K Selector 截取分数最高的若干条；第七步对入选的候选再做一轮额外的"昂贵注水"；第八步用 3 个串行的 Filter 做最后一道把关；第九步通过 `tokio::spawn` 异步触发一系列副作用，包括写 Kafka、更新缓存、上报监控等。

整个过程中最值得留意的设计原则是"能并行的就并行，必须串行的才串行"。Query Hydrator、Source、Candidate Hydrator 这三个阶段都是并行的，原因在于它们各自向不同的下游服务发起请求，相互之间没有数据依赖。Filter 与 Scorer 则必须串行，原因在于后一个组件经常会用到前一个组件写入到候选对象上的字段。Side Effect 阶段通过 `tokio::spawn` 异步触发，意味着主响应不需要等待这些副作用完成，用户感知到的延迟与"是否写完 Kafka"完全无关。

## 两条嵌套的 Pipeline

读源码的过程中很容易绕住的一个细节，是仓库里实际上存在两条相互嵌套的流水线，而不是只有一条。外层那一条叫做 `ForYouCandidatePipeline`，它把核心的内层流水线 `PhoenixCandidatePipeline` 当作自身的一个 Source 来调用。两层各自负责截然不同的事情。

| 层 | 文件 | 干什么 |
|---|---|---|
| 外层（For You） | `home-mixer/candidate_pipeline/for_you_candidate_pipeline.rs` | 负责把"已经排好序的有机帖"与广告、Who-To-Follow、Prompts、Push-to-Home 等额外卡片一起塞进 `BlenderSelector` 进行版面编排，控制广告插入的具体位置以及品牌安全方面的间隔约束 |
| 内层（Phoenix 核心） | `home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs` | 真正承担推荐相关的核心工作：召回、注水、过滤、打分、Top-K 选择、复滤 |

之所以要做这样一种"外层负责版面、内层负责相关性"的拆分，主要是为了让广告团队可以独立地调整广告插入策略，而不需要去触碰核心排序逻辑。同时这种拆分也让有机推荐部分可以被复用到其它面板上，例如 Search Home、Profile 推荐等场景。在大规模系统中，把"业务无关的核心能力"与"业务强相关的产品规则"清晰地分层是非常常见的做法。

## 两个 gRPC 入口

与上面两条流水线相对应，整个 Home Mixer 对外暴露的 gRPC 入口也有两个。两者职责不同，对应了不同的使用场景。

| 服务 | 路径 | 用途 |
|---|---|---|
| `ScoredPostsService` | `home-mixer/scored_posts_server.rs` | 暴露纯粹的有机帖打分能力，由 For You 内部调用，同时也供其它面板（例如 Search Home、Profile 推荐）复用 |
| `ForYouService` | `home-mixer/for_you_server.rs` | 暴露最终的完整 Feed，也就是有机帖与广告、WTF、Prompts 等共同混排之后的结果 |

简单来说，`ScoredPostsService` 是"内核"，`ForYouService` 是"产品"。其它团队如果只需要"按相关性给我一批帖子"的能力，可以直接调用 `ScoredPostsService`，不必关心广告与其它产品元素。

## Sources 阶段：候选源（共 11 路，并行执行）

候选源是整条流水线的输入端，每一个 Source 都向后续阶段提供一批候选。本仓库目前定义了十一个 Source，它们彼此独立、并行执行。前六个属于内层 Phoenix 流水线，后四个则属于外层 For You 流水线。

| Source | 文件 | 类型 |
|---|---|---|
| `ThunderSource` | `sources/thunder_source.rs` | 关注网内的内存索引召回 |
| `PhoenixSource` | `sources/phoenix_source.rs` | Phoenix 双塔的关注网外召回 |
| `PhoenixTopicsSource` | `sources/phoenix_topics_source.rs` | Phoenix 在主题维度上做的召回 |
| `PhoenixMOESource` | `sources/phoenix_moe_source.rs` | Phoenix 的混合专家路径召回 |
| `TweetMixerSource` | `sources/tweet_mixer_source.rs` | 复用 Tweet Mixer 服务已经准备好的一批候选 |
| `CachedPostsSource` | `sources/cached_posts_source.rs` | 复用上一次请求所缓存下来的候选，用以平滑刷新体验 |
| `AdsSource` | `sources/ads_source.rs` | 广告候选，属于外层流水线 |
| `WhoToFollowSource` | `sources/who_to_follow_source.rs` | 推荐你关注模块，属于外层 |
| `PromptsSource` | `sources/prompts_source.rs` | 顶部提示卡，属于外层 |
| `PushToHomeSource` | `sources/push_to_home_source.rs` | 把推送过的内容投到 Home 的特殊候选源，属于外层 |

这里有几个名词需要简单展开一下。"关注网内"（In-Network）指的是当前用户所关注的那些账号发出的内容；"关注网外"（Out-of-Network，简称 OON）指的是用户没有关注、但是通过模型检索发现可能感兴趣的内容。在 X 的产品形态里，For You 信息流要做的恰恰是把这两类内容统一打分排序后混合呈现，因此两类候选都需要召回。"主题维度的召回"指的是按照内容主题（例如体育、科技、娱乐）做一类专门的召回，让用户感兴趣的主题不会因为某一类候选源覆盖不到而被遗漏。"混合专家"（Mixture of Experts，简称 MoE）是一种神经网络架构，让模型内部有多个"专家"子网络并行工作，根据输入路由到不同专家上，从而提升整体表达能力。

## Scorers 阶段：打分器（共 3 个，串行执行）

打分器这一层与候选源不同，它是严格串行执行的。原因很直接：后一个打分器经常会用到前一个打分器写入到候选对象上的字段，没有顺序就无法工作。代码里的实例化顺序可以从 `phoenix_candidate_pipeline.rs` 直接读出来。

```rust
// from phoenix_candidate_pipeline.rs
let scorers = vec![phoenix_scorer, ranking_scorer, vm_ranker];
```

三个打分器各自承担不同的任务。

| 顺序 | Scorer | 作用 |
|---|---|---|
| 1 | `PhoenixScorer` | 通过 gRPC 调用远端的 Phoenix 服务，拿到每一种用户动作的概率，写入候选的 `phoenix_scores` 字段 |
| 2 | `RankingScorer` | 内部由三个子打分器组合而成：先由 `WeightedScorer` 把多种动作的概率按预设权重组合为单一分数，再由 `AuthorDiversityScorer` 对同一作者的候选进行衰减，最后由 `OONScorer` 对关注网外的候选施加折扣 |
| 3 | `VMRanker` | 把候选交给 VMRanker 服务，做最后一层"价值最大化重排"，目前仍处于线上 A/B 测的阶段 |

打分器之所以一定要串行，可以从 `candidate-pipeline/candidate_pipeline.rs:394` 中 `score()` 函数的实现里看得很清楚：后一个 scorer 拿到的候选对象已经被前一个 scorer 修改过。这种依赖关系决定了不存在并行执行的可能性。

这里出现了几个新名词。"A/B 测试"是指在线上把流量按比例切给两个或多个不同版本，通过对比各版本的关键指标来评估哪一个版本更优；"价值最大化重排"是产品层面的术语，意指在相关性的基础上再叠加一层考量，让排序结果同时优化平台希望强调的产品目标，例如某些类型帖子的曝光配额、商业化目标等。

## 选择、复滤与最终返回

打分完成之后，流水线就进入了最后一段。这一段相对简短，但每一步都有自己明确的职责。

Selector 阶段由 `TopKScoreSelector` 负责，它按候选最终的分数从高到低排序，截取到设定好的 `RESULT_SIZE` 条为止。这一阶段相当于一个简单但关键的"漏斗收口"，把前面几个阶段精心准备出来的几千条候选压缩到最终的几十条。

Post-Selection Hydrators 阶段则在已经入选的候选之上再补一层字段。这一阶段所拉取的字段往往代价较高，只对真正会被展示的候选才值得拉，例如可见性判定、品牌安全相关属性、近期互动指标、好友的互关 Jaccard 系数等。这里的 Jaccard 系数是一个集合相似度的度量，计算公式是"两个集合的交集大小除以并集大小"，取值范围在 0 到 1 之间，越大表示两个集合越相似；在推荐场景里常用来度量两位用户的关注列表的重合程度。把这一类字段放到候选被截断之后再拉取，可以避免在被丢弃的候选上做无用功。

Post-Selection Filters 阶段串行执行三个过滤器，依次是 `VFFilter`（可见性过滤）、`AncillaryVFFilter`（次级可见性过滤）以及 `DedupConversationFilter`（同一会话的去重）。这一层过滤的存在是为了拦住那些之前阶段没办法识别、但展示出来会引发问题的候选。可见性过滤本身是一个很大的话题，涉及到平台内容审核、用户偏好、合规等多方面的规则。

Side Effects 阶段做的是收尾工作。它包括把当次的 Phoenix 请求快照写回缓存、向 Kafka 写入相应的事件、更新用户的已展示历史，以及上报各种监控指标。这一阶段非常特殊，它是通过 `tokio::spawn` 异步触发的，并不阻塞主响应的返回。这一设计让用户感知到的延迟与这些写入操作完全解耦。Kafka 是一种分布式消息队列，常用于把"发生过的事件"分发给多个下游消费者；在推荐系统里，Kafka 通常承担"训练数据收集"与"事件广播"两类工作。

完整的端到端流程会在第 4 章里逐阶段展开，可以参见[端到端流水线阶段](pipeline-stages.md)一节。
