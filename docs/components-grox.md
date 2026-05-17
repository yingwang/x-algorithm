# 3.5 Grox · 内容理解服务（Python）

**目录：** [`grox/`](https://github.com/yingwang/x-algorithm/tree/main/grox) — 2026-05-15 新增。

Grox 是这一版仓库新增加的一项服务，它承担的角色与 Phoenix、Thunder 都不相同。它的存在是为了给推荐系统与安全策略提供一份统一的"内容理解"能力，避免每一个下游需要的时候都自己重复造一遍轮子。换句话说，Grox 把所有与"理解一篇帖子在讲什么、是否符合规则、应该被打上什么标签"相关的逻辑集中到一处，让其它服务可以直接消费它产出的结果，而不必各自维护一套类似的能力。

什么叫做"内容理解"？在推荐系统的语境里，这是一个相对宽泛的术语，包含但不限于以下几方面的工作：判断一篇帖子的主题（例如属于体育、科技还是娱乐）、识别其中是否包含违规内容（垃圾信息、暴力、仇恨等）、生成其语义表示（embedding 向量）以供下游召回与排序使用、必要时还可以生成摘要、标题、回复建议等等。这些任务彼此独立但又共享底层的语言理解能力，因此特别适合集中到同一个服务里统一管理。

## 内部结构

Grox 的目录结构按照"能力"划分得非常清晰。下面这份清单把所有关键子目录的角色都列了出来。

```
grox/
  engine.py             调度引擎入口
  dispatcher.py         按 plan 与 schedule 把任务派发到对应的 task
  main.py               进程入口
  classifiers/content/  各类内容分类器（垃圾内容识别、品类判定、PTOS 策略）
  embedder/             把内容向量化
  summarizer/
    summarizer.py
    post_embedding_summarizer.py
    eapi_summarizer.py
  generators/           生成类能力（标题、推荐理由等）
  tasks/                所有"原子任务"的实现
  plans/                把若干个任务组合成完整的"业务计划"
  schedules/            什么时刻触发哪一个 plan
  data_loaders/         训练与推理阶段的数据装载
```

整套架构可以这样概括：底层是一组 task 原子，中间是把多个 task 串起来的 plan，再往上是决定什么时候启动哪个 plan 的 schedule。所有调度行为最终交给一个统一的 engine + dispatcher 配合执行。这种"任务、计划、调度"的三层架构在企业级数据处理系统中相当常见，与 Apache Airflow、Prefect 等流行的工作流引擎在思路上比较接近。

## tasks 目录：干活的最小单元

`grox/tasks/` 目录下每一个 `task_*.py` 文件都对应着一种"原子能力"。下面这张表把主要文件的职责都做了说明。

| 文件 | 做的事 |
|---|---|
| `task_spam_detection.py` | 垃圾信息识别 |
| `task_safety_ptos_policy.py` 与 `task_safety_ptos_category.py` | PTOS（Platform Terms of Service）策略检查与品类判定 |
| `task_post_safety_screen_deluxe.py` | 综合的安全筛查任务 |
| `task_write_safety_post_annotations_result_sink.py` | 把安全标签写回 sink |
| `task_grok_upa_action_with_labels.py` | 调用 Grok 模型给帖子打 UPA（User Performed Action）类动作标签 |
| `task_multimodal_post_embedding.py` 与 `task_write_mm_embedding_sink.py` | 多模态 embedding 的生成与落库 |
| `task_summarizer_for_post_embedding.py` 与 `task_load_post_with_summary.py` | 让帖子先经过摘要再 embedding，目的是节省 token 并提升抗噪能力 |
| `task_post_embedding_pub.py` 等一系列 `task_*_pub.py` | 把计算完的结果写入 Pub 或者下游 sink |
| `task_load_post_with_not_found_retry.py` | 帖子加载的同时处理 not-found 重试 |
| `task_load_user_recent_posts.py` | 加载用户近期发表的帖子，例如给"banger 屏"使用 |
| `task_banger_screen.py` 以及对应的 `task_initial_banger`（属于 plan 一侧） | 高互动帖筛选 |
| `task_rank_replies.py` | 回复排序 |
| `task_rate_limit.py` | 速率限制 |
| `task_asr.py` | 自动语音识别，主要用于把视频内容转写成文字 |
| `task_media.py` | 媒体相关的处理任务 |
| `task_filters.py` 与 `disable_rules.py` | 任务级别的过滤工具与 disable 规则 |
| `task_pub.py` | 通用的 publish 工具 |
| `task_embedding_pub.py` | embedding 专用的 publish 工具 |

这里出现了几个名词需要解释。"PTOS"是 Platform Terms of Service 的缩写，指的是平台的服务条款；判断一条帖子是否违反平台条款属于平台合规层面的工作。"多模态 embedding"是指同时融合多种模态信息（文字、图片、视频等）来生成统一向量表示的技术，比单纯只用文字 embedding 更能反映帖子的完整语义。"sink"在数据工程中是一个常用术语，指的是数据流的下游目的地（例如数据库、消息队列、对象存储等）。"ASR"（Automatic Speech Recognition）是自动语音识别，把音频中的语音转换成文字。"Pub"是 publish 的简称，指的是把数据发布到某个消息系统的动作。

`task.py` 文件本身是所有 task 的基类，规定了统一的生命周期接口；`task_filters.py` 提供组合与短路的工具方法，让 task 之间可以以声明的方式互相配合。

## plans 目录：把 tasks 串成业务流

单一的 task 只承担一种原子能力，把它们组合成完整的业务逻辑则由 plan 来负责。下面这张表列出了主要的 plan 与它们各自的用途。

| Plan | 用途 |
|---|---|
| `plan_master.py` | 总入口，承担顶层 plan 的路由职责 |
| `plan_post_safety.py`、`plan_safety_ptos.py`、`plan_spam_comment.py` | 安全与合规相关 |
| `plan_initial_banger.py` | 挑选"高互动帖" |
| `plan_reply_ranking.py` | 回复排序 |
| `plan_post_embedding_v5.py` 与 `plan_post_embedding_v5_for_reply.py` | v5 版本的帖子 embedding |
| `plan_post_embedding_with_summary.py` 与 `..._for_reply.py` | 先摘要再 embedding 的变体 |

把"内容理解"切分到 task 与 plan 这样两层粒度，相比于"一个庞大的 monolith 服务"有几处明显的好处。"monolith"指的是把所有功能塞进同一个进程的单体架构，与"微服务"形成对比。每一个 task 都可以单独地编写测试用例、独立重启、独立做 A/B 实验；plan 层面的改动不需要触碰具体 task 的实现细节。这种切分方式让大规模团队同时在不同方向上推进迭代成为可能。

## engine.py 与 dispatcher.py

整套调度逻辑的核心由两个文件承担。`engine.py` 负责在进程启动时注册所有的 task、plan 与 schedule；`dispatcher.py` 则在运行时接收事件，按照配置把请求路由到对应的 task。这两份文件配合起来形成了一套"插件框架"：业务方按规范写好 task 与 plan，注册到 engine，dispatcher 负责把外部事件分发到合适的 task 上。

这一框架强制所有 task 走同一套生命周期，依次是 init、run、emit、ack 四个阶段。`init` 阶段做必要的初始化工作；`run` 阶段是核心计算逻辑；`emit` 阶段把计算结果输出到下游；`ack` 阶段则向上游确认事件已经处理完毕。这种统一的生命周期约束让框架可以对每一个 task 做一致的可观测性与重试控制：如果 ack 没有按时到达，框架可以触发重试；如果某一个 task 反复失败，可以按规则进入"熔断"状态停止接收新事件。

## 在 For You 链路里的角色

值得特别强调的一点是，Grox 是一个完全独立的服务，并不会被 Home Mixer 同步调用。它所做的工作更接近离线或者准实时的 enrichment（信息富化），与在线请求路径有清晰的边界。

具体到工作模式：当一篇新的帖子被发布出来之后，相关事件会异步地流入 Grox，由它给这条帖子打安全标签、判定品类、生成 embedding，并把这些结果写到下游的存储层。当用户随后发起一次刷新请求时，Home Mixer 会通过 `VFCandidateHydrator`、`FilteredTopicsHydrator`、`AdsBrandSafetyHydrator` 等一系列 hydrator 从下游存储拉取这些"已经被打过标签"的字段，而不是临时去调用 Grox 重新计算。

也就是说，Grox 与 Home Mixer 之间是一种生产者与消费者的关系，二者通过 Strato、VF、Safety Label Store 等数据层进行解耦。这里的 Strato 是 Twitter 系内部的图与数据查询服务，VF 是 Visibility Filtering 的缩写代表可见性标签服务，Safety Label Store 顾名思义是用来集中存储安全标签的服务。这种解耦让 Grox 内部的复杂计算（多模态 embedding、Grok 调用、ASR 转写）不会出现在用户感知的延迟链路中。"在线计算"与"离线计算"分离是大规模实时系统的一种基础设计模式：能在事件到达时一次性计算好的东西就提前算好存起来，避免每次请求都临时计算。

## 与 Grok 模型的关系

整个仓库里 Grok 模型同时承担了两个相互独立、却又彼此呼应的角色。`task_grok_upa_action_with_labels.py` 这一任务会直接调用 Grok 模型，让它给帖子打上 UPA 类标签。"UPA"是 User Performed Action 的缩写，指的是与用户曾经做过的动作相关的语义标签，例如这条帖子很可能引发"喜欢"、"转推"或者"举报"等。这一用法体现了 xAI 在工程上的一种典型策略：同一份基础架构（Grok-1 风格的 Transformer，包括它的注意力机制、RoPE 位置编码、统一词表）既被 Phoenix 用来做相关性排序，也被 Grox 用来做内容理解。

两个完全不同的下游任务都从同一份底层框架中受益，研发资源得以集中投入，迭代节奏也因此加快。从更宏观的视角看，这一策略呼应了近年来 AI 领域"基础模型加上下游任务适配"的主流范式：先训练一份通用的大模型，再针对不同任务做轻量级的微调或者改头换面，从而让一份模型在多种业务里发挥最大价值。
