<div align="center">

# 从 Claude Code 学习 Agent 设计模式

**通过研究世界上最成熟的开源 AI 编程助手，学习如何构建生产级智能体系统。**

🌐 [English Version](README.en.md)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Language](https://img.shields.io/badge/语言-中文%20%2F%20English-green)](#)
[![Source](https://img.shields.io/badge/源码-Claude%20Code-blue)](https://github.com/PoorOtterBob/claude-code)

</div>

---

## 🧭 快速导航

| 你想做什么 | 去哪里 |
|-----------|--------|
| 从零开始学习，理解整体架构 | [📖 教程（9章）→](#教程章节) |
| 我要设计工具/Agent，需要模板 | [🔧 工具包（6个模板）→](#智能体设计工具包) |
| 在 Claude Code 里直接使用技能 | [🤖 技能（4个斜杠命令）→](#claude-code-技能) |
| 深入查阅某个模块的技术细节 | [📚 参考文档（中文）→](#参考文档) |

---

## 🌟 我们做了什么，为什么重要

### 背景

Claude Code 是 Anthropic 开发的 AI 编程助手，其源代码包含 **1332 个 TypeScript 文件**，实现了：
- 30+ 个工具（文件操作、Shell 执行、网络搜索、多智能体协调……）
- 完整的多智能体编排系统（子智能体、Fork、Swarm）
- 权限与安全系统
- 自定义终端 UI 框架

这些代码在 2026 年初被短暂开源（[github.com/PoorOtterBob/claude-code](https://github.com/PoorOtterBob/claude-code)），随后撤回。

### 我们的贡献

**我们做的事情：将 1332 个文件逆向工程为结构化的学习资源。**

在我们之前，没有人系统地记录 Claude Code 的内部架构。要理解它，你需要自己读完 1332 个文件，花几周时间。

我们用多智能体系统并行分析了全部源代码，提炼出：

| 产出 | 数量 | 价值 |
|------|------|------|
| 逐步教程（中英双语） | 9 章 | 从零到理解完整架构 |
| 设计模板（中英双语） | 6 个 | 可直接复制的实现模式 |
| Claude Code 技能 | 4 个 | 设计时直接调用获取建议 |
| 架构参考文档 | 10 EN + 10 ZH | 每个模块的深度文档 |
| 可复用设计模式 | 12 个 | 生产验证的 Agent 架构模式 |

### 重大意义

1. **降低门槛**：任何人都能在几小时内掌握生产级 Agent 架构，而不是几周
2. **模式提炼**：12 个设计模式经过真实生产验证（Claude Code 被数百万开发者使用）
3. **中文社区**：为中文 AI 开发者社区提供高质量的 Agent 设计参考
4. **开箱即用**：6 个模板 + 4 个技能，设计时直接引用，不需要从头发明

> **⚠️ 版权说明**：本仓库中除 `` 文件夹外的所有文件，均来自曾短暂开源的 Claude Code 源码（[github.com/PoorOtterBob/claude-code](https://github.com/PoorOtterBob/claude-code)），版权归 Anthropic 所有，并非我们的贡献。**我们的全部贡献在 `` 文件夹中。**

---

## 📖 教程章节

> **路径**: [`tutorial/zh-CN/`](tutorial/zh-CN/) · 新手友好 · 建议顺序阅读

| # | 章节 | 你会学到什么 | 链接 |
|---|------|------------|------|
| 1 | **架构总览** | QueryEngine（查询引擎）、Tools（工具）、Tasks（任务）的关系；一个请求怎么从输入变成输出 | [→](tutorial/zh-CN/01-架构总览.md) |
| 2 | **工具系统深入** | 工具是怎么定义的？buildTool() 模式；权限验证两阶段；如何设计自己的工具 | [→](tutorial/zh-CN/02-工具系统深入.md) |
| 3 | **多智能体系统** | 为什么需要多个 Agent？如何生成子 Agent？Context Fork 是什么？ | [→](tutorial/zh-CN/03-多智能体系统.md) |
| 4 | **任务生命周期** | Task 是什么？后台任务怎么运行？AsyncLocalStorage 隔离原理 | [→](tutorial/zh-CN/04-任务生命周期.md) |
| 5 | **权限与安全** | 如何保护危险操作？权限冒泡是什么？沙箱机制 | [→](tutorial/zh-CN/05-权限与安全.md) |
| 6 | **终端 UI 框架** | 为什么终端也能用 React？Ink 框架原理；无 GC 渲染技巧 | [→](tutorial/zh-CN/06-终端UI框架.md) |
| 7 | **命令与插件** | 斜杠命令怎么工作？技能系统；如何扩展 Claude Code | [→](tutorial/zh-CN/07-命令与插件.md) |
| 8 | **远程连接** | Bridge 是什么？会话管理；JWT 认证；崩溃恢复 | [→](tutorial/zh-CN/08-远程连接.md) |
| 9 | **12个设计模式** | 完整模式手册，含问题→方案→适用场景 | [→](tutorial/zh-CN/09-设计模式总结.md) |

📖 [查看英文教程](tutorial/en/) ·  [什么是 Claude Code？→ 官方文档](https://docs.anthropic.com/zh-CN/docs/claude-code/overview)

---

## 🔧 智能体设计工具包

> **路径**: [`toolkit/zh-CN/`](toolkit/zh-CN/) · 设计时直接参考 · 含决策矩阵和检查清单

| 模板 | 解决什么问题 | 链接 |
|------|------------|------|
| **工具设计模板** | 设计一个新工具：schema 定义、权限、进度上报、渲染 | [→](toolkit/zh-CN/工具设计模板.md) |
| **智能体生成模板** | 选择生成模式（前台/后台/Fork）、上下文继承、通信方式 | [→](toolkit/zh-CN/智能体生成模板.md) |
| **权限系统模板** | 两阶段验证、权限规则、破坏性操作检测、多智能体权限冒泡 | [→](toolkit/zh-CN/权限系统模板.md) |
| **协调器工作流模板** | 研究→综合→实现→验证，四阶段编排，防确认偏差 | [→](toolkit/zh-CN/协调器工作流模板.md) |
| **Swarm/团队模板** | 共享工作队列，智能体自主领取任务，权限同步 | [→](toolkit/zh-CN/Swarm团队模板.md) |
| **任务系统模板** | 任务状态机、全局注册、僵尸保护、AsyncLocalStorage | [→](toolkit/zh-CN/任务系统模板.md) |

📖 [查看英文工具包](toolkit/en/)

---

## 🤖 Claude Code 技能

> **路径**: [`skills/`](skills/) · 安装一次，设计时随时调用

### 安装

```bash
cp -r skills/agent-design         ~/.claude/skills/
cp -r skills/find-pattern         ~/.claude/skills/
cp -r skills/tool-design          ~/.claude/skills/
cp -r skills/multi-agent-workflow ~/.claude/skills/
```

重启 Claude Code，技能自动出现。

### 可用技能

| 技能 | 命令 | 你会得到 |
|------|------|---------|
| **agent-design** | `/agent-design 我要构建XXX` | 完整架构设计：适用模式、结构图、实现路线图 |
| **find-pattern** | `/find-pattern 我遇到了XXX问题` | 精准匹配对应设计模式 + 指向具体实现文件 |
| **tool-design** | `/tool-design 我需要一个XXX工具` | 完整 buildTool() 规范：schema、验证、安全检查 |
| **multi-agent-workflow** | `/multi-agent-workflow 我要完成XXX复杂任务` | 完整编排方案：智能体角色、通信、阶段、故障恢复 |

📖 [技能安装详细说明](skills/README.md)

---

## 📚 参考文档（速查手册）

> 与教程的区别：**教程**（`tutorial/`）是按学习路线组织的逐步讲解，每章 400-800 行，适合从头读到尾。**参考文档**（`zh-CN/`、`en/`）是按模块组织的简明速查，每篇 60-230 行，适合"我知道要找什么"时快速翻阅。

### 中文参考 [`zh-CN/`](zh-CN/)

> 每个模块的深度文档，适合已理解基础后查阅细节

[00 总览](zh-CN/00-总览.md) · [01 工具系统](zh-CN/01-工具系统.md) · [02 多智能体](zh-CN/02-多智能体.md) · [03 任务系统](zh-CN/03-任务系统.md) · [04 工具目录](zh-CN/04-工具目录.md) · [05 命令系统](zh-CN/05-命令系统.md) · [06 终端UI](zh-CN/06-终端UI.md) · [07 远程连接](zh-CN/07-远程连接.md) · [08 工具与核心](zh-CN/08-工具与核心.md) · [09 设计模式](zh-CN/09-设计模式.md)

### English Reference [`en/`](en/)

[00 Overview](en/00-OVERVIEW.md) · [01 Tool System](en/01-TOOL-SYSTEM.md) · [02 Multi-Agent](en/02-MULTI-AGENT.md) · [03 Tasks](en/03-TASK-SYSTEM.md) · [04 Tools Catalog](en/04-TOOLS-CATALOG.md) · [05 Commands](en/05-COMMAND-SYSTEM.md) · [06 Ink UI](en/06-INK-UI.md) · [07 Bridge](en/07-BRIDGE-SERVER.md) · [08 Utils](en/08-UTILS-CORE.md) · [09 Patterns](en/09-PATTERNS.md)

---

## ⚡ 12 个核心设计模式

> 从 Claude Code 生产代码中提炼，可直接用于你的 Agent 系统设计

| # | 模式 | 解决的问题 | 详见 |
|---|------|-----------|------|
| 1 | **工具接口 + 安全默认值** | 30+ 工具扩展无样板代码 | [→](tutorial/zh-CN/02-工具系统深入.md) |
| 2 | **上下文 Fork** | 父子 Agent 共享 Prompt 缓存，零额外 Token 成本 | [→](tutorial/zh-CN/03-多智能体系统.md) |
| 3 | **扇出→综合→实现→验证** | 复杂多阶段任务编排，防确认偏差 | [→](toolkit/zh-CN/协调器工作流模板.md) |
| 4 | **两阶段权限验证** | 逻辑检查（validateInput）与授权（checkPermissions）分离 | [→](toolkit/zh-CN/权限系统模板.md) |
| 5 | **AsyncLocalStorage 隔离** | 同进程多 Agent，无需手动传递上下文 | [→](tutorial/zh-CN/04-任务生命周期.md) |
| 6 | **渐进式结果交付** | 长时间工具边运行边显示进度，不阻塞 UI | [→](toolkit/zh-CN/工具设计模板.md) |
| 7 | **延迟工具加载** | 工具太多浪费 Token，按需发现加载 | [→](zh-CN/01-工具系统.md) |
| 8 | **XML 异步通知** | 后台 Agent 完成时主动通知主对话 | [→](toolkit/zh-CN/智能体生成模板.md) |
| 9 | **共享工作队列（Swarm）** | Agent 自主领取任务，无中心瓶颈 | [→](toolkit/zh-CN/Swarm团队模板.md) |
| 10 | **多来源命令注册** | 内置+技能+插件统一发现，按场景过滤 | [→](tutorial/zh-CN/07-命令与插件.md) |
| 11 | **无 GC 打包缓冲区** | 高频终端渲染不卡顿，消除垃圾回收暂停 | [→](tutorial/zh-CN/06-终端UI框架.md) |
| 12 | **协调者 + 草稿板** | 跨 Agent 共享发现，避免通过提示词传大量文本 | [→](toolkit/zh-CN/协调器工作流模板.md) |

---

## 🏗️ 架构速览

```
                    ┌──────────────────────────┐
                    │       用户输入             │
                    └─────────┬────────────────┘
                              │
                    ┌─────────▼────────────────┐
                    │    命令注册中心             │  src/commands.ts
                    │    （斜杠命令解析）          │
                    └─────────┬────────────────┘
                              │
                    ┌─────────▼────────────────┐
                    │      QueryEngine           │  src/QueryEngine.ts
                    │   （对话循环 · 大脑）        │
                    └──────┬──────┬─────────────┘
                           │      │
               ┌───────────┘      └───────────┐
               │                              │
     ┌─────────▼──────────┐       ┌───────────▼──────────┐
     │     工具层           │       │     任务层             │
     │   src/tools/ (30+)  │       │   src/tasks/          │
     │                     │       │                       │
     │  BashTool            │       │  LocalShellTask        │
     │  FileEditTool        │       │  LocalAgentTask        │
     │  GrepTool            │       │  InProcessTeammate     │
     │  AgentTool ──────────┼───────┼──► 生成子 Agent        │
     │  MCPTool             │       │                       │
     │                     │       │  AsyncLocalStorage     │
     │  权限系统             │       │  （隔离）              │
     └─────────┬───────────┘       └───────────┬──────────┘
               └──────────────┬────────────────┘
                              │
                    ┌─────────▼────────────────┐
                    │    Ink UI 框架             │  src/ink/
                    │  （CLI 的 React）           │
                    └──────────────────────────┘
```

---

## 📂 本仓库结构说明

```
/（仓库根目录）
│
├──           ← ✅ 我们的贡献（本项目全部原创内容）
│   ├── README.md        ← 你正在看的这个文件（中文主版）
│   ├── README.en.md     ← English version
│   ├── tutorial/
│   │   ├── zh-CN/       ← 中文教程（9章，新手友好）
│   │   └── en/          ← English tutorial
│   ├── toolkit/
│   │   ├── zh-CN/       ← 中文设计模板（6个）
│   │   └── en/          ← English templates
│   ├── skills/          ← Claude Code 技能（4个斜杠命令）
│   ├── zh-CN/           ← 中文架构参考（10篇）
│   └── en/              ← English reference（10 docs）
│
└── src/ 等其他目录      ← ⚠️ 来自曾短暂开源的 Claude Code 源码
                           （版权归 Anthropic 所有，非我们贡献）
                           来源：github.com/PoorOtterBob/claude-code
```

---

## 📊 项目统计

| 指标 | 数量 |
|------|------|
| 分析的 Claude Code 源文件 | 1,332 个 TypeScript 文件 |
| 文档化的工具 | 30+ |
| 内置智能体类型 | 6 |
| 中文教程章节 | 9 |
| 设计模板 | 6 个（中英各版） |
| Claude Code 技能 | 4 个斜杠命令 |
| 架构参考文档 | 10 EN + 10 ZH |
| 可复用设计模式 | 12 |
| 参考文件总数 | 50+ |

---

## 贡献

欢迎提 PR！如发现错误、想补充模式或改进教程，请提交 Issue 或 PR。保持中英双语格式。

## 许可证

`` 目录下的内容：MIT 许可证

其余目录（Claude Code 源码部分）：版权归 Anthropic 所有

## 致谢

本项目分析了 Anthropic 的 Claude Code 架构。所有洞察来自曾公开的源代码。这是独立的教育项目，与 Anthropic 无关联。
