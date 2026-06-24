---
title: Kun 能力清单
created: 2026-06-24
tags: [reference, kun-capabilities, workflow]
source: thr_current
status: active
para: Resources
---

# Kun 能力清单

## 📂 文件操作
- 读取、创建、编辑、搜索工作区文件
- 文件内容搜索（grep）、文件查找（find）、目录浏览（ls）
- 代码编辑（edit，精确定位替换）

## 💬 对话与交互
- 回答技术问题，写代码，调试
- 需要时向用户提问（request_user_input / user_input）
- 创建/更新/完成目标（create_goal / update_goal）

## 📚 知识库（knowledge-base）
- 按 PARA 结构写入新条目
- 更新索引 `_index.md`
- 双向同步到 Obsidian vault（push / pull）

## 🔧 工作流与自动化
- 执行计划（plan mode）
- 运行/管理工作流（list_workflows / run_workflow）
- 创建/管理定时任务（scheduled tasks）
- 管理待办事项（todo_list / todo_write）

## 🌐 网页
- 抓取网页内容（web_fetch）
- 控制浏览器（control-in-app-browser skill）

## 🛠️ 技能（Skills）
- 创建/编辑 Word 文档（documents）
- 创建/编辑 PDF（pdf）
- 创建/编辑 PPT（presentations）
- 创建/分析 Excel/CSV（spreadsheets）
- 创建表情包（generating-memes）
- 生成微信表情包系列（wechat-sticker-series）
- 创建/更新个人模板（template-creator）
- 搜索和安装技能（find-skills）

## 🧠 记忆
- 记住用户偏好/事实（memory_create）
- 更新/删除记忆（memory_update / memory_delete）

## ⚙️ 调度
- 查看、创建、更新、删除定时任务
- 支持一次性（at）、每日（daily）、间隔（interval）

## 📊 系统
- 汇报当前目标状态
- 执行结果返回与用量统计
