---
name: eino-learning
description: Eino framework learning guide and tutorials. Use when a user wants to learn Eino from scratch, follow a structured learning path, needs beginner-friendly explanations, or asks about A2UI protocol (Agent to UI streaming rendering). Covers getting started, component tutorials, orchestration concepts, agent development, A2UI protocol, advanced topics, and exercises.
---

# Eino Learning Guide

Welcome to the Eino learning guide! This skill contains tutorials and explanations for learning Eino from beginner to advanced.

## Learning Path

### Stage 1: Quick Start
- [快速开始](附录/快速开始.md) - 5 minutes to run your first program

### Stage 2: Components
- [教程-模型.md](02-组件/教程-模型.md) - ChatModel basics
- [教程-工具.md](02-组件/教程-工具.md) - Tool development
- [教程-内存.md](02-组件/教程-内存.md) - Memory management
- [教程-检索.md](02-组件/教程-检索.md) - RAG and retrieval
- [教程-回调.md](02-组件/教程-回调.md) - Callback mechanism

### Stage 3: Orchestration
- [教程-编排.md](03-编排/教程-编排.md) - Graph and Workflow
- [教程-流编排.md](03-编排/教程-流编排.md) - Stream processing
- [教程-批处理.md](03-编排/教程-批处理.md) - Batch processing

### Stage 4: Agent
- [教程-Agent.md](04-智能体/教程-Agent.md) - ReAct Agent basics
- [教程-ADK.md](04-智能体/教程-ADK.md) - ADK complete guide
- [指南-ADK.md](04-智能体/指南-ADK.md) - ADK best practices

### Stage 5: Advanced
- [进阶-人机交互.md](05-进阶/进阶-人机交互.md) - Interrupt and checkpoint
- [进阶-MultiAgent.md](05-进阶/进阶-MultiAgent.md) - Multi-agent systems
- [A2UI-协议.md](05-进阶/A2UI-协议.md) - Agent to UI streaming protocol
- [核心概念.md](05-进阶/核心概念.md) - Core concepts
- [进阶专题.md](05-进阶/进阶专题.md) - Advanced topics

### Exercises
- [练习册.md](06-练习/练习册.md) - Practice problems

### Reference
- [常见问题.md](附录/常见问题.md) - FAQ (24+ Q&A)
- [术语表.md](附录/术语表.md) - Glossary (80+ terms)
- [速查手册.md](01-入门/速查手册.md) - Quick reference

## Document Statistics

| Section | Content | Lines |
|---------|---------|-------|
| 01-入门 | Hand-on tutorial, quick reference | ~2300 |
| 02-组件 | Model, Tool, Memory, Retrieval, Callback | ~4500 |
| 03-编排 | Graph, Workflow, Stream, Batch | ~3000 |
| 04-智能体 | Agent, ADK, Best practices | ~2500 |
| 05-进阶 | HITL, MultiAgent, A2UI, Core concepts | ~5000 |
| 06-练习 | Practice problems | ~500 |
| 附录 | Quick start, FAQ, Glossary | ~1500 |

**Total: ~19,000 lines**

## How to Use

When helping users learn Eino:

1. **For beginners**: Start with [附录/快速开始.md](附录/快速开始.md) or [01-入门/手把手教程.md](01-入门/手把手教程.md)
2. **For concept understanding**: Direct to relevant tutorial in the learning path
3. **For troubleshooting**: Check [附录/常见问题.md](附录/常见问题.md)

## Core Learning Modules

### Quick Start Path
→ [快速开始](附录/快速开始.md) → [手把手教程](01-入门/手把手教程.md) → [速查手册](01-入门/速查手册.md)

### Component Fundamentals
→ [模型](02-组件/教程-模型.md) → [工具](02-组件/教程-工具.md) → [内存](02-组件/教程-内存.md)

### Advanced Skills
→ [Agent 开发](04-智能体/教程-Agent.md) → [ADK 完整指南](04-智能体/教程-ADK.md) → [人机交互](05-进阶/进阶-人机交互.md)

### Protocol Reference
→ [A2UI 协议](05-进阶/A2UI-协议.md) - Agent to UI 流式渲染协议

## Official Skills (eino-skills)

For production development, use the official Eino skills package:

| Skill | Focus | When to Use |
|-------|-------|-------------|
| `/eino-skills` | Official skill package | Contains all official skills below |
| `/eino-guide` | Framework overview | Architecture, navigation |
| `/eino-component` | Component reference | Models, embeddings, tools, retrievers |
| `/eino-compose` | Orchestration reference | Graph, Chain, Workflow |
| `/eino-agent` | Agent development | ADK, agents, middleware, runners |

**Usage**: Say `/eino-skills` or `/eino-component` etc. to activate official technical skills.

## Reference Files

Read these files on-demand for detailed learning content:

- [附录/快速开始.md](附录/快速开始.md) -- 5-minute quick start
- [01-入门/手把手教程.md](01-入门/手把手教程.md) -- 10-step hands-on tutorial
- [02-组件/教程-模型.md](02-组件/教程-模型.md) -- ChatModel tutorial
- [04-智能体/教程-ADK.md](04-智能体/教程-ADK.md) -- ADK complete guide
- [05-进阶/进阶-人机交互.md](05-进阶/进阶-人机交互.md) -- Human-in-the-loop
- [05-进阶/进阶-MultiAgent.md](05-进阶/进阶-MultiAgent.md) -- Multi-agent systems
- [05-进阶/A2UI-协议.md](05-进阶/A2UI-协议.md) -- A2UI protocol
- [附录/常见问题.md](附录/常见问题.md) -- 24+ common questions
- [附录/术语表.md](附录/术语表.md) -- 80+ terms
