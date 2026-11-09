# 开源复用台账

## 复用策略

“基于开源项目开发”是默认实现方式。每个基础能力按顺序评估：官方包/SDK → 可独立部署的上游服务 → 兼容许可证的模块化源码 → 自研适配器。不得为了接口偏好而先重写成熟能力。

复用分三种，评审时必须标清：

- **依赖上游**：通过包或服务使用，保持升级能力，首选。
- **移植/派生代码**：只在许可证允许且模块边界清晰时采用，记录原始 commit、变更和归并方式。
- **设计参考**：只借鉴行为、概念或 UX，不复制代码；Codex 属于此类，除非确认有可复用且明确许可的独立开源组件。

每个引入项记录仓库 URL、精确版本/commit、许可证、维护状态、采用模块、改动、漏洞响应和升级责任。每次发布前检查依赖许可证与安全公告。若上游许可证与 Apache-2.0 发布目标不兼容，采用进程/协议隔离并进行法律审查，不能仅凭“独立进程”自行断言兼容。

## 首选映射

### 主运行时决策

pi-mono 是项目唯一的主 Agent runtime，负责 agent loop、模型流式调用、上下文构建与压缩、工具循环。LangGraph、OpenAI Agents SDK、OpenHands、AstrBot 等只在 pi-mono 缺少一项明确能力时作为独立补充或设计参考；不得与 pi-mono 并列编排同一条 Agent 执行链，也不得重复实现其已有能力。若补充项目引入第二套 runtime，必须先说明 pi-mono 的具体缺口、隔离边界和移除条件；否则不引入。

### 能力映射

| 能力 | 主采用 | 补充参考/复用边界 |
|---|---|---|
| 目标理解、验收和任务输出 | pi-mono runtime + 本项目任务契约 | OpenAI Agents SDK、OpenHands 参考结构化验收；不替换主 runtime |
| 任务分解与协作 | pi-mono 主 runtime；群聊调度复用现有 `agent/` 逻辑 | LangGraph/AstrBot 仅补 pi-mono 缺失的 durable orchestration 能力 |
| 上下文构建与压缩 | pi-mono | AstrBot/Codex 作行为和验收参考，不再加一套压缩器 |
| 记忆 | Mem0 | Graphiti/Letta 只在 Mem0 缺失具体需求时评估，不同时维护多套记忆引擎 |
| 工具调用 | pi-mono + MCP TypeScript SDK | Playwright 提供浏览器执行，不另写工具循环 |
| 错误恢复与长任务 | P0 验证后选择一个 durable workflow 项目 | LangGraph、Temporal、Restate 作为候选，不同时引入 |
| 结果验证 | Playwright/工具原生断言 + 本项目验收契约 | OpenHands、Agents SDK 参考验证结构 |
| 权限与副作用控制 | 本项目 policy adapter | Codex CLI、OpenHands Sandbox、AstrBot Sandbox 参考；实现复用前核验许可证 |
| 终止与交付 | pi-mono + 本项目 run/approval 状态 | LangGraph/OpenHands 仅作生命周期设计参考 |
| 可观测性与评测 | OpenTelemetry + 选定的 Langfuse 或 Phoenix | OpenHands/Agent benchmark 参考用例组织 |

| 能力 | 首选上游 | 采用方式 | 本项目负责的部分 |
|---|---|---|---|
| Agent loop、模型流式、上下文管理/压缩 | [pi-mono](https://github.com/badlogic/pi-mono) | TypeScript 包/API 为主；仅在必要且许可兼容时维护小补丁 | Agent 配置、群聊编排适配、UI 事件映射 |
| 现有群聊、@ 解析、快照并行、PASS | `E:/dsh-data/personal-agent/agent`，并核验用户所指 dsh/ds-bot 对应上游 | 先测试后移植可复用模块，不整体复制原型 | Agent roster、权限接入、产品 UI 和可配置策略 |
| 记忆提取与召回 | [Mem0](https://github.com/mem0ai/mem0) | 优先官方 SDK/本地运行模式；不从零实现记忆 CRUD 和检索 | 用户审核、来源呈现、撤销、隐私策略、温度实验层 |
| MCP | [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk) | 官方 SDK | 权限代理、插件市场生命周期和 UX |
| Browser | [Playwright](https://github.com/microsoft/playwright) / 官方 Playwright MCP | 官方依赖或独立 server | 工具策略、审批和状态呈现 |
| 审批与本地执行 UX | Codex、Claude Code | 设计参考；只复用明确开放的独立组件 | 项目权限策略与审计适配 |
| QQ / IM | AstrBot、NapCat 等 | 后续通过独立渠道适配器；逐项确认许可证、平台规则和 API | 身份绑定、消息映射、个人记忆隔离 |

## 原型迁移

`E:/dsh-data/personal-agent/agent` 已有 loop、群聊、@ 解析、工具和 JSONL store。迁移时为每块分别跑现有测试、确认来源和许可证、评估与 pi-mono/MCP 的重叠，再决定保留、适配或替换。禁止把已有 loop 与 pi-mono loop 并行维护。

`server/` 的通知分诊和 Android 采集端是后续事件源候选，不进入首个核心闭环；稳定后再按渠道/事件 Source 接口迁移。

## 上游变更流程

1. 记录依赖的精确版本和许可证。
2. 在适配层升级并运行对应契约测试。
3. 本地补丁必须有来源 commit、理由、测试和上游合并/放弃状态。
4. 定期检查安全更新；插件和模型 Provider 依赖需支持可复现锁文件安装。

