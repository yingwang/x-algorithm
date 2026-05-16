# 8. 本地运行 Phoenix 推理

## 8.1 安装依赖

推荐用 [uv](https://docs.astral.sh/uv/getting-started/installation/)：

```bash
cd phoenix
uv sync
```

或者直接 pip：

```bash
pip install jax jaxlib dm-haiku numpy
```

需要的库：

| 库 | 用途 |
|---|---|
| `jax` + `jaxlib` | 张量计算 + JIT |
| `dm-haiku` | 函数式神经网络层（Phoenix 全用 hk.Module） |
| `numpy` | corpus / sequence 加载 |

> 没有 GPU 也能跑，jax 会 fallback 到 CPU。首次 jit 编译要等 30~60 秒，编译完后是毫秒级。

## 8.2 下载并解压模型 artifacts

`phoenix/artifacts/oss-phoenix-artifacts.zip` 通过 **Git LFS** 分发，约 3 GB。如果 clone 时没装 Git LFS：

```bash
git lfs install
git lfs pull
```

然后解压：

```bash
cd phoenix
unzip artifacts/oss-phoenix-artifacts.zip -d artifacts/
```

得到的目录结构：

```
oss-phoenix-artifacts/
  retrieval/
    model_params.npz            ~3 MB   召回 Transformer + Candidate Tower 权重
    embedding_tables.npz        ~1.4 GB user / item / author 多哈希表
    config.json                 模型超参 + 哈希函数参数
  ranker/
    model_params.npz            ~3 MB   排序 Transformer + action head
    embedding_tables.npz        ~1.4 GB 与 retrieval 同构（独立训练）
    config.json
  sports_corpus.npz             537K 条 sports 帖（含 post_id / author_id / 预计算的 [537K, 128] embedding）
  example_sequence.json         示例用户互动（3 条：NFL / NBA / NHL）
```

## 8.3 跑端到端推理

```bash
uv run run_pipeline.py --artifacts_dir artifacts/oss-phoenix-artifacts
```

`run_pipeline.py` 做的事（按顺序）：

```
1. load_model_params(retrieval/) → 召回 Transformer + Candidate Tower
2. load_model_params(ranker/)    → 排序 Transformer + action head
3. load_embedding_table(...)     → 统一 embedding 表
4. 读 example_sequence.json      → 拿用户最近 3 条互动
5. build_hash_functions(config)  → 把 ID 哈希到 vocab 空间
6. 构造 RecsysBatch + RecsysEmbeddings
7. 跑召回：user_emb · corpus_emb.T → top_k(200)
8. 跑排序：把 200 个候选喂排序模型 → [1, 200, 19] logits + [1, 200, 8] continuous
9. sigmoid 转概率，按多动作展示前 top_k_display 条
```

输出示意：

```
=== Top 30 ranked candidates ===
rank=1   post_id=...  P(fav)=0.183 P(reply)=0.024 P(rt)=0.072 P(dwell)=0.812 P(vqv)=0.640 ...
rank=2   post_id=...  ...
```

## 8.4 常用参数

| 参数 | 默认 | 含义 |
|---|---|---|
| `--artifacts_dir` | （必填） | 解压后的目录 |
| `--sequence_file` | `<artifacts>/example_sequence.json` | 用户历史 |
| `--corpus_file` | `<artifacts>/sports_corpus.npz` | 召回语料 |
| `--top_k_retrieval` | 200 | 召回后留多少进入排序 |
| `--top_k_display` | 30 | 排序后展示多少条 |

放宽召回数量看看模型对边缘候选的判断：

```bash
uv run run_pipeline.py --artifacts_dir artifacts/oss-phoenix-artifacts \
    --top_k_retrieval 500 --top_k_display 50
```

## 8.5 换成自己的用户历史

编辑 `example_sequence.json`，每条 history 形如：

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

动作 index 对应 proto `ActionName` 枚举（节选）：

| index | 动作 |
|---|---|
| 1 | favorite |
| 4 | reply |
| 5 | quote |
| 6 | retweet |
| 11 | dwell（停留） |
| 13 | video quality view |

值通常是 0/1（multi-hot）。dwell 也支持连续值（写成 `"history_continuous_actions"` 字段，模型会走 `_project_continuous_value_to_embedding`）。

## 8.6 跑测试

```bash
uv run pytest test_recsys_model.py test_recsys_retrieval_model.py
```

会跑：

- 排序模型输入形状 / 注意力掩码正确性 / 多动作 logits shape
- 召回模型 user_repr 形状 / L2 范数 / top_k 正确性
- Hash-based embedding lookup 端到端

## 8.7 常见问题

| 现象 | 原因 / 解 |
|---|---|
| `git clone` 后 `artifacts/oss-phoenix-artifacts.zip` 只有几 KB | Git LFS 未安装。装上 LFS 再 `git lfs pull` |
| 首次 run 卡几十秒 | JAX 在 jit 编译。再调一次会快 |
| OOM | 1.4 GB embedding × 2 + 中间张量需 ≥ 6 GB 内存。换小机型不够 |
| GPU 没用上 | 装的是 `jaxlib` 不是 `jaxlib[cuda]`；按 [JAX 官方装 CUDA 版](https://docs.jax.dev/en/latest/installation.html) |
| corpus 加载报 `npz` 缺字段 | `sports_corpus.npz` 没解压完整；重新 `unzip` |

## 8.8 不要试图把它当线上跑

Demo 完全在内存里：

- 召回是**精确稠密**点积（537K × 128 mat-mul），换成线上数亿语料必须接 ANN 索引。
- 排序是 batch=1 推理；线上要 batch、缓存 User KV、按 candidate 切分。
- 没有任何**过滤**：Demo 的 200 个候选直接全打分；线上 home-mixer 会先剔掉重复、太老、被屏蔽的。

Demo 的价值在于**理解数据流**，不是替代线上服务。
