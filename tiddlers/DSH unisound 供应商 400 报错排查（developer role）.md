## 现象

在 DSH 里选用 unisound 供应商（模型 `u2-flash`）发请求时返回：

```
400: {"message":"'messages[0].role' must be one of: system, user, assistant, tool.",
"type":"invalid_request_error","param":"messages[0].role","code":"invalid_parameter"}
```

## 根因

unisound 的 MaaS 网关只接受 `system / user / assistant / tool` 四种 role，不认识 OpenAI 较新的 `developer` role。

而 DSH 的 `openai-completions` 适配路径（pi-ai）在组装请求时，只要模型被判定为 reasoning 模型，就把系统提示词以 `developer` 发送：

`packages/llm/llm-pi-ai` 依赖的 pi-ai 中，`convertMessages` 里：

```js
const useDeveloperRole = model.reasoning && compat.supportsDeveloperRole
const role = useDeveloperRole ? "developer" : "system"
```

`compat.supportsDeveloperRole` 的取值来自 pi-ai 的 `detectCompat`，按 baseURL 猜：

```js
supportsDeveloperRole: isOpenRouterDeveloperRoleModel || (!isNonStandard && !isOpenRouter)
```

`https://maas-api.unisound.com/v1` 既不在 `isNonStandard` 名单里，也不是 openrouter，于是被判定为支持 developer role。再加上配置里 `reasoningEfforts` 声明了 high/max，模型 reasoning=true，最终 `messages[0].role = "developer"`，网关直接拒绝。

即：**私有/非标准网关的 URL 无法被 pi-ai 识别，检测结果按"标准 OpenAI"回退，这对大多数 OpenAI 兼容网关都是错的。**

## 实测验证

用 curl/node 直接打该端点：

| 请求 | 结果 |
| --- | --- |
| `messages[0].role = developer` | HTTP 400，报错文本与现象逐字一致 |
| `messages[0].role = system` | HTTP 200 |
| `reasoning_effort: high` | HTTP 200 |
| `max_tokens` / `max_completion_tokens` | 均 HTTP 200 |
| `store: false` | HTTP 200 |
| `stream_options: {include_usage:true}` | HTTP 200 |
| tool 定义（有无 `strict`） | 均 HTTP 200 |
| 回放带 `reasoning_content` 的 assistant 消息 | HTTP 200 |
| tool 调用往返（assistant.tool_calls + tool 结果） | HTTP 200 |

结论：整条链路上**唯一**不兼容的字段就是 developer role，其余字段都不需要关。

## 修复

在 profile 的路由级加一个兼容开关（`~/.dsh/profiles/desktop/cordis.patch.yml`，`llm-pi-ai` 插件的 `providers.unisound` 下）：

```yaml
unisound:
  displayName: unisound
  apiKeyEnv: UNISOUND_API_KEY
  api: openai-completions
  baseURL: https://maas-api.unisound.com/v1
  compat:
    supportsDeveloperRole: false
  models:
    - id: u2-flash
      ...
```

改为 false 后 pi-ai 回退到 `system`，`reasoning_effort` 等推理参数照常发送。

配在**路由级**（而非某个 model 下）是因为 `supportsDeveloperRole` 是网关的协议能力，路由上所有模型共享。该字段在 `COMPLETIONS_COMPAT_GATE` 里的 disposition 是 `offer`，即允许在 profile 里配置。

## 同类问题

这个字段适用于所有"非标准 OpenAI 兼容网关 + reasoning 模型"的组合。pi-ai README 也明确建议：对 Ollama、vLLM、SGLang 之类自建服务设 `compat.supportsDeveloperRole: false`；若再不支持 `reasoning_effort`，一并设 `compat.supportsReasoningEffort: false`。

判断某网关是否需要关：直接发一条 `role: developer` 的最小请求，看是否返回上面的 400。
