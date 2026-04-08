---
name: eino-learning
description: Eino framework learning guide and tutorials. Use when a user wants to learn Eino from scratch, follow a structured learning path, or needs beginner-friendly explanations. Covers getting started, component tutorials, orchestration concepts, agent development, advanced topics, and exercises. This is a tutorial/guide skill, while /eino-agent, /eino-component, /eino-compose are technical reference skills.
---

# Eino Learning Guide

Welcome to the Eino learning guide! This skill contains tutorials and explanations for learning Eino from beginner to advanced.

## Learning Path

### Stage 1: Quick Start
- [快速开始](study/附录/快速开始.md) - 5 minutes to run your first program

### Stage 2: Components
- [教程-模型.md](study/02-组件/教程-模型.md) - ChatModel basics
- [教程-工具.md](study/02-组件/教程-工具.md) - Tool development
- [教程-内存.md](study/02-组件/教程-内存.md) - Memory management
- [教程-检索.md](study/02-组件/教程-检索.md) - RAG and retrieval
- [教程-回调.md](study/02-组件/教程-回调.md) - Callback mechanism

### Stage 3: Orchestration
- [教程-编排.md](study/03-编排/教程-编排.md) - Graph and Workflow
- [教程-流编排.md](study/03-编排/教程-流编排.md) - Stream processing
- [教程-批处理.md](study/03-编排/教程-批处理.md) - Batch processing

### Stage 4: Agent
- [教程-Agent.md](study/04-智能体/教程-Agent.md) - ReAct Agent basics
- [教程-ADK.md](study/04-智能体/教程-ADK.md) - ADK complete guide
- [指南-ADK.md](study/04-智能体/指南-ADK.md) - ADK best practices

### Stage 5: Advanced
- [进阶-人机交互.md](study/05-进阶/进阶-人机交互.md) - Interrupt and checkpoint
- [进阶-MultiAgent.md](study/05-进阶/进阶-MultiAgent.md) - Multi-agent systems
- [核心概念.md](study/05-进阶/核心概念.md) - Core concepts
- [进阶专题.md](study/05-进阶/进阶专题.md) - Advanced topics

### Exercises
- [练习册.md](study/06-练习/练习册.md) - Practice problems

### Reference
- [常见问题.md](study/附录/常见问题.md) - FAQ (24+ Q&A)
- [术语表.md](study/附录/术语表.md) - Glossary (80+ terms)
- [速查手册.md](study/01-入门/速查手册.md) - Quick reference

## Document Statistics

| Section | Content | Lines |
|---------|---------|-------|
| 01-入门 | Hand-on tutorial, quick reference | ~2300 |
| 02-组件 | Model, Tool, Memory, Retrieval, Callback | ~4500 |
| 03-编排 | Graph, Workflow, Stream, Batch | ~3000 |
| 04-智能体 | Agent, ADK, Best practices | ~2500 |
| 05-进阶 | HITL, MultiAgent, Core concepts | ~5000 |
| 06-练习 | Practice problems | ~500 |
| 附录 | Quick start, FAQ, Glossary | ~1500 |

**Total: ~19,000 lines**

## How to Use

When helping users learn Eino:

1. **For beginners**: Start with [附录/快速开始.md](study/附录/快速开始.md) or [01-入门/手把手教程.md](study/01-入门/手把手教程.md)
2. **For concept understanding**: Direct to relevant tutorial in the learning path
3. **For API details**: Recommend using `/eino-agent`, `/eino-component`, or `/eino-compose`
4. **For troubleshooting**: Check [附录/常见问题.md](study/附录/常见问题.md)

## Relationship with Other Skills

| Skill | Focus | When to Use |
|-------|-------|-------------|
| /eino-learning | Learning & tutorials | Beginners, understanding concepts |
| /eino-guide | Framework overview | Architecture, navigation |
| /eino-component | Technical reference | API details, component config |
| /eino-compose | Technical reference | Graph, Chain, Workflow |
| /eino-agent | Technical reference | ADK, Agent, Middleware |

## Key Differences from Technical Skills

- **This skill**: Tutorial-first, explains concepts with examples, learning path
- **Technical skills**: Code-first, API references, production usage

## Reference Files

Read these files on-demand for detailed learning content:

- [study/附录/快速开始.md](study/附录/快速开始.md) -- 5-minute quick start
- [study/01-入门/手把手教程.md](study/01-入门/手把手教程.md) -- 10-step hands-on tutorial
- [study/02-组件/教程-模型.md](study/02-组件/教程-模型.md) -- ChatModel tutorial
- [study/04-智能体/教程-ADK.md](study/04-智能体/教程-ADK.md) -- ADK complete guide
- [study/05-进阶/进阶-人机交互.md](study/05-进阶/进阶-人机交互.md) -- Human-in-the-loop
- [study/05-进阶/进阶-MultiAgent.md](study/05-进阶/进阶-MultiAgent.md) -- Multi-agent systems
- [study/附录/常见问题.md](study/附录/常见问题.md) -- 24+ common questions
- [study/附录/术语表.md](study/附录/术语表.md) -- 80+ terms
