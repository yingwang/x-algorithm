# 3.4 Candidate Pipeline · 流水线框架（Rust）

**目录：** [`candidate-pipeline/`](https://github.com/yingwang/x-algorithm/tree/main/candidate-pipeline)

这一份 crate 把"如何组装一个推荐流水线"这件事抽象成了七类 trait。"crate" 是 Rust 的代码组织单位，相当于其它语言里的"包"或者"库"。业务方在使用时只需要实现这些 trait，框架自身则负责并发调度、错误隔离、可观测性以及在超时情况下的优雅降级。Home Mixer 中的 For You 流水线、广告流水线，以及未来可能出现的其它面板（例如 Search Home 与 Profile 推荐），都会复用这同一套基础设施。

把"流水线"抽象成框架的好处在于：所有面板的推荐链路都按同一种模式构建，开发者在不同面板之间切换时不需要重新理解执行模型；同时，跨面板的横向工作（例如新增一种监控指标、新增一种灰度策略）只需要在框架内部改动一次，所有面板就能同时受益。这种"框架与业务分离"的工程实践在大型代码库里非常重要，能够显著减少重复劳动，提高整体演进速度。

## 七个 trait

整个框架的对外接口由七个 trait 组成。下面这段代码列出了它们各自的位置。

```rust
// candidate-pipeline/lib.rs
pub mod candidate_pipeline;
pub mod filter;
pub mod hydrator;
pub mod query_hydrator;
pub mod scorer;
pub mod selector;
pub mod side_effect;
pub mod source;
pub mod util;
```

每一个 trait 都对应着流水线中的一类阶段，它们的职责与执行模型可以汇总成下面这张表。

| Trait | 文件 | 职责 | 执行模型 |
|---|---|---|---|
| `QueryHydrator<Q>` | `query_hydrator.rs` | 给请求补充用户级上下文字段 | 全部并行 |
| `Source<Q, C>` | `source.rs` | 拉取一类候选 | 全部并行 |
| `Hydrator<Q, C>` | `hydrator.rs` | 给候选补充字段 | 全部并行 |
| `Filter<Q, C>` | `filter.rs` | 把不合规的候选剔除掉 | 按顺序串行，每一个 filter 接收上一个的输出 |
| `Scorer<Q, C>` | `scorer.rs` | 对候选打分 | 按顺序串行，后一个 scorer 可以读取前一个 scorer 写入的字段 |
| `Selector<Q, C>` | `selector.rs` | 排序加截断，返回入选与落选两份列表 | 单一实现 |
| `SideEffect<Q, C>` | `side_effect.rs` | 异步副作用 | 通过 `tokio::spawn` 在后台执行，不阻塞主路径返回 |

这里的 `<Q, C>` 是 Rust 的泛型参数语法，`Q` 代表 Query 类型，`C` 代表 Candidate 类型。泛型让 trait 的定义与具体类型解耦：同一份框架既可以承载推荐请求的 Pipeline，也可以承载广告请求的 Pipeline，只需要换不同的 Query 与 Candidate 类型即可。

"并行的 fan-out 与串行的 dependency chain"这种区分并不是从理论上推导出来的，而是从实际工程经验里积累出来的。能够并发完成的工作就尽量并发，从而尽可能压缩端到端延迟；但是过滤与打分这一段必然存在严格的依赖关系，前一个 scorer 写入的字段恰恰是后一个 scorer 的输入，因此必须按顺序执行。理解这种"哪里能并行、哪里必须串行"的分界，是大型异步系统设计的核心能力之一。

## 主循环

整套框架最核心的调度逻辑集中在 `candidate-pipeline/candidate_pipeline.rs:execute` 这一函数里。去掉日志与错误处理之后，它的骨架大致是下面这样。

```rust
async fn execute(&self, query: Q) -> PipelineResult<Q, C> {
    let q = self.hydrate_query(query).await;                    // ① 并行
    let q = self.hydrate_dependent_query(q).await;              // ② 可选，并行
    let candidates = self.fetch_candidates(&q).await;           // ③ 并行
    let candidates = self.hydrate(&q, candidates).await;        // ④ 并行
    let (kept, mut removed) = self.filter(&q, candidates);      // ⑤ 串行

    let scored = self.score(&q, kept).await;                    // ⑥ 串行
    let SelectResult { selected, non_selected } = self.select(&q, scored);  // ⑦

    let selected = self.hydrate_post_selection(&q, selected).await;  // ⑧ 并行
    let (mut final_, ps_removed) = self.filter_post_selection(&q, selected);  // ⑨ 串行
    removed.extend(ps_removed);

    // 截断到 result_size
    let truncated = final_.split_off(self.result_size().min(final_.len()));
    let mut non_selected = non_selected;
    non_selected.extend(truncated);
    self.finalize(&q, &mut final_);
    self.stat_result_size(&final_);

    // ⑩ 异步副作用，不进行 await
    self.run_side_effects(Arc::new(SideEffectInput {
        query: Arc::new(q.clone()),
        selected_candidates: final_.clone(),
        non_selected_candidates: non_selected,
    }));

    PipelineResult { retrieved_candidates: ..., filtered_candidates: removed,
                     selected_candidates: final_, query: Arc::new(q) }
}
```

这一段代码里有几个值得展开来谈的细节。

第一个细节是 `PipelineResult` 同时带回了三份候选列表：retrieved（召回出来的全量）、filtered（被过滤掉的部分）、selected（最终入选的部分）。落选的候选之所以仍然要保留，是因为下游的某些副作用需要把它们写出去，例如供训练管线使用的负样本。"负样本"在机器学习里指的是"被模型当作不应当选择的那一类样本"，对训练监督学习模型至关重要。这种"完整保留所有候选"的接口设计让后续的数据落地工作不会被框架内部的中间状态阻挡住。

第二个细节是 `result_size()` 之后被截掉的尾巴并不会被直接丢弃，而是会被合并到 `non_selected_candidates` 中。这种处理方式与上一条同源：所有曾经被考虑过的候选都要在响应中留下痕迹，仅仅没有被展示出来而已。

第三个细节是 `finalize` 钩子。"钩子"（hook）是一种允许子类在父类某个特定时机插入自定义逻辑的设计模式。这里 `finalize` 有一份默认空实现，子类可以在这一钩子中做"最后一刻"的整理工作，例如重新生成 traceId（请求追踪 ID），或是在副作用启动之前做一些不可逆的修订。

## 框架自带的监控

框架在每一个阶段上都套上了 `#[tracing::instrument]` 宏，可以自动记录下面这一组 span 字段。"span"是分布式追踪中的一个概念，代表一次操作的时间区间，包含起止时间以及附加的标签信息。

```
total_count       该阶段一共配置了多少个组件
enabled_count     启用的数量，剩下的是被 feature switch 关掉了的
disabled          被关闭的组件名，逗号分隔
latency_ms        该阶段的耗时
size              对于 Source 与 Hydrator 来说，是通过该阶段后的候选数量
kept_count        对于 Filter 来说，是被保留下来的候选数量
removed_count     对于 Filter 来说，是被剔除掉的候选数量
filter_rate       对于 Filter 来说，是剔除比例
```

除此之外，整条流水线在收尾阶段还会做一次额外的统计，专门用于监控"空结果"这一异常状况。

```rust
fn stat_result_size(&self, final_candidates: &[C]) {
    let response_size = final_candidates.len();
    let metric_name = format!("{}.execute", self.name());
    receiver.observe(metric_name, ..., response_size, HistogramBuckets::Bucket0To50);
    if response_size == 0 {
        receiver.incr(metric_name, ..., 1u64);   // 单独计数"空结果"，用于报警
    }
}
```

之所以要把"`response_size == 0`"单独记一个计数器，是因为空结果对用户体验来说是一种严重事故。哪怕仅仅是一万次请求中出现一次，运维同学也需要专门盯着这条曲线，确认问题没有蔓延。这种"为关键异常单独埋指标"的做法是生产系统可观测性建设的关键，让团队可以在问题尚未严重之前就提前发现并干预。

## 错误处理策略

框架内部对每一个组件的失败，采取的都是"忽略并继续"这一策略。下面把不同组件的失败处理方式都列了出来。

- QueryHydrator 与 Hydrator 返回 `Err` 时不写入字段，下游消费方按"该字段缺失"处理。
- Source 返回 `Err` 时该路候选缺失，其它路候选照常合并。
- Filter 与 Scorer 在 trait 签名上不允许直接返回 `Err`，函数签名是 `fn run -> Vec<C>`，业务自己需要在内部消化所有可能的失败。
- Selector 在 `enable(query) == false` 时会把全部候选当作 selected 直接返回，这一行为用于灰度时安全地关闭选择逻辑。

整体哲学是"单点失败不传染"。当某一个下游依赖发生抖动时，For You 仍然能够返回一份降级版本的 Feed，而不是直接抛出 503 错误让用户看到空白屏。"503" 是 HTTP 状态码中表示"服务不可用"的标准错误，对用户来说意味着完全无法访问。这种容忍策略对于一个由众多远程依赖支撑起来的复杂系统来说，是工程可用性的根基。在分布式系统理论里，这种设计哲学常常被概括为"失败开放"或"防御性容错"。

## `PipelineCandidate` 与 `PipelineQuery` trait

这两个 trait 是整套框架的最底层抽象，它们的定义非常简洁。

```rust
pub trait PipelineQuery: Clone + Send + Sync + 'static {
    fn params(&self) -> &xai_feature_switches::Params;
    fn decider(&self) -> Option<&xai_decider::Decider>;
}

pub trait PipelineCandidate: Clone + Send + Sync + 'static {}
impl<T> PipelineCandidate for T where T: Clone + Send + Sync + 'static {}
```

`PipelineQuery` 要求实现者能够提供两样东西：一是 `Params`（来自 `xai_feature_switches` 模块），用于读取所有的 feature switch 配置；二是 `Decider`（来自 `xai_decider` 模块），用于参与 A/B 灰度判定。"Decider" 是一种典型的"概率开关"组件，可以根据配置以一定比例返回 true 或 false，从而实现把流量按比例分配到不同实验组的能力。任何 `enable()` 钩子在判断是否启用某一组件时都会用到这两样信息。

`PipelineCandidate` 是一个 blanket implementation，它对任何满足 `Clone + Send + Sync + 'static` 这一组 trait bound 的类型都自动实现。"blanket implementation" 是 Rust 中一种特殊的 impl 写法，对一大类满足某些条件的类型批量实现某个 trait。这意味着业务侧根本不需要手动去实现 `PipelineCandidate`，只需要定义好自己的候选结构体类型即可。

这里出现的 `Clone + Send + Sync + 'static` 是 Rust 的几个标准 trait。`Clone` 表示类型可以被克隆出独立副本；`Send` 表示类型的值可以安全地在线程之间转移所有权；`Sync` 表示类型的引用可以安全地在多线程之间共享；`'static` 是生命周期约束，表示该类型不包含任何短于程序生命周期的引用。这一组约束加在一起意味着候选类型可以被自由地传递给异步任务，是异步 Rust 中常见的组合。

## 框架对业务的承诺

把整套框架的工程价值汇总起来，可以列出这样一份"承诺清单"。每一项承诺背后都有相应的代码实现来兑现。

| 承诺 | 怎么兑现 |
|---|---|
| 新增候选源时不需要改动调度代码 | 实现 `Source` trait，往 `vec![...]` 装配清单里加一行即可 |
| 单一组件失败不会让整次请求崩溃 | 框架内部以 `Result::unwrap_or` 风格吞掉 `Err`，让其它组件继续运行 |
| 阶段级监控开箱即用 | `#[tracing::instrument]` 宏自动记录所有关键字段 |
| Feature switch 与 A/B 灰度 | 每一类 trait 都有一个 `enable(&query)` 钩子供检查 |
| 副作用不阻塞主路径 | 所有副作用都通过 `tokio::spawn` 后台执行 |
| 空结果可报警 | 单独记入 `FINAL_RESULT_EMPTY_SCOPE` 这一计数器 |

这份清单看起来简短，但是它代表了一份成熟基础设施所需要承担的全部工作。一个新加入团队的工程师只需要按照 trait 的形状实现自己的组件，剩下的并发、容错、监控、灰度都是框架"免费"提供的。这种"框架抬升下限、业务专注价值"的分工方式，是大型工程团队提高人均产出的关键手段。
