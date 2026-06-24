---
title: NovelWizard 项目分析
created: 2026-06-24
tags: [project-analysis, novel-wizard, vue3, monorepo, ai-writing]
source: C:\temp\workspaceV2\novelWizard\NovelWizard_new
status: active
para: Resources
---

# Novel Wizard 项目分析

## 一句话定位

AI 小说创作系统 —「可生长的故事世界操作系统」：从灵感种子生长出故事世界，非一键成书。

## 技术栈

| 层 | 选型 |
|----|------|
| 前端 | Vue 3 + Vite + Vue Router + TypeScript |
| UI 库 | Naive UI |
| 后端 | Express（Node.js） |
| 数据库 | SQLite（sql.js） |
| 测试 | Vitest + Supertest |
| 构建 | esbuild + rollup |
| 类型 | vue-tsc + Zod |
| 工作区 | npm workspaces monorepo |

## 仓库结构

```
NovelWizard_new/
├── packages/         ★ 17 个共享包（核心领域逻辑）
├── apps/             ★ 4 个可运行应用
├── docs/             文档与架构
├── experiments/      Lab / 可视化工具
├── ops/              配置与启动脚本
└── legacy/           0.0.x 冻结原型
```

## 共享包（packages/）

| 包 | 职责 |
|----|------|
| domain | 核心领域类型、Zod schema、normalizer（seed/project/session/entity/inspiration 等） |
| db | SQLite 策略、规范 schema、表目录 |
| ai-provider | 多 AI 提供方注册、对话服务、SSE、计费 |
| chat-domain | 对话领域模型：轮次类型、人格、消息组装 |
| chat-runtime | 对话 Agent 运行时：单轮编排、委托、HTTP 客户端 |
| chat-shell-ui | 对话壳层 UI、会话控制器、Agent 模块、观测台 |
| context-builder | 需求上下文组装器：分层打包与清单 |
| llm-utils | LLM 策略规则与安全 JSON 解析 |
| knowledge-index | 标签索引 schema 与查找 |
| skills | Skill 目录 SSOT + 斜杠命令 |
| app-store | 应用状态 store |
| repositories | 数据仓库层（封装 db） |
| silicon-reflection |「硅灵反射」规划器与执行器之间的方法调用协议 |
| zero-memory |「硅灵记忆提取」知识库/书架/档案/文档统一获取 |
| scoped-state | 作用域状态：会话/Agent 隔离、生命周期 |
| gui-env | GUI 环境工具、Naive theme、Vue composable |
| gui-theme | 共享设计 token 与 CSS |

## 应用层（apps/）

| 应用 | 端口 | 用途 |
|------|------|------|
| gui（产品） | 4990 | 主产品 UI |
| gui-api | 4991 | Express API 服务 |
| gui-dev | 4992 | 开发者观测台 |
| gui-console | 4993 |「故事世界操作系统」主壳 |

## 核心概念

- **种子（Seed）** → **故事世界** 的演化管道
- **会话版本链**（Session version chain）管理写作过程
- **Skills 斜杠命令系统**扩展能力
- **Agent 运行时**编排 AI 推理
- **硅灵反射协议**：规划器 ↔ 执行器间的通信
- **标签知识索引**（knowledge-index）驱动检索

## 代码规模

- TypeScript 源文件：~668 个（不含 node_modules/legacy）
- 包数量：17 个 packages + 4 个 apps
- 状态：Ver.0.1.0 重构线，活跃开发中
