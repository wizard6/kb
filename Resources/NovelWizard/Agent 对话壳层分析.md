---
title: NovelWizard Agent 对话壳层分析
created: 2026-06-24
tags: [architecture, chat-shell-ui, agent, ui, analysis]
source: packages/chat-shell-ui/src 源码分析
status: active
para: Resources/NovelWizard
---

# NovelWizard Agent 对话壳层分析

> 对 `@novel-wizard/chat-shell-ui` 包的设计分析。
> 这是整个项目中最被低估的包——它实际上实现了一套完整的 **Agent UI 框架**。

---

## 一、包定位

```
描述: 对话壳层 UI · 会话控制器 · Agent 模块 · 观测台
用途: gui-dev / gui-console 共用
```

这是一个 **Vue 3 组件包**，提供可复用的 Agent 对话界面。任何应用只要接入这个包，就能获得：

- 一个完整的 AI 对话界面
- 多 Agent 切换（灵感创作/规划器/开发测试/壳层桩）
- 每轮对话的**链路追踪可视化**
- 管线步骤开关调试
- 观测台（上下文查看、路由分析）

---

## 二、整体架构

```
┌── 应用层（gui-dev / gui-console）─────────────┐
│  configureChatShellUi(ports)  ← 注入端口        │
├── chat-shell-ui 包 ─────────────────────────────┤
│                                                  │
│  ┌─ Ports（端口层）────────────────────────────┐ │
│  │  运行时端口（dispatchTurn / postChat 等）    │ │
│  │  Agent 注册表端口                            │ │
│  │  链路执行端口                                 │ │
│  └──────────────────────────────────────────────┘ │
│                                                  │
│  ┌─ Agents（Agent 层）─────────────────────────┐ │
│  │  5 个内置 Agent                              │ │
│  │  每个 Agent = runTurn + buildShellView        │ │
│  │  统一注册表（registry.ts）                    │ │
│  └──────────────────────────────────────────────┘ │
│                                                  │
│  ┌─ Shell（壳层 UI 层）────────────────────────┐ │
│  │  观测台（shellView 构建）                     │ │
│  │  链路步骤显示（linkStepDisplay）              │ │
│  │  Trace 构建                                   │ │
│  └──────────────────────────────────────────────┘ │
│                                                  │
│  ┌─ Composables（Vue 组合式）───────────────────┐ │
│  │  会话控制器（session controller）             │ │
│  │  管线设置                                    │ │
│  │  浮动面板                                    │ │
│  └──────────────────────────────────────────────┘ │
│                                                  │
│  ┌─ LinkEngine（链路图引擎）───────────────────┐  │
│  │  声明式链路图执行                              │ │
│  │  约束检查                                    │ │
│  │  图存储                                      │ │
│  └──────────────────────────────────────────────┘ │
│                                                  │
│  ┌─ Trace（追踪）───────────────────────────────┐│
│  │  Pipeline 追踪 + 可视化                       ││
│  │  步骤级开关                                   ││
│  └──────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

---

## 三、核心设计模式：端口注入（Ports）

`chat-shell-ui` 不直接依赖运行时。它定义端口接口，由宿主应用注入：

```typescript
// ports/configure.ts
interface ChatShellRuntimePort {
  dispatchWorkshopTurn: (input) => Promise<Result>;
  sendPlannerMessage: (input) => Promise<Result>;
  createAgentRuntime: (options) => ChatAgentRuntime;
  agentRegistry: AgentRegistryPort;
  // ...
}

// 应用层注入：
configureChatShellUi({
  runtime: { dispatchWorkshopTurn, sendPlannerMessage, ... },
  onlineAgents: { onlineAgentIds, loadOnlineAgentIds },
  workbenchContext: { getContext, getSessionId },
});
```

| 端口 | 谁实现 | 用途 |
|------|--------|------|
| `ChatShellRuntimePort` | chat-runtime 包 | 对话分发、Agent 注册表 |
| `ChatShellOnlineAgentsPort` | 应用层 | Agent 在线状态管理 |
| `ChatShellWorkbenchContextPort` | 应用层 | 种子/会话上下文 |
| `ChatShellLinkExecutionPort` | 应用层 | 链路执行回调 |

**评价**：✅ 这是正确的解耦方式。chat-shell-ui 不关心运行时实现，只关心接口。

---

## 四、Agent 模块系统

### 4.1 Agent 定义

每个 Agent 是一个 `ChatAgentModule`：

```typescript
interface ChatAgentModule {
  id: string;                    // Agent ID
  definition: ChatAgentDefinition;  // runTurn + buildShellView
  traceProfile: ChatShellTraceProfile;  // 追踪配置
  shellMeta: ChatShellMeta;      // 壳层元数据
}

interface ChatAgentDefinition {
  id: string;
  label: string;                 // 显示名
  description: string;           // 描述
  personas: AgentPersonaOption[]; // 可选人格
  defaultPersonaId: string;
  runTurn: (input) => Promise<ChatAgentTurnResult>;  // 核心：执行一轮对话
  buildShellView: (input) => ChatShellViewModel;      // 核心：构建 UI 视图
}
```

### 4.2 5 个内置 Agent

| Agent | ID | 用途 | runTurn | 观测台 UI |
|-------|-----|------|---------|-----------|
| **灵感激荡** | `workshop-discussion` | 默认创作 Agent | dispatchWorkshopTurn | 完整对话 + 追踪树 |
| **规划器** | `planner-agent` | 解析意图生成计划 | sendPlannerMessage | 计划 + 步骤列表 |
| **壳层路由** | `shell-router` | 切换 Agent | N/A（路由） | Agent 选择面板 |
| **壳层桩** | `blank-shell` | UI 流程验证 | runBlankShellTurn（桩回复） | 最小化 UI |
| **开发测试** | `dev-test-agent` | Skill/管线调试 | sendDevTestAgentMessage | 全链路追踪 + FlowMap |

### 4.3 runTurn + buildShellView 分离

这是最核心的设计决策。每个 Agent 同时提供：

```
runTurn(input) → ChatAgentTurnResult
  ↑ 这是 Agent 的「脑子」：它怎么处理输入、调 AI、返回结果

buildShellView(input) → ChatShellViewModel
  ↑ 这是 Agent 的「脸」：它在界面上长什么样
```

**为什么这么设计**：
- runTurn 是纯逻辑，可以单元测试
- buildShellView 是 UI 纯函数，输入 turns → 输出视图模型
- 两者通过 `ChatAgentTurnResult`（含 turns / error / draft）通信
- 不依赖 Vue、不依赖 DOM，纯数据驱动

---

## 五、会话控制器（Session Controller）

### 5.1 接口

```typescript
interface ChatShellSessionController {
  turns: Ref<DevChatTurn[]>;           // 对话历史
  draft: Ref<string>;                  // 当前输入框
  running: Ref<boolean>;               // 是否正在执行
  error: Ref<string | null>;           // 错误信息
  shellView: Ref<ChatShellViewModel>;  // 当前的 UI 视图

  sendMessage(): Promise<void>;        // 发送消息
  clearTurns(): void;                  // 清空对话
  unloadTurn(index: number): void;     // 撤回某轮
  reloadTurn(index: number): void;     // 重试某轮
  openDialog(): void;
  closeDialog(): void;

  // 链路执行
  linkExecution: ChatShellLinkExecutionPort;
}
```

### 5.2 创建方式

```typescript
// 方式 1：按 Agent ID（推荐）
const session = useChatShellSession({ agentId: 'workshop-discussion' });

// 方式 2：自定义
const session = createChatShellSession({
  agent: myCustomAgent,
  traceProfile: myTraceProfile,
  shellMeta: myShellMeta,
});

// 方式 3：多 Agent（开发者观测台）
const session = createMultiAgentChatShellSession({
  agentIds: ['workshop-discussion', 'dev-test-agent', 'blank-shell'],
  defaultAgentId: 'workshop-discussion',
  shellMeta: BLANK_SHELL_META,
});
```

---

## 六、链路执行（LinkEngine）

这是一个值得注意的组件——它已经有「声明式图执行」的雏形：

```
linkEngine/
├── types.ts         （节点/边的类型定义）
├── executor.ts      （图执行器）
├── constraints.ts   （约束检查）
└── graphStorage.ts  （图持久化）
```

当前它用于 dev-test-agent 的链路步骤展示（linkStepDisplay），而不是完整的流程执行。但它的类型设计**与 Agent 工作流引擎设计文档中的 FlowDefinition 非常接近**——这是一个可复用的基础。

---

## 七、可观测性设计

### 7.1 Pipeline Trace

```
trace/pipeline/
├── index.ts                 （主入口）
├── registry.ts              （追踪模块注册表）
├── types.ts                 （追踪数据类型）
├── buildTrace.ts            （追踪构建）
├── buildTraceRailTree.ts    （追踪树可视化）
├── context.ts               （上下文追踪）
├── formatTraceValue.ts      （值格式化）
├── intentDecompose.ts       （意图分解追踪）
├── intentRoute.ts           （意图路由追踪）
├── providerPrefs.ts         （Provider 配置追踪）
├── sanitizeUserText.ts      （文本清洗追踪）
├── tracePipelineSettings.ts （管线设置）
├── traceRoute.ts            （路由追踪）
├── resolveTraceStepVisualKind.ts（步骤可视化类型）
└── modules/                 （各步骤模块）
    ├── sanitize.ts
    ├── intentDecompose.ts
    ├── intentHeuristic.ts
    ├── intentSmart.ts
    ├── flowRoute.ts
    ├── skillSelect.ts
    ├── knowledgeRetrieve.ts
    ├── capabilityUse.ts
    ├── reply.ts
    ├── request.ts
    ├── validate.ts
    ├── persistInfo.ts
    └── assemble.ts
```

每个步骤模块独立追踪输入/输出，在 gui-dev 中展示为追踪树或"铁路图"（rail tree）。

### 7.2 步骤开关

与 `RuntimeStepGate` 联动，观测台中可关掉某一步的追踪显示。

---

## 八、设计评价

### ✅ 做得好的

| 设计 | 评价 |
|------|------|
| **Ports 端口模式** | 正确的依赖倒置。chat-shell-ui 不依赖运行时实现 |
| **runTurn + buildShellView 分离** | 逻辑与 UI 解耦。Agent 可以无头运行（仅测试）、也可以带 UI 运行 |
| **可组合的会话控制器** | 支持单 Agent、多 Agent、自定义 Agent 三种模式 |
| **LinkEngine 的早期存在** | 已经有了图执行的基础，可以扩展为完整的 FlowDefinition 引擎 |
| **Trace 模块化** | 每个管线步骤独立追踪注册 |

### ⚠️ 需要关注的

| 问题 | 说明 |
|------|------|
| **包体积大** | `chat-shell-ui` 依赖 Vue 3、Naive UI，且包含 workshop-discussion 等所有 Agent 的 UI。如果只想用「空白壳层」，仍然引入了全部 |
| **LinkEngine 未用满** | 当前只用于 linkStepDisplay（展示），没有用于真正的流程执行。它的 executor.ts 有能力驱动节点执行，但没被调用 |
| **观测台与产品 UI 未分离** | 观测台代码（trace/observatory）与壳层代码混在一起。如果是产品环境，不应该包含观测台代码 |
| **Agent 模块在 UI 层** | Agent 的 `runTurn` 实现在 chat-shell-ui 包中，理论上应该在 chat-runtime 包。这意味着如果想换 UI 框架（Vue → React），Agent 逻辑也要搬 |

### 核心设计问题：壳层与 Agent 的耦合

> **你想解除的不是「壳层不知道 Agent」，而是「壳层可以显示任何 Agent」——即壳层不需要在代码里硬编码每个 Agent 长什么样。**

当前 `chat-shell-ui/agents/` 存在的根本矛盾：

| 当前状态 | 问题 |
|----------|------|
| 每个 Agent 把 `runTurn`（业务）和 `buildShellView`（UI）打包在一起 | 业务逻辑被困在 UI 层 |
| 壳层硬编码 5 个 Agent 的 buildShellView | 新增 Agent 必须改壳层代码 |
| Agent 的视图代码在壳层包内部 | 换 UI 框架需要搬整个 Agent 定义 |

正确的解耦方向不是「壳层完全不知道 Agent」——因为壳层显示的内容**本质就是 Agent 运行时的可视化**，不可能完全无关。真正的目标是：**壳层只认 `agentId → viewBuilder` 的映射，不认具体的 Agent 实现**。

```typescript
// 壳层核心——不认识任何具体 Agent
function renderChatShell(agentId: string, data: ChatAgentObservatoryInput) {
  const buildView = agentViewRegistry[agentId];  // 查表
  if (!buildView) return renderGenericView(data); // 兜底通用视图
  return buildView(data);
}
```

这需要：

1. `runTurn`（业务）迁移到 `agent-definitions` 包，无 UI 依赖
2. `buildShellView`（UI）留在 `chat-shell-ui` 但作为**可注册的视图插件**
3. 壳层核心只维护 `agentViewRegistry: Record<string, AgentViewBuilder>`
4. 应用层在 `configureChatShellUi()` 时注入哪些 Agent 视图可用

### 与 Agent 工作流引擎设计的关系

当前的 `chat-shell-ui` 恰好为新的引擎设计提供了**坚实的 UI 基础**：

| 新引擎需要的 | chat-shell-ui 已有什么 |
|-------------|----------------------|
| 节点执行可视化 | ✅ Trace + linkStepDisplay |
| Agent 注册表 | ✅ registry.ts |
| 会话控制器 | ✅ session controller |
| 图执行器 | ⚠️ LinkEngine（已存在，需扩展） |
| 人类中断 UI | ❌ 需要新增 |
| 可视化编辑器 | ❌ 需要新增 |
## 九、壳层功能清单

此处为 `chat-shell-ui` 包提供的全部功能，按模块分组。来源：`src/index.ts` 导出 + 源码分析。

### 9.1 会话管理（Session Controller）

```typescript
// 创建会话
createChatShellSession(agent, traceProfile, shellMeta)   // 自定义单 Agent 会话
createMultiAgentChatShellSession({ agentIds, ... })       // 多 Agent 切换
createInspirationWorkshopSession()                        // 产品灵感工坊

// 会话控制
session.openDialog() / .closeDialog()           // 开/关对话框
session.sendMessage()                           // 发送 → runTurn → 追加 turns
session.clearTurns()                            // 清空对话
session.unloadTurn(id) / .reloadTurn(id)        // 撤回/重试某轮

// 状态（均为 Ref，Vue 响应式）
session.turns          // DevChatTurn[] 对话历史
session.draft          // string 当前输入
session.running        // boolean 是否执行中
session.error          // string | null 错误
session.shellView      // ChatShellViewModel 自动计算的 UI 模型
session.personaId      // 当前人格 ID
session.personas       // 可选人格列表
```

### 9.2 视图渲染（View Building）

纯函数，输入 turns → 输出 UI 数据模型，无 Vue 依赖。

```typescript
// 主对话区
buildDevChatMainView({ turns, running, personaId })
  → DevChatMainView = DevChatWelcomeView | DevChatThreadView
  // 无对话时：欢迎页（标题 + 引导 + 快捷提示词）
  // 有对话时：消息线程（turn 列表 + typing 指示器）

// 单条气泡
buildChatTurnRowView(turn, { personaLabel, ... })
  → DevChatTurnRowView = { role, avatarGlyph, content, bubbleClass, foot, linkStepDisplay }

// 上下文面板
buildDevChatContextView({ turns, personaId, seed, draft, docRawInput, agentId })
  → DevChatContextItem[] = 分层上下文列表
  // system-prompt / tool-defs / rules / skills / mcp / subagents /
  // external(seed+doc) / summary / conversation / pending

// 上下文拆分
splitDevChatContextItems(items)
  → { system, layers[], turns[], pending, nextCount, historyWindowStart, usage }
  // usage = token 用量仪表盘（{ usedTokens, budgetTokens, percent, slices[] }）
```

### 9.3 Trace 追踪（Observability）

```typescript
// 步骤开关
useTracePipelineSettings(traceProfile)
  → { isStepEnabled, disabledSet, toggleStep, resetAllEnabled, disabledIds }

// 路径追踪
useTraceFlowPath()
  → traceFlowPath 当前路径
  → setTraceFlowPath / clearTraceFlowPath / isTraceFlowPath

// 浮动面板
useDevFloatingPanel()
  → open / close / toggle / isOpen

// Vue 组件
DevChatTraceCanvas         // 追踪画布
DevChatTraceFlowMap        // FlowMap 链路图
DevChatTraceRailTree       // 铁路树
DevChatTraceModuleDetail   // 步骤详情
DevChatTraceSchemeView     // 概览
DevChatTraceSettingsDialog // 设置对话框
```

### 9.4 链路执行（Link Engine）

```typescript
// 链路执行端口
createChatShellLinkExecution({ turns, running, error, openDialog, nextTurnId })
  → ChatShellLinkExecutionPort
  → runLinkNode(nodeId, opts)         // 执行单个节点
  → runLinkPipeline(nodeIds, opts)    // 执行整条管线
  → compareLinkNode(a, b, opts)       // 对比两次执行

// 链路步骤展示
buildLinkStepTurn(turn)              // 构建折叠行
createLinkStepDisplayPort()          // 创建展示端口
publishLinkNodeStep(nodeId, data)    // 发布步骤数据
getLinkStepDisplayPort()             // 获取已有端口
```

### 9.5 Agent 管理（Agent Registry）

```typescript
// 注册表
AGENT_MODULES            // 5 个内置 Agent 模块（workshop-discussion / planner-agent / shell-router / blank-shell / dev-test-agent）
getAgentModule(id)       // 按 ID 获取完整模块
getChatAgent(id)         // 按 ID 获取 Agent 定义
listChatAgents()         // 列出所有 Agent
listAgentModules()       // 列出所有模块

// Persona
getPersonasForAgent(id)         // Agent 的可选人格
getDefaultPersonaForAgent(id)   // 默认人格
resolveAgentPersona(agentId, personaId)
buildAgentPersonaSystemBlock(agentId, personaId)

// 上线门控
onlineAgentIds          // 当前在线 Agent ID 列表（响应式）
loadOnlineAgentIds()    // 刷新在线列表

// 会话注册
registerDevChatAgentSession(agentId, turns, sessionId)   // 注册
unregisterDevChatAgentSession(agentId, sessionId)         // 注销
```

### 9.6 端口注入（Ports）

```typescript
// 宿主注入
configureChatShellUi({
  runtime: ChatShellRuntimePort,                   // dispatchWorkshopTurn / sendPlannerMessage / createAgentRuntime
  onlineAgents?: ChatShellOnlineAgentsPort,        // onlineAgentIds / loadOnlineAgentIds
  workbenchContext?: ChatShellWorkbenchContextPort, // getContext / getSessionId
  agentSessionRegistry?: ChatShellAgentSessionRegistryPort,  // register / unregister
  createLinkExecution?: (host) => ChatShellLinkExecutionPort,
})

// 读取注入的端口
getChatShellUiPorts()              // 抛出异常如果没有注入
tryGetChatShellUiPorts()           // 返回 null 如果没有注入
tryGetChatShellOnlineAgents()
tryGetChatShellWorkbenchContext()
tryGetChatShellAgentSessionRegistry()
```

### 9.7 不提供的功能

以下功能不属于壳层，由宿主应用通过 ports 注入：

| 功能 | 由谁提供 | 通过哪个 port |
|------|---------|-------------|
| 调 AI | chat-runtime | `runtime.dispatchWorkshopTurn` |
| 调 Skill | chat-runtime / skills | `runtime.dispatchWorkshopTurn` |
| 知识检索 | chat-runtime | `runtime` (KnowledgeRetriever) |
| 种子上下文 | 应用层 | `workbenchContext.getContext()` |
| Agent 在线状态 | 应用层 | `onlineAgents.loadOnlineAgentIds()` |
| DB 读写 | 应用层 | 不经过壳层 |
| 业务决策 | 应用层 | 不经过壳层 |

### 当前问题：壳层直接写状态

从 `useChatShellSession.ts` 中发现的直接写操作：

```typescript
// 壳层直接修改 Agent 状态（违反单向数据流）
turns.value = [];                                    // clearTurns
turns.value = turns.value.map(t => ({ ...t, ... })); // patchTurn
turns.value = result.turns;                           // sendMessage 后写回
turns.value = await agent.validatePair(...);          // validatePair 后写回
```

### 正确方向：Agent 管理状态，壳层只读

```typescript
// Agent 拥有自己的状态
interface AgentState {
  turns: ChatTurn[];
  draft: string;
  status: 'idle' | 'running' | 'error';
  error: string | null;
}

// Agent 提供操作方法（副作用由 Agent 自己管理）
interface AgentController {
  getState(): Readonly<AgentState>;
  sendMessage(draft: string): Promise<void>;
  clearTurns(): void;
  patchTurn(turnId: string, patch: Partial<ChatTurn>): void;
  onStateChange(callback: (state: AgentState) => void): void;
}

// 壳层只读取状态，不直接修改
function renderShell(agent: AgentController) {
  // 读取
  const state = agent.getState();
  
  // 渲染（只读）
  return buildChatShellView(state);
  
  // 用户操作 → 通知 Agent 执行
  // agent.sendMessage(text);  // ← 由 Agent 自己管理状态变更
  // agent.clearTurns();       // ← 同上
}
```

### 数据流

```
用户操作（点击发送、撤回、清空）
  → 壳层调用 agent.sendMessage() / agent.clearTurns()
    → Agent 修改自己的状态（turns / running / error）
      → Agent 触发 onStateChange 回调
        → 壳层重新渲染（只读读取最新状态）
```

这样：
- Agent 可以在无头环境运行（CLI / 测试 / API），不依赖壳层
- 壳层只是一个「状态显示器」，不拥有也不直接修改状态
- 状态变更路径唯一，可追踪、可测试

### 最终原则：壳层是纯 UI，零逻辑

> **壳层仅是一个简单的 UI，不涉及任何的逻辑处理。**

这意味着壳层不应该包含：

| 不应该有的 | 例子 | 说明 |
|-----------|------|------|
| ❌ 状态管理 | `ref()`, `computed()`, `watch()` | 状态由 Agent 管理，壳层只接收 |
| ❌ 数据转换 | `map()`, `filter()`, `reduce()` | 数据在交给壳层前已格式化好 |
| ❌ 决策逻辑 | `if/else` 判断显示什么 | 显示什么由上层决定，壳层只渲染 |
| ❌ 生命周期 | `onMounted`, `onUnmounted` | 壳层组件没有自己的生命周期逻辑 |
| ❌ 副作用 | `fetch`, `localStorage` | 壳层不发起任何外部调用 |

壳层应该只有：

| 应该有的 | 说明 |
|---------|------|
| ✅ 纯渲染组件 | 输入 props → 输出 DOM |
| ✅ 事件转发 | 用户点击 → 调用上层传入的回调 |
| ✅ 样式管理 | CSS、主题、动画 |

### 理想接口

```typescript
// 壳层组件 = 纯函数
function ChatShellView(props: {
  state: AgentState;           // 已经完全格式化好的 UI 数据
  onSend: (text: string) => void;   // 事件回调，由上层实现
  onClear: () => void;
  onUnload: (turnId: string) => void;
  onReload: (turnId: string) => void;
}): JSX.Element;

// 上层（controller / connector）负责：
// 1. 从 Agent 获取状态
// 2. 将状态转换为壳层需要的 props
// 3. 将壳层的事件回调映射为 Agent 的操作
```

### 当前现状（需要逐步收敛）

```
useChatShellSession.ts 中的逻辑：
  ref() / computed() / watch()                          ← 状态管理
  turns.value = ... / draft.value = ...                  ← 直接写状态  
  buildDevChatMainView({ turns, running, ... })          ← 数据转换
  buildDevChatContextView({ seed, draft, ... })          ← 数据转换
  reconcileAndValidate()                                 ← 决策逻辑
  createLinkExecution()                                  ← 副作用
```

这些都是需要逐步迁移到 Agent 层或 controller 层的逻辑。
