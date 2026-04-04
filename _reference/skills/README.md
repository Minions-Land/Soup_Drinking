# Agent Design Skills / 智能体设计技能包

> 🗺️ **Navigation / 导航**: [← README](../README.md) | [Tutorial](../tutorial/) | [Toolkit](../toolkit/) | [Reference EN](../en/) | [Reference ZH](../zh-CN/)

A set of Claude Code slash-command skills that let you directly apply the patterns from this repository when designing your own agent systems.

一套 Claude Code 斜杠命令技能，让你在设计智能体系统时直接套用本仓库的架构模式。

---

## Installation / 安装

Copy the skill directories to your Claude Code skills folder:

将技能目录复制到你的 Claude Code 技能文件夹：

```bash
# Mac / Linux
cp -r skills/agent-design       ~/.claude/skills/
cp -r skills/find-pattern       ~/.claude/skills/
cp -r skills/tool-design        ~/.claude/skills/
cp -r skills/multi-agent-workflow ~/.claude/skills/

# Verify / 验证
ls ~/.claude/skills/
```

Then restart Claude Code or open a new session. The skills appear automatically.

然后重启 Claude Code 或开启新会话，技能会自动出现。

---

## Available Skills / 可用技能

| Skill | Trigger | Purpose |
|-------|---------|---------|
| **agent-design** | `/agent-design <what you're building>` | Full system design from description |
| **find-pattern** | `/find-pattern <problem>` | Match a problem to the right pattern |
| **tool-design** | `/tool-design <what the tool does>` | Complete buildTool() spec |
| **multi-agent-workflow** | `/multi-agent-workflow <complex task>` | Orchestration design (Coordinator/Swarm/Fork) |

| 技能 | 触发命令 | 用途 |
|------|---------|------|
| **agent-design** | `/agent-design <你要构建的东西>` | 从描述生成完整系统设计 |
| **find-pattern** | `/find-pattern <问题描述>` | 将问题映射到正确的设计模式 |
| **tool-design** | `/tool-design <工具的功能>` | 生成完整 buildTool() 规范 |
| **multi-agent-workflow** | `/multi-agent-workflow <复杂任务>` | 编排设计（协调者/Swarm/Fork 模型） |

---

## Usage Examples / 使用示例

```
/agent-design I need a system that analyzes a codebase in parallel and produces a refactoring plan
/agent-design 我需要一个并行分析代码库并生成重构计划的系统

/find-pattern How do I share context between parent and child agents without re-tokenizing?
/find-pattern 如何在父子智能体之间共享上下文而不重复消耗 token？

/tool-design A tool that queries PostgreSQL and returns formatted results
/tool-design 一个查询 PostgreSQL 并返回格式化结果的工具

/multi-agent-workflow Analyze a large codebase, find all security vulnerabilities, generate a fix for each
/multi-agent-workflow 分析大型代码库，找出所有安全漏洞，为每个漏洞生成修复方案
```

---

## How Skills Work / 技能工作原理

Each skill is a `SKILL.md` file containing a structured prompt. When invoked:

每个技能是一个包含结构化提示词的 `SKILL.md` 文件。调用时：

1. Claude reads the skill prompt / Claude 读取技能提示词
2. The skill tells Claude to read specific reference files in this repo / 技能指示 Claude 读取本仓库中的特定参考文件
3. Claude applies the patterns to your specific problem / Claude 将模式应用到你的具体问题
4. You get a concrete, implementation-ready design / 你得到一个可立即实现的具体设计

The skills reference this repository's files at runtime, so they stay up-to-date automatically as the reference docs improve.

技能在运行时引用本仓库的文件，因此随着参考文档的改进，技能会自动保持最新。

---

## Skill Files / 技能文件

```
skills/
├── README.md                        ← This file / 本文件
├── agent-design/
│   └── SKILL.md                     ← Full system design advisor
├── find-pattern/
│   └── SKILL.md                     ← Pattern matcher
├── tool-design/
│   └── SKILL.md                     ← Tool spec generator
└── multi-agent-workflow/
    └── SKILL.md                     ← Orchestration designer
```
