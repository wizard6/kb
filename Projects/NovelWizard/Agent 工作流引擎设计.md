---
title: NovelWizard Agent 工作流引擎设计
created: 2026-06-24
tags: [architecture, agent, workflow-engine, dag, design]
source: 与用户讨论梳理
status: active
para: Projects/NovelWizard
---

# NovelWizard Agent 工作流引擎设计

> 下一代 Agent 会话链路的底层基础——可视化 DAG 工作流引擎 + Agent Loop。
> 当前版本（声明式路由图）的全面升级。

---

## 一、设计目标

下一代引擎的核心目标：

| 目标 | 说明 |
|------|------|
| **无固定路径** | 不预设节点执行顺序，AI 在每个节点动态决策下一步去哪 |
| **节点自描述** | 每个节点内置：它能做什么、输出什么、可选下一跳、选择规则、禁止规则 |
| **复合节点** | 节点可以展开为子图，对内是完整流程，对外是一个能力单元 |
| **并行执行** | 节点的可选下一跳可以标记为并行，多个分支同时执行 |
| **人类介入** | 特定节点类型（human_decision）执行时暂停，等用户输入后再继续 |
| **可视化编排** | 流程在画布上拖拽编辑，运行时高亮当前节点 |

---

## 二、核心概念

### 1.1 设计理念

| 原则 | 说明 |
|------|------|
| **AI 在约束下自主决策** | 没有固定路径。AI 在每个节点根据内置的可选列表和选择规则决定下一跳 |
| **节点自描述** | 每个节点自带：执行体、输出端口、可选下一跳列表、选择规则、禁止规则 |
| **复合节点** | 节点可以展开为子图，子图细节对外部不可见 |
| **并行执行** | 节点可标记为并行，多个节点同时运行后汇聚 |
| **人类介入点** | 特定节点类型强制暂停，等待人类输入后继续 |
| **可视化编排** | 整个流程在画布上拖拽编辑，运行时高亮当前节点 |

### 1.2 与当前架构的关系

| 当前版本（声明式路由图） | 新版本（Agent 工作流引擎） |
|-------------------------|--------------------------|
| 固定 5 个路由节点 | 任意多个节点，动态组合 |
| 节点在代码中硬编码 | 节点是数据（JSON/DB），运行时加载 |
| 顺序遍历，命中即返回 | DAG 图，AI 动态决策路径 |
| 无并行概念 | 支持并行节点 |
| 无人类介入点 | human_decision 节点暂停等待 |
| 图不可编辑（代码级） | 图可视化编辑（拖拽画布） |

---

## 二、数据模型

### 2.1 FlowDefinition（流程定义）

```typescript
interface FlowDefinition {
  id: string;
  name: string;
  description: string;
  version: string;
  nodes: FlowNode[];
  edges: FlowEdge[];
  createdAt: number;
  updatedAt: number;
}
```

### 2.2 FlowNode（节点）

```typescript
interface FlowNode {
  id: string;
  type: NodeType;          // atomic | composite
  kind: NodeKind;          // ai-call | human-decision | skill-exec | condition | transform

  label: string;
  description: string;

  // 节点配置（按 kind 不同而不同）
  config: {
    // ai-call:  { systemPrompt, model?, temperature? }
    // human-decision: { question, options[], allowCustom? }
    // skill-exec: { skillId, params? }
    // condition: { expression }
    // transform: { code? }
    [key: string]: unknown;
  };

  // 输入输出声明
  inputs: PortDef[];
  outputs: PortDef[];

  // 可选下一跳列表（节点向外暴露的「可去哪儿」）
  nextOptions: NextOption[];

  // 内置选择规则（AI 决策时的约束）
  selectionRules: SelectionRule[];

  // 嵌套子图（仅 composite 类型）
  subGraph?: FlowDefinition;

  // 位置信息（供可视化编辑器使用）
  position?: { x: number; y: number };
}

interface PortDef {
  name: string;
  type: string;
  description: string;
}

interface NextOption {
  targetNodeId: string;
  label: string;
  condition?: string;       // 建议条件描述，供 AI 决策参考
  parallel?: boolean;       // 是否并行执行
  description?: string;
}

interface SelectionRule {
  type: 'prefer' | 'avoid' | 'require' | 'forbid';
  target: string;            // 节点 id 或节点标签
  condition?: string;        // 触发规则的上下文条件
  priority?: number;         // 优先级
}
```

### 2.3 FlowEdge（连线）

```typescript
interface FlowEdge {
  id: string;
  sourceNodeId: string;
  targetNodeId: string;
  sourcePort?: string;      // 从哪个输出端口出
  targetPort?: string;      // 到哪个输入端口
  label?: string;
}
```

### 2.4 FlowInstance（运行时状态）

```typescript
interface FlowInstanceState {
  // 一次执行实例
  flowId: string;
  instanceId: string;
  status: 'running' | 'paused' | 'completed' | 'failed' | 'cancelled';

  // 当前在哪
  currentNodeId: string | null;
  history: NodeExecutionRecord[];

  // 上下文数据
  context: Record<string, unknown>;

  // 暂停信息（当 status === 'paused' 时）
  pauseInfo?: {
    nodeId: string;
    question: string;
    options: HumanDecisionOption[];
    createdAt: number;
  };

  // 并行执行状态
  parallelBranches?: {
    branchId: string;
    nodeId: string;
    status: 'running' | 'completed' | 'failed';
    result?: unknown;
  }[];

  createdAt: number;
  updatedAt: number;
}

interface NodeExecutionRecord {
  nodeId: string;
  startTime: number;
  endTime?: number;
  status: 'running' | 'completed' | 'failed' | 'skipped';
  input?: unknown;
  output?: unknown;
  decision?: {               // AI 的决策理由
    nextNodeId: string;
    reason: string;
  };
  error?: string;
}
```

---

## 三、执行引擎设计

### 3.1 Agent Loop（核心循环）

```
FlowInstance 创建 → status = running
  │
  ├→ Agent Loop:
  │    │
  │    ├→ ① 构建上下文
  │    │    ├ 当前状态（已执行记录、当前上下文数据）
  │    │    ├ 当前节点的可选下一跳列表（nextOptions）
  │    │    └ 选择规则（selectionRules）
  │    │
  │    ├→ ② AI 决策
  │    │    输入: 上下文 + 用户目标
  │    │    输出: { nextNodeId, reason }
  │    │    约束: 受 selectionRules 限制
  │    │
  │    ├→ ③ 执行节点
  │    │    ├ 原子节点 → 直接执行 handler
  │    │    ├ 复合节点 → 递归创建子 FlowInstance
  │    │    └ 人类决策节点 → 暂停，等用户
  │    │
  │    ├→ ④ 并行分叉
  │    │    如果选中的节点标记了 parallel，
  │    │    创建多个分支同时执行
  │    │
  │    ├→ ⑤ 记录结果
  │    │    写入 FlowInstanceState.history
  │    │
  │    └→ ⑥ 判断终止
  │         没有可选下一跳 → 完成
  │       ← 用户取消 → 取消
  │       ← 出错 → 回退/失败
  │       ← 否则 → 回到 ①
  │
  └→ 完成
```

### 3.2 节点执行器

```typescript
interface NodeExecutor {
  kind: NodeKind;
  execute(input: {
    node: FlowNode;
    context: Record<string, unknown>;
    signal: AbortSignal;
  }): Promise<{
    outputs: Record<string, unknown>;
    status: 'completed' | 'failed';
    error?: string;
  }>;
}
```

内置执行器：

| kind | 执行行为 |
|------|----------|
| `ai-call` | 组装 prompt → 调 LLM → 解析输出 |
| `human-decision` | 暂停实例 → 记录 pauseInfo → 等用户回复 → 继续 |
| `skill-exec` | 查 Skill 注册表 → 调对应的 handler |
| `condition` | 评估表达式 → 返回 true/false |
| `transform` | 数据变换（过滤/映射/聚合） |

### 3.3 复合节点的递归执行

```
复合节点被选中
  → 创建子 FlowInstance（subGraph）
  → 子实例独立跑 Agent Loop
  → 子实例完成
    → 取子实例的最终 context 作为输出
    → 回到父实例继续
```

子实例对父实例完全透明。父节点看到的只是：

```
输入: rawText
输出: structuredSeed
```

子实例内部可能跑了 5 个原子节点 + 2 次人类追问——父实例不需要知道。

### 3.4 并行执行

```
当前节点输出了 3 个 parallel 的 nextOptions
  → 同时创建 3 个子分支
  → 每个分支独立执行
  → 全部完成后合并结果
  → 继续下一个节点

A → [B, C, D] 同时跑 → 合并 → E
```

合并策略：

| 策略 | 行为 |
|------|------|
| `wait-all` | 所有分支完成后合并 |
| `wait-any` | 任意一个完成即继续（其他取消） |
| `wait-n` | 等待 N 个完成 |

---

## 四、人类介入机制

### 4.1 暂停与恢复

```
执行到 human_decision 节点
  → 暂停实例（status = 'paused'）
  → 记录 pauseInfo { nodeId, question, options[] }
  → 通过 WebSocket/SSE 通知前端
  → 前端展示对话框
  → 用户选择或输入
  → 调用 resume API
  → 实例继续执行
```

### 4.2 API

```typescript
// 暂停实例
PATCH /api/flows/:instanceId/pause
  → { nodeId: string }

// 恢复实例（用户给出决策后）
PATCH /api/flows/:instanceId/resume
  body: {
    nodeId: string;         // 确认是哪个节点
    decision: string;       // 用户选中的选项值
    input?: string;         // 用户补充的文本
  }
```

### 4.3 暂停 vs 非暂停节点

| | AI 节点 | 人类决策节点 |
|--|---------|-------------|
| 执行流程 | 自动执行 | 暂停等待 |
| 决策者 | AI | 人类 |
| 响应时间 | 秒级 | 任意（用户说了算） |
| 可否跳过 | 可（AI 选别的路径） | 可（如果节点可选） |

---

## 五、工具箱与节点注册

### 5.1 NodeTypeRegistry

```typescript
interface NodeTypeRegistration {
  kind: NodeKind;
  label: string;
  description: string;
  icon: string;
  category: string;        // 'ai' | 'human' | 'skill' | 'flow' | 'data'
  executor: NodeExecutor;
  // 可视化编辑器的配置面板 UI
  configSchema: Record<string, unknown>;  // Zod schema
  defaultConfig: Record<string, unknown>;
}
```

### 5.2 内置节点类型（初始）

| 类别 | 节点类型 | 说明 |
|------|----------|------|
| AI | `ai-call` | 调用 LLM，可配 system prompt |
| AI | `skill-exec` | 调用已注册的 Skill |
| 人类 | `human-decision` | 暂停等用户选择 |
| 流程 | `condition` | 条件判断 |
| 流程 | `transform` | 数据变换 |
| 流程 | `composite` | 复合节点（子图） |
| 流程 | `start` | 流程入口 |
| 流程 | `end` | 流程终点 |

---

## 六、可视化编辑器约定

### 6.1 编辑器（gui-dev 中）

```
画布区域（基于 vue-flow 或自研）
├── 节点（矩形/圆角）
│   ├── 标题栏（节点名 + 类型图标）
│   ├── 输入端口（左侧圆点）
│   ├── 输出端口（右侧圆点）
│   └── 状态指示（运行中高亮/已完成绿色/失败红色）
├── 连线（箭头线）
│   ├── 串行（实线）
│   └── 并行（虚线，带并行标记）
├── 工具栏
│   ├── 节点工具箱（可拖拽）
│   └── 缩放/适应
└── 配置面板（选中节点后显示）
    ├── 节点名称
    ├── config 参数
    ├── nextOptions 管理
    └── selectionRules 管理
```

### 6.2 运行时状态可视化

```
节点高亮：
  ██ 当前正在执行的节点（蓝色闪烁）
  ██ 已完成的节点（绿色）
  ██ 失败的节点（红色）
  ██ 暂停的节点（黄色，等待人类）
  ░░ 未执行的节点（灰色）

连线动态：
  → 走过的路径高亮
  → 并行分支同时高亮
  → 人类等待时连线闪烁
```

---

## 七、持久化存储

### 7.1 新增 SQLite 表

```sql
-- 流程定义
CREATE TABLE flow_definitions (
  id           TEXT PRIMARY KEY,
  name         TEXT NOT NULL,
  description  TEXT NOT NULL DEFAULT '',
  version      TEXT NOT NULL DEFAULT '1.0',
  definition   TEXT NOT NULL,          -- FlowDefinition JSON
  created_at   INTEGER NOT NULL,
  updated_at   INTEGER NOT NULL
);

-- 流程实例（运行时状态）
CREATE TABLE flow_instances (
  id            TEXT PRIMARY KEY,
  flow_id       TEXT NOT NULL,
  status        TEXT NOT NULL DEFAULT 'running',
  state         TEXT NOT NULL,          -- FlowInstanceState JSON
  created_at    INTEGER NOT NULL,
  updated_at    INTEGER NOT NULL,
  FOREIGN KEY (flow_id) REFERENCES flow_definitions(id)
);

-- 暂停等待的人类决策
CREATE TABLE flow_pauses (
  id            TEXT PRIMARY KEY,
  instance_id   TEXT NOT NULL,
  node_id       TEXT NOT NULL,
  question      TEXT NOT NULL,
  options       TEXT NOT NULL,          -- JSON array
  status        TEXT NOT NULL DEFAULT 'pending',  -- pending | resolved | cancelled
  decision      TEXT,                   -- 用户的选择
  user_input    TEXT,                   -- 用户补充输入
  created_at    INTEGER NOT NULL,
  resolved_at   INTEGER,
  FOREIGN KEY (instance_id) REFERENCES flow_instances(id)
);
```

---

## 八、与现有代码的迁移策略

### 8.1 兼容层

当前的 `WORKSHOP_TURN_ROUTING_GRAPH` 可以翻译为一个等价的 `FlowDefinition`：

```typescript
// 当前的 5 节点路由图 → 一个预定义的 FlowDefinition
const legacyGraph: FlowDefinition = {
  id: 'workshop-turn-routing',
  nodes: [
    { id: 'delegate',    kind: 'condition',  nextOptions: [...] },
    { id: 'flow-continue', kind: 'condition', nextOptions: [...] },
    { id: 'skill-select', kind: 'skill-exec', nextOptions: [...] },
    { id: 'model-route',   kind: 'ai-call',    nextOptions: [...] },
    { id: 'fallback-chat', kind: 'ai-call',    nextOptions: [] },  // 终点
  ],
  edges: []
};
```

### 8.2 分阶段迁移

| 阶段 | 内容 |
|------|------|
| **Phase 1** | 新引擎与旧路由并存。现有代码走旧路由，新流程走新引擎 |
| **Phase 2** | 旧路由翻译为 FlowDefinition，跑在新引擎上 |
| **Phase 3** | 旧路由代码移除，全部统一为新引擎 |

---

## 九、与现有代码的关系

| 现有组件 | 关系 |
|----------|------|
| `packages/skills/src/turnFlow/` | 概念被新引擎继承，代码逐步替代 |
| `packages/chat-runtime/src/pipeline/` | 管线中的 sendWorkshopMessage / sendSkillMessage / delegate 逻辑 → 成为新引擎的内置节点驱动 |
| `packages/chat-runtime/src/agentCatalog/` | Agent 目录继续保留，Agent 作为复合节点的一种特例 |
| `@novel-wizard/chat-domain` | 对话消息组装逻辑继续使用 |
| `RuntimeStepGate` | 概念保留，扩展为节点的 gates 字段 |
| `session_state` 持久化 | 扩展为 FlowInstanceState |
| `apps/gui-dev` | FlowMap 概念 → 变为可视化编辑器 |

### 不兼容变更

| 变更 | 原因 |
|------|------|
| `graph.ts` 硬编码路由图 → 删除 | 被 FlowDefinition 替代 |
| `TurnFlowResolution` → 简化 | AI 直接决策，不需要规则链 |
| step gate 的 `isStepEnabled()` → 扩展 | 每个节点自描述 gates |

---

## 十、新增包结构

```
packages/flow-engine/               ← 新包：通用工作流引擎
├── src/
│   ├── index.ts
│   ├── types.ts                    (FlowDefinition, FlowNode, FlowEdge, FlowInstanceState)
│   ├── registry.ts                 (NodeTypeRegistry)
│   ├── engine/                     (Agent Loop 核心)
│   │   ├── agent-loop.ts           (Observe → Think → Act 循环)
│   │   ├── node-executor.ts        (节点执行调度)
│   │   └── parallel-scheduler.ts   (并行分支管理)
│   ├── drivers/                    (内置节点执行器)
│   │   ├── ai-call.ts
│   │   ├── human-decision.ts
│   │   ├── skill-exec.ts
│   │   ├── condition.ts
│   │   └── transform.ts
│   ├── persistence/                (持久化)
│   │   ├── flow-store.ts
│   │   └── instance-store.ts
│   └── api/                        (对外 API)
│       ├── create-flow.ts
│       ├── start-instance.ts
│       ├── resume-instance.ts
│       └── get-instance-state.ts
```

---

## 修订记录

| 版本 | 日期 | 说明 |
|------|------|------|
| v0.1 | 2026-06-24 | 初稿，基于与用户讨论的设计方案 |
