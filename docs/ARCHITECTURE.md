# 架构与数据流

## 技术基线

- Monorepo：pnpm workspace；运行时 TypeScript + Node.js LTS。
- API：Fastify + WebSocket 事件流。
- UI：React + Vite。
- 本地持久化：SQLite；由上游记忆引擎管理其专用索引/存储，不重复造检索引擎。
- **主 Agent 运行时：pi-mono。** Agent loop、Provider 流式调用和上下文构建/压缩统一由 pi-mono 承担，通过 `AgentRuntime` 适配接口接入。
- **补充项目规则：** 其他 Agent 框架只用于补齐 pi-mono 明确缺少的独立能力（例如 durable workflow、评测或沙箱）；不得并列承担主运行时，也不得重复实现 pi-mono 已覆盖的 loop、上下文压缩和工具循环。引入补充项目必须在 `OSS_REUSE.md` 记录缺口、采用范围和退出/替换边界。
- 工具互通：官方 MCP TypeScript SDK；不自定义 MCP 协议。
- 浏览器：Playwright 上游实现或其官方 MCP server，不自写浏览器控制引擎。
- 记忆：以 Mem0 开源记忆引擎为基线，使用本地配置运行；Personal Agent 提供候选审核、来源展示、隐私策略和 UI 适配。
- 本地插件：独立 Node 子进程，经 JSON-RPC stdio 通信；外部 MCP server 作为另一种插件来源接入。

这些组件的实际版本、许可证和维护状态在依赖落地时记录到 `OSS_REUSE.md`；禁止未核验即复制源码。

## 模块边界

- `apps/server`：认证之外的本地 API、会话、事件、配置和依赖编排。
- `apps/web`：聊天、群聊、记忆审核、Agent 编辑器、插件管理和审批 UI。
- `packages/contracts`：事件、Provider、Agent、Plugin、Tool、Memory、Approval 的公开类型。
- `packages/runtime-adapters`：pi-mono 主运行时及 Mem0、MCP 等上游的薄适配层；不得在此并列接入第二套 Agent harness。
- `packages/plugin-host`：插件安装、版本锁定、进程监管、RPC 和权限执行。
- `packages/policy`：资源范围授权、逐次审批和审计记录。
- `packages/workflows`：声明式工作流校验、调度和 checkpoint；优先复用成熟 durable/workflow 组件，评估结果记入复用台账。

## 主要数据流

### 对话与群聊

Web UI → WebSocket command → Session store → 群聊 orchestrator → pi-mono runtime → Provider/MCP/plugin tools → 事件流 → Web UI。并行 Agent 使用同一轮输入快照；群聊轮次默认最多三轮，用户可随时停止。群聊 orchestrator 只负责选人、轮次和协作，不实现第二套 agent loop。

### 记忆

会话消息 → 上游记忆引擎异步提取候选 → 本地候选队列 → 用户审核/撤销 → 已确认记忆 → 上游召回 → 上下文适配层。记忆引擎调用失败不阻塞对话；候选保留待重试并可在 UI 显示失败状态。

### 插件工具

Git source → 安装到隔离目录 → 校验 manifest/锁定版本 → 用户查看权限 → 启动子进程 → Core 验证每个工具调用权限 → 执行并记录审计事件。插件不能直接访问会话数据库、密钥或未授权文件。

## 稳定接口

- `AgentRuntime`: `run(input, context, emit, abortSignal)`，统一产出文本、工具请求、usage、压缩状态和错误事件。
- `MemoryProvider`: `extract(messages)`, `search(query, scope)`, `review(candidateId, decision)`, `delete(memoryId)`。
- `PluginHost`: `install(source, ref)`, `inspect(pluginId)`, `enable(pluginId)`, `disable(pluginId)`, `update(pluginId, ref)`, `rollback(pluginId)`。
- 所有运行事件具有 `id`, `type`, `sessionId`, `runId`, `timestamp`, `payload`；前端按事件重连恢复运行状态。

