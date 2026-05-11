# SciForge / AgentServer 功能分析报告

生成时间：2026-05-11

## 1. 项目定位

SciForge 是一个面向科学研究的本地 workspace-backed Agent 工作台。它不是单纯聊天 UI，也不是某个模型 API 的前端壳，而是把用户请求、科研场景、工作区文件、AgentServer 后端、结构化 artifact、执行日志、验证结果、UI 组件和反馈修复链路组织成一个可审计的科研操作系统原型。

当前主线可以概括为：

```text
Backend-first, Contract-enforced, Capability-driven, Harness-governed
```

含义是：

- Backend-first：用户意图理解、任务规划、能力选择主要交给 AgentServer/backend。
- Contract-enforced：SciForge 用 schema、refs、WorkEvidence、ExecutionUnit、ContractValidationFailure 等结构保证输出可检查。
- Capability-driven：系统能力通过 capability manifests、skills、actions、verifiers、views、scenario packages 声明，而不是散落在 UI 逻辑里。
- Harness-governed：未来 agent 行为治理，包括上下文预算、工具预算、验证强度和修复策略，会收敛到 agent-harness profile/contract。

## 2. 本地运行结构

本地桌面当前有两个互相配合的项目：

```text
/Users/user/Desktop/SciForge
/Users/user/Desktop/AgentServer
```

SciForge 默认服务：

```text
UI:                 http://127.0.0.1:5173
Workspace Writer:   http://127.0.0.1:5174
AgentServer:        http://127.0.0.1:18080
```

配置文件：

```text
SciForge:    /Users/user/Desktop/SciForge/config.local.json
AgentServer: /Users/user/Desktop/AgentServer/openteam.json
```

当前模型配置：

```text
provider: openai-compatible
model: bailian/deepseek-v4-flash
baseUrl: http://35.220.164.252:3888/v1
```

## 3. 总体调用链路

一次普通用户请求大致走这条路径：

```text
React ChatPanel
  -> runPromptOrchestrator
  -> SciForge tool stream API
  -> Workspace Runtime Gateway
  -> Python conversation-policy
  -> Context Envelope + Capability Broker Brief
  -> AgentServer / backend
  -> task generation 或 direct response
  -> workspace task execution
  -> artifact materialization
  -> validation / repair loop
  -> UIManifest + ExecutionUnits + WorkEvidence
  -> React ResultsRenderer / interactive views
```

这条链路的关键点是：SciForge 不靠前端关键词猜用户想要什么，而是把当前场景、refs、workspace、capability brief 和模型配置交给 backend，再把 backend 输出变成可审计对象。

## 4. SciForge UI 功能

### 4.1 App Shell

SciForge UI 是 React + Vite 应用，主要负责页面布局、状态管理、配置面板、运行状态展示和结果渲染。

主要入口：

```text
src/ui/src/app/SciForgeApp.tsx
src/ui/src/app/sciforgeApp/SciForgeWorkbench.tsx
src/ui/src/app/appShell/ShellPanels.tsx
```

核心功能：

- 选择内置或 workspace scenario。
- 配置 Workspace Path、AgentServer URL、模型 provider/baseUrl/model/apiKey。
- 显示 runtime health，包括 AgentServer、workspace writer 和模型配置状态。
- 保存/读取本地 config.local.json。
- 管理 workspace state。
- 在 Dashboard、Workbench、Feedback Inbox、Component Workbench 等页面之间切换。

### 4.2 Chat / Run 面板

主要文件：

```text
src/ui/src/app/ChatPanel.tsx
src/ui/src/app/chat/*
```

功能：

- 输入自然语言科研任务。
- 上传文件并生成 uploaded artifact refs。
- 选择目标 scenario。
- 支持引用已有 artifact、run、file、object refs。
- 展示运行中的 AgentServer stream 事件。
- 展示 tool call、tool result、progress、read/write refs、failure 和 recovery 建议。
- 支持 target instance 选择，用于双实例互修。
- 展示 context window meter，避免超上下文。
- 支持 acceptance / repair / continue 相关交互。

### 4.3 Results Renderer

主要文件：

```text
src/ui/src/app/ResultsRenderer.tsx
src/ui/src/app/results/*
packages/presentation/components/*
```

功能：

- 渲染 backend response。
- 展示 artifact cards、ExecutionUnit、WorkEvidence。
- 根据 UIManifest 选择注册组件。
- 对未知 artifact 使用 unknown-artifact-inspector。
- 支持 workspace 文件 preview、raw preview、derivative preview。
- 支持 handoff controls、preview actions、artifact controls。
- 支持 provenance、refs、audit details 和 validation failure 展示。

### 4.4 Dashboard / Scenario Builder

主要文件：

```text
src/ui/src/app/Dashboard.tsx
src/ui/src/app/ScenarioBuilderPanel.tsx
packages/scenarios/core/*
```

功能：

- 展示内置 scenario 和 workspace scenario library。
- 创建/编辑 scenario draft。
- 编译 scenario package。
- 导入/导出 scenario package manifest。
- 发布、归档、恢复、删除 workspace scenario。
- 展示 scenario 的 UI slots、skill plan、tests、quality report。
- 管理 skill promotion proposals。

### 4.5 Feedback Inbox

主要文件：

```text
src/ui/src/app/sciforgeApp/FeedbackInboxPage.tsx
src/runtime/workspace-server.ts
```

功能：

- 把 UI 评论、artifact 评论、run 评论组织成 issue bundle。
- 从 workspace writer 读取 feedback issues。
- 可与 GitHub issue 同步。
- 服务双实例互修：把反馈转成另一个实例可接手的修复任务。

### 4.6 Component Workbench

主要文件：

```text
src/ui/src/app/ComponentWorkbenchPage.tsx
packages/presentation/components/*
```

功能：

- 查看已注册 UI component manifests。
- 验证 artifact/component 兼容性。
- 调试 interactive views。
- 帮助开发新的科研组件。

## 5. 内置科研场景

内置场景来自：

```text
packages/scenarios/core/src/scenarioSpecs.ts
```

### 5.1 文献证据评估

id：

```text
literature-evidence-review
```

目标：

- 根据自然语言 query 检索生命科学文献。
- 生成 paper-list artifact。
- 提取事实、推断、假设和证据等级。
- 展示 paper-card-list、evidence-matrix、notebook-timeline。

边界：

- 不做系统综述最终裁判。
- 不提取付费全文。
- 不输出临床建议。

### 5.2 结构探索

id：

```text
structure-exploration
```

目标：

- 根据 PDB ID、UniProt、蛋白/基因名或自然语言请求获取结构。
- 支持 PDB / AlphaFold DB 等结构来源。
- 生成 structure-summary artifact。
- 用 molecule-viewer / structure-viewer 展示结构。

边界：

- 不做分子动力学。
- 不做结合自由能计算。
- 不把结构观察直接升级成机制结论。

### 5.3 组学差异分析

id：

```text
omics-differential-exploration
```

目标：

- 读取 workspace 中的表达矩阵和 metadata。
- 执行差异表达分析。
- 生成 volcano、heatmap、UMAP 等 artifact。
- 保留可复现实验记录。

边界：

- 不无界处理原始 FASTQ。
- 没有明确设计矩阵时不声称 publication-grade batch correction。

### 5.4 生物医学知识图谱

id：

```text
biomedical-knowledge-graph
```

目标：

- 围绕基因、蛋白、疾病、化合物查询数据库事实。
- 生成 knowledge-graph artifact。
- 展示 graph/table/evidence 视图。
- 支持向文献、结构、组学场景 handoff。

边界：

- 缺少 connector 时返回 unsupported/failed-with-reason。
- 不把数据库事实伪装成因果证明。

## 6. Interactive Views / 科学 UI 组件

组件 registry：

```text
packages/presentation/components/manifest-registry.ts
```

当前注册了 26 个 view manifest：

```text
report-viewer
paper-card-list
evidence-matrix
execution-unit-table
notebook-timeline
record-table
graph-viewer
point-set-viewer
matrix-viewer
structure-viewer
scientific-plot-viewer
sequence-viewer
alignment-viewer
time-series-viewer
model-eval-viewer
schema-form-editor
comparison-viewer
genome-track-viewer
image-annotation-viewer
spatial-omics-viewer
plate-layout-viewer
prediction-reviewer
protocol-editor
publication-figure-builder
statistical-annotation-layer
unknown-artifact-inspector
```

这些组件的共同 contract：

- 输入是 artifact data、schema、view props 和 refs。
- 输出是可见 UI、用户选择事件、object refs。
- 组件不能直接写文件、调用 AgentServer 或执行外部动作。
- 组件可以发出 select、inspect、filter-change、annotation-add、verify-accept 等事件。
- 真实动作必须交给 action provider。
- 验证结论必须交给 verifier provider。

## 7. Workspace Writer / 本地文件功能

Workspace writer 是本地 HTTP 服务：

```text
src/runtime/workspace-server.ts
src/runtime/server/workspace-file-api.ts
```

主要 API：

```text
GET  /health
GET  /api/sciforge/config
POST /api/sciforge/config
GET  /api/sciforge/workspace/list
GET  /api/sciforge/workspace/file
POST /api/sciforge/workspace/file
POST /api/sciforge/workspace/file-action
POST /api/sciforge/workspace/open
GET  /api/sciforge/workspace/snapshot
POST /api/sciforge/workspace/snapshot
GET  /api/sciforge/preview/raw
GET  /api/sciforge/preview/descriptor
GET  /api/sciforge/preview/derivative
POST /api/sciforge/tools/run
POST /api/sciforge/tools/run/stream
```

文件能力：

- 列目录。
- 读文本文件或小型二进制 preview。
- 写文件。
- 创建文件/文件夹。
- rename。
- delete。
- reveal/open/copy path。
- 保存 workspace snapshot。
- 保存 artifacts、sessions、versions。

workspace 下常见状态目录：

```text
.sciforge/workspace-state.json
.sciforge/task-attempts/
.sciforge/capability-evolution-ledger/
.sciforge/scenarios/
.sciforge/skill-proposals/
.sciforge/evolved-skills/
.sciforge/repair-worktrees/
```

## 8. AgentServer Gateway / Runtime 功能

SciForge runtime 负责和 AgentServer 连接：

```text
src/runtime/generation-gateway.ts
src/runtime/gateway/*
```

核心功能：

- 构建 AgentServer handoff payload。
- 读取本地 config.local.json 的模型 endpoint。
- 选择 backend。
- 构建 context envelope。
- 构建 capability broker compact brief。
- 把当前 run、refs、artifacts、previous attempts、validation failures 交给 backend。
- 处理 AgentServer stream。
- 处理 direct text、generated task、fenced task、path-only task 等返回形态。
- 执行生成的 workspace task。
- 做 output validation。
- 失败时生成 ContractValidationFailure。
- 支持 compact repair、supplemental fallback、timeout resume、acceptance repair。

## 9. Capability Broker / Capability Manifest

Capability 是 SciForge 的能力声明单元。

主要文件：

```text
src/runtime/capability-manifest-registry.ts
src/runtime/capability-broker.ts
src/runtime/capability-manifest-skill-package-projection.ts
packages/contracts/runtime/capability-manifest.ts
```

能力类型覆盖：

- AgentServer generation。
- artifact resolver/read/render。
- workspace read/write。
- command/python task。
- vision observe。
- computer-use action。
- report/evidence views。
- schema verifier。
- package skills。
- package views。
- package verifiers。

关键设计：

- 默认只给 backend compact brief。
- schema/examples/repair hints lazy expansion。
- provider availability、required config、side effects、risk、validators 都进入 manifest。
- 用户显式选择能力只提高优先级，不能绕过 safety/config/budget gate。

## 10. Contract Validation / Repair 功能

SciForge 不把 backend 输出直接当成功，而是经过多层校验：

- payload schema validation。
- artifact schema validation。
- reference validation。
- UIManifest validation。
- WorkEvidence validation。
- verifier validation。
- runtime verification gate。

失败时产生：

```text
ContractValidationFailure
recoverActions
failureReason
relatedRefs
repairContext
```

这些失败会进入下一轮 context，让 backend 可以修复，而不是让 UI 静默吞掉。

## 11. WorkEvidence / ExecutionUnit / Audit

SciForge 的科研价值很大一部分在执行审计：

- ExecutionUnit：记录某一步真实执行了什么。
- WorkEvidence：记录 search/fetch/read/write/command/validate 等事实证据。
- task attempts：记录 AgentServer 生成任务、执行、失败、修复。
- capability evolution ledger：记录能力组合、验证、失败、晋升候选。
- budget debits：记录某个 capability invocation 消耗了什么预算并关联哪些 refs。

这使得“这个结论怎么来的”可以被追溯。

## 12. Skills / Actions / Observe / Verifiers

### 12.1 Skills

目录：

```text
packages/skills
```

功能：

- 用 SKILL.md 描述 agent 可选择的工作策略。
- 包含领域 skills、pipeline skills、meta skills、tool skills。
- Runtime discovery 会递归读取 skills。
- 稳定 workspace task 可晋升为 skill proposal。

### 12.2 Actions

目录：

```text
packages/actions
```

功能：

- 声明会改变外部环境的 action provider。
- 例如 GUI、浏览器、远程桌面、文件系统、notebook、外部 API、未来实验设备。
- 高风险 action fail closed。
- action 必须输出 trace，不能替代 verifier。

### 12.3 Computer Use

目录：

```text
packages/actions/computer-use
src/runtime/computer-use
```

功能：

- 通用 GUI action loop。
- 最小闭环：observe -> planner -> safety -> locate -> execute -> verify -> trace。
- 支持 click、type_text、press_key、scroll 等操作。
- 支持窗口绑定、截图、定位、执行、验证和 trace。
- 发送、删除、支付、授权、发布、上传等高风险动作需要明确确认。

### 12.4 Observe / Vision Sense

目录：

```text
packages/observe/vision
```

功能：

- 把截图/图像/视觉模态转成可审计 text-response。
- 支持视觉规划、KV-Ground 定位、视觉 grounder fallback。
- 保留 file-ref-only 视觉记忆，不内联大图。
- deepseek-v4-flash 这类普通文本模型不能作为 VLM；视觉 planner/grounder 要用支持图像输入的模型。

### 12.5 Verifiers

目录：

```text
packages/verifiers
```

功能：

- agent-rubric verifier。
- human-approval verifier fixture。
- minimal schema verifier example。
- verifier result 会进入 audit/debit refs。

## 13. Scenario Package / Skill Promotion

Scenario package 功能：

- 把一个科研任务域封装成 scenario.json、skill-plan、ui-plan、tests、quality-report。
- 支持 draft、publish、archive、restore、delete。
- 支持导入/导出 package。
- 支持 workspace-specific scenario。

Skill promotion 功能：

- 成功的 workspace task 可生成 proposal。
- 用户可以 accept/reject/archive/validate。
- accept 后复制到 `.sciforge/evolved-skills/`。
- validate 会按 manifest 再跑一次 smoke。

## 14. 双实例互修 / 自我进化

SciForge 支持两个隔离实例互相修复：

```text
A  UI 5173 / writer 5174
B  UI 5273 / writer 5274
AgentServer shared 18080
```

核心功能：

- 一个稳定实例读取另一个目标实例的 issue bundle。
- 在目标 repo 的 `.sciforge/repair-worktrees/<run>` 中创建隔离修复 worktree。
- 执行修复后产生 patch、测试日志、diff、repair result。
- stable version promote/sync-plan 负责稳定版本同步。
- repair-handoff-runner 会 fail-closed，避免执行方和目标方 workspace 路径互相包含。

## 15. AgentServer 功能

AgentServer 是 SciForge 的后端 Agent 编排层。

主要能力：

- 对外暴露统一 SDK/HTTP API。
- 支持多个 backend id。
- 统一 run/task/session/agent lifecycle。
- 标准化 stream events。
- 标准化 canonical tool primitives。
- 支持 per-request modelProvider/modelName/llmEndpoint。
- 支持 multi-stage orchestration。
- 支持 backend adapter、runtime supervisor、managed launcher。
- 支持 openteam_agent direct backend。
- 支持 Codex、Claude Code、Gemini、Hermes Agent、OpenClaw 等 adapter 路线。

主要 HTTP API：

```text
GET  /api/agent-server/agents
POST /api/agent-server/agents
GET  /api/agent-server/agents/:agentId
GET  /api/agent-server/agents/:agentId/runs
POST /api/agent-server/runs
POST /api/agent-server/runs/stream
GET  /api/agent-server/runs/:runId
```

标准事件：

```text
status
text-delta
tool-call
tool-result
permission-request
stage-result
result
error
usage-update
contextWindowState
```

支持 backend：

```text
openteam_agent
claude-code
codex
gemini
hermes-agent
openclaw
```

## 16. 验证与质量保障

常用命令：

```bash
npm run typecheck
npm run test
npm run smoke:all
npm run build
npm run verify
```

项目有大量 smoke：

- AgentServer generation。
- AgentServer unavailable diagnostics。
- Context window。
- Complex multiturn chat。
- Artifact followup。
- Compact repair。
- Supplemental fallback。
- Backend matrix。
- Direct text bridge。
- LLM endpoint。
- Timeout resume。
- Self-evolving skill。
- Repair handoff。
- Dual instance。
- Stable version registry。
- Workspace file API。
- Vision sense runtime。
- Capability broker。
- Capability budget debits。
- No legacy paths。
- Module boundaries。
- Long file budget。

这说明项目不只追求功能跑通，也很重视架构边界、无旧链路、无场景特例、长文件治理和验证链条。

## 17. 当前成熟度判断

### 已经比较完整的部分

- 本地 React 工作台。
- workspace writer 和本地持久化。
- AgentServer 对接和 stream 协议。
- config.local.json / openteam.json 配置链路。
- 内置 scenario contract。
- artifact / UIManifest / ExecutionUnit / WorkEvidence 抽象。
- 26 个 interactive view manifest。
- scenario package 与 skill proposal 机制。
- 双实例互修主路径。
- 大量 smoke/test 边界。

### 仍偏原型或研发中的部分

- 很多科研能力目前是 contract/manifest/fixture 优先，真实数据库 connector 和真实科学计算 provider 还需要进一步接入。
- Computer Use 已有完整设计和 runtime 骨架，但真实桌面 bridge、VLM planner、KV-Ground 服务需要单独配置。
- Agent harness 是明确终局方向，但仍处在逐步收敛阶段。
- 多 backend 的生产质量取决于对应 adapter、launcher、账号和本机环境。
- 项目没有最终 License，正式产品化/分发前需要补。

## 18. 使用价值

这个项目适合：

- 做科研 Agent 工作台原型。
- 做论文复现、文献证据评估、组学分析、结构分析、知识图谱等科研 workflow。
- 比较不同 backend/model 在科研任务中的表现。
- 沉淀可审计的科研执行轨迹。
- 建立 self-evolving agent/software 修复闭环。
- 作为你自己的科研自动化平台底座继续扩展。

短板是：它不是开箱即用的“成熟 SaaS 产品”，而是一个架构很完整、工程野心很大的研发原型。真正用于生产科研任务时，需要继续补真实 connector、真实 verifier、真实 VLM/KV-Ground 配置、以及稳定的 backend runtime。
