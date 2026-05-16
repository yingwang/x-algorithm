# 3.4 Candidate Pipeline · 流水线框架（Rust）

**目录：** [`candidate-pipeline/`](https://github.com/yingwang/x-algorithm/tree/main/candidate-pipeline)

把"组装一个推荐流"抽象成 7 类 trait，业务方只需实现 trait，框架负责并发、错误隔离、监控、超时降级。Home Mixer、广告、未来其他面（Search Home / Profile 推荐）都复用这一套。

## 7 个 trait

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

| Trait | 文件 | 职责 | 执行模型 |
|---|---|---|---|
| `QueryHydrator<Q>` | `query_hydrator.rs` | 给请求（query）补字段：用户上下文 | **全部并行** |
| `Source<Q, C>` | `source.rs` | 拉取一类候选 | **全部并行** |
| `Hydrator<Q, C>` | `hydrator.rs` | 给候选补字段 | **全部并行** |
| `Filter<Q, C>` | `filter.rs` | 把不合规的候选剔掉 | **按顺序串行**（每个 filter 接收上一个的输出） |
| `Scorer<Q, C>` | `scorer.rs` | 对候选打分 | **按顺序串行**（后一个 scorer 可读前一个写入的字段） |
| `Selector<Q, C>` | `selector.rs` | 排序 + 截断，返回 `(selected, non_selected)` | 单一实现 |
| `SideEffect<Q, C>` | `side_effect.rs` | 异步副作用 | **`tokio::spawn` 后台执行**，不阻塞返回 |

> 这种"并行的 fan-out / 串行的 dependency chain"分类是从实际经验里出来的：拉信息能并发就并发，但过滤和打分必须按顺序（前一个 scorer 写的字段是后一个 scorer 的输入）。

## 主循环

`candidate-pipeline/candidate_pipeline.rs:execute` 是整套流水线的"调度函数"。脱去日志和错误处理后大致是：

```rust
async fn execute(&self, query: Q) -> PipelineResult<Q, C> {
    let q = self.hydrate_query(query).await;                    // ① 并行
    let q = self.hydrate_dependent_query(q).await;              // ②* 并行（可选）
    let candidates = self.fetch_candidates(&q).await;           // ③ 并行
    let candidates = self.hydrate(&q, candidates).await;        // ④ 并行
    let (kept, mut removed) = self.filter(&q, candidates);      // ⑤ 串行

    let scored = self.score(&q, kept).await;                    // ⑥ 串行
    let SelectResult { selected, non_selected } = self.select(&q, scored);  // ⑦

    let selected = self.hydrate_post_selection(&q, selected).await;  // ⑧ 并行
    let (mut final_, ps_removed) = self.filter_post_selection(&q, selected);  // ⑨ 串行
    removed.extend(ps_removed);

    // 截到 result_size
    let truncated = final_.split_off(self.result_size().min(final_.len()));
    let mut non_selected = non_selected;
    non_selected.extend(truncated);
    self.finalize(&q, &mut final_);
    self.stat_result_size(&final_);

    // ⑩ 异步副作用，不 await
    self.run_side_effects(Arc::new(SideEffectInput {
        query: Arc::new(q.clone()),
        selected_candidates: final_.clone(),
        non_selected_candidates: non_selected,
    }));

    PipelineResult { retrieved_candidates: ..., filtered_candidates: removed,
                     selected_candidates: final_, query: Arc::new(q) }
}
```

注意几个细节：

1. **`PipelineResult` 同时带回了三份候选**：`retrieved` / `filtered` / `selected`。落选的也保留是因为有些 side effect 要写它们（比如供训练用的负样本）。
2. **截到 `result_size()`** 后被砍掉的尾巴并入 `non_selected_candidates`，避免被丢弃。
3. **`finalize` 是个 hook**：默认空实现，子类可重写做"最后一刻"的整理（如重新打 traceId）。

## 框架自带的监控

每个阶段都有 `#[tracing::instrument]`，会自动 record 这些 span 字段：

```
total_count       该阶段一共配了多少个组件
enabled_count     启用的数量（剩下的被 feature switch 关掉）
disabled          被关掉的组件名（逗号分隔）
latency_ms        该阶段耗时
size              （Source / Hydrator）通过该阶段后的候选数
kept_count        （Filter）保留多少
removed_count     （Filter）剔掉多少
filter_rate       （Filter）剔掉占比
```

外加最终统计：

```rust
fn stat_result_size(&self, final_candidates: &[C]) {
    let response_size = final_candidates.len();
    let metric_name = format!("{}.execute", self.name());
    receiver.observe(metric_name, ..., response_size, HistogramBuckets::Bucket0To50);
    if response_size == 0 {
        receiver.incr(metric_name, ..., 1u64);   // 0-result 单独计数，用于报警
    }
}
```

`response_size == 0` 单独记一个 counter，是因为"空结果"是用户体验事故，运维要专门看这条曲线。

## 错误处理策略

框架内部对每个组件的失败都是**"忽略并继续"**：

- QueryHydrator / Hydrator 返回 `Err`：不写值，下游当成没有该字段。
- Source 返回 `Err`：该路候选缺失，其他路照常合并。
- Filter / Scorer 在框架层不允许 fail（trait 签名是 `fn run -> Vec<C>`，业务自己内部消化）。
- Selector 是 `enable(query) == false` 时直接把全部 candidates 当 selected 返回（用于灰度关闭）。

这种"单点失败不传染"的策略让 For You 在依赖飘的时候仍能出**降级的 Feed**，而不是直接 503。

## `PipelineCandidate` 与 `PipelineQuery` trait

两者的边界条件很简洁：

```rust
pub trait PipelineQuery: Clone + Send + Sync + 'static {
    fn params(&self) -> &xai_feature_switches::Params;
    fn decider(&self) -> Option<&xai_decider::Decider>;
}

pub trait PipelineCandidate: Clone + Send + Sync + 'static {}
impl<T> PipelineCandidate for T where T: Clone + Send + Sync + 'static {}
```

- `PipelineQuery` 必须能拿到 `Params`（feature switches）和 `Decider`（A/B 灰度），用来让任何 `enable()` 决定能根据当前实验配置开关。
- `PipelineCandidate` 是 blanket impl，对任何 `Clone + Send + Sync + 'static` 的类型都自动满足 —— 业务端不需要手动实现。

## 框架对业务的承诺

| 承诺 | 怎么兑现 |
|---|---|
| 加新候选源不动调度代码 | 实现 `Source` trait，往 `vec![...]` 加一行 |
| 单组件失败不爆 | `Result::unwrap_or` 风格，框架吞掉 Err |
| 阶段级监控开箱即用 | `#[tracing::instrument]` 自动 record |
| Feature switch / A/B | `enable(&query)` 钩子 |
| 副作用不阻塞主路径 | `tokio::spawn` |
| 0-result 报警 | `FINAL_RESULT_EMPTY_SCOPE` counter |
