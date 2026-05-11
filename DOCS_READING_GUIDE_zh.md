# SciForge docs/ 导读

生成时间：2026-05-11

## 0. 一句话理解 docs/

`docs/` 不是普通用户手册，而是 SciForge 的项目级“架构真相源”。模块内部细节仍然放在各 package 的 README 或源码旁边，`docs/` 只解释项目级边界、运行链路、扩展方式、Agent Harness 标准和多轮对话恢复机制。

当前 `docs/` 下的核心文档：

```text
docs/README.md
docs/Usage.md
docs/Architecture.md
docs/AgentHarnessStandard.md
docs/Extending.md
docs/SciForgeConversationSessionRecovery.md
```

## 1. 推荐阅读顺序

### 第一遍：先建立全局地图

```text
1. docs/README.md
2. docs/Usage.md
3. docs/Architecture.md
```

读完这三篇，你应该知道：

- SciForge 是什么。
- 怎么启动。
- 主要服务有哪些。
- 用户请求如何从 UI 走到 AgentServer。
- workspace、artifact、ExecutionUnit、WorkEvidence 是什么。
- 为什么项目强调 backend-first、contract-enforced、capability-driven。

### 第二遍：理解怎么扩展

```text
4. docs/Extending.md
```

读完你应该知道：

- 如何新增 scenario package。
- 如何新增 capability。
- observe/action/verifier/view/skill 分别应该放哪里。
- `src/` 和 `packages/` 的边界。
- 为什么不能随便在 UI 或 runtime 里硬编码场景语义。

### 第三遍：理解下一阶段研究重点

```text
5. docs/AgentHarnessStandard.md
6. docs/SciForgeConversationSessionRecovery.md
```

读完你应该知道：

- Agent Harness 要解决什么问题。
- HarnessRuntime / HarnessProfile / HarnessCallback / HarnessContract / HarnessTrace 的关系。
- 多轮对话里当前请求、历史、refs、digest、recovery 如何协同。
- Python conversation-policy 和 TypeScript runtime 怎么分工。

## 2. docs/README.md 讲什么

定位：

```text
项目级文档索引 + 代码真相源索引
```

它告诉你：

- 哪些文档是权威文档。
- 哪些代码文件是实现真相源。
- 当前项目状态是什么。
- 哪些 smoke guard 代表当前架构已经收敛。

最重要的一段：

```text
capability registry 是能力真相源
harness policy 是行为治理真相源
runtime gateway 是生命周期和 enforcement 真相源
agent backend 是推理和组合真相源
```

这句话基本就是理解 SciForge 的钥匙。

对应代码：

```text
package.json
src/ui/src/config.ts
src/ui/src/api/sciforgeToolsClient.ts
src/runtime/workspace-server.ts
src/runtime/generation-gateway.ts
packages/scenarios/core/src
packages/contracts/runtime/capabilities.ts
packages/presentation/components/README.md
packages/skills/README.md
packages/observe/vision/README.md
packages/actions/computer-use/README.md
```

## 3. Usage.md 讲什么

定位：

```text
如何启动、配置、使用和运维
```

核心内容：

- 快速启动。
- 配置字段。
- 常用工作流。
- workspace 产物。
- 双实例互修。
- Computer Use。
- Skill 晋升。
- 验证命令。

默认服务：

```text
UI:               http://127.0.0.1:5173
Workspace writer: http://127.0.0.1:5174
AgentServer:      http://127.0.0.1:18080
```

配置字段重点：

```text
agentServerBaseUrl
workspaceWriterBaseUrl
workspacePath
agentBackend
modelProvider
modelBaseUrl
modelName
apiKey
requestTimeoutMs
maxContextWindowTokens
peerInstances
feedbackGithubRepo
feedbackGithubToken
```

常用工作流重点：

```text
ChatPanel
  -> runPromptOrchestrator
  -> sendSciForgeToolMessage
  -> /api/sciforge/tools/run/stream
  -> runWorkspaceRuntimeGateway
  -> Python conversation-policy
  -> context envelope + capability broker brief
  -> AgentServer/backend
  -> validation / repair loop
  -> ToolPayload + artifacts + ExecutionUnits
```

你读这篇时要重点理解：

- SciForge 不只是前端，它有 workspace writer。
- 所有长期状态都应该沉淀到 workspace 的 `.sciforge/`。
- 真实请求最终会走 AgentServer/backend。
- 失败和修复是正常工作流的一部分。

## 4. Architecture.md 讲什么

定位：

```text
项目架构总说明
```

这篇是最重要的文档。它回答：

- SciForge 当前边界是什么。
- 为什么 SciForge 不是第二套 agent。
- Backend-first Capability Architecture 是什么。
- Harness-governed Scientific Agent OS 是什么。
- `src/` 和 `packages/` 怎么分工。
- Runtime 请求链路怎么走。
- AgentServer Contract 是什么。
- Conversation Policy 如何工作。
- WorkEvidence 和 Task Project 如何审计。
- Context 和恢复机制如何设计。
- Feedback 与双实例互修边界是什么。
- Vision/Computer Use 兼容面在哪里。

### 4.1 当前边界

SciForge 当前是：

```text
本地 workspace-backed 科研 Agent 工作台
```

它不应该：

- 维护硬编码回复模板。
- 通过 UI 关键词猜用户语义。
- 为某个 prompt、provider、scenario 写特殊分支。

它应该：

- 组织用户请求。
- 组织 workspace refs。
- 组织 scenario contract。
- 组织 capability brief。
- 组织 backend stream。
- 持久化 artifact、ExecutionUnit、WorkEvidence。
- 把 validation failure 结构化返回给 backend 修复。

### 4.2 Backend-first Capability Architecture

核心链路：

```text
User intent
  -> agent backend understands the goal
  -> capability broker provides compact relevant capability briefs
  -> backend selects capabilities and writes glue code when needed
  -> SciForge runtime executes capabilities behind safe boundaries
  -> validators check protocol, refs, artifacts and evidence
  -> validation failures return to backend as repair context
  -> stable artifacts, refs, views and evidence are persisted
```

理解重点：

- 后端负责理解和规划。
- SciForge 负责声明能力、执行边界、验证和持久化。
- UI 不应该成为语义路由层。
- Scenario package 只做 policy，不写执行代码或 prompt 特例。

### 4.3 Harness-governed Scientific Agent OS

这是项目下一阶段核心方向。

它要解决的问题不是“系统有什么能力”，而是：

```text
每一轮 Agent 应该如何使用能力
```

例如：

- 应该读多少历史。
- 应该给多少工具预算。
- 应该用哪些 capability。
- 应该多强验证。
- 失败时修复、补充、人工确认还是 fail closed。
- 用户应该看到什么进度。

### 4.4 src 与 packages 边界

最重要规则：

```text
src/       回答“系统怎么运行”
packages/  回答“系统能做什么”
```

`src/` 适合：

- app shell
- workspace writer
- runtime server
- request/stream transport
- backend run lifecycle
- capability broker 主流程
- validation / repair loop
- artifact persistence
- permission / safety boundary

`packages/` 适合：

- observe providers
- skills
- actions
- verifiers
- views
- scenario packages
- importers / exporters
- capability manifests
- schemas
- validators
- repair hints

## 5. Extending.md 讲什么

定位：

```text
怎么给 SciForge 增加新能力
```

它是开发者最该反复看的文档。

### 5.1 Scenario Package

Scenario package 是一个可复用科研场景包，结构包括：

```text
scenario
skillPlan
uiPlan
validationReport
qualityReport
tests
versions
```

workspace 位置：

```text
<workspace>/.sciforge/scenarios/<safe-id>/
```

可以单文件存：

```text
package.json
```

也可以拆分存：

```text
scenario.json
skill-plan.json
ui-plan.json
tests.json
versions.json
validation-report.json
quality-report.json
```

### 5.2 Capability Brief

能力以 `CapabilityManifest` 为统一真相源。

能力类型包括：

```text
observe
skill
composed
action
verifier
view
memory
importer
exporter
runtime-adapter
```

默认给 backend 的只是 compact brief，不给 full schema、examples、repair hints 或完整日志。选中能力后再 lazy expansion。

### 5.3 Observe / Action / Verifier

Observe：

- 只读观察。
- 不产生副作用。
- 例如 vision、OCR、网页/文件观察。
- 输入输出都应走 refs 和 bounded text。

Action：

- 会改变外部环境。
- 例如 workspace task、命令运行、Computer Use、文件写入。
- 必须记录副作用边界、stdout/stderr、失败原因和可恢复动作。

Verifier：

- 负责验证结果。
- verdict 包括 pass、fail、uncertain、needs-human、unverified。
- 高风险、外部副作用、科学 claim 应显式进入 verifier。

### 5.4 UIManifest 与 View Composition

UIManifest slot 告诉前端：

```text
用哪个 component 渲染哪个 artifact
```

核心字段：

```text
componentId
title
props
artifactRef
priority
encoding
layout
selection
sync
transform
compare
```

组件只负责显示和交互事件，不负责写 workspace、调用 AgentServer 或做 verifier verdict。

### 5.5 Skill 与 Promotion

Skill 是 agent 可选的工作策略。稳定任务可以晋升为 skill：

```text
successful task
  -> skill proposal
  -> user accept
  -> validation smoke
  -> .sciforge/evolved-skills/
```

## 6. AgentHarnessStandard.md 讲什么

定位：

```text
未来 Agent 行为治理的编程标准
```

它把 Agent Harness 类比 PyTorch Lightning：

- Runtime 主循环稳定。
- 策略通过 callbacks/profiles 注入。
- 每次策略选择有 trace。
- 不再到处改 prompt、gateway、UI、repair 分支。

### 6.1 核心对象

```text
HarnessRuntime
HarnessProfile
HarnessCallback
HarnessEvaluation
HarnessContract
HarnessTrace
HarnessDecision
HarnessContext
```

最重要的是：

```text
HarnessContract 是本轮唯一行为契约
```

Context builder、broker、prompt renderer、validator、repair loop 和 UI 都应该消费这个 contract。

### 6.2 分级 hooks

文档把 hook 分成 8 层：

```text
Level 0: Runtime Lifecycle
Level 1: Planning
Level 2: Capability Planning
Level 3: Dispatch
Level 4: Execution
Level 5: Validation and Repair
Level 6: UX and Interaction
Level 7: Audit and Research
```

你可以把它理解成：

```text
请求进入
  -> 规划
  -> 选能力
  -> 派发
  -> 执行
  -> 验证/修复
  -> 用户交互
  -> 研究审计
```

### 6.3 为什么它重要

如果未来要发 AI conference 论文，这篇文档里的思想很可能就是理论核心之一：

```text
Contract-governed scientific agents
Harness-governed agent behavior
Auditable capability-budgeted scientific workflows
```

## 7. SciForgeConversationSessionRecovery.md 讲什么

定位：

```text
多轮对话、session 恢复、上下文选择和 Python 策略层
```

一句话模型：

```text
SciForge 把对话看成一条可恢复的研究时间线
```

每轮用户消息会经过：

```text
UserGoalSnapshot
  -> Python policy engine
  -> goal/context/memory/reference/capability/handoff/recovery/userVisiblePlan
  -> TypeScript runtime
  -> AgentServer
  -> ToolPayload
  -> session state
```

最重要原则：

```text
当前用户请求永远是主语
历史只能当证据，不能反客为主
```

### 7.1 核心对象

```text
Session
Message
Run
Artifact
ExecutionUnit
Reference
Conversation Ledger
Current Reference Digest
Capability Brief
Process Progress
Acceptance
```

### 7.2 Python 策略层

当前策略算法主要在：

```text
packages/reasoning/conversation-policy/
```

模块职责：

```text
goal_snapshot.py
context_policy.py
memory.py
reference_digest.py
artifact_index.py
capability_broker.py
handoff_planner.py
acceptance.py
recovery.py
process_events.py
```

这篇文档尤其适合理解：

- 为什么不把全历史塞进模型。
- 为什么大文件走 digest。
- 为什么 failure 也要可恢复。
- 为什么运行过程必须用户可见。
- 为什么 TypeScript 不应该再写一套语义判断算法。

## 8. 一张总心智图

```text
User request
  -> UI app shell
  -> workspace writer / stream API
  -> conversation-policy
       goal / context / memory / refs / acceptance / recovery
  -> runtime gateway
       context envelope / capability broker / payload validation
  -> AgentServer backend
       reasoning / planning / task generation / tool calls
  -> SciForge runtime execution
       workspace task / artifact materialization / WorkEvidence
  -> validation and repair
       ContractValidationFailure / repair context / supplement
  -> UI rendering
       UIManifest / interactive views / object refs
  -> feedback and evolution
       comments / issue bundle / repair handoff / skill promotion
```

## 9. 你现在应该抓住的 10 个关键词

1. `workspace-backed`：本地 workspace 是长期事实源。
2. `AgentServer`：backend 推理和任务生成入口。
3. `CapabilityManifest`：能力的唯一真相源。
4. `Capability Broker`：把当前轮相关能力以 compact brief 给 backend。
5. `HarnessContract`：未来每轮 agent 行为的唯一契约。
6. `ToolPayload`：backend 输出归一后的运行结果。
7. `Artifact`：结构化科研产物。
8. `ExecutionUnit`：可审计执行单元。
9. `WorkEvidence`：证明系统实际做过什么的证据。
10. `ContractValidationFailure`：失败不是崩溃，而是可修复上下文。

## 10. 适合你的阅读路线

### 如果你想先跑起来

```text
docs/Usage.md
README.md
package.json scripts
```

### 如果你想理解架构

```text
docs/README.md
docs/Architecture.md
docs/SciForgeConversationSessionRecovery.md
```

### 如果你想扩展科研场景

```text
docs/Extending.md
packages/scenarios/core/src/scenarioSpecs.ts
packages/scenarios/core/src/scenarioPackage.ts
packages/presentation/components/README.md
packages/skills/README.md
```

### 如果你想做 AI 会议论文

```text
docs/AgentHarnessStandard.md
docs/Architecture.md
docs/SciForgeConversationSessionRecovery.md
src/runtime/capability-broker.ts
packages/contracts/runtime/capability-manifest.ts
packages/reasoning/conversation-policy/
```

### 如果你想做自动科研 / 论文复现

```text
docs/Usage.md
docs/Extending.md
docs/SciForgeConversationSessionRecovery.md
packages/scenarios/core/src/scenarioSpecs.ts
packages/skills/
packages/verifiers/
packages/presentation/components/
```

## 11. 我的理解建议

不要把 SciForge 先理解成“一个 App”。更准确的理解是：

```text
SciForge = 科研对象工作台 + AgentServer gateway + capability contract system + audit/repair/evolution loop
```

它真正有价值的地方不是 UI，也不是模型调用，而是：

- 科学对象结构化。
- Agent 行为可治理。
- 执行结果可验证。
- 失败可修复。
- 技能可晋升。
- 系统自身可被另一个实例修复。

如果你后面要拿它做科学复现或 AI 论文，最应该深挖的是：

```text
paper -> claim -> data -> task -> artifact -> evidence -> validation -> repair -> reusable skill
```

这条链路正好穿过 docs/ 里的所有核心概念。
