---
title: Android Skills 谷歌官方 AI Agent 技能库
source: raw_20261017_233134_46b4b9d3
source_url: https://github.com/android/skills
created: '2026-10-17'
updated: '2026-10-17'
tags:
- AI
- Agent
- Android开发
- 技能系统
- 提示工程
status: processed
note_type: content
related:
- SkillOpt 微软开源 LLM Agent 技能优化器
- Scientific Agent Skills 科研 Agent 技能库
- Agent Browser：面向 AI Agent 的 Rust 浏览器自动化 CLI
- agent-teams-ai 多智能体协作框架
- CubeSandbox AI Agent 安全沙箱
- ECC AI Agent 工具集：Claude Code 的工程化操作系统
- TencentDB Agent Memory 本地化 AI 智能体长期记忆框架
- AI Job Search 求职自动化框架
- ai-memory 跨AI编码代理的长期记忆系统
- Claude Code 官方插件市场
---

# Android Skills 谷歌官方 AI Agent 技能库

## 核心结论
> Android Skills 是 Google 维护的开源 AI Agent 技能仓库，通过遵循开放标准的 SKILL.md 指令文件向 LLM 注入 Android 开发领域的专业工作流知识。其选题策略聚焦于评估显示 LLM 表现欠佳的场景（如 R8 分析、Wear Compose），不重复 LLM 已熟练的基础能力。技能可通过 CLI 安装或作为 Claude Code、Codex 插件加载，Agent 会根据任务语义自动激活匹配技能。

## 详细内容
## 概述

[[Android Skills]] 是 Google 官方维护的开源仓库，汇集了面向 [[Android 开发]] 场景优化的 AI Agent 技能指令集。每个技能以遵循 [agentskills.io](https://agentskills.io/home) 开放标准的 [[SKILL.md]] 文件形式存在，向 LLM 提供专业领域知识与工作流规范，让 Agent 能更准确地执行复杂任务。

## 设计理念

项目核心策略是**补齐短板而非锦上添花**：只针对评估显示 LLM 表现欠佳的场景制作技能，对于 [[Jetpack Compose]] 基础最佳实践这类 LLM 已熟练的领域不做覆盖。这种基于实证差距的选题方式使技能投入聚焦于高价值场景，典型例子包括 [[R8 分析]]（如 `r8-analyzer`）与 [[Wear Compose M3]] 等。

## 安装方式

技能可通过三种渠道加载：

- **Android CLI**：使用 `android skills add <name>` 安装单个技能，或 `android skills add --all` 安装全部；默认会写入 `~/.gemini/antigravity/skills/`，供 Gemini 与 Antigravity 读取。常用参数包括 `--agent`（指定 Agent 列表）、`--project`（指定项目根目录）
- **Claude Code 插件市场**：通过 `/plugin install android-skills --marketplace android/skills` 加载
- **Codex 插件市场**：使用 `codex plugin marketplace add android/skills` 添加，再用 `codex plugin add` 启用

更新已安装技能可执行 `android skills update --all` 或 `android skills update <skill-name>`。

## 激活机制

Agent 会在用户下达与技能主题相关的任务时自动检索并调用匹配的技能，例如提示"让我的应用 UI 实现 edge-to-edge"就会触发对应技能，无需手动指定。这种[[基于任务的自动激活]]机制依赖 Agent 自身的技能发现能力，本质上属于[[提示工程]]领域的技能检索增强（RAG）模式。

## 局限与治理

README 明确声明 AI 输出仍可能出错，需人工复核。项目遵循 [[Apache 2.0]] 许可，但**暂不接受外部公开贡献**，反馈、问题或新技能请求需通过 GitHub Issue 提交。社区行为规范遵循 Google 开源社区准则。

## 相关笔记
- [[SkillOpt 微软开源 LLM Agent 技能优化器]]
- [[Scientific Agent Skills 科研 Agent 技能库]]
- [[Agent Browser：面向 AI Agent 的 Rust 浏览器自动化 CLI]]
- [[agent-teams-ai 多智能体协作框架]]
- [[CubeSandbox AI Agent 安全沙箱]]
- [[ECC AI Agent 工具集：Claude Code 的工程化操作系统]]
- [[TencentDB Agent Memory 本地化 AI 智能体长期记忆框架]]
- [[AI Job Search 求职自动化框架]]
- [[ai-memory 跨AI编码代理的长期记忆系统]]
- [[Claude Code 官方插件市场]]