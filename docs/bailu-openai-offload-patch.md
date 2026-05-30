# Bailu / OpenAI-compatible offload compatibility patch

## 背景

该仓库在 `context-offload` 的 `local` 模式下，会通过 `ai` + `@ai-sdk/openai` 的 OpenAI-compatible 调用链路执行 L1 / L1.5 / L2。

在 OpenClaw 环境里，Bailu 常见 provider 配置为：

- `baseUrl = https://bailucode.com/openapi`
- `model = bailu-apex-openclaw`

但 OpenAI-compatible SDK 需要的实际根路径应为：

- `https://bailucode.com/openapi/v1`

否则会错误请求到：

- `/openapi/chat/completions`

并报 `Not Found`。

另外，L2 Mermaid 生成批次较重时，仓库原代码将超时写死为 `120_000ms`，在较大 batch 下容易报：

- `L2 FAILED ... timeout`

## 本次改动

### 1. Bailu local-llm baseUrl 兼容修复

修改文件：

- `src/offload/index.ts`

改动要点：

- 在 local LLM 模式初始化 `LocalLlmClient` 前，新增 `localLlmBaseUrl`
- 当 `providerKey === "bailu"` 且 `baseUrl` 以 `/openapi` 结尾时：
  - 去掉结尾多余 `/`
  - 自动补成 `${baseUrl}/v1`
- 最终把 `localLlmBaseUrl` 传给 `LocalLlmClient`
- 保留 debug 日志：
  - `adjusted Bailu local-llm baseUrl for OpenAI-compatible API`

### 2. L2 timeout 放宽并改为可继承配置

修改文件：

- `src/offload/local-llm/index.ts`

改动要点：

- 原先 `l2Generate()` 里写死：
  - `timeoutMs: 120_000`
- 现改为：
  - `timeoutMs: Math.max(this.config.timeoutMs, 180_000)`

含义：

- 优先使用更大的配置值
- 即使配置值偏小，L2 也至少保底 180 秒

## 为什么这样改

### Bailu 路径问题

OpenAI-compatible SDK 会自动拼接 `chat/completions`。

- 错误：`https://bailucode.com/openapi/chat/completions`
- 正确：`https://bailucode.com/openapi/v1/chat/completions`

因此必须在插件 local-llm 这一层对 Bailu 做 `/v1` 适配，而不是改 OpenClaw 主 provider 的 `api: anthropic-messages` 语义。

### L2 timeout 问题

L2 比 L1/L1.5 更重：

- 要吃 `existingMmd`
- 要吃 `recentHistory`
- 要吃 `currentTurn`
- 要处理一批 `newEntries`
- 还要输出 Mermaid + JSON + node_mapping

在较大 batch 下，120 秒可能不够，因此需要提高最小超时下限。

## 验证建议

### A. Bailu OpenAI-compatible 路径验证

使用：

- `bailu-apex-openclaw`
- OpenAI-compatible baseURL：`https://bailucode.com/openapi/v1`

发送：

- `请只回复：PLUGIN_PATH_OK`

预期：

- `content === "PLUGIN_PATH_OK"`
- `finishReason === "stop"`

### B. 运行态观察

观察插件日志中以下报错是否消失：

- `L1.5 FAILED ... Not Found`
- `AI_APICallError: Not Found`

以及 L2 是否还会频繁出现：

- `The operation was aborted due to timeout`

## 若后续升级后再次回滚，应重打的补丁

升级后如果上游源码覆盖了本地修改，优先检查并重打这两处：

1. `src/offload/index.ts`
   - `LocalLlmClient` 初始化前的 Bailu `/openapi -> /openapi/v1` 兼容逻辑
2. `src/offload/local-llm/index.ts`
   - `l2Generate()` 中的超时从固定 `120_000` 改为：
     - `Math.max(this.config.timeoutMs, 180_000)`

## 备注

- `bailu-apex-openclaw` 已验证可用于 OpenAI-compatible 路径
- `bailu-2.7` 在该路径下可能只返回 `reasoning_content`，不适合作为 offload local-llm 的稳定验证模型
