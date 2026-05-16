# 10. 术语对照表

本章把全书中出现的所有专业术语集中起来，按主题分门别类地给出中英文对照与简明含义。在阅读其它章节时如果对某一术语感到陌生，可以随时翻回本章查阅。

## 系统与流程

下面这一组术语涉及系统层面的组件与流程概念，是整套推荐架构的"骨架词汇"。

| 英文 | 中文 | 含义 |
|---|---|---|
| For You | For You | X 平台的算法推荐 Feed |
| In-Network | 关注网内 | 来自当前用户所关注账号的内容 |
| Out-of-Network（OON） | 关注网外 | 通过模型从全量语料中检索到的候选 |
| Home Mixer | 编排层 | 整个 For You 链路的入口服务 |
| Thunder | 雷霆 | 关注网内候选的内存索引服务 |
| Phoenix | 凤凰 | 召回与排序的双模型系统 |
| Grox | 内容理解服务 | 包含安全分类、embedding、摘要等能力 |
| Candidate Pipeline | 候选流水线（框架） | 基于 trait 的 Pipeline 抽象 |
| Source | 候选源 | 拉取一类候选的组件 |
| Hydration 与 Hydrator | 注水与注水器 | 给候选补全字段 |
| Filter | 过滤器 | 把不合规的候选剔除 |
| Scorer | 打分器 | 给候选打分 |
| Selector | 选择器 | 排序加截断到 Top-K |
| Side Effect | 副作用 | 异步执行的缓存、日志、埋点 |
| Pre-Scoring 与 Post-Selection | 打分前与选择后 | 两段 filter 各自的位置 |
| Result Size | 结果数 | 一次请求最终返回多少帖子 |

## 模型与算法

这一组术语属于模型与算法层面，理解它们对于深入阅读 Phoenix 相关章节非常关键。

| 英文 | 中文 | 含义 |
|---|---|---|
| Two-Tower | 双塔 | 用户塔与物品塔分别编码后做点积 |
| User Tower | 用户塔 | 编码用户特征与历史的 Transformer |
| Candidate Tower | 候选塔 | 编码候选 post 与 author 的轻量 MLP |
| ANN | 近似最近邻 | Approximate Nearest Neighbor，召回端的工程实现 |
| Candidate Isolation | 候选隔离 | 排序时让候选互相不可见的注意力掩码 |
| Attention Mask | 注意力掩码 | 控制 Transformer 中哪些 token 可以互相看见 |
| Hash-Based Embedding | 哈希 Embedding | 用多哈希查表代替每一个 ID 单独一条 embedding |
| RoPE | 旋转位置编码 | Rotary Position Embedding |
| Right-Anchored RoPE | 右锚定 RoPE | 让最新一条 history 拿到固定位置编码，仓库内可选启用 |
| Layer Norm | 层归一化 | Transformer 每一层之后的归一化操作 |
| Unembedding | 反嵌入 | 把 hidden state 投影回 vocab 或 action 空间 |
| Multi-Action 与 Multi-Task | 多动作与多任务 | 一次预测多个动作的概率 |
| BCE | 二分类交叉熵 | Binary Cross-Entropy，离散动作的 loss |
| MAE | 平均绝对误差 | Mean Absolute Error |
| Tweedie | Tweedie 分布损失 | 适用于重尾连续值（例如 dwell time）的损失函数选项 |
| Logit | 对数几率 | 模型直接输出，尚未经 sigmoid |
| KV Cache | KV 缓存 | Transformer 注意力中 key 与 value 的缓存，用于推理加速 |

## 候选与内容

这一组术语涉及帖子、作者、话题等内容侧的概念。

| 英文 | 中文 | 含义 |
|---|---|---|
| Post 与 Tweet | 帖子与推文 | X 平台上的一条原创内容 |
| Repost 与 Retweet | 转推 | 不带评论地转发 |
| Quote | 引用帖 | 带评论地转发 |
| Reply | 回复 | 在某一条帖子下的回复 |
| LightPost | 轻量帖 | Thunder 内部存储的简化帖结构 |
| TinyPost | 极简帖 | 只包含 post_id 与 created_at 的索引项 |
| Author | 作者 | 帖子的发布者 |
| Topic | 话题 | 帖子的分类标签 |
| Starter Pack | 入门包 | 用户加入的一组账号合集 |
| Product Surface | 产品面 | 帖子展示的位置，例如 Home、Profile、Search 等 |

## 互动与动作

这一组术语对应着 Phoenix 模型所预测的各种用户动作。

| 英文 | 中文 | 含义 |
|---|---|---|
| Favorite 与 Like | 点赞 | 单击爱心按钮 |
| Click | 点击 | 点开帖子详情 |
| Profile Click | 头像点击 | 点开作者主页 |
| Photo Expand | 放大图片 | 点开大图 |
| VQV（Video Quality View） | 视频质量观看 | 视频被有效观看（达到时长阈值） |
| Dwell | 停留 | 用户在某一帖上停留 |
| Dwell Time | 停留时长 | 连续值，归一化之后的秒数 |
| Quoted Click | 引用帖点击 | 点开嵌入的引用帖 |
| Follow Author | 关注作者 | 点击关注按钮 |
| Share | 分享 | 通用分享 |
| Share via DM | 私信分享 | 通过 DM 分享给特定用户 |
| Share via Copy Link | 复制链接分享 | 通过复制链接的方式分享 |
| Not Interested | 不感兴趣 | "X 不感兴趣"按钮 |
| Block Author | 拉黑作者 | 屏蔽该作者的所有内容 |
| Mute Author | 静音作者 | 静音该作者的发文 |
| Report | 举报 | 提交举报 |

## 安全与合规

这一组术语集中在内容安全与合规这一主题上。

| 英文 | 中文 | 含义 |
|---|---|---|
| VF（Visibility Filtering） | 可见性过滤 | 针对删除态、垃圾、暴力、仇恨等内容的过滤 |
| PTOS | Platform Terms of Service | 平台政策合规检查 |
| Brand Safety | 品牌安全 | 广告位的内容环境安全保障 |
| Safe Gap | 安全间隔 | 广告与敏感内容之间的最小距离约束 |
| Bloom Filter | 布隆过滤器 | 概率数据结构，用于"已展示去重" |

## 服务与基础设施

这一组术语涉及外部依赖服务与基础设施层面。

| 英文 | 中文 | 含义 |
|---|---|---|
| gRPC | gRPC | Home Mixer 与 Phoenix、Thunder 之间的通信协议 |
| Kafka | Kafka | 消息流，用于帖子事件与训练日志等 |
| Redis | Redis | KV 缓存，用于候选缓存以及 Phoenix 请求缓存 |
| Strato | Strato | xAI 与 Twitter 内部的图与数据查询服务 |
| Gizmoduck | Gizmoduck | 用户档案服务，包含认证状态与粉丝数等 |
| TES | Tweet Entity Service | 帖子详情服务 |
| SocialGraph | 社交图 | 关注、block、mute 等关系图的存储 |
| Manhattan | Manhattan | Twitter 系的高吞吐 KV 数据库 |
| Egress Sidecar | egress 边车 | 出向流量的网络代理，用于实验路径 |

## 模型族与历史

这一组术语涉及模型族系与历史名称，便于读者厘清不同代际的关系。

| 英文 | 中文 | 含义 |
|---|---|---|
| Grok | Grok | xAI 的大语言模型，Phoenix 主干由 Grok-1 移植而来 |
| MaskNet | MaskNet | Twitter 时代的排序模型族 |
| SimClusters | SimClusters | 老一代 OON 召回方案（基于社区聚类） |
| TwHIN | TwHIN | 老一代 OON 召回方案（基于图嵌入） |
| EarlyBird | EarlyBird | 老一代关注网内的实时索引（基于 Lucene 改造） |
| the-algorithm | the-algorithm | Twitter 2023 年开源的版本，本仓库的前身 |

## 缩写速查

下面这张表汇总了全书中常见的几个英文缩写及其全称，对应链接指向相关章节。

| 缩写 | 全称 | 章节 |
|---|---|---|
| ANN | Approximate Nearest Neighbor | [5.1 节](phoenix-retrieval.md) |
| OON | Out-of-Network | [5.3 节](phoenix-multiaction.md) |
| VF | Visibility Filtering | [第 6 章](filters.md) |
| VQV | Video Quality View | [5.3 节](phoenix-multiaction.md) |
| MoE | Mixture of Experts | [3.1 节](components-home-mixer.md) |
| WTF | Who To Follow | [3.1 节](components-home-mixer.md) |
| PTOS | Platform Terms of Service | [3.5 节](components-grox.md) |
| RoPE | Rotary Position Embedding | [5.2 节](phoenix-ranking.md) |
| TES | Tweet Entity Service | [3.1 节](components-home-mixer.md) |
| BCE | Binary Cross-Entropy | [5.2 节](phoenix-ranking.md) |
| MAE | Mean Absolute Error | [5.3 节](phoenix-multiaction.md) |
