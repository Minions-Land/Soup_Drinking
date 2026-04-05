<div align="center">

# Agent Design Patterns from Claude Code

**Learn to build production-grade AI agent systems by studying the world's most sophisticated open-source agent implementation.**

🌐 [中文版](README.md)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Language](https://img.shields.io/badge/Language-EN%20%2F%20ZH-green)](#)
[![Source](https://img.shields.io/badge/Source-Claude%20Code-blue)](https://github.com/PoorOtterBob/claude-code)

</div>

---

## 🧭 Quick Navigation

| What you want to do | Where to go |
|--------------------|-------------|
| Learn from scratch, understand the architecture | [📖 Tutorial (9 chapters) →](#tutorial) |
| Design tools/agents, need templates | [🔧 Toolkit (6 templates) →](#toolkit) |
| Use skills directly inside Claude Code | [🤖 Skills (4 slash commands) →](#skills) |
| Deep-dive a specific module | [📚 Reference Docs →](#reference) |

---

## 🌟 What We Did and Why It Matters

### Background

Claude Code is Anthropic's AI coding assistant. Its source code contains **1,332 TypeScript files** implementing:
- 30+ tools (file operations, shell execution, web search, multi-agent coordination…)
- A complete multi-agent orchestration system (sub-agents, Fork, Swarm)
- A permission and security system
- A custom terminal UI framework

The source code was briefly open-sourced in early 2026 ([github.com/PoorOtterBob/claude-code](https://github.com/PoorOtterBob/claude-code)), then withdrawn.

### Our Contribution

**We reverse-engineered 1,332 files into structured, actionable learning resources.**

Before this project, there was no systematic documentation of Claude Code's internal architecture. Understanding it required reading all 1,332 files yourself — weeks of work.

We used a multi-agent system to analyze the entire codebase in parallel, distilling:

| Output | Count | Value |
|--------|-------|-------|
| Bilingual tutorials (EN + ZH) | 9 chapters | From zero to complete architecture understanding |
| Bilingual design templates | 6 | Copy-ready implementation patterns |
| Claude Code skills | 4 | Get design advice directly during development |
| Architecture reference docs | 10 EN + 10 ZH | Deep documentation for every module |
| Reusable design patterns | 12 | Production-validated Agent architecture patterns |

### Why This Matters

1. **Lowers the bar**: Anyone can grasp production-grade Agent architecture in hours, not weeks
2. **Distilled patterns**: 12 design patterns proven in production (Claude Code serves millions of developers)
3. **Community resource**: High-quality Agent design reference for the global developer community
4. **Ready to use**: 6 templates + 4 skills you can reference immediately while building

> **⚠️ Attribution**: All files in this repository EXCEPT `` come from the briefly open-sourced Claude Code ([github.com/PoorOtterBob/claude-code](https://github.com/PoorOtterBob/claude-code)), copyright Anthropic. That code is not our contribution. **Our entire contribution is in the `` folder.**

---

## 📖 Tutorial <a name="tutorial"></a>

> **Path**: [`tutorial/en/`](tutorial/en/) · Beginner-friendly · Read sequentially

| # | Chapter | What you'll learn | Link |
|---|---------|------------------|------|
| 1 | **Architecture Overview** | QueryEngine, Tools, Tasks — how a request flows from input to output | [→](tutorial/en/01-architecture-overview.md) |
| 2 | **Tool System Deep Dive** | Tool interface, buildTool() pattern, two-stage validation, permissions | [→](tutorial/en/02-tool-system-deep-dive.md) |
| 3 | **Multi-Agent System** | Why multiple agents? Spawning, Context Fork, SendMessage, Coordinator | [→](tutorial/en/03-multi-agent-system.md) |
| 4 | **Task Lifecycle** | What is a Task? Background execution, AsyncLocalStorage isolation | [→](tutorial/en/04-task-lifecycle.md) |
| 5 | **Permission & Security** | Permission modes, destructive action detection, sandbox, bubbling | [→](tutorial/en/05-permission-security.md) |
| 6 | **Terminal UI Framework** | Why React in the terminal? Ink, GC-free rendering tricks | [→](tutorial/en/06-terminal-ui.md) |
| 7 | **Commands & Plugins** | Slash commands, skill system, how to extend Claude Code | [→](tutorial/en/07-command-plugin-system.md) |
| 8 | **Remote Connection** | Bridge, session management, JWT auth, crash recovery | [→](tutorial/en/08-remote-bridge.md) |
| 9 | **12 Design Patterns** | Complete pattern cookbook: Problem → Solution → When to Use | [→](tutorial/en/09-design-patterns-summary.md) |

📖 [中文教程版本](tutorial/zh-CN/)

---

## 🔧 Agent Design Toolkit <a name="toolkit"></a>

> **Path**: [`toolkit/en/`](toolkit/en/) · Use as copy-paste reference when building

| Template | What problem it solves | Link |
|----------|----------------------|------|
| **Tool Design Template** | Design a new tool: schema, permissions, progress, rendering | [→](toolkit/en/tool-template.md) |
| **Agent Spawning Template** | Choose spawn mode (foreground/background/fork), context inheritance | [→](toolkit/en/agent-spawning-template.md) |
| **Permission System Template** | Two-stage validation, rules, destructive detection, bubbling | [→](toolkit/en/permission-system-template.md) |
| **Coordinator Workflow Template** | Research→Synthesis→Implement→Verify, anti-confirmation-bias | [→](toolkit/en/coordinator-workflow-template.md) |
| **Swarm/Team Template** | Shared work queue, agent autonomy, permission sync | [→](toolkit/en/swarm-team-template.md) |
| **Task System Template** | State machine, global registry, zombie protection, AsyncLocalStorage | [→](toolkit/en/task-system-template.md) |

📖 [中文工具包](toolkit/zh-CN/)

---

## 🤖 Claude Code Skills <a name="skills"></a>

> **Path**: [`skills/`](skills/) · Install once, use anytime during development

### Installation

```bash
cp -r skills/agent-design         ~/.claude/skills/
cp -r skills/find-pattern         ~/.claude/skills/
cp -r skills/tool-design          ~/.claude/skills/
cp -r skills/multi-agent-workflow ~/.claude/skills/
```

Restart Claude Code — the skills appear automatically.

### Available Skills

| Skill | Command | What you get |
|-------|---------|-------------|
| **agent-design** | `/agent-design <what you're building>` | Full architecture: patterns, structure, roadmap |
| **find-pattern** | `/find-pattern <your problem>` | Precise pattern match + implementation pointer |
| **tool-design** | `/tool-design <what it should do>` | Complete buildTool() spec with security checklist |
| **multi-agent-workflow** | `/multi-agent-workflow <complex task>` | Full orchestration design: agents, phases, recovery |

📖 [Skills installation guide](skills/README.md)

---

## 📚 Architecture Reference <a name="reference"></a>

### English Reference [`en/`](en/)

[00 Overview](en/00-OVERVIEW.md) · [01 Tool System](en/01-TOOL-SYSTEM.md) · [02 Multi-Agent](en/02-MULTI-AGENT.md) · [03 Tasks](en/03-TASK-SYSTEM.md) · [04 Tools Catalog](en/04-TOOLS-CATALOG.md) · [05 Commands](en/05-COMMAND-SYSTEM.md) · [06 Ink UI](en/06-INK-UI.md) · [07 Bridge](en/07-BRIDGE-SERVER.md) · [08 Utils](en/08-UTILS-CORE.md) · [09 Patterns](en/09-PATTERNS.md)

### 中文参考 [`zh-CN/`](zh-CN/)

[00 总览](zh-CN/00-总览.md) · [01 工具系统](zh-CN/01-工具系统.md) · [02 多智能体](zh-CN/02-多智能体.md) · [03 任务系统](zh-CN/03-任务系统.md) · [04 工具目录](zh-CN/04-工具目录.md) · [05 命令系统](zh-CN/05-命令系统.md) · [06 终端UI](zh-CN/06-终端UI.md) · [07 远程连接](zh-CN/07-远程连接.md) · [08 工具与核心](zh-CN/08-工具与核心.md) · [09 设计模式](zh-CN/09-设计模式.md)

---

## ⚡ The 12 Design Patterns

| # | Pattern | Problem Solved |
|---|---------|---------------|
| 1 | **Tool Interface + Safe Defaults** | Scaling to 30+ tools without boilerplate |
| 2 | **Context Forking** | Share prompt cache between parent/child agents, zero extra token cost |
| 3 | **Fan-Out → Synthesize → Implement → Verify** | Complex multi-phase orchestration, anti-confirmation-bias |
| 4 | **Two-Stage Validation** | Separate logic checks (validateInput) from authorization (checkPermissions) |
| 5 | **AsyncLocalStorage Isolation** | Multiple agents in one process, no manual context threading |
| 6 | **Progressive Result Delivery** | Long tools show progress while running, don't block UI |
| 7 | **Deferred Tool Loading** | Too many tools waste prompt tokens, discover on demand |
| 8 | **XML Async Notifications** | Background agents notify main conversation when done |
| 9 | **Shared Work Queue (Swarm)** | Agents self-claim tasks, no central bottleneck |
| 10 | **Multi-Source Command Registry** | Built-in + skills + plugins unified discovery |
| 11 | **GC-Free Packed Buffers** | High-frequency terminal rendering without GC pauses |
| 12 | **Coordinator + Scratchpad** | Cross-agent knowledge sharing without passing large text |

---

## 📂 Repository Structure

```
/ (repo root)
│
├──           ← ✅ OUR CONTRIBUTION (all original content)
│   ├── README.md        ← Chinese primary README
│   ├── README.en.md     ← This file (English README)
│   ├── tutorial/
│   │   ├── zh-CN/       ← Chinese tutorial (9 chapters, beginner-friendly)
│   │   └── en/          ← English tutorial
│   ├── toolkit/
│   │   ├── zh-CN/       ← Chinese design templates (6)
│   │   └── en/          ← English templates
│   ├── skills/          ← Claude Code skills (4 slash commands)
│   ├── zh-CN/           ← Chinese reference docs (10)
│   └── en/              ← English reference docs (10)
│
└── src/ etc.            ← ⚠️ From briefly open-sourced Claude Code
                            Copyright Anthropic — NOT our contribution
                            Source: github.com/PoorOtterBob/claude-code
```

---

## 📊 Stats

| Metric | Count |
|--------|-------|
| Claude Code source files analyzed | 1,332 TypeScript files |
| Tools documented | 30+ |
| Built-in agent types | 6 |
| Tutorial chapters (bilingual) | 9 |
| Design templates (bilingual) | 6 |
| Claude Code skills | 4 slash commands |
| Architecture reference docs | 10 EN + 10 ZH |
| Reusable patterns | 12 |

---

## Contributing

Contributions welcome. Open a PR to fix errors, add patterns, or improve tutorials. Please maintain the bilingual (EN/ZH) format for all content in ``.

## License

Content in ``: MIT License

Everything else (Claude Code source): Copyright Anthropic

## Acknowledgments

This project analyzes Anthropic's Claude Code architecture. All insights are derived from publicly available source code. This is an independent educational project not affiliated with or endorsed by Anthropic.
