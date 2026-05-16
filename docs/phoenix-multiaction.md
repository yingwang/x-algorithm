# 5.3 多动作预测与加权打分

**代码：** [`home-mixer/scorers/weighted_scorer.rs`](https://github.com/yingwang/x-algorithm/blob/main/home-mixer/scorers/weighted_scorer.rs) · [`author_diversity_scorer.rs`](https://github.com/yingwang/x-algorithm/blob/main/home-mixer/scorers/author_diversity_scorer.rs) · [`oon_scorer.rs`](https://github.com/yingwang/x-algorithm/blob/main/home-mixer/scorers/oon_scorer.rs)

## Phoenix 输出的全部动作

| 类别 | 字段 | 含义 |
|---|---|---|
| 正向显式 | `favorite_score` · `reply_score` · `retweet_score` · `quote_score` | 点赞、回复、转推、引用 |
| 正向阅读 | `click_score` · `profile_click_score` · `photo_expand_score` · `quoted_click_score` | 点开帖详情 / 头像 / 大图 / 引用帖 |
| 正向视频 | `vqv_score` (Video Quality View) | 视频质量观看 |
| 正向分享 | `share_score` · `share_via_dm_score` · `share_via_copy_link_score` | 分享 / DM / 复制链接 |
| 正向行为 | `dwell_score` · `dwell_time` · `follow_author_score` | 停留 / 停留时长（连续值）/ 关注作者 |
| 负向 | `not_interested_score` · `block_author_score` · `mute_author_score` · `report_score` | 不感兴趣 / 拉黑 / 静音 / 举报 |

`dwell_time` 是**连续值**（归一化的停留秒数），其余是 0~1 的概率。

## 加权公式

`WeightedScorer::compute_weighted_score`：

```rust
combined = favorite_score      * FAVORITE_WEIGHT
         + reply_score         * REPLY_WEIGHT
         + retweet_score       * RETWEET_WEIGHT
         + photo_expand_score  * PHOTO_EXPAND_WEIGHT
         + click_score         * CLICK_WEIGHT
         + profile_click_score * PROFILE_CLICK_WEIGHT
         + vqv_score           * vqv_weight_eligibility(candidate)   // 见下
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

权重值在 `params.rs` 里（`p::FAVORITE_WEIGHT` 等常量），由产品 / 实验团队调。

## VQV 门槛（视频权重的工程取舍）

```rust
fn vqv_weight_eligibility(candidate: &PostCandidate) -> f64 {
    if candidate.video_duration_ms.is_some_and(|ms| ms > p::MIN_VIDEO_DURATION_MS) {
        p::VQV_WEIGHT
    } else {
        0.0
    }
}
```

为什么这样：

- 短于 `MIN_VIDEO_DURATION_MS` 的视频（比如 1~2 秒的自动播放卡片），"质量观看"信号太容易触发，会污染排序。
- 直接把 vqv_score 在这些候选上权重置零，比"在模型里学一个隐式阈值"更可控。

> 这是典型的"模型预测一切，但工程上对极端 case 做硬规则"。

## 负权机制：让"用户大概率讨厌的内容"被拉低

`not_interested` / `block` / `mute` / `report` 四个动作的权重是**负数**。

例：用户对某帖 P(block_author) = 0.3，乘以 BLOCK_AUTHOR_WEIGHT = -100，combined 直接 -30。

`offset_score` 把负 combined 归一化到 `[0, NEGATIVE_SCORES_OFFSET]` 区间：

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

含义：

- 正数 combined → 直接加 offset，保持正数。
- 负数 combined → 线性映射到 `[0, NEGATIVE_SCORES_OFFSET]`，让"被讨厌的帖"也是非负小数，但永远低于"中性帖 + offset"。

这样下游排序 / 阈值判断都能在统一的正数空间里工作。

## 为什么多任务比单一相关性更合理

| 单一相关性分（旧式） | 多动作 + 加权（本仓库） |
|---|---|
| 只能学"相关" | 能区分**为什么相关**（点开 vs 转推 vs 长停留） |
| 无法表达"用户讨厌" | 显式负权直接表达 |
| 不同业务目标无法独立调 | 改 `RETWEET_WEIGHT` 一行就能提升二次传播 |
| 实验难解释 | 看哪个动作概率变了就知道模型变了什么 |

## 作者多样性的几何含义

`AuthorDiversityScorer::multiplier`：

```rust
fn multiplier(&self, position: usize) -> f64 {
    (1.0 - self.floor) * self.decay_factor.powf(position as f64) + self.floor
}
```

- position = 0（同作者第 1 条）：multiplier = 1
- position = 1：multiplier = floor + (1 - floor) * decay
- position → ∞：multiplier → floor

`floor` 防止"封禁式"惩罚 —— 即使第 10 条同作者帖，也至少有 floor 倍的分。

实现细节（容易看错）：

```rust
let mut ordered: Vec<(usize, &PostCandidate)> = candidates.iter().enumerate().collect();
ordered.sort_by(|(_, a), (_, b)| b.weighted_score.partial_cmp(&a.weighted_score)...);
for (original_idx, candidate) in ordered { ... }
```

**先按当前分数排好序再编号**，意味着每个作者的最高分那条 multiplier=1（不衰减），分数低的那几条才被打折。这是"保留好的、压制重复的"取向。

## OON 调整

`OONScorer` 很简单：

```rust
if c.in_network == Some(false):
    score *= OON_WEIGHT_FACTOR
```

`OON_WEIGHT_FACTOR` 通常 < 1，比如 0.75，让关注网外候选默认轻微吃亏。一旦它们的相关性显著高（模型分高得足以补偿），仍然能进 Top-K。

> 这是"先验偏好"而非"硬性切分" —— 比"按比例固定填充 OON 内容"灵活。

## 串成一行公式

把三个 scorer 链起来：

```
score = OON_FACTOR(c) × diversity_multiplier(c, rank_among_same_author)
                     × offset_score(Σ_i  weight_i × P(action_i | user, c))
```

最后这个 `score` 才是 `TopKScoreSelector` 排序用的分。再后面还有 `VMRanker` 做一层产品化重排，那一层是黑盒服务，开源代码里只有 client。
