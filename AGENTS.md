# Folio 项目 Agent 指南

## 工作约定

- 本文件适用于整个仓库；子目录中的 `AGENTS.md` 可补充或覆盖对应范围的规则。
- 默认使用简体中文沟通和编写文档；代码保持所在文件的命名、注释和格式风格。
- 开始前阅读 `CONTRIBUTING.md`，按任务查阅相关文档。`CLAUDE.md` 提供历史上下文；命令和行为以当前代码、`package.json` 与 CI 为准。
- 新功能或较大改动先澄清需求和方案；只允许使用 `superpowers:brainstorming`，其他 `superpowers:*` 技能禁用。明确的小改动可以直接推进。
- 只修改完成当前需求所必需的内容，不顺手重构、重排目录或更新无关依赖。
- 本地提交和远程推送分别取得用户授权；默认均不执行。不得擅自更改 Git 配置、跳过钩子或 force push。

## 项目与目录

Folio 是本地优先的 AI 投资研究桌面工作台，使用 Bun workspaces、TypeScript strict、Electron、React 18、Jotai、Tailwind CSS 和 Pi Agent。产品仅提供只读研究与决策支持，不提供下单或交易能力。

- `apps/electron/src/main/`：主进程、IPC、内核接线、凭证与服务生命周期。
- `apps/electron/src/preload/`：白名单 `contextBridge`；`src/renderer/`：React 入口与客户端桥接。
- `packages/core/src/`：公共领域类型与协议；公共契约优先放在这里。
- `packages/shared/src/`：内核、Agent 适配器、提供商、能力注册表、研究、论点、提醒、风险与持久化。
- `packages/ui/src/`：React 组件、布局与 Jotai atoms。
- `packages/longbridge-tools/src/`：Longbridge CLI 参数执行、校验、解析与规范化。
- `packages/pi-extension/src/`、`.pi/extensions/`：Pi 工具与运行时扩展。
- `packages/skill-hub/src/`、`skills/`：技能加载、安全资源访问与技能包。
- `packages/i18n/src/`：本地化资源与工具。
- `docs/`：产品、架构、UI、评测与发布文档；`scripts/`：评测及发布脚本。

沿用现有 monorepo 布局。单元测试通常与源码同目录，命名为 `*.test.ts` / `*.test.tsx`；Electron 集成与端到端测试位于 `apps/electron/e2e/`。

## 开发与验证命令

在仓库根目录执行；使用 Bun 和现有 `bun.lock`，不新增其他包管理器的锁文件。CI 的 Bun 版本见 `.github/workflows/pr.yml`（当前为 `1.4.2`）。依赖安装在项目本地。

```sh
bun install --frozen-lockfile
bun run dev                         # Vite 渲染预览；不会自动启动 Electron 主进程
bun run build                       # 工作区与 Electron 渲染/预加载/主进程构建
bun run typecheck
bun test packages/shared --isolate  # 示例：针对改动所在工作区运行测试
bun run test:unit                    # 全仓隔离单元测试
bun run eval:smoke                   # 确定性 Agent smoke 评测
bun run test:e2e                     # Electron 端到端验证
bun run i18n:check
```

- 添加或更新依赖时使用 Bun，并同步对应 `package.json` 与 `bun.lock`。
- 改动代码至少运行相关测试和类型检查；Agent/工具执行路径增加 `eval:smoke`，UI 流程按需运行 E2E，本地化改动运行 `i18n:check`。
- 纯文档改动核对引用路径、命令和差异即可，不宣称未运行的测试已通过。
- 真实提供商、付费评测和发布打包按任务需求运行，不把外部服务凭证当作离线验证的前置条件。
- Windows 上设置环境变量使用 PowerShell 语法，例如 `$env:FINAGENT_AGENT_PROVIDER='local'`；执行后恢复原值。

## 架构与安全边界

- `AgentKernel` 在 Electron 主进程中拥有会话、运行和持久化；渲染进程的 Jotai 状态只是视图缓存。会话状态使用现有按会话隔离的 atoms，避免上下文串扰。
- UI 通过客户端和白名单 IPC 调用服务，不直接访问 `fs`、子进程、凭证存储或 Pi 运行时。保持 `contextIsolation: true`、`nodeIntegration: false` 与 sandbox 边界。
- 提供商适配器 → ProviderRouter → 能力注册表/执行器 → 产品流程。业务层和 UI 消费类型化能力契约，不直接接入厂商 SDK；结果保留来源、时间与部分失败信息。
- Longbridge CLI 由用户安装，不打包进应用。沿用 `validator.ts` 的标的校验与 `executor.ts` 的 `execa('longbridge', args)` 数组参数，禁止拼接 shell 命令。
- 执行流程必须正确处理超时、取消、工具失败和终态；新增行为不要留下永久 `running` 的任务。
- 数据缺失保持空态或错误态；示例数据必须明确标注，不编造行情、财务数值、引用或证据。
- 凭证由主进程 `CredentialStore` / Electron `safeStorage` 管理，渲染进程只接收必要元数据。新增日志、诊断和 trace 使用现有脱敏机制。
- 不提交 `.env`、真实账户 fixtures、凭证、会话数据或敏感截图。技能资源加载保持路径穿越、绝对路径和符号链接逃逸防护。
- `.pi/extensions/langsmith/vendor/` 是 vendored 代码；仅在任务确实需要时修改。

## 按需查阅与交付

- 产品与架构：`docs/PRD.md`、`docs/architecture.zh-CN.md`。
- UI 与本地化：`docs/UI-SYSTEM.zh-CN.md`、`docs/I18N.md`。
- Longbridge：`docs/longbridge-auth.zh-CN.md`、`docs/api-reference.md`。
- Agent 评测：`docs/EVALUATION.md`、`docs/EVALUATION-CI.md`。
- 安全与发布：`docs/security-prompt-injection.md`、`docs/release-gates.zh-CN.md`。
- 工具/扩展工作按需使用 `pi-coding-agent`，前端/架构工作按需使用 `craft-agent-template`；不可用时说明情况，不擅自全局安装。
- 提交信息沿用中文 Conventional Commit 风格，说明目的，例如 `docs: 明确开发边界以减少接入误用`。
- PR 按 `CONTRIBUTING.md` 提供可复现的验证报告：环境、实际命令、结果及失败说明；只在可见 UI 发生变化时附截图。
- 完成时说明改动、验证结果及未验证部分；不得把已知平台差异或未执行的检查描述为通过。
