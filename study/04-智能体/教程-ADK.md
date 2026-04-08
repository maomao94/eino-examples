# Eino ADK 智能体开发指南

本文档详细讲解 Eino ADK（Agent Development Kit）的使用方法，包括基础组件、Agent 类型、人机交互、多 Agent 协作等核心功能。所有代码示例均配有详细中文注释。

---

## 目录

1. [ADK 概述](#1-adk-概述)
2. [Runner 运行器](#2-runner-运行器)
3. [Agent 类型详解](#3-agent-类型详解)
4. [ChatModelAgent 详解](#4-chatmodelagent-详解)
5. [Tool 工具开发](#5-tool-工具开发)
6. [人机交互 (Human-in-the-Loop)](#6-人机交互-human-in-the-loop)
7. [多 Agent 协作模式](#7-多-agent-协作模式)
8. [Supervisor 监督者模式](#8-supervisor-监督者模式)
9. [Plan-Execute-Replan 规划执行模式](#9-plan-execute-replan-规划执行模式)
10. [Checkpoint 检查点与恢复](#10-checkpoint-检查点与恢复)
11. [中间件 Middleware](#11-中间件-middleware)
12. [Callback 回调机制](#12-callback-回调机制)

---

## 1. ADK 概述

### 1.1 ADK 是什么

ADK（Agent Development Kit）是 Eino 框架中用于开发智能体应用的核心工具包。它提供了：

| 组件 | 说明 |
|------|------|
| `Runner` | Agent 的运行容器，管理执行生命周期 |
| `Agent` | 智能体的核心接口定义 |
| `ChatModelAgent` | 基于 ChatModel 的标准 Agent 实现 |
| `SequentialAgent` | 顺序执行多个子 Agent |
| `ParallelAgent` | 并行执行多个子 Agent |
| `LoopAgent` | 循环执行 Agent（含批评机制） |

### 1.2 目录结构

```
adk/
├── intro/                          # 入门示例
│   ├── agent_with_summarization/   # 带摘要功能的 Agent
│   ├── chatmodel/                 # ChatModel 使用
│   ├── custom/                     # 自定义 Agent
│   ├── http-sse-service/           # HTTP SSE 服务
│   ├── session/                    # Session 状态管理
│   ├── transfer/                   # Agent 间转接
│   └── workflow/                   # 工作流
│       ├── sequential/            # 顺序执行
│       ├── parallel/              # 并行执行
│       └── loop/                  # 循环执行
├── human-in-the-loop/              # 人机交互
│   ├── 1_approval/                 # 批准机制
│   ├── 2_review-and-edit/         # 审核编辑
│   ├── 3_feedback-loop/            # 反馈循环
│   ├── 4_follow-up/                # 跟进机制
│   ├── 5_supervisor/               # 监督者模式
│   ├── 6_plan-execute-replan/      # 规划执行模式
│   ├── 7_deep-agents/              # 深度 Agent
│   └── 8_supervisor-plan-execute/  # 监督者+规划执行
├── multiagent/                      # 多 Agent 示例
│   ├── deep/                       # 深度 Agent 系统
│   └── integration-excel-agent/    # Excel 集成 Agent
├── common/                          # 公共组件
│   ├── model/                      # 模型配置
│   ├── prints/                     # 输出格式化
│   ├── store/                      # 状态存储
│   └── tool/                       # 工具基类
└── middlewares/                     # 中间件
    ├── dynamictool/                # 动态工具
    └── skill/                      # 技能系统
```

---

## 2. Runner 运行器

### 2.1 Runner 作用

Runner 是 Agent 的"运行环境"，负责：
- 管理 Agent 的生命周期
- 处理输入输出
- 支持流式输出（SSE）
- 支持断点恢复（Checkpoint）

### 2.2 基本使用

```go
import "github.com/cloudwego/eino/adk"

// 创建 Agent
agent, err := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    // Agent 配置...
})

// 创建 Runner
runner := adk.NewRunner(ctx, adk.RunnerConfig{
    EnableStreaming: true,  // 开启流式输出
    Agent:           agent, // 绑定的 Agent
})

// 查询
iter := runner.Query(ctx, "用户的问题")

// 处理返回的事件流
for {
    event, ok := iter.Next()
    if !ok {
        break // 处理完成
    }

    // event 包含不同类型的事件
    if event.Err != nil {
        // 处理错误
        log.Fatal(event.Err)
    }

    prints.Event(event) // 打印事件详情

    // 获取最终输出
    if event.Output != nil {
        msg, _, _ := adk.GetMessage(event)
        fmt.Println(msg.Content)
    }
}
```

### 2.3 Runner 配置项

```go
runner := adk.NewRunner(ctx, adk.RunnerConfig{
    // 是否启用流式输出（SSE）
    // 开启后支持打字机效果
    EnableStreaming: true,

    // Agent 实例
    Agent: agent,

    // 检查点存储（用于断点恢复）
    // 使用内存存储（测试环境）
    CheckPointStore: store.NewInMemoryStore(),
    // 生产环境应使用分布式存储，如 Redis
    // CheckPointStore: redisStore,
})
```

### 2.4 带检查点的查询

```go
// 使用唯一的 CheckPointID 来标识这次执行
// 用于后续的断点恢复
iter := runner.Query(ctx, "用户的查询",
    adk.WithCheckPointID("unique-session-id-001"),
)

// 处理完成后，可以通过相同的 CheckPointID 恢复执行
iter, err := runner.ResumeWithParams(ctx, "unique-session-id-001", &adk.ResumeParams{
    Targets: map[string]any{
        // 中断点的 ID -> 恢复数据
        interruptID: approvalResult,
    },
})
```

---

## 3. Agent 类型详解

### 3.1 Agent 类型总览

```
┌─────────────────────────────────────────────────────────────┐
│                        adk.Agent                            │
│                      (接口定义)                             │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ ChatModelAgent│   │SequentialAgent│   │ ParallelAgent │
│ (基础类型)     │   │ (顺序执行)     │   │ (并行执行)     │
└───────────────┘   └───────────────┘   └───────────────┘
                            │                   │
                            ▼                   ▼
                    ┌───────────────┐
                    │  LoopAgent    │
                    │ (循环+批评)    │
                    └───────────────┘
```

### 3.2 ChatModelAgent（基础类型）

最常用的 Agent 类型，基于 ChatModel 实现 ReAct 模式。

```go
agent, err := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Name:        "my_agent",           // Agent 名称
    Description: "我的第一个 Agent",     // 描述（用于 Supervisor 路由）
    Instruction: "你是一个助手...",      // 系统提示词
    Model:       chatModel,            // 使用的模型
    ToolsConfig: adk.ToolsConfig{},    // 工具配置
    MaxIterations: 30,                 // 最大迭代次数
})
```

### 3.3 SequentialAgent（顺序执行）

按顺序执行多个子 Agent，每个 Agent 完成后将结果传递给下一个。

```go
// 创建子 Agent
planAgent := subagents.NewPlanAgent()     // 规划 Agent
writerAgent := subagents.NewWriterAgent() // 写作 Agent

// 创建顺序 Agent
seqAgent, err := adk.NewSequentialAgent(ctx, &adk.SequentialAgentConfig{
    Name:        "ResearchAgent",        // 名称
    Description: "研究助手，按顺序规划并写作", // 描述
    SubAgents: []adk.Agent{
        planAgent,   // 第一步：规划
        writerAgent, // 第二步：写作
    },
})

// 使用
runner := adk.NewRunner(ctx, adk.RunnerConfig{
    EnableStreaming: true,
    Agent:           seqAgent,
})

iter := runner.Query(ctx, "写一篇关于 AI 的报告")
// 执行流程: START → PlanAgent → WriterAgent → END
```

### 3.4 ParallelAgent（并行执行）

同时执行多个子 Agent，收集所有结果后汇总。

```go
// 创建子 Agent
stockAgent := subagents.NewStockDataCollectionAgent()    // 股票数据
newsAgent := subagents.NewNewsDataCollectionAgent()        // 新闻数据
socialAgent := subagents.NewSocialMediaInfoCollectionAgent() // 社交媒体

// 创建并行 Agent
paraAgent, err := adk.NewParallelAgent(ctx, &adk.ParallelAgentConfig{
    Name:        "DataCollectionAgent",
    Description: "数据收集助手，从多个来源收集数据",
    SubAgents: []adk.Agent{
        stockAgent,   // 并行执行
        newsAgent,    // 并行执行
        socialAgent,  // 并行执行
    },
})

// 执行流程: START → [三个 Agent 并行] → END
```

### 3.5 LoopAgent（循环+批评）

主 Agent 执行任务，批评 Agent 评估结果，如果不满意则循环。

```go
mainAgent := subagents.NewMainAgent()      // 主 Agent（执行任务）
criticAgent := subagents.NewCritiqueAgent() // 批评 Agent（评估）

loopAgent, err := adk.NewLoopAgent(ctx, &adk.LoopAgentConfig{
    Name:          "reflection_agent",
    Description:   "反思 Agent，通过批评改进结果",
    SubAgents: []adk.Agent{
        mainAgent,   // 执行 Agent
        criticAgent, // 批评 Agent
    },
    MaxIterations: 5, // 最多循环 5 次
})

// 执行流程:
// 1. mainAgent 执行
// 2. criticAgent 评估
// 3. 如果不满意 → 返回步骤 1
// 4. 满意或达到最大循环 → 结束
```

---

## 4. ChatModelAgent 详解

### 4.1 完整配置项

```go
agent, err := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    // ========== 基础信息 ==========
    Name:        "assistant",           // Agent 名称
    Description: "通用助手",             // 描述（重要！Supervisor 根据这个路由）

    // ========== 提示词 ==========
    Instruction: `你是一个专业的助手。
要求：
1. 回答简洁明了
2. 如果需要工具，使用工具来完成任务
3. 完成后给出总结`,
    // 或者使用 Prompt 对象
    // Prompt: myPrompt,

    // ========== 模型配置 ==========
    Model: chatModel, // 必须：支持工具调用的 ChatModel

    // ========== 工具配置 ==========
    ToolsConfig: adk.ToolsConfig{
        // ToolsNodeConfig 与 compose 包一致
        compose.ToolsNodeConfig{
            Tools: []tool.BaseTool{
                myTool1,
                myTool2,
            },
        },
    },

    // ========== 执行控制 ==========
    MaxIterations: 30,     // 最大迭代次数（防止无限循环）
    Timeout:       5*time.Minute, // 超时时间

    // ========== 中间件 ==========
    // 用于扩展 Agent 行为，如日志、监控、摘要等
    Middlewares: []adk.AgentMiddleware{
        myMiddleware1,
        myMiddleware2,
    },

    // ========== 退出机制 ==========
    // Exit 工具：让 Agent 主动结束对话
    Exit: &adk.ExitTool{},

    // ========== 未知工具处理 ==========
    // 当 LLM 调用不存在的工具时的处理
    UnknownToolHandler: myUnknownToolHandler,
})
```

### 4.2 带中间件的示例

```go
// 摘要中间件：自动压缩过长的对话历史
sumMW, err := summarization.New(ctx, &summarization.Config{
    Model:                      chatModel,  // 用于生成摘要的模型
    MaxTokensBeforeSummary:     10 * 1024,  // 超过这个 token 数开始摘要
    MaxTokensForRecentMessages: 2 * 1024,   // 保留最近 2K token
})

agent, err := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Name:        "report_writer",
    Description: "长篇报告助手",
    Instruction: `你是一个长篇报告写作助手。
分步骤思考，通过工具扩展内容，20 次工具调用后生成最终总结。`,
    Model:       chatModel,
    Middlewares: []adk.AgentMiddleware{sumMW}, // 添加中间件
    ToolsConfig: adk.ToolsConfig{
        compose.ToolsNodeConfig{
            Tools: []tool.BaseTool{
                NewRepeatSectionsTool(), // 重复段落扩展工具
            },
        },
    },
    MaxIterations: 30,
})
```

---

## 5. Tool 工具开发

### 5.1 工具基础

在 Eino 中，工具是 Agent 与外部交互的桥梁。

```go
import "github.com/cloudwego/eino/components/tool"
import "github.com/cloudwego/eino/components/tool/utils"
```

### 5.2 使用 utils.InferTool 创建工具（推荐）

最简单的方式，自动推断参数类型：

```go
// 定义请求结构体
type SearchReq struct {
    Query string `json:"query" jsonschema_description:"搜索关键词"`
}

type SearchResp struct {
    Results []string `json:"results"`
    Total   int     `json:"total"`
}

// 实现工具函数
searchFunc := func(ctx context.Context, req *SearchReq) (*SearchResp, error) {
    results := []string{"结果1", "结果2", "结果3"}
    return &SearchResp{
        Results: results,
        Total:   len(results),
    }, nil
}

// 创建工具
searchTool, err := utils.InferTool(
    "search",                  // 工具名称
    "搜索互联网获取信息",        // 工具描述
    searchFunc,                // 工具函数
)
// 自动生成 JSON Schema，用于 LLM 理解参数
```

### 5.3 使用 schema.NewRunnableTool 创建工具

更灵活的控制：

```go
import "github.com/cloudwego/eino/schema"

tool := schema.NewRunnableTool(
    // 工具信息
    func(ctx context.Context) (*schema.ToolInfo, error) {
        return &schema.ToolInfo{
            Name:        "calculate",
            Desc:        "执行数学计算",
            Parameters:  myJSONSchema,
        }, nil
    },
    // 执行函数
    func(ctx context.Context, input string) (string, error) {
        // 解析参数
        var req CalculateRequest
        json.Unmarshal([]byte(input), &req)

        // 执行计算
        result := req.A + req.B

        return fmt.Sprintf("%d", result), nil
    },
)
```

### 5.4 InvokableTool vs RunnableTool

| 类型 | 接口 | 适用场景 |
|------|------|---------|
| `InvokableTool` | `InvokableRun(ctx, args JSON) (string, error)` | 参数类型确定 |
| `RunnableTool` | `Run(ctx, input any) (any, error)` | 参数类型灵活 |

```go
// InvokableTool - 传入和返回都是 JSON 字符串
type InvokableTool interface {
    Info(ctx context.Context) (*schema.ToolInfo, error)
    InvokableRun(ctx context.Context, argumentsInJSON string, opts ...Option) (string, error)
}

// RunnableTool - 传入和返回是泛型
type RunnableTool[Input any, Output any] interface {
    Info(ctx context.Context) (*schema.ToolInfo, error)
    Run(ctx context.Context, input Input) (Output, error)
}
```

### 5.5 工具中间件

工具也可以使用中间件来处理错误：

```go
import "github.com/cloudwego/eino/components/tool/middlewares"

// JSON 修复中间件：自动修复 LLM 返回的错误 JSON
fixer := jsonfix.New()

// 错误移除中间件：自动过滤错误消息中的噪声
remover := errorremover.New()

wrappedTool := fixer.Wrap(originalTool)
wrappedTool = remover.Wrap(wrappedTool)
```

---

## 6. 人机交互 (Human-in-the-Loop)

### 6.1 概述

人机交互允许 Agent 在执行过程中暂停，等待人类确认后再继续。

```
Agent 执行
    │
    ├──▶ Step 1
    ├──▶ Step 2
    │
    ▼
[中断点] ──────────────────┐
    │                       │
    ▼                       │
等待人类批准                 │
    │                       │
    ├── [批准] ──────────────┼──▶ 继续执行
    │                       │
    └── [拒绝] ──────────────┴──▶ 返回错误
```

### 6.2 Approval 批准工具

使用 `InvokableApprovableTool` 包装需要批准的敏感操作：

// 使用 InvokableApprovableTool 包装需要批准的敏感操作
// 注：工具函数通常直接在当前包中定义，或从官方库导入
// 官方路径：github.com/cloudwego/eino/adk/common/tool

// 创建基础工具
transferTool, err := utils.InferTool("transfer_funds", "转账工具", transferFunc)
if err != nil {
    return nil, err
}

// 包装为需要批准的工具
// 当 Agent 调用此工具时，会自动触发中断
approvableTool := &tool.InvokableApprovableTool{
    InvokableTool: transferTool,
}

// Agent 配置
return adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Name:        "transaction_agent",
    Description: "处理金融交易",
    Model:       chatModel,
    ToolsConfig: adk.ToolsConfig{
        compose.ToolsNodeConfig{
            Tools: []tool.BaseTool{approvableTool},
        },
    },
})
```

### 6.3 处理中断

```go
func main() {
    runner := adk.NewRunner(ctx, adk.RunnerConfig{
        EnableStreaming: true,
        Agent:          myAgent,
        CheckPointStore: store.NewInMemoryStore(),
    })

    iter := runner.Query(ctx, "帮我转账 500 元",
        adk.WithCheckPointID("tx-001"),
    )

    var lastEvent *adk.AgentEvent

    for {
        event, ok := iter.Next()
        if !ok {
            break
        }

        if event.Err != nil {
            log.Fatal(event.Err)
        }

        prints.Event(event)
        lastEvent = event
    }

    // 检查是否是中断事件
    if lastEvent.Action != nil && lastEvent.Action.Interrupted != nil {
        // 获取中断上下文
        interruptCtx := lastEvent.Action.Interrupted.InterruptContexts[0]
        interruptID := interruptCtx.ID

        fmt.Println("需要批准的操作:")
        // 解析中断信息
        if info, ok := interruptCtx.Info.(*tool.ApprovalInfo); ok {
            fmt.Printf("工具: %s\n", info.ToolName)
            fmt.Printf("参数: %s\n", info.ArgumentsInJSON)
        }

        // 等待用户输入
        fmt.Print("是否批准? (Y/N): ")
        scanner := bufio.NewScanner(os.Stdin)
        scanner.Scan()
        input := scanner.Text()

        var approvalResult *tool.ApprovalResult
        if strings.ToUpper(input) == "Y" {
            approvalResult = &tool.ApprovalResult{Approved: true}
        } else {
            fmt.Print("请输入拒绝原因: ")
            scanner.Scan()
            reason := scanner.Text()
            approvalResult = &tool.ApprovalResult{
                Approved:         false,
                DisapproveReason: &reason,
            }
        }

        // 恢复执行
        iter, err = runner.ResumeWithParams(ctx, "tx-001", &adk.ResumeParams{
            Targets: map[string]any{
                interruptID: approvalResult,
            },
        })

        // 继续处理剩余事件...
    }
}
```

### 6.4 自定义中断信息

```go
import "github.com/cloudwego/eino/components/tool"

// 定义中断时传递的信息
type MyInterruptInfo struct {
    Action    string
    Reason    string
    Timestamp time.Time
}

// 在工具中触发中断
func (t *MyTool) InvokableRun(ctx context.Context, args string, opts ...tool.Option) (string, error) {
    toolInfo, _ := t.Info(ctx)

    // 检查是否是从中断恢复
    wasInterrupted, _, storedArgs := tool.GetInterruptState[string](ctx)
    if !wasInterrupted {
        // 第一次执行，触发中断
        return "", tool.StatefulInterrupt(ctx, &MyInterruptInfo{
            Action: "DELETE_USER",
            Reason: "删除用户需要批准",
        }, args)
    }

    // 恢复执行，检查批准结果
    isTarget, hasData, decision := tool.GetResumeContext[*MyDecision](ctx)
    if isTarget && hasData {
        if decision.Approved {
            // 执行删除操作
            return t.doDelete(ctx, storedArgs)
        }
        return fmt.Sprintf("操作被拒绝: %s", *decision.Reason), nil
    }

    return "", nil
}
```

### 6.5 中断类型

| 类型 | 说明 | 使用场景 |
|------|------|---------|
| `tool.StatefulInterrupt` | 带状态的中断 | 需要传递上下文信息 |
| `compose.Interrupt` | Workflow 中断 | 在 Workflow/Graph 中使用 |
| `ErrApprovalRequired` | 需要批准 | 敏感操作 |

---

## 7. 多 Agent 协作模式

### 7.1 SequentialAgent 顺序执行

适用场景：任务有明确的先后依赖

```go
// 创建子 Agent
planner := createPlannerAgent()  // 规划
executor := createExecutorAgent() // 执行
reviewer := createReviewerAgent() // 审核

seqAgent, _ := adk.NewSequentialAgent(ctx, &adk.SequentialAgentConfig{
    Name:        "ReviewWorkflow",
    Description: "审核工作流：规划→执行→审核",
    SubAgents: []adk.Agent{
        planner,   // 第一步：分析任务并规划
        executor,  // 第二步：按照计划执行
        reviewer,  // 第三步：审核结果
    },
})

// 执行流程:
// User Query → Planner → Executor → Reviewer → User Response
```

### 7.2 ParallelAgent 并行执行

适用场景：任务可以分解为独立的子任务

```go
researcher := createResearcherAgent()  // 研究员
writer := createWriterAgent()          // 作家
coder := createCoderAgent()           // 程序员

paraAgent, _ := adk.NewParallelAgent(ctx, &adk.ParallelAgentConfig{
    Name:        "TeamAssistant",
    Description: "团队助手，同时处理研究、写作、编程任务",
    SubAgents: []adk.Agent{
        researcher,
        writer,
        coder,
    },
})

// 执行流程:
// User Query → [ Researcher ] ─┐
//                → [ Writer    ] ├→ Combined Response
//                → [ Coder     ] ─┘
```

### 7.3 LoopAgent 循环反思

适用场景：需要迭代改进的任务

```go
mainAgent := createMainAgent()    // 主 Agent
criticAgent := createCriticAgent() // 批评 Agent

loopAgent, _ := adk.NewLoopAgent(ctx, &adk.LoopAgentConfig{
    Name:          "ReflectionAgent",
    Description:   "反思助手，通过批评持续改进",
    SubAgents: []adk.Agent{
        mainAgent,   // 执行任务
        criticAgent, // 评估结果
    },
    MaxIterations: 5, // 最多 5 次循环
})

// 执行流程:
// ┌─────────────────────────────────────────┐
// │                                         │
// ▼                                         │
// MainAgent 执行 ──▶ CriticAgent 评估       │
//        ↑                    │            │
//        │                    │            │
//        │      不满意         │            │
//        │◀───────────────────┘            │
//        │                                  │
//        │           满意                   │
//        └──────────────────────────────────┼──▶ 结束
```

---

## 8. Supervisor 监督者模式

### 8.1 Supervisor 概述

Supervisor 是一个"调度员"Agent，它：
- 理解用户任务
- 将任务分配给合适的子 Agent
- 协调多个子 Agent 的执行
- 整合结果返回给用户

### 8.2 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                      User Query                             │
│           "帮我查一下账户余额，然后转账"                       │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                     Supervisor                               │
│         "需要调用 account_agent 和 transaction_agent"        │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ SYSTEM: 你是一个财务监督者，管理两个 Agent：          │    │
│  │   - account_agent: 查账户余额                        │    │
│  │   - transaction_agent: 执行转账                      │    │
│  │ 分析请求，委托给合适的 Agent，完成后总结结果           │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────┬───────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          │               │               │
          ▼               ▼               ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │ Account  │    │Transaction│    │   ...    │
    │  Agent   │    │   Agent   │    │          │
    └────┬─────┘    └────┬─────┘    └──────────┘
         │               │
         ▼               ▼
    查询余额          可能需要批准
         │               │
         └───────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                     Supervisor                               │
│              "账户余额 5000，转账成功"                        │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                      Response                               │
│         "您的 checking 账户余额为 5000 USD，                 │
│          已成功转账 500 USD 到 savings 账户"                  │
└─────────────────────────────────────────────────────────────┘
```

### 8.3 实现示例

```go
// 步骤 1: 创建子 Agent
accountAgent, err := buildAccountAgent(ctx)
if err != nil {
    return nil, err
}

transactionAgent, err := buildTransactionAgent(ctx)
if err != nil {
    return nil, err
}

// 步骤 2: 创建 Supervisor Agent
// Supervisor 本身也是一个 ChatModelAgent
supervisorAgent, err := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Name:        "financial_supervisor",
    Description: "财务监督者，协调账户和交易任务",
    Instruction: `你是一个财务顾问监督者，管理两个 Agent：

- account_agent: 处理账户相关查询，如查余额
- transaction_agent: 处理交易任务，如转账（需要批准）

要求：
1. 分析用户请求，委托给合适的 Agent
2. 如果请求涉及查余额和转账，先查余额，再转账
3. 每次只委托给一个 Agent，不要并行
4. 不要自己做工作，总是委托给 Agent
5. 任务完成后，为用户总结结果`,

    Model: chatModel,
    // Supervisor 需要 Exit 工具来结束对话
    Exit: &adk.ExitTool{},
})

// 步骤 3: 组合成 Supervisor
financialSV, err := supervisor.New(ctx, &supervisor.Config{
    Supervisor: supervisorAgent,        // 监督者 Agent
    SubAgents: []adk.Agent{            // 子 Agent 列表
        accountAgent,
        transactionAgent,
    },
})

return financialSV, nil
```

### 8.4 子 Agent 实现要点

```go
func buildAccountAgent(ctx context.Context) (adk.Agent, error) {
    // 余额查询工具
    balanceTool, _ := utils.InferTool(
        "check_balance",
        "查询账户余额",
        func(ctx context.Context, req *BalanceReq) (*BalanceResp, error) {
            balances := map[string]float64{
                "checking": 5000.00,
                "savings":  15000.00,
            }
            balance := balances[req.AccountID]
            return &BalanceResp{
                AccountID: req.AccountID,
                Balance:   balance,
                Currency:  "USD",
            }, nil
        },
    )

    return adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
        Name:        "account_agent",  // 重要！Supervisor 根据这个名称调用
        Description: "负责查询账户信息和余额",
        Instruction: `你是一个账户信息 Agent。

职责：
- 只处理账户相关查询，如查余额
- 使用 check_balance 工具获取账户信息
- 完成工作后，直接向监督者返回结果
- 只返回结果，不要添加其他文字。`,

        Model: chatModel,
        ToolsConfig: adk.ToolsConfig{
            compose.ToolsNodeConfig{
                Tools: []tool.BaseTool{balanceTool},
            },
        },
    })
}
```

### 8.5 处理 Supervisor 的中断

Supervisor 中的子 Agent 触发中断时，需要在主循环中处理：

```go
func processEvents(iter *adk.AsyncIterator[*adk.AgentEvent]) (*adk.AgentEvent, bool) {
    var lastEvent *adk.AgentEvent

    for {
        event, ok := iter.Next()
        if !ok {
            break
        }

        if event.Err != nil {
            log.Fatal(event.Err)
        }

        prints.Event(event)
        lastEvent = event
    }

    // 检查是否中断
    if lastEvent.Action != nil && lastEvent.Action.Interrupted != nil {
        return lastEvent, true
    }

    return lastEvent, false
}

// 主循环
for {
    lastEvent, interrupted := processEvents(iter)
    if !interrupted {
        break // 完成
    }

    // 处理中断
    interruptCtx := lastEvent.Action.Interrupted.InterruptContexts[0]
    interruptID := interruptCtx.ID

    // 获取用户批准...
    approval := getUserApproval(interruptCtx)

    // 恢复执行
    iter, err = runner.ResumeWithParams(ctx, checkpointID, &adk.ResumeParams{
        Targets: map[string]any{
            interruptID: approval,
        },
    })
}
```

---

## 9. Plan-Execute-Replan 规划执行模式

### 9.1 模式概述

这是一种"先规划、后执行、发现问题再调整"的智能模式：

1. **Planner（规划器）**: 将复杂任务分解为步骤
2. **Executor（执行器）**: 按步骤执行，使用工具
3. **Replanner（反思器）**: 检查结果，决定是继续还是调整计划

### 9.2 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                     User Task                               │
│               "帮我规划一次北京三日游"                        │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                       PLANNER                               │
│  "将任务分解为：1.查天气 2.订酒店 3.订机票 4.查景点"         │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                      EXECUTOR                               │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Step 1: 查天气                                       │   │
│  │   → 调用 get_weather(北京)                          │   │
│  │   → 天气：晴，15-25°C                               │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Step 2: 订酒店                                       │   │
│  │   → 调用 book_hotel(北京, 日期)                     │   │
│  │   → 需要用户批准                                     │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                      REPLANNER                              │
│  "步骤 1-2 完成，检查是否还有未完成的步骤"                    │
│                                                              │
│  如果还有步骤 → 返回 EXECUTOR 继续                            │
│  如果全部完成 → 返回 "最终答案" 结束                          │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                      Final Result                           │
│              "北京三日游规划：天气良好，酒店已订..."          │
└─────────────────────────────────────────────────────────────┘
```

### 9.3 实现示例

```go
// 步骤 1: 创建 Planner
func NewPlanner(ctx context.Context) (adk.Agent, error) {
    return planexecute.NewPlanner(ctx, &planexecute.PlannerConfig{
        // 使用支持工具调用的模型
        ToolCallingChatModel: chatModel,
    })
}

// 步骤 2: 创建 Executor
func NewExecutor(ctx context.Context) (adk.Agent, error) {
    travelTools, _ := GetAllTravelTools(ctx)

    return planexecute.NewExecutor(ctx, &planexecute.ExecutorConfig{
        Model: chatModel,
        ToolsConfig: adk.ToolsConfig{
            compose.ToolsNodeConfig{
                Tools: travelTools, // 旅行相关工具
            },
        },
        // 自定义输入格式化
        GenInputFn: func(ctx context.Context, in *planexecute.ExecutionContext) ([]adk.Message, error) {
            // in.Plan: 当前计划
            // in.ExecutedSteps: 已完成的步骤
            // in.UserInput: 用户原始输入

            firstStep := in.Plan.FirstStep()

            prompt := fmt.Sprintf(`
## 目标
%s

## 计划
%s

## 已完成步骤
%s

## 你的任务
执行第一步: %s
`, in.UserInput[0].Content, planJSON, executedStepsJSON, firstStep)

            return []adk.Message{{Content: prompt}}, nil
        },
    })
}

// 步骤 3: 创建 Replanner
func NewReplanner(ctx context.Context) (adk.Agent, error) {
    return planexecute.NewReplanner(ctx, &planexecute.ReplannerConfig{
        ChatModel: chatModel,
    })
}

// 步骤 4: 组合
func NewTravelPlanningAgent(ctx context.Context) (adk.Agent, error) {
    planner, _ := NewPlanner(ctx)
    executor, _ := NewExecutor(ctx)
    replanner, _ := NewReplanner(ctx)

    return planexecute.New(ctx, &planexecute.Config{
        Planner:       planner,   // 规划器
        Executor:      executor,  // 执行器
        Replanner:     replanner, // 反思器
        MaxIterations: 20,        // 最大迭代次数
    })
}
```

### 9.4 Plan-Execute 节点关系

```
START → Planner → Plan → Executor → Tool → Executor → Replanner
                                                      │
                                      ┌───────────────┼───────────────┐
                                      │               │               │
                                      ▼               ▼               ▼
                                    END          Executor        Executor
                                  (完成)        (继续执行)       (继续执行)
```

---

## 10. Checkpoint 检查点与恢复

### 10.1 概述

Checkpoint 允许 Agent 在中断后从断点继续执行，适用于：
- 人机交互场景
- 长任务分阶段执行
- 故障恢复

### 10.2 CheckPointStore 接口

```go
// 检查点存储接口
type CheckPointStore interface {
    // Get 获取检查点数据
    Get(ctx context.Context, id string) ([]byte, bool, error)
    // Set 保存检查点数据
    Set(ctx context.Context, id string, data []byte) error
}

// 内存存储（测试用）
store.NewInMemoryStore()

// Redis 存储（生产用）
redisStore, _ := NewRedisCheckPointStore(ctx, &RedisConfig{
    Addr:     "localhost:6379",
    Password: "",
    DB:       0,
})
```

### 10.3 使用示例

```go
// 创建 Runner 时配置 CheckPointStore
runner := adk.NewRunner(ctx, adk.RunnerConfig{
    EnableStreaming: true,
    Agent:          agent,
    CheckPointStore: store.NewInMemoryStore(), // 或 Redis 存储
})

// 第一阶段：开始执行
checkpointID := "my-task-001"
iter := runner.Query(ctx, "复杂任务",
    adk.WithCheckPointID(checkpointID),
)

// 处理事件...
for {
    event, ok := iter.Next()
    if !ok {
        break
    }

    if event.Err != nil {
        // 如果是中断
        if isInterruptError(event.Err) {
            // 保存状态，继续处理
            continue
        }
        log.Fatal(event.Err)
    }
}

// 第二阶段：从断点恢复
// 注意：可以是另一个进程或机器
iter, err := runner.ResumeWithParams(ctx, checkpointID, &adk.ResumeParams{
    Targets: map[string]any{
        interruptID: resumeData,
    },
})
```

### 10.4 断点信息提取

```go
import "github.com/cloudwego/eino/compose"

// 从错误中提取中断信息
info, ok := compose.ExtractInterruptInfo(err)
if ok {
    for _, ctx := range info.InterruptContexts {
        fmt.Printf("中断 ID: %s\n", ctx.ID)
        fmt.Printf("中断地址: %v\n", ctx.Address)
        fmt.Printf("中断信息: %v\n", ctx.Info)
    }

    // 准备恢复数据
    resumeData := make(map[string]any)
    for _, iCtx := range info.InterruptContexts {
        resumeData[iCtx.ID] = myDecision
    }

    // 恢复执行
    resumeCtx := compose.BatchResumeWithData(ctx, resumeData)
    result, err = runner.Invoke(resumeCtx, nil,
        compose.WithCheckPointID(checkpointID),
    )
}
```

---

## 11. 中间件 Middleware

### 11.1 AgentMiddleware 接口

```go
// AgentMiddleware 扩展 Agent 行为
type AgentMiddleware interface {
    // Wrap 包装一个 Agent，返回新的 Agent
    Wrap(agent Agent) Agent
}
```

### 11.2 摘要中间件示例

// 创建摘要中间件
// 注：实际使用时，摘要功能来自官方库 github.com/cloudwego/eino/adk/middleware/summarization
// 此处演示中间件的使用模式
sumMW, err := summarization.New(ctx, &summarization.Config{
    Model:                      chatModel, // 用于生成摘要的模型
    MaxTokensBeforeSummary:     10 * 1024,  // 超过 10K token 开始摘要
    MaxTokensForRecentMessages: 2 * 1024,   // 保留最近 2K token
})

// 包装 Agent
wrappedAgent := sumMW.Wrap(originalAgent)

// 使用包装后的 Agent
runner := adk.NewRunner(ctx, adk.RunnerConfig{
    EnableStreaming: true,
    Agent:           wrappedAgent, // 使用包装后的 Agent
})
```

### 11.3 常见中间件场景

| 中间件 | 功能 |
|--------|------|
| 摘要中间件 | 自动压缩过长的对话历史 |
| 日志中间件 | 记录所有交互 |
| 限流中间件 | 控制调用频率 |
| 监控中间件 | 收集性能指标 |

---

## 12. Callback 回调机制

### 12.1 Callback Handler

```go
import "github.com/cloudwego/eino/callbacks"

// 定义 Callback 处理器
type MyCallback struct {
    callbacks.HandlerBuilder
}

func (cb *MyCallback) OnStart(ctx context.Context, info *callbacks.RunInfo, input callbacks.CallbackInput) context.Context {
    fmt.Printf("[开始] %s/%s\n", info.Component, info.Name)
    return ctx
}

func (cb *MyCallback) OnEnd(ctx context.Context, info *callbacks.RunInfo, output callbacks.CallbackOutput) context.Context {
    fmt.Printf("[结束] %s/%s\n", info.Component, info.Name)
    return ctx
}

func (cb *MyCallback) OnError(ctx context.Context, info *callbacks.RunInfo, err error) context.Context {
    fmt.Printf("[错误] %s/%s: %v\n", info.Component, info.Name, err)
    return ctx
}

// 使用
runner := adk.NewRunner(ctx, adk.RunnerConfig{
    EnableStreaming: true,
    Agent:          agent,
    // Runner 的 Callbacks
    Callbacks: callbacks.Handlers(&MyCallback{}),
})
```

### 12.2 Agent Option 中的 Callbacks

```go
// 在调用时传入 Callbacks
opt := []agent.AgentOption{
    agent.WithComposeOptions(compose.WithCallbacks(myCallback)),
}

iter := runner.Query(ctx, query, opt...)
```

### 12.3 回调钩子列表

| 钩子 | 时机 | 用途 |
|------|------|------|
| `OnStart` | 开始执行 | 记录开始日志 |
| `OnEnd` | 执行完成 | 记录结束日志 |
| `OnError` | 执行出错 | 错误告警 |
| `OnStartWithStreamInput` | 流输入开始 | 处理流输入 |
| `OnEndWithStreamOutput` | 流输出结束 | 处理流输出 |

---

## 总结

本文档涵盖了 Eino ADK 的核心功能：

| 主题 | 关键点 |
|------|--------|
| **Runner** | Agent 的运行环境，管理生命周期和流式输出 |
| **Agent 类型** | ChatModelAgent（基础）、Sequential、Parallel、Loop |
| **ChatModelAgent** | 基于 ChatModel + Tools 的 ReAct 实现 |
| **Tool** | 工具是 Agent 与外部交互的桥梁 |
| **人机交互** | Approval + Interrupt 实现需批准的操作 |
| **Supervisor** | 多 Agent 协调器，智能分配任务 |
| **Plan-Execute** | 先规划后执行，支持动态调整 |
| **Checkpoint** | 断点恢复，支持分布式执行 |
| **Middleware** | 扩展 Agent 行为 |
| **Callback** | 监控和日志 |

---

## 进阶阅读

| 示例 | 路径 | 说明 |
|------|------|------|
| HelloWorld | `adk/helloworld/` | 最小可用示例 |
| 摘要 Agent | `adk/intro/agent_with_summarization/` | 中间件使用 |
| 顺序执行 | `adk/intro/workflow/sequential/` | SequentialAgent |
| 并行执行 | `adk/intro/workflow/parallel/` | ParallelAgent |
| 循环反思 | `adk/intro/workflow/loop/` | LoopAgent |
| 批准机制 | `adk/human-in-the-loop/1_approval/` | 人机交互 |
| Supervisor | `adk/human-in-the-loop/5_supervisor/` | 监督者模式 |
| 规划执行 | `adk/human-in-the-loop/6_plan-execute-replan/` | 规划执行模式 |
