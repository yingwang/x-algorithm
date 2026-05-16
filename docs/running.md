# 8. 本地运行 Phoenix 推理

仓库内提供了一份可以在本地直接跑通的小规模推理 Demo。借助它，读者可以在自己的机器上完整观察一次"召回再到排序"的端到端推理过程，亲眼看到每一阶段所接收的输入与产出的输出。本章把跑通这一 Demo 所需要的全部步骤分小节展开说明。

## 8.1 安装依赖

推荐使用 [uv](https://docs.astral.sh/uv/getting-started/installation/) 这一现代化的 Python 包管理工具。

```bash
cd phoenix
uv sync
```

如果习惯使用传统 pip，也可以直接安装四个核心依赖。

```bash
pip install jax jaxlib dm-haiku numpy
```

每一个依赖在 Demo 中所承担的角色如下表所示。

| 库 | 用途 |
|---|---|
| `jax` 与 `jaxlib` | 张量计算与 JIT 编译 |
| `dm-haiku` | 函数式神经网络层，Phoenix 全部基于 `hk.Module` 实现 |
| `numpy` | 加载 corpus 与 sequence 文件 |

没有 GPU 也完全可以跑通，JAX 会自动 fallback 到 CPU 执行。首次 JIT 编译需要等待 30 到 60 秒，编译完成之后再次调用就会进入毫秒级延迟。

## 8.2 下载并解压模型 artifacts

`phoenix/artifacts/oss-phoenix-artifacts.zip` 这一文件通过 Git LFS 分发，体积约为 3 GB。如果在 clone 仓库时还没有安装 Git LFS，需要先安装并拉取一次。

```bash
git lfs install
git lfs pull
```

随后把压缩包解压到 artifacts 目录之下。

```bash
cd phoenix
unzip artifacts/oss-phoenix-artifacts.zip -d artifacts/
```

解压之后会得到下面这样一份目录结构。

```
oss-phoenix-artifacts/
  retrieval/
    model_params.npz            约 3 MB     召回 Transformer 加上 Candidate Tower 权重
    embedding_tables.npz        约 1.4 GB   user、item、author 三张多哈希表
    config.json                 模型超参数与哈希函数参数
  ranker/
    model_params.npz            约 3 MB     排序 Transformer 加上 action head
    embedding_tables.npz        约 1.4 GB   与 retrieval 同构，单独训练
    config.json
  sports_corpus.npz             53.7 万条 sports 帖，包含 post_id、author_id 与预先算好的 [537K, 128] embedding
  example_sequence.json         示例用户互动序列（三条历史，依次为 NFL、NBA、NHL）
```

## 8.3 跑端到端推理

```bash
uv run run_pipeline.py --artifacts_dir artifacts/oss-phoenix-artifacts
```

`run_pipeline.py` 执行时所做的事情按顺序展开来就是下面这一组步骤。

```
1. load_model_params(retrieval/) → 加载召回 Transformer 与 Candidate Tower
2. load_model_params(ranker/)    → 加载排序 Transformer 与 action head
3. load_embedding_table(...)     → 加载统一 embedding 表
4. 读取 example_sequence.json     → 取出用户最近的三条互动
5. build_hash_functions(config)  → 把 ID 哈希到 vocab 空间
6. 构造 RecsysBatch 与 RecsysEmbeddings
7. 跑召回：user_emb 与 corpus_emb.T 做点积，再 top_k(200)
8. 跑排序：把 200 个候选喂入排序模型，得到 [1, 200, 19] 的 logits 与 [1, 200, 8] 的连续值
9. 通过 sigmoid 转成概率，按多动作展示前 top_k_display 条
```

跑完之后，控制台上会看到类似下面这样的输出。

```
=== Top 30 ranked candidates ===
rank=1   post_id=...  P(fav)=0.183 P(reply)=0.024 P(rt)=0.072 P(dwell)=0.812 P(vqv)=0.640 ...
rank=2   post_id=...  ...
```

## 8.4 常用参数

`run_pipeline.py` 提供了几个常用的命令行参数，下面这张表把它们一并列出。

| 参数 | 默认值 | 含义 |
|---|---|---|
| `--artifacts_dir` | 必填 | 解压之后的目录 |
| `--sequence_file` | `<artifacts>/example_sequence.json` | 用户历史文件 |
| `--corpus_file` | `<artifacts>/sports_corpus.npz` | 召回阶段使用的语料 |
| `--top_k_retrieval` | 200 | 召回阶段保留多少条候选进入排序 |
| `--top_k_display` | 30 | 排序之后展示多少条 |

如果想要更宽松地观察模型对边缘候选的判断，可以放宽召回数量。

```bash
uv run run_pipeline.py --artifacts_dir artifacts/oss-phoenix-artifacts \
    --top_k_retrieval 500 --top_k_display 50
```

## 8.5 换成自己的用户历史

如果想要用自己构造的用户互动序列跑一次模型，可以编辑 `example_sequence.json`。每一条 history 的字段大致是下面这样。

```json
{
  "post_id": 1234567890,
  "author_id": 9876543210,
  "actions": {
    "1": 1,    // favorite
    "11": 1,   // dwell
    "13": 1    // video quality view
  }
}
```

动作 index 对应着 proto 中的 `ActionName` 枚举。下面这张表节选了几种最常用的动作及其对应索引。

| index | 动作 |
|---|---|
| 1 | favorite |
| 4 | reply |
| 5 | quote |
| 6 | retweet |
| 11 | dwell（停留） |
| 13 | video quality view |

字段值通常是 0 或者 1（多热编码）。dwell 这一动作还可以以连续值的形式给出，写在 `history_continuous_actions` 字段中，模型会通过 `_project_continuous_value_to_embedding` 函数把它投影到 embedding 空间。

## 8.6 跑测试

```bash
uv run pytest test_recsys_model.py test_recsys_retrieval_model.py
```

测试会覆盖下面这些方面。

- 排序模型的输入形状、注意力掩码的正确性、多动作 logits 的 shape
- 召回模型的 user_repr 形状、L2 范数、top_k 取值正确性
- Hash-based embedding lookup 的端到端正确性

## 8.7 常见问题

下面这张表汇总了在本地跑 Demo 时经常会遇到的几种问题及其处理方式。

| 现象 | 原因与解法 |
|---|---|
| `git clone` 之后 `artifacts/oss-phoenix-artifacts.zip` 只有几 KB | Git LFS 没有安装。先 `git lfs install` 然后 `git lfs pull` |
| 首次运行时卡几十秒 | JAX 正在做 JIT 编译。再调一次就会快 |
| OOM | 1.4 GB embedding 两份加上中间张量，整体需要 6 GB 以上内存。换成更小的机型会装不下 |
| 装了 GPU 却没有用上 | 安装的是 `jaxlib` 而不是带 CUDA 的版本。按照 [JAX 官方文档](https://docs.jax.dev/en/latest/installation.html)安装 CUDA 版本即可 |
| 加载 corpus 时报 `npz` 缺字段 | `sports_corpus.npz` 解压不完整，重新 `unzip` 即可 |

## 8.8 不要试图把它当线上服务跑

最后还有一段重要的提醒。本仓库提供的 Demo 完全在内存里跑，并不能直接用于线上服务，原因有以下几条。

第一，召回阶段使用的是精确稠密点积（537K × 128 的矩阵乘法）。这种做法在 53.7 万级别的语料上还能撑住，但一旦换到线上数亿级别的语料就完全不可行，必须接入 ANN 索引服务。

第二，排序阶段使用的是 batch 大小为 1 的推理。线上服务必须批量化推理，缓存 User 段的 KV，按 candidate 维度切分。

第三，整个 Demo 没有任何过滤逻辑。Demo 中的 200 条候选会被全部送入排序模型，而线上服务的 Home Mixer 会先把重复、过老、被屏蔽的候选剔除掉，再交给排序模型。

简而言之，Demo 的价值在于帮助读者直观地理解数据流，并不是用来替代线上服务的。把它当作"理解推荐系统结构"的可视化工具来使用，才能发挥它真正的价值。
