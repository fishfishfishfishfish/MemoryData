# 本地推理服务配置（vLLM）

仓库 `config/` 下的 YAML 预设默认面向 **vLLM 暴露的 OpenAI 兼容接口**，端口与模型短名已经写死在配置中。只要用 vLLM 按下面的方式起好服务，`--served-model-name` 与配置里的短名对上，就能**不改任何 config 直接跑**。

> **提示：vLLM 无需与代码同环境。** vLLM 服务可以安装在独立的 conda 环境（如 `vllm-env`）中启动，代码的 benchmark 执行环境（如 `memory-bench`，Python 3.11）通过 OpenAI 兼容 HTTP 接口访问即可。二者无需共用同一个 conda 环境，也不要求 Python 版本一致——这是本仓库推荐的用法：vLLM 服务于 `vllm-env`，benchmark 代码跑在 `memory-bench`，互不干扰。

## 需要起的两类服务

| 类型 | 模型 | 端口 | 说明 |
| --- | --- | --- | --- |
| 对话模型 | `Qwen3-8B`（少数用 `Qwen2.5-7B`） | 9908 / 9909 / 9910 / 9911 | 同一模型被拆成 4 个"泳道"跑不同方法 |
| 嵌入模型 | `Qwen3-Embedding-4B` | 9009（另 9010 用于 letta/memtree） | 大部分用到向量检索的方法共用 |

## 各端口的方法归属

| 端口 | 使用它的预设 |
| --- | --- |
| 9908 | reference_long_context_agent、reference_embedding_rag、reference_simple_rag_bm25 |
| 9909 | sequential_mem0、topological_cognee、topological_hippo_rag、topological_zep_local、topological_raptor、topological_zep、hybrid_memo_rag |
| 9910 | sequential_memochat、topological_memtree、hybrid_memoryos、hybrid_lightmem |
| 9911 | sequential_memagent、hybrid_simplemem、hybrid_letta |
| 9009 嵌入 | 绝大多数用到向量检索的方法 |
| 9010 嵌入 | hybrid_letta、topological_memtree |

> 例外（不依赖本地 vLLM）：
> - `hybrid_memos.yaml` 用 OpenRouter 云端（`openrouter.ai`），不需要本地服务。
> - `hybrid_lightmem`、`sequential_memochat` 的嵌入分别用本地 HuggingFace 加载 / hashing，不起嵌入服务也能跑。

## 起服务

### 对话模型（Qwen3-8B）

假设权重在 `Qwen/Qwen3-8B`，短名对齐 config 里的 `Qwen3-8B`：

```bash
export OPENAI_API_KEY=sk-local-anything   # 本地可随便填，但必须存在

vllm serve Qwen/Qwen3-8B \
  --host 127.0.0.1 --port 9908 \
  --served-model-name Qwen3-8B \
  --max-model-len 32768 \
  --gpu-memory-utilization 0.9
```

端口 9909 / 9910 / 9911 是同一模型，区别只在 `--port` 参数。想完全按仓库配置跑，就分别再起 3 个实例（换 `--port` 即可）。

### 嵌入模型（Qwen3-Embedding-4B）

```bash
vllm serve Qwen/Qwen3-Embedding-4B \
  --host 127.0.0.1 --port 9009 \
  --served-model-name Qwen3-Embedding-4B \
  --task embed
```

## 关键点

- **`--served-model-name` 必须与 config 里的 `model:` / `embedding_model:` 完全一致**，否则会报 `model not found`。config 里已经是短名（`Qwen3-8B`、`Qwen3-Embedding-4B`），照抄即可。
- **`OPENAI_API_KEY` 本地可任意填**，但必须存在（多数预设通过 `api_key_env: OPENAI_API_KEY` 读取）。
- **部分预设的 `model:` 是 `Qwen2.5-7B`**（`topological_raptor`、`topological_zep`、`hybrid_memo_rag`、`hybrid_memos`）。若要用这几个预设，需另起一个 `--served-model-name Qwen2.5-7B` 的服务，或将它们的 `model:` 改为 `Qwen3-8B`。
- **省显存：合并到单实例**。4 个对话端口其实是同一模型，作者只是为了并行实验才分泳道。单机显存有限时可只起 1 个对话服务（如 9908）+ 1 个嵌入服务（9009），然后把所有 config 的 `base_url` / `embedding_base_url` 统一指向这两个端口。这是唯一需要动 config 的地方。