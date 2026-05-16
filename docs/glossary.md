# 10. 术语对照表

## 系统 / 流程

| 英文 | 中文 | 含义 |
|---|---|---|
| For You | For You | X 的算法推荐 Feed |
| In-Network | 关注网内 | 来自你关注的账号 |
| Out-of-Network (OON) | 关注网外 | 通过模型从全量语料检索到的候选 |
| Home Mixer | 编排层 | 整个 For You 链路的入口服务 |
| Thunder | 雷霆 | 关注网内候选的内存索引服务 |
| Phoenix | 凤凰 | 召回 + 排序的双模型系统 |
| Grox | 内容理解服务 | 安全分类、embedding、摘要等 |
| Candidate Pipeline | 候选流水线（框架） | trait-based Pipeline 抽象 |
| Source | 候选源 | 拉取一类候选的组件 |
| Hydration / Hydrator | 注水 / 注水器 | 给候选补全字段 |
| Filter | 过滤器 | 把不合规的候选剔掉 |
| Scorer | 打分器 | 给候选打分 |
| Selector | 选择器 | 排序 + 截断到 Top-K |
| Side Effect | 副作用 | 异步执行的缓存 / 日志 / 埋点 |
| Pre-Scoring / Post-Selection | 打分前 / 选择后 | 两个 filter 段的位置 |
| Result Size | 结果数 | 一次请求最终返回多少帖 |

## 模型 / 算法

| 英文 | 中文 | 含义 |
|---|---|---|
| Two-Tower | 双塔 | 用户塔 + 物品塔，分别编码后做点积 |
| User Tower | 用户塔 | 编码用户特征 + 历史的 Transformer |
| Candidate Tower | 候选塔 | 编码候选 post + author 的轻量 MLP |
| ANN | 近似最近邻 | Approximate Nearest Neighbor，召回端的工程实现 |
| Candidate Isolation | 候选隔离 | 排序时候选互相不可见的注意力掩码 |
| Attention Mask | 注意力掩码 | 控制 Transformer 哪些 token 能互相看见 |
| Hash-Based Embedding | 哈希 Embedding | 用多哈希查表代替每 ID 一条 embedding |
| RoPE | 旋转位置编码 | Rotary Position Embedding |
| Right-Anchored RoPE | 右锚定 RoPE | 让最新 history 拿固定位置（仓库可选启用） |
| Layer Norm | 层归一化 | Transformer 每层后归一化 |
| Unembedding | 反嵌入 | 把 hidden state 投影回 vocab / action 空间 |
| Multi-Action / Multi-Task | 多动作 / 多任务 | 一次预测多个动作的概率 |
| BCE | 二分类交叉熵 | Binary Cross-Entropy，离散动作 loss |
| MAE | 平均绝对误差 | Mean Absolute Error |
| Tweedie | Tweedie 分布损失 | 重尾连续值（如 dwell time）的损失选项 |
| Logit | 对数几率 | 模型直接输出，未经 sigmoid |
| KV Cache | KV 缓存 | Transformer 注意力的 key/value 缓存（推理加速） |

## 候选 / 内容

| 英文 | 中文 | 含义 |
|---|---|---|
| Post / Tweet | 帖子 / 推文 | X 上的一条原创内容 |
| Repost / Retweet | 转推 | 不带评论地转发 |
| Quote | 引用帖 | 带评论地转发 |
| Reply | 回复 | 在某条帖下的回复 |
| LightPost | 轻量帖 | Thunder 里存的简化帖结构 |
| TinyPost | 极简帖 | 只含 post_id + created_at 的索引项 |
| Author | 作者 | 帖子的发布者 |
| Topic | 话题 | 帖子的分类标签 |
| Starter Pack | 入门包 | 用户加入的一组账号合集 |
| Product Surface | 产品面 | 帖子展示的位置（Home / Profile / Search 等） |

## 互动 / 动作

| 英文 | 中文 | 含义 |
|---|---|---|
| Favorite / Like | 点赞 | 单击爱心 |
| Click | 点击 | 点开帖详情 |
| Profile Click | 头像点击 | 点开作者主页 |
| Photo Expand | 放大图片 | 点开大图 |
| VQV (Video Quality View) | 视频质量观看 | 视频被有效观看（达到时长阈值） |
| Dwell | 停留 | 用户在帖子上停留 |
| Dwell Time | 停留时长 | 连续值，归一化的秒数 |
| Quoted Click | 引用帖点击 | 点开嵌入的引用帖 |
| Follow Author | 关注作者 | 点关注 |
| Share | 分享 | 通用分享 |
| Share via DM | 私信分享 | |
| Share via Copy Link | 复制链接分享 | |
| Not Interested | 不感兴趣 | "X 不感兴趣"按钮 |
| Block Author | 拉黑作者 | |
| Mute Author | 静音作者 | |
| Report | 举报 | |

## 安全 / 合规

| 英文 | 中文 | 含义 |
|---|---|---|
| VF (Visibility Filtering) | 可见性过滤 | 删除态 / 垃圾 / 暴力 / 仇恨等过滤 |
| PTOS | Platform Terms of Service | 平台政策合规检查 |
| Brand Safety | 品牌安全 | 广告位的内容环境安全 |
| Safe Gap | 安全间隔 | 广告与敏感内容的最小距离 |
| Bloom Filter | 布隆过滤器 | 概率数据结构，用于"已展示去重" |

## 服务 / 基础设施

| 英文 | 中文 | 含义 |
|---|---|---|
| gRPC | gRPC | Home Mixer 与 Phoenix / Thunder 之间的通信协议 |
| Kafka | Kafka | 消息流（帖子事件、训练日志） |
| Redis | Redis | KV 缓存（候选缓存、Phoenix 请求缓存） |
| Strato | Strato | xAI / Twitter 内部的图 / 数据查询服务 |
| Gizmoduck | Gizmoduck | 用户档案服务（认证、粉丝数） |
| TES | Tweet Entity Service | 帖子详情服务 |
| SocialGraph | 社交图 | 关注 / block / mute 图存储 |
| Manhattan | Manhattan | Twitter 系的高吞吐 KV 数据库 |
| Egress Sidecar | egress 边车 | 出向流量的网络代理（实验路径用） |

## 模型族 / 历史

| 英文 | 中文 | 含义 |
|---|---|---|
| Grok | Grok | xAI 的大语言模型，Phoenix 主干由 Grok-1 移植 |
| MaskNet | MaskNet | Twitter 时代的排序模型族 |
| SimClusters | SimClusters | 老 OON 召回（社区聚类） |
| TwHIN | TwHIN | 老 OON 召回（图嵌入） |
| EarlyBird | EarlyBird | 老 in-network 实时索引（Lucene 改造） |
| the-algorithm | the-algorithm | Twitter 2023 年开源的版本，本仓库的前身 |

## 缩写速查

| 缩写 | 全称 | 章节 |
|---|---|---|
| ANN | Approximate Nearest Neighbor | [§5.1](phoenix-retrieval.md) |
| OON | Out-of-Network | [§5.3](phoenix-multiaction.md) |
| VF | Visibility Filtering | [§6](filters.md) |
| VQV | Video Quality View | [§5.3](phoenix-multiaction.md) |
| MoE | Mixture of Experts | [§3.1](components-home-mixer.md) |
| WTF | Who To Follow | [§3.1](components-home-mixer.md) |
| PTOS | Platform Terms of Service | [§3.5](components-grox.md) |
| RoPE | Rotary Position Embedding | [§5.2](phoenix-ranking.md) |
| TES | Tweet Entity Service | [§3.1](components-home-mixer.md) |
| BCE | Binary Cross-Entropy | [§5.2](phoenix-ranking.md) |
| MAE | Mean Absolute Error | [§5.3](phoenix-multiaction.md) |
