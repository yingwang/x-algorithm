# 5.3 多动作预测与加权打分

**代码：** [`home-mixer/scorers/weighted_scorer.rs`](https://github.com/yingwang/x-algorithm/blob/main/home-mixer/scorers/weighted_scorer.rs)、[`author_diversity_scorer.rs`](https://github.com/yingwang/x-algorithm/blob/main/home-mixer/scorers/author_diversity_scorer.rs)、[`oon_scorer.rs`](https://github.com/yingwang/x-algorithm/blob/main/home-mixer/scorers/oon_scorer.rs)

排序模型预测出来的并不是一个单一的相关性分数，而是一组覆盖了多种用户动作的概率与连续值。如何把这一组多维输出聚合成一个最终的排序分，正是本节要展开讨论的话题。

## Phoenix 输出的全部动作

下面这张表按动作的类别归纳了 Phoenix 模型所输出的所有字段，每一项都给出了对应的实际含义。

| 类别 | 字段 | 含义 |
|---|---|---|
| 正向显式 | `favorite_score`、`reply_score`、`retweet_score`、`quote_score` | 点赞、回复、转推、引用 |
| 正向阅读 | `click_score`、`profile_click_score`、`photo_expand_score`、`quoted_click_score` | 点开帖子详情、点开头像、点开大图、点开引用帖 |
| 正向视频 | `vqv_score`（Video Quality View） | 视频质量观看 |
| 正向分享 | `share_score`、`share_via_dm_score`、`share_via_copy_link_score` | 分享、通过 DM 分享、通过复制链接分享 |
| 正向行为 | `dwell_score`、`dwell_time`、`follow_author_score` | 停留、停留时长（连续值）、关注作者 |
| 负向 | `not_interested_score`、`block_author_score`、`mute_author_score`、`report_score` | 不感兴趣、拉黑、静音、举报 |

需要特别注意的一点是：`dwell_time` 是一个连续值（归一化之后的停留秒数），而其它字段都是 0 到 1 之间的概率。这一区别在后续的加权环节会通过不同的处理方式体现出来。

## 加权公式

`WeightedScorer::compute_weighted_score` 函数承担了把多动作预测聚合成单一分数的工作。下面这段代码体现了它的完整结构。

```rust
combined = favorite_score      * FAVORITE_WEIGHT
         + reply_score         * REPLY_WEIGHT
         + retweet_score       * RETWEET_WEIGHT
         + photo_expand_score  * PHOTO_EXPAND_WEIGHT
         + click_score         * CLICK_WEIGHT
         + profile_click_score * PROFILE_CLICK_WEIGHT
         + vqv_score           * vqv_weight_eligibility(candidate)   // 详见下面一节
         + share_score         * SHARE_WEIGHT
         + share_via_dm_score  * SHARE_VIA_DM_WEIGHT
         + share_via_copy_link_score * SHARE_VIA_COPY_LINK_WEIGHT
         + dwell_score         * DWELL_WEIGHT
         + quote_score         * QUOTE_WEIGHT
         + quoted_click_score  * QUOTED_CLICK_WEIGHT
         + dwell_time          * CONT_DWELL_TIME_WEIGHT
         + follow_author_score * FOLLOW_AUTHOR_WEIGHT
         + not_interested_score * NOT_INTERESTED_WEIGHT       // 负权
         + block_author_score   * BLOCK_AUTHOR_WEIGHT          // 负权
         + mute_author_score    * MUTE_AUTHOR_WEIGHT           // 负权
         + report_score         * REPORT_WEIGHT;               // 负权
```

每一项的权重值都定义在 `params.rs` 文件中（例如 `p::FAVORITE_WEIGHT` 这类常量），由产品团队与实验团队根据 A/B 结果共同调节。

## VQV 门槛：视频权重的工程取舍

视频观看分数 `vqv_score` 的权重并不是一个固定常量，而是带有一道工程上的门槛。

```rust
fn vqv_weight_eligibility(candidate: &PostCandidate) -> f64 {
    if candidate.video_duration_ms.is_some_and(|ms| ms > p::MIN_VIDEO_DURATION_MS) {
        p::VQV_WEIGHT
    } else {
        0.0
    }
}
```

这一道门槛背后有两个清晰的考量。第一个是：那些短于 `MIN_VIDEO_DURATION_MS` 的视频，例如一两秒的自动播放卡片，本身的"质量观看"信号太容易触发，如果不加约束地把这一类候选的 vqv_score 计入权重，会污染整个排序。第二个是：直接把这一类候选上的 vqv 权重置为零，比让模型自己学习一个隐式的阈值更加可控，工程上的复杂度也低得多。

这是典型的"模型负责预测一切，工程层在极端 case 上叠加硬规则"的做法。这种做法在大规模推荐系统里非常常见，因为模型再好也无法预知所有的边界情况，工程层留一道安全网会让整体表现更稳。

## 负权机制：让"用户大概率讨厌的内容"被拉低

`not_interested`、`block`、`mute`、`report` 这四个动作所对应的权重都是负数。这一处设计才是整个加权公式真正区别于传统单一相关性的关键所在。

举一个具体的例子。假设用户对某一条帖子的 `P(block_author)` 概率被模型预测为 0.3，再乘以 `BLOCK_AUTHOR_WEIGHT = -100`，combined 这一项就直接产生了 -30 的贡献。

当 combined 出现负值时，紧随其后的 `offset_score` 函数会把它归一化到 `[0, NEGATIVE_SCORES_OFFSET]` 这一区间内。

```rust
fn offset_score(combined: f64) -> f64 {
    if WEIGHTS_SUM == 0.0 {
        combined.max(0.0)
    } else if combined < 0.0 {
        (combined + NEGATIVE_WEIGHTS_SUM) / WEIGHTS_SUM * NEGATIVE_SCORES_OFFSET
    } else {
        combined + NEGATIVE_SCORES_OFFSET
    }
}
```

这一段映射的语义可以这样理解。当 combined 是正数时，直接加上一个 offset，保持正数即可。当 combined 是负数时，线性映射到 `[0, NEGATIVE_SCORES_OFFSET]` 这一区间，使得"被用户讨厌的帖子"也是一个非负的小数，但它永远低于"中性帖子加上 offset"所产生的分数。

这种归一化方式让下游所有的排序与阈值判断都可以在统一的正数空间里工作，不需要分别处理正负两套分数。

## 为什么多任务比单一相关性更合理

把"多动作加权"这一做法与传统的"单一相关性分"做对比，可以看到下面这些差别。

| 单一相关性分（旧式） | 多动作加权（本仓库） |
|---|---|
| 只能学到"相关"这一个概念 | 能够区分"为什么相关"，例如点开、转推、长停留各自的差异 |
| 无法表达"用户讨厌某类内容" | 通过显式的负权直接表达 |
| 不同业务目标无法独立调节 | 改一行 `RETWEET_WEIGHT` 就能针对性地提升二次传播 |
| 实验结果难以解释 | 看哪一种动作的概率发生了变化就能直接判断模型在哪里发生了变化 |

## 作者多样性的几何含义

`AuthorDiversityScorer::multiplier` 函数实现了同作者衰减的核心逻辑。

```rust
fn multiplier(&self, position: usize) -> f64 {
    (1.0 - self.floor) * self.decay_factor.powf(position as f64) + self.floor
}
```

把这一公式在不同 position 上的取值列出来，可以直观地看出衰减的形状。

- 当 position 为 0 时（即同一作者的第一条），multiplier 等于 1
- 当 position 为 1 时，multiplier 等于 `floor + (1 - floor) * decay`
- 当 position 趋向于无穷时，multiplier 趋向于 `floor`

`floor` 参数的存在是为了避免"封禁式"惩罚。即便是同一作者的第十条帖子，它的分数也至少会保留 `floor` 倍。这种"衰减但不归零"的处理方式既能避免同一作者刷屏，也能保证好作者的多条优质内容不会被完全压制。

实现上有一个非常容易看错的细节：

```rust
let mut ordered: Vec<(usize, &PostCandidate)> = candidates.iter().enumerate().collect();
ordered.sort_by(|(_, a), (_, b)| b.weighted_score.partial_cmp(&a.weighted_score)...);
for (original_idx, candidate) in ordered { ... }
```

这一段代码先把候选按当前分数倒序排好，再按这一顺序进行编号。这意味着每一位作者得分最高的那一条 multiplier 等于 1，根本不会被衰减；只有分数较低的几条才会被打折。这一处理方式背后的取向是"保留好的、压制重复的"。

## OON 调整

相比前面两个 scorer，`OONScorer` 的逻辑非常简单。

```rust
if c.in_network == Some(false):
    score *= OON_WEIGHT_FACTOR
```

`OON_WEIGHT_FACTOR` 这一参数通常会被设置为略小于 1 的值，例如 0.75，目的是让关注网外的候选默认稍微吃点亏。但是这一惩罚是温和的：一旦某一条关注网外候选的相关性足够高（也就是模型分数高得足以补偿这一折扣），它仍然有机会进入最终的 Top-K。

这是一种"先验偏好"而非"硬性切分"的设计。它与"按比例固定填充 OON 内容"那一类做法相比要灵活得多。

## 串成一行公式

把上面三个 scorer 的逻辑串起来，可以写成下面这样一行公式。

```
score = OON_FACTOR(c) × diversity_multiplier(c, rank_among_same_author)
                     × offset_score(Σ_i  weight_i × P(action_i | user, c))
```

这一最终的 `score` 才是 `TopKScoreSelector` 在选择阶段实际用来排序的分数。在此之后还会有 `VMRanker` 做一层产品化重排，但是这一层属于黑盒服务，开源代码中只包含 client，模型本身并未对外公开。
