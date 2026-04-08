# ADK 目录学习指南

本文档帮助你按照 `adk/` 目录结构系统学习 Eino ADK。

---

## 目录结构总览

```
adk/
├── helloworld/              # 🚀 第一步：HelloWorld
├── intro/                   # 📚 入门示例
│   ├── agent_with_summarization/   # 带摘要的 Agent
│   ├── chatmodel/          # ChatModel 使用
│   ├── custom/             # 自定义 Agent
│   ├── http-sse-service/   # HTTP SSE 服务
│   ├── session/            # Session 状态
│   ├── transfer/            # Agent 间转接
│   └── workflow/            # 工作流模式
│       ├── sequential/     # 顺序执行
│       ├── parallel/       # 并行执行
│       └── loop/           # 循环执行
├── human-in-the-loop/      # 🤝 人机交互
│   ├── 1_approval/         # 批准机制
│   ├── 2_review-and-edit/  # 审核编辑
│   ├── 3_feedback-loop/    # 反馈循环
│   ├── 4_follow-up/        # 跟进机制
│   ├── 5_supervisor/       # 监督者模式
│   ├── 6_plan-execute-replan/  # 规划执行
│   ├── 7_deep-agents/      # 深度 Agent
│   └── 8_supervisor-plan-execute/  # 监督者+规划执行
├── multiagent/              # 👥 多 Agent 系统
│   ├── deep/               # Deep Agent
│   └── integration-excel-agent/  # Excel 集成
├── common/                  # 🔧 公共组件
│   ├── model/              # 模型配置
│   ├── prints/             # 输出格式化
│   ├── store/              # 状态存储
│   └── tool/               # 工具基类
└── middlewares/            # 🔌 中间件
    ├── dynamictool/         # 动态工具
    └── skill/              # 技能系统
```

---

## 学习路径

### 路径一：快速入门

```
1. helloworld/          → 5 分钟跑通第一个 Agent
2. intro/workflow/      → 理解 3 种工作流模式
3. 教程-ADK.md      → 掌握 ADK 核心概念
```

### 路径二：人机交互

```
1. human-in-the-loop/1_approval/     → 批准机制
2. human-in-the-loop/5_supervisor/   → Supervisor 模式
3. human-in-the-loop/6_plan-execute/ → Plan-Execute 模式
4. 教程-ADK.md (人机交互章节)     → 深入理解
```

### 路径三：多 Agent 系统

```
1. intro/workflow/sequential/  → 顺序协作
2. intro/workflow/parallel/   → 并行协作
3. intro/workflow/loop/       → 循环反思
4. human-in-the-loop/5_supervisor/  → 监督者模式
5. multiagent/deep/           → Deep Agent 系统
```

### 路径四：高级特性

```
1. human-in-the-loop/7_deep-agents/           → 深度 Agent
2. human-in-the-loop/8_supervisor-plan-execute/ → 组合模式
3. multiagent/integration-excel-agent/       → 实际项目
4. ../05-进阶/进阶专题.md                               → 进阶专题
```

---

## 示例详解

### 1. helloworld - 最小可用示例

**文件**: `adk/helloworld/helloworld.go`

最简单的 Agent 示例，5 分钟跑通。

```go
// 核心代码
agent, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Model: model,
})

runner := adk.NewRunner(ctx, adk.RunnerConfig{
    EnableStreaming: true,
    Agent:          agent,
})

iter := runner.Query(ctx, "你好")
```

**学习重点**:
- Runner 的创建和使用
- AsyncIterator 遍历事件
- 流式输出处理

---

### 2. intro/workflow/sequential - 顺序执行

**文件**: `adk/intro/workflow/sequential/`

按顺序执行多个 Agent，上一个的输出作为下一个的输入。

```go
seqAgent, _ := adk.NewSequentialAgent(ctx, &adk.SequentialAgentConfig{
    SubAgents: []adk.Agent{
        subagents.NewPlanAgent(),   // 第一步：规划
        subagents.NewWriterAgent(), // 第二步：写作
    },
})
```

**执行流程**:
```
START → PlanAgent → WriterAgent → END
```

**学习重点**:
- SequentialAgent 配置
- 子 Agent 之间的数据传递
- 适合有明确依赖的任务

---

### 3. intro/workflow/parallel - 并行执行

**文件**: `adk/intro/workflow/parallel/`

同时执行多个 Agent，收集所有结果。

```go
paraAgent, _ := adk.NewParallelAgent(ctx, &adk.ParallelAgentConfig{
    SubAgents: []adk.Agent{
        subagents.NewStockDataCollectionAgent(),
        subagents.NewNewsDataCollectionAgent(),
        subagents.NewSocialMediaInfoCollectionAgent(),
    },
})
```

**执行流程**:
```
                    → [股票数据] ─┐
START → [并行执行] → [新闻数据] ─┼→ [汇总] → END
                    → [社交数据] ─┘
```

**学习重点**:
- ParallelAgent 配置
- 并行收集结果
- 适合独立任务的并行处理

---

### 4. intro/workflow/loop - 循环反思

**文件**: `adk/intro/workflow/loop/`

主 Agent 执行，批评 Agent 评估，不满意则循环。

```go
loopAgent, _ := adk.NewLoopAgent(ctx, &adk.LoopAgentConfig{
    SubAgents: []adk.Agent{
        subagents.NewMainAgent(),     // 执行
        subagents.NewCritiqueAgent(), // 批评
    },
    MaxIterations: 5,
})
```

**执行流程**:
```
┌────────────────────────────────┐
│                                │
│  MainAgent → CritiqueAgent ────┼──→ 满意 → END
│     ↑              │           │
│     │              │ 不满意     │
│     └──────────────┘           │
│         (循环)                 │
└────────────────────────────────┘
```

**学习重点**:
- LoopAgent 配置
- 反思机制的实现
- MaxIterations 防止无限循环

---

### 5. human-in-the-loop/1_approval - 批准机制

**文件**: `adk/human-in-the-loop/1_approval/`

敏感操作需要人类批准才能执行。

**关键代码**:

```go
// 工具包装为需要批准
approvableTool := &tool.InvokableApprovableTool{
    InvokableTool: transferTool,
}

// Agent 配置
agent, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    ToolsConfig: adk.ToolsConfig{
        compose.ToolsNodeConfig{
            Tools: []tool.BaseTool{approvableTool},
        },
    },
})
```

**处理中断**:

```go
// 检查中断
if lastEvent.Action.Interrupted != nil {
    interruptID := lastEvent.Action.Interrupted.InterruptContexts[0].ID

    // 获取用户批准
    approval := getUserApproval()

    // 恢复执行
    iter, _ = runner.ResumeWithParams(ctx, "checkpoint-1", &adk.ResumeParams{
        Targets: map[string]any{interruptID: approval},
    })
}
```

**学习重点**:
- InvokableApprovableTool 使用
- 中断检测和处理
- ResumeWithParams 恢复执行
- CheckPointStore 状态持久化

---

### 6. human-in-the-loop/5_supervisor - 监督者模式

**文件**: `adk/human-in-the-loop/5_supervisor/`

一个监督者 Agent 协调多个专业子 Agent。

**架构**:

```
用户请求
    ↓
┌─────────────────────────────────────┐
│         Supervisor                   │
│  "分析任务，分配给合适的 Agent"       │
└─────────────────────────────────────┘
    ↓
┌──────────────┐    ┌──────────────┐
│ Account Agent│    │ Transaction  │
│  (查余额)    │    │ Agent (转账) │
└──────────────┘    └──────────────┘
```

**关键代码**:

```go
// 1. 创建子 Agent
accountAgent, _ := buildAccountAgent(ctx)
transactionAgent, _ := buildTransactionAgent(ctx)

// 2. 创建 Supervisor Agent
supervisorAgent, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Name:        "financial_supervisor",
    Description: "财务监督者",
    Instruction: `你是财务顾问监督者...
- account_agent: 处理账户查询
- transaction_agent: 处理转账`,
    Exit: &adk.ExitTool{}, // 结束对话
})

// 3. 组合
sv, _ := supervisor.New(ctx, &supervisor.Config{
    Supervisor: supervisorAgent,
    SubAgents:  []adk.Agent{accountAgent, transactionAgent},
})
```

**子 Agent 实现要点**:

```go
return adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Name:        "account_agent", // 重要！Supervisor 通过名称调用
    Description: "负责账户查询",
    Instruction: `你是账户信息 Agent...
完成后直接返回结果，不要添加其他文字。`,
    Model: chatModel,
    ToolsConfig: adk.ToolsConfig{...},
})
```

**学习重点**:
- Supervisor 创建流程
- 子 Agent 的 Name 和 Description 重要性
- SupervisorAgent 的 Exit 工具配置
- 协调多个专业 Agent

---

### 7. human-in-the-loop/6_plan-execute-replan - 规划执行

**文件**: `adk/human-in-the-loop/6_plan-execute-replan/`

先规划、后执行、发现问题再调整。

**架构**:

```
用户任务
    ↓
┌────────────┐
│   Planner  │ → 分解任务为步骤
└─────┬──────┘
      ↓
┌────────────┐
│  Executor  │ → 执行当前步骤
└─────┬──────┘
      ↓
┌────────────┐
│ Replanner  │ → 检查是否完成
└────────────┘
      ↓
  继续/结束
```

**关键代码**:

```go
// 创建三个 Agent
planner, _ := NewPlanner(ctx)
executor, _ := NewExecutor(ctx)
replaner, _ := NewReplanner(ctx)

// 组合
travelAgent, _ := planexecute.New(ctx, &planexecute.Config{
    Planner:       planner,
    Executor:      executor,
    Replanner:     replaner,
    MaxIterations: 20,
})
```

**学习重点**:
- Planner/Executor/Replanner 分工
- 循环执行直到完成
- 动态调整计划

---

### 8. human-in-the-loop/7_deep-agents - 深度 Agent

**文件**: `adk/human-in-the-loop/7_deep-agents/`

更复杂的 Agent 系统，支持多轮对话和深度思考。

**特点**:
- 深度问题分析
- 多轮工具调用
- 复杂任务分解

---

### 9. multiagent/deep - Deep Agent 系统

**文件**: `adk/multiagent/deep/`

完整的 Deep Agent 实现，包括：
- 代码执行
- 文件操作
- 搜索工具
- 提交结果

---

## 公共组件

### common/model - 模型配置

```go
// 统一的模型获取方式
model := model.NewChatModel()
```

### common/store - 状态存储

```go
// 内存存储（测试）
store.NewInMemoryStore()

// 检查点保存
checkpointStore.Set(ctx, id, data)

// 检查点恢复
data, ok, _ := checkpointStore.Get(ctx, id)
```

### common/tool - 工具基类

```go
// 批准信息
type ApprovalInfo struct {
    ToolName        string
    ArgumentsInJSON string
}

// 批准结果
type ApprovalResult struct {
    Approved         bool
    DisapproveReason *string
}

// 需要批准的工具包装
&InvokableApprovableTool{InvokableTool: myTool}
```

---

## 中间件

### middlewares/skill - 技能系统

```go
// 技能中间件
skillMW, _ := skill.New(ctx, &skill.Config{
    Skills: []skill.Skill{...},
})
```

---

## 总结

| 示例 | 复杂度 | 核心概念 |
|------|--------|---------|
| helloworld | ⭐ | Runner 基础 |
| sequential | ⭐⭐ | 顺序执行 |
| parallel | ⭐⭐ | 并行执行 |
| loop | ⭐⭐ | 循环反思 |
| approval | ⭐⭐⭐ | 人机交互 |
| supervisor | ⭐⭐⭐ | 监督者模式 |
| plan-execute | ⭐⭐⭐ | 规划执行 |
| deep-agents | ⭐⭐⭐⭐ | 深度 Agent |

---

## 下一步

1. **快速体验**: 运行 `helloworld`
2. **理解模式**: 学习 `workflow/` 下的三种模式
3. **实战应用**: 实现 `human-in-the-loop/` 中的场景
4. **深入研究**: 阅读 `multiagent/` 完整实现
5. **理论提升**: 阅读 [教程-ADK.md](教程-ADK.md)
