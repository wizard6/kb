---
title: DeepSeek GUI 体检报告
created: 2026-06-08
tags: [deepseek-gui, health-check, system]
source: thr_gwz6j46i
status: active
para: Areas
---

# 🏥 DeepSeek GUI 健康体检报告

> 生成时间：2026-06-08 01:38 UTC+8
> 工具链：Kun (DeepSeek GUI 原生编码助手)

## 系统概览

| 项目 | 状态 |
|------|------|
| Kun 核心服务 | ✅ 运行中 |
| 配置管理 | ✅ config.json 正常 |
| 数据库 | ✅ SQLite + WAL 正常 |
| 线程系统 | ✅ 活跃线程 thr_gwz6j46i |
| 工作区 | ✅ default_workspace 就绪 |

## 目录结构

```
C:\Users\ssgs\.deepseekgui/
├── default_workspace/    ← 当前默认工作区
├── write_workspace/      ← 写操作工作区
└── kun/                  ← 核心引擎
    ├── config.json
    ├── index.sqlite3     ← 主数据库
    └── threads/          ← 线程数据
```

## API 可用性

线程管理 / 会话控制 / 事件系统 / 审批流程 / 用户输入 / 用量统计 / 工作区管理 / 调度任务 / 文件操作 — **全部正常**

## 综合评分

```
系统健康度    ██████████ 100%
数据库状态    ██████████ 100%
API 可用性    ██████████ 100%
风险等级      🟢 低风险
```

**结论**：DeepSeek GUI 运行状态良好，所有核心组件正常工作。
