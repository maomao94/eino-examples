# 04-智能体

本章节深入讲解 ADK 智能体开发套件。

---

## 文件列表

| 文件 | 说明 |
|------|------|
| [教程-ADK.md](教程-ADK.md) | ADK 核心：Runner、Supervisor、Plan-Execute |
| [指南-ADK.md](指南-ADK.md) | 按目录结构学习 adk/ 示例 |
| [教程-Agent.md](教程-Agent.md) | Agent 基础：ReAct、ChatModelAgent |

---

## 学习内容

### ADK 核心

#### Runner 运行器

- Agent 的运行环境
- 生命周期管理
- 流式输出
- 断点恢复

#### Agent 类型

- ChatModelAgent：基于 ChatModel
- SequentialAgent：顺序执行
- ParallelAgent：并行执行
- LoopAgent：循环反思

#### Supervisor 监督者

- 任务分配
- 子 Agent 协调
- 结果整合

#### Plan-Execute 规划执行

- Planner：任务分解
- Executor：执行步骤
- Replanner：检查调整

### 人机交互

- Approval 批准机制
- Interrupt 中断
- Checkpoint 检查点

---

## 下一步

开始动手练习 [06-练习](../06-练习/README.md)
