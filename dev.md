# magpie-middleware-strip-empty-content

Magpie 网关中间件：在请求到达供应商之前，丢弃 content 为空的 user 消息，避免严格渠道（vLLM 系）返回 400 "message content cannot be empty"。

## 背景

旧版会话中残留的空 user 墓碑消息（content 为空字符串或空数组）会被序列化为 `content:""`，vLLM 系渠道直接拒绝整条请求。DSH 本体已在 `llm-pi-ai/src/context.ts` 修复，本中间件在网关层兜底，覆盖所有供应商和 agent。

## 技术栈

- 纯 JavaScript（Magpie moejs 引擎，无 Node API、无 fetch、无定时器）
- 单文件 `empty-content.middleware.js`，声明即用

## Magpie 中间件要点

来源：https://usemagpie.ai/docs/zh/plugins#middleware

### 声明方式

文件名 `*.middleware.js` 或 `*.middleware.mjs`，或 `package.json` 写 `"magpie": { "middleware": "./mw.js" }`。

### 钩子

- `onRequest(body, ctx)` — 路由前改写请求体，返回修改后的 body 或不返回（原样发送）。`ctx.reject(status, msg)` 拒绝请求。
- `onEvent(event, ctx)` — 流式事件逐个处理，返回 null 丢弃。本中间件不需要。
- `onResponse(body, ctx)` — 非流式完整回复。本中间件不需要。
- `export const events = [...]` — 限制 onEvent 只处理指定事件名。

### ctx 字段

`protocol`（`anthropic` / `chat` / `responses` / `gemini`）、`model`、`stream`、`path`、`agent`、`options`、`state`（每请求独立对象）。

### 运行约束

- 无 fetch / 无 Node API / 无定时器，只能用标准 JS + console.log（每分钟 60 行上限）
- 只能用相对路径导入插件目录内文件，无 node_modules
- 钩子超时：请求 250ms，事件 50ms，加载 1s
- 抛异常/超时/返回非预期值 → 按原样发送（安全降级）

## 过滤规则

按 `ctx.protocol` 区分请求体结构：

| 协议 | 消息位置 | user 的空 content 判定 |
|------|----------|------------------------|
| `chat` | `body.messages` | `role==="user"` 且 `content===""` 或 `content` 为空数组 |
| `anthropic` | `body.messages` | `role==="user"` 且 `content` 为空数组，或所有 text block 的 text 均为空 |
| `responses` | `body.input` | `type==="message"` 且 `role==="user"` 且 content 数组为空或全空文本 |
| `gemini` | `body.contents` | `role==="user"` 且 `parts` 为空数组或所有 part 无有效内容 |

只过滤 user 角色。assistant 的 `content:null`（tool_calls 场景）不动。

## 开发与测试

```bash
# 添加（沙箱隔离用独立 HOME）
magpie plugin add ./empty-content.middleware.js

# 验证：构造含空 user 消息的请求发往 vLLM 渠道，确认不再 400

# 查看状态
magpie plugin
```

开发时修改文件后 magpie 每秒自动检测变更并重载，无需手动操作。

## 约定

- 单文件，零依赖，不建 npm 包
- `ponytail` 风格：最短能跑的实现
- 中间件逻辑纯同步，不用 async（没有可等的东西）
