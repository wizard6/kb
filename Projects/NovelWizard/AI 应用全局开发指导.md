---
title: AI 应用全局开发指导
created: 2026-06-24
tags: [architecture-guide, best-practices, ai-application, development]
source: NovelWizard 项目逆向 + 通用工程原则
status: active
para: Projects/NovelWizard
---

# AI 应用全局开发指导

> 基于 NovelWizard Ver.0.1.0 的项目逆向，结合通用工程原则整理。
> NovelWizard 是**参考案例**，不是圣经。文档中会标明：✅ 值得借鉴 / ⚠️ 有条件采纳 / ❌ 建议避开。

---

## 一、指导思想

### 1.1 四条铁律

| # | 原则 | 为什么 |
|---|------|--------|
| 1 | **AI 应用的本质是控制流驱动信息流** | 技术栈会变，模型会换，但「编排逻辑 → 数据处理 → 模型调用 → 结果校验」的流程不会变。架构的核心是管好这条链。 |
| 2 | **可观测性不是附加功能，是基础设施** | AI 的输出不可预测。没有观测台，你就是在盲调一个黑盒。每一步的输入、输出、路由决策都必须可追溯。 |
| 3 | **契约先行，类型即文档** | 当 AI 参与开发，类型定义和接口契约是它理解代码的最可靠来源。Zod / TypeScript / OpenAPI 的投入回报率极高。 |
| 4 | **模块化优先，效率兜底** | 所有功能在保证效率的前提下全部模块化——每个模块边界清晰、可替换、可独立演进。模块化不是「尽量做」，而是架构纪律；效率是唯一可接受的让步理由。 |

### 1.2 设计决策优先级

```
正确性 > 可观测性 > 可测试性 > 模块化 > 性能
```

模块化单独列在性能之前：因为良好的模块化设计（边界清晰、接口契约、按需加载）本身就能带来可维护性的提升，而性能瓶颈通常出现在具体实现而非模块边界上。**先模块化，再针对热点做性能优化。**

---

## 二、项目结构

### 2.1 推荐方案：按边界明确拆分的 monorepo

```
my-app/
├── packages/          # 共享库：纯逻辑、无 UI
│   ├── domain         # 领域类型 + Schema + Normalizer
│   ├── db             # 数据库策略 + Repository
│   ├── ai-provider    # AI 模型抽象
│   └── shared-ui      # 共享 UI 组件
├── apps/              # 可独立运行的应用
│   ├── web            # 主产品
│   ├── api            # API 服务
│   └── dev-tools      # 开发者工具
├── docs/              # 架构文档
├── ops/               # 配置 + 脚本
└── package.json       # npm workspaces 根
```

✅ **NovelWizard 值得借鉴**：
- 17 个 package 的拆分解耦程度高，依赖方向单向（domain → db → repositories → api）
- 每个 package 有自己的 package.json，可独立构建和测试
- `apps/` 和 `packages/` 的分离清晰：能复用的放 packages，能独立运行放 apps

⚠️ **有条件采纳**：
- 17 个 package 对一个小团队来说可能过度拆分。**建议 5-8 个起步**（domain / db / ai / skills / api / web / dev-tools），按需拆分
- 前端 UI 组件单独成包需要成熟的组件库和明确的复用场景，否则过早抽离增加维护成本

### 2.2 目录命名规范

```
packages/<scope>/src/
  ├── index.ts          # 公开 API（仅导出需要暴露的内容）
  ├── types.ts          # 类型定义
  ├── contracts.ts      # 接口契约
  ├── ports.ts          # 端口定义（依赖倒置）
  ├── *.ts              # 实现
  └── __tests__/        # 测试
```

✅ 推荐每个 package 只暴露 index.ts。内部文件不对外可见，这是模块化的底线。

---

## 三、数据层

### 3.1 实体建模

每一步都应该是先有**类型定义**，后有**实现**。

```
需求分析
  → 定义 TypeScript interface（是什么）
    → Zod schema 校验（合法吗）
      → Normalizer 转换（怎么存）
        → SQL schema 落地
```

✅ **NovelWizard 实践**：domain package 中每个实体都有 types.ts → schema.ts → normalize.ts 的完整链路。

### 3.2 Schema 即 AI 文档

TypeScript 类型和 Zod schema 是 AI 理解你数据模型的最佳入口：

```typescript
// ✅ 好的：类型精确，AI 一看就懂
interface WorldSeed {
  id: WorldSeedId;
  title: string;
  genre: string;
  tone: string;
  protagonist: string;
  characters?: SeedCharacter[];
  createdAt: number;
}

// ❌ 差的：any 遍地，AI 只能猜
interface WorldSeed {
  id: any;
  data: Record<string, any>;
}
```

### 3.3 数据库选型

| 场景 | 推荐 | 理由 |
|------|------|------|
| 原型/单机/小团队 | SQLite（sql.js） | 零运维，嵌入运行，文件级备份 |
| 多用户/服务端 | PostgreSQL | 成熟生态，JSON 字段支持好 |
| AI 应用的特殊需求 | 支持 JSON 字段存储 | 种子/版本/状态的 JSON 数据很常见 |

✅ **NovelWizard 的 SQLite 选型**对原型阶段合理。WAL 模式 + 外键索引也是正确的实践。

⚠️ **注意**：SQLite 的并发写入能力有限。如果未来需要多用户实时协作，需要迁移到 PG 或加一层读写分离。

---

## 四、AI 编排层（核心复杂度所在）

这是 AI 应用区别于传统软件的关键。编排层负责决定「什么时候调 AI、调哪个 AI、结果怎么处理」。

### 4.1 推荐：声明式路由图

不要用 if-else 链来编排 AI 流程。用路由图：

```
Node 0: hardcoded:delegate      ← 明确的委托规则
Node 1: hardcoded:flow-continue ← AI 自述的下一步
Node 2: hardcoded:skill-select  ← 关键词/命令匹配
Node 3: llm:route               ← 模型自主判断
Node 4: fallback:chat           ← 兜底
```

| 方式 | 可读性 | AI 可修改 | 可观测性 | 推荐 |
|------|--------|-----------|----------|------|
| if-else 链 | ❌ | ❌ | ❌ | ❌ |
| 声明式路由图 | ✅ | ✅ | ✅ | ✅ |
| 状态机 | ⚠️ 适合固定流程 | ❌ | ✅ | 部分场景 |

✅ **NovelWizard 的 `orchestrator/engine.ts` 是好的参考**：按 order 排序，逐个尝试，首个命中即返回，每个节点可配置 gate（开关）。

### 4.2 管线步骤标准化

每轮 AI 交互应该走标准化的步骤序列：

```
① skill-select    ← 预处理：是否需要触发 Skill
② flow-route      ← 路由决策：走哪条路径
③ knowledge-retrieve ← 知识检索（可选）
④ document-load   ← Skill 文档挂载（可选）
⑤ chat/execute    ← AI 调用
⑥ validate        ← 输出校验（可选）
```

每个步骤应该：
- 有稳定的 `stepId`
- 可独立开关（RuntimeStepGate 模式）
- 输入输出可记录（trace）

### 4.3 Skill / 插件系统

Skill 是可扩展的 AI 操作单元。设计要点：

| 设计点 | 推荐做法 |
|--------|----------|
| **注册** | 统一 catalog，元数据与实现分离 |
| **触发** | 斜杠命令 + 关键词匹配 + LLM 路由 |
| **执行** | mock/LLM 双路径，开发期不依赖真实模型 |
| **通道** | 简单 Skill 走标准 API，复杂 Skill 走独立路由，动态能力走文档挂载 |
| **结果** | 自然语言摘要 + 结构化数据双输出 |

✅ **NovelWizard 的 `execution-channel` 设计（skill-run / dedicated-api / document）值得借鉴**。三种通道覆盖了从简单到复杂的 Skill 需求。

### 4.4 上下文管理

AI 的上下文窗口是有限的。必须精心管理什么放进对话、什么裁剪掉。

```
┌─────────────────────────────┐
│  System: 种子上下文 (恒定)    │  ← 每一轮都在
│  System: Skill 文档 (按需)   │  ← 匹配时注入
│  System: 知识检索 (按需)     │  ← 命中时注入
│  User/Assistant: 对话历史    │  ← 按窗口大小裁剪
└─────────────────────────────┘
```

✅ **值得借鉴**：context-builder 的分层打包机制 - 每层可独立开关，适合调试。

---

## 五、API 设计

### 5.1 模块化注册

不要用一整个 `routes/index.ts`。按 API 模块拆分：

```
routes/
├── health.mjs       // 健康检查
├── projects.mjs     // 书架项目
├── sessions.mjs     // 灵感会话
├── skills.mjs       // Skill 执行
├── agents.mjs       // Agent 管理
└── dev.mjs          // 开发者工具
```

✅ **NovelWizard 的模块化设计**：每个路由文件是一个独立函数 `createXxxRouter(repos)`，通过 mount 系统按需挂载。还支持环境变量 `NW_API_MODULES` 控制启用哪些模块——这在开发/测试中非常有用。

### 5.2 错误体统一

```json
{
  "error": "描述信息",
  "code": "ERROR_CODE",
  "runtimeTier": "core|extended|planned",
  "phase": "P3|P4|P5"
}
```

✅ **推荐**：统一的错误结构让前端和 AI 都能一致地处理错误。NovelWizard 的 A1-2 待做项正是这个。

---

## 六、可观测性

### 6.1 步骤开关（Step Gate）

每个管线步骤应该有一个开关。这不仅是功能开关，更是调试工具——在观测台中可以一键开启/关闭某一步。

```typescript
type RuntimeStepGate = (stepId: string) => boolean;
// stepId: 'skill-select' | 'flow-model-route' | 'knowledge-retrieve' | 'validate'
```

### 6.2 链路追踪

每一轮对话应该记录：
1. 用户输入原文
2. 经过的管线步骤及每一步的输入/输出
3. 路由决策及原因
4. AI 回复原文
5. 耗时和 token 用量

✅ **NovelWizard 的 trace + step gate 组合**是 AI 应用可观测性的好参考。

### 6.3 开发者工具独立成应用

```
gui-dev :4992      ← 面向开发者的观测台
```

不要把调试工具塞在主产品里。独立的应用可以：
- 暴露更多内部信息
- 不影响主产品的用户体验
- 随时可移除（生产环境不部署）

---

## 七、测试策略

### 7.1 分层测试

```
┌─────────────────────┐
│  E2E (少量)          │  ← 关键用户流程
├─────────────────────┤
│  集成测试 (适量)      │  ← API 路由 + DB
├─────────────────────┤
│  单元测试 (大量)      │  ← 核心逻辑 + normalizer
└─────────────────────┘
```

✅ **NovelWizard 推荐实践**：
- 每个 package 有独立的 vitest.config.ts
- 根 `verify:workspace` 命令一键跑全部测试
- 使用 Supertest 做 API 集成测试

❌ **注意避开**：不要为了「测试覆盖率」数字去测 getter/setter。测试应该覆盖「AI 可能会改错」的逻辑——normalizer、路由决策、数据校验。

### 7.2 Mock 策略

对 AI 应用来说，Mock 不是替代方案，是**开发模式**：

```
开发环境：全程 Mock，秒级反馈
测试环境：Mock AI，测试编排逻辑
预发布：真实 AI，但可观测
生产：真实 AI，完整 trace
```

✅ 每个 Skill 都应该支持 `mock: true` 参数。这不只是测试工具，更是让前端开发者、设计师等不依赖 AI 也能工作。

---

## 八、常见陷阱

### 8.1 过度架构

❌ **问题**：NovelWizard 在 P0-P2 阶段花了大量精力在基础架构上，P3-P5 业务功能全部暂缓。

⚠️ **建议**：在架构和业务之间找平衡。基础架构做到「可扩展即停」：
- 模块拆分到位即可，不必提前实现所有扩展点
- 编排层有骨架即可，不必提前实现规划器/调度器
- 测试覆盖核心路径即可，不必追求 100%

### 8.2 AI 输出完全信任

❌ **问题**：假设 LLM 返回的一定是合法 JSON。

✅ **解决**：
- 每次解析 JSON 都用 try-catch + 修复策略
- 定义清晰的 output schema，让 LLM 按 schema 输出
- 使用 `parseJsonSafe` 类工具做防御性解析
- 关键路径加 validate 步骤

### 8.3 上下文窗口无管理

❌ **问题**：对话越长，上下文越大，直到超出模型限制。

✅ **解决**：
- 只保留活跃对话（activeChatTurns）
- 历史摘要替代原始对话
- knowledge-retrieve 替代将所有上下文塞进 prompt

### 8.4 类型不一致

❌ **问题**：API 层用 camelCase，DB 层用 snake_case，没有转换层。

✅ **解决**：定义 DTO 类型并保持明确的转换函数。NovelWizard 的 `WorldSeedDto` + `normalize` 模式是正确的做法。

---

## 九、什么时候参考 NovelWizard，什么时候别

### ✅ 适合参考的场景

| 场景 | 参考内容 |
|------|----------|
| 设计 AI 编排层 | 声明式路由图 + step gate + trace |
| 构建 Skill/插件系统 | catalog + handler 注册表 + 多执行通道 |
| 管理 AI 上下文 | context-builder 分层打包 |
| 模块化 API | mount 系统 + env 控制模块开关 |
| AI 应用测试策略 | mock/LLM 双路径 + verify:workspace |
| 数据建模 | types → schema → normalize → SQL 链路 |

### ⚠️ 需要批判性采纳的场景

| 场景 | 问题 | 建议 |
|------|------|------|
| 项目初期拆分粒度 | 17 个 package 过多 | 5-8 个起步 |
| SQLite 持久化 | 不适合多用户协作 | 原型用 SQLite，上线用 PG |
| 业务暂缓策略 | 架构先行可能导致用户看不到价值 | 核心功能先做通，再优化架构 |
| .mjs 与 .ts 混用 | 部分 package 用 .mjs 而非 .ts | 统一 TypeScript，build 时编译 |

### ❌ 不建议照搬的场景

| 场景 | 原因 |
|------|------|
| 非 AI 应用 | 编排层/Skill 系统/context-builder 等设计专门针对 AI 场景 |
| 团队 < 2 人 | monorepo 的工具链成本可能超过收益 |
| 需要强实时协作 | SQLite + session_state JSON 不是协作友好的方案 |

---

## 十、推荐实施路线

### Phase 1 — 跑通核心（2-4 周）

```
□ 搭建 monorepo 骨架（domain / db / api / web）
□ 定义核心数据模型 + Zod schema
□ 实现种子输入 → AI 解析 → 存储的核心流程
□ 基础 UI：输入框 + 结果显示
□ verify 脚本：构建 + 核心测试
```

### Phase 2 — 可观测（2-3 周）

```
□ 加入 step gate + trace 机制
□ 搭建开发者工具应用
□ Mock 模式支持
□ API 模块化 + 统一错误体
□ 集成测试覆盖关键路径
```

### Phase 3 — 可扩展（2-4 周）

```
□ Skill 系统骨架（catalog + handler 注册表 + mock 执行）
□ 声明式路由图编排
□ 文档挂载机制
□ 上下文管理（分层打包 + 裁剪）
```

### Phase 4 — 打磨（持续）

```
□ 真实 LLM prompt 调优
□ 完善业务功能
□ 性能优化
□ 用户体验打磨
```

---

## 附录 A：评价 NovelWizard 的关键设计决策

| 决策 | 评价 | 理由 |
|------|------|------|
| 声明式路由图 | ✅ 优秀 | 将编排逻辑从代码提升为数据，AI 可读、人可改 |
| 17 个 package | ⚠️ 看阶段 | 对 0.1.0 版本偏多，但方向正确 |
| SQLite 嵌入 | ✅ 合理 | 原型阶段快速迭代的最优解 |
| Skill 系统 | ✅ 优秀 | catalog + handler + 多通道的设计成熟 |
| 可观测性 | ✅ 突出 | RuntimeStepGate + trace + dev-gui 套件 |
| 架构先行策略 | ⚠️ 双刃剑 | 基础架构稳固，但业务功能被推迟 |
| 类型全栈 | ✅ 优秀 | Zod + TypeScript 贯穿全链路 |
| 测试覆盖 | ⚠️ 部分不足 | 部分 repository 缺少测试 |

---

## 修订记录

| 版本 | 日期 | 说明 |
|------|------|------|
| v0.1 | 2026-06-24 | 初稿，基于 NovelWizard 项目逆向提炼 |
