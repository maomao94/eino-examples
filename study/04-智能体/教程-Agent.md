# Agent 专题教程

> 本教程详细讲解 Eino 中 Agent 组件的创建和使用，包括 ReAct 模式、ChatModelAgent、自定义 Agent 等。

---

## 目录

1. [概述](#1-概述)
2. [ReAct Agent](#2-react-agent)
3. [ChatModelAgent](#3-chatmodelagent)
4. [Agent 接口](#4-agent-接口)
5. [自定义 Agent](#5-自定义-agent)
6. [Runner](#6-runner)
7. [最佳实践](#7-最佳实践)

---

## 1. 概述

### 什么是 Agent

Agent（智能体）是能够自主决策使用工具的 AI 系统，它能够：
- 理解用户意图
- 决定是否需要使用工具
- 执行工具获取信息
- 基于结果生成回复

```
┌─────────────────────────────────────────────────────────────────┐
│                      Agent 执行模型                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────┐                                                 │
│  │ 用户输入  │                                                 │
│  └─────┬─────┘                                                 │
│        │                                                         │
│        ▼                                                         │
│  ┌───────────┐     ┌───────────┐     ┌───────────┐              │
│  │  思考     │ ──► │  决定    │ ──► │  行动    │              │
│  │(Reasoning)│     │(Action?)  │     │(Tool)    │              │
│  └─────┬─────┘     └─────┬─────┘     └─────┬─────┘              │
│        │                 │                 │                    │
│        │       ┌─────────┴─────────┐       │                    │
│        │       │                   │       │                    │
│        │       ▼                   ▼       ▼                    │
│        │    ┌─────┐              ┌─────────┐                    │
│        │    │ Yes │              │   No    │                    │
│        │    └──┬──┘              └────┬────┘                    │
│        │       │                      │                          │
│        └───────┼──────────────────────┘                          │
│                ▼                                                 │
│          ┌───────────┐                                          │
│          │ 生成回复  │                                          │
│          └───────────┘                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Eino 中的 Agent 类型

| 类型 | 说明 | 源码 |
|------|------|------|
| `react.NewAgent` | ReAct 模式，支持工具调用 | `flow/agent/react` |
| `adk.NewChatModelAgent` | 基于 ChatModel 的 Agent | `adk/chatmodel.go` |
| `adk.NewLoopAgent` | 循环 Agent（反思模式） | `adk/workflow.go` |
| `adk.NewSequentialAgent` | 顺序 Agent | `adk/workflow.go` |
| `adk.NewParallelAgent` | 并行 Agent | `adk/workflow.go` |

---

## 2. ReAct Agent

### 2.1 快速开始

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "io"
    "log"

    "github.com/cloudwego/eino/components/model/openai"
    "github.com/cloudwego/eino/components/tool/utils"
    "github.com/cloudwego/eino/compose"
    "github.com/cloudwego/eino/flow/agent/react"
    "github.com/cloudwego/eino/schema"
)

type SearchRequest struct {
    Query string `json:"query"`
}

func main() {
    ctx := context.Background()

    // 1. 创建模型
    model, err := openai.NewChatModel(ctx, &openai.ChatModelConfig{
        Model:  "gpt-4o",
        APIKey: "your-key",
    })
    if err != nil {
        log.Fatal(err)
    }

    // 2. 创建工具
    searchTool, err := utils.InferTool(
        "search",
        "搜索互联网获取信息",
        func(ctx context.Context, req *SearchRequest) (string, error) {
            // 实际项目中调用搜索 API
            return fmt.Sprintf("[搜索结果] 关于「%s」: 这是搜索返回的信息", req.Query), nil
        },
    )
    if err != nil {
        log.Fatal(err)
    }

    // 3. 创建 ReAct Agent
    agent, err := react.NewAgent(ctx, &react.AgentConfig{
        Model: model,
        ToolsConfig: compose.ToolsNodeConfig{
            Tools: []tool.BaseTool{searchTool},
        },
        MaxStep: 10,  // 最大迭代次数
    })
    if err != nil {
        log.Fatal(err)
    }

    // 4. 执行
    messages := []*schema.Message{
        schema.SystemMessage("你是一个有帮助的助手，可以使用搜索工具。"),
        schema.UserMessage("北京今天的天气怎么样？"),
    }

    stream, err := agent.Stream(ctx, messages)
    if err != nil {
        log.Fatal(err)
    }
    defer stream.Close()

    // 5. 处理流
    for {
        chunk, err := stream.Recv()
        if errors.Is(err, io.EOF) {
            break
        }
        if err != nil {
            log.Fatal(err)
        }
        fmt.Print(chunk.Content)
    }
    fmt.Println()
}
```

### 2.2 ReAct Agent 配置

```go
react.NewAgent(ctx, &react.AgentConfig{
    Model: model,              // 模型

    // 工具配置
    ToolsConfig: compose.ToolsNodeConfig{
        Tools: []tool.BaseTool{tool1, tool2},
        // 工具执行顺序
        // true = 顺序执行，false = 并行执行
        ExecuteSequentially: false,
    },

    // 最大迭代次数（防止死循环）
    MaxStep: 10,

    // 工具返回后直接结束（不继续循环）
    ToolReturnDirectly: map[string]struct{}{
        "exit": {},
    },
})
```

### 2.3 ReAct 执行详解

```
用户: "北京今天天气怎么样？"

┌─────────────────────────────────────────────────────────────────┐
│                    ReAct 执行详解                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Step 1: LLM Generate                                           │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Assistant:                                                 │  │
│  │                                                           │  │
│  │ Thought: 用户问天气，我需要搜索一下                        │  │
│  │ Action: search({"query": "北京天气今天"})                 │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                        │                                         │
│                        ▼                                         │
│  Step 2: Tool Execute                                           │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Tool: search                                               │  │
│  │ Input: {"query": "北京天气今天"}                         │  │
│  │ Output: "北京: 晴, 28°C, 湿度60%"                       │  │
│  └───────────────────────────────────────────────────────────┘  │
│                        │                                         │
│                        ▼                                         │
│  Step 3: LLM Generate (带工具结果)                              │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Assistant:                                                 │  │
│  │                                                           │  │
│  │ 今天北京天气晴朗，气温28摄氏度，湿度约60%。                │  │
│  │ 空气质量良好，适合户外活动。                               │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. ChatModelAgent

### 3.1 快速开始

```go
package main

import (
    "context"
    "log"

    "github.com/cloudwego/eino/adk"
    "github.com/cloudwego/eino/components/model/openai"
    "github.com/cloudwego/eino/compose"
)

func main() {
    ctx := context.Background()

    model, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
        Model:  "gpt-4o-mini",
        APIKey: "your-key",
    })

    // 创建 ChatModelAgent
    agent, err := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
        Name:        "assistant",
        Description: "我的助手",
        Instruction: `你是一个有帮助的助手。
请用简洁的语言回答用户的问题。`,
        Model: model,
    })
    if err != nil {
        log.Fatal(err)
    }

    // 创建 Runner
    runner := adk.NewRunner(ctx, adk.RunnerConfig{
        EnableStreaming: true,
        Agent:          agent,
    })

    // 执行
    iter := runner.Query(ctx, "你好，请介绍一下自己")

    for event := range iter.Iter() {
        if event.Output != nil {
            if msg, _, _ := adk.GetMessage(event); msg != nil {
                println(msg.Content)
            }
        }
    }
}
```

### 3.2 ChatModelAgent 配置

```go
adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    // 基本信息
    Name:        "my_agent",
    Description: "我的助手",

    // 系统提示词
    Instruction: `你是一个有帮助的助手...`,

    // 模型
    Model: model,

    // 工具配置
    ToolsConfig: adk.ToolsConfig{
        ToolsNodeConfig: compose.ToolsNodeConfig{
            Tools: []tool.BaseTool{tool1, tool2},
        },
        // 工具直接返回
        ReturnDirectly: map[string]bool{
            "exit": true,
        },
    },

    // 输入转换
    GenModelInput: func(ctx context.Context, state *ChatModelAgentState) ([]*schema.Message, error) {
        // 自定义输入生成
        return state.Messages, nil
    },

    // 退出工具
    Exit: exitTool,

    // 最大迭代次数
    MaxIterations: 20,

    // 中间件
    Handlers:    []adk.ChatModelAgentMiddleware{},
    Middlewares: []adk.Middleware{},

    // 模型重试配置
    ModelRetryConfig: &adk.ModelRetryConfig{
        MaxRetries: 3,
        RetryDelay: time.Second,
    },
})
```

---

## 4. Agent 接口

### 4.1 Agent 接口定义

```go
// Agent 接口
type Agent interface {
    // Agent 名称
    Name(ctx context.Context) string

    // Agent 描述
    Description(ctx context.Context) string

    // 执行入口
    Run(ctx context.Context, input *AgentInput,
        options ...AgentRunOption) *AsyncIterator[*AgentEvent]
}

// 可恢复的 Agent
type ResumableAgent interface {
    Agent
    // 从中断点恢复
    Resume(ctx context.Context, info *ResumeInfo,
           options ...AgentRunOption) *AsyncIterator[*AgentEvent]
}
```

### 4.2 AgentInput

```go
type AgentInput struct {
    Query        string                      // 用户输入
    Conversation []*schema.Message          // 对话历史
    State        map[string]any              // 自定义状态
}
```

### 4.3 AgentEvent

```go
type AgentEvent struct {
    AgentName string         // Agent 名称
    RunPath   []RunStep      // 执行路径
    Output    *AgentOutput   // 输出内容
    Action    *AgentAction  // 特殊动作
    Err       error         // 错误
}

// Agent 动作
type AgentAction struct {
    Exit            bool                  // 自然结束
    Interrupted     *InterruptInfo       // 被中断
    TransferToAgent *TransferToAgentAction // 转移到其他 Agent
    BreakLoop       *BreakLoopAction     // 跳出循环
    CustomizedAction any                  // 自定义动作
}
```

---

## 5. 自定义 Agent

### 5.1 实现基本 Agent

```go
package main

import (
    "context"
    "fmt"
    "sync"

    "github.com/cloudwego/eino/adk"
    "github.com/cloudwego/eino/schema"
)

// ============ 自定义 Agent ============
type MyAgent struct {
    name        string
    description string
    model       model.ChatModel
    tools       []tool.BaseTool
}

func NewMyAgent(model model.ChatModel, tools []tool.BaseTool) *MyAgent {
    return &MyAgent{
        name:        "my_agent",
        description: "我的自定义 Agent",
        model:       model,
        tools:       tools,
    }
}

func (a *MyAgent) Name(ctx context.Context) string {
    return a.name
}

func (a *MyAgent) Description(ctx context.Context) string {
    return a.description
}

func (a *MyAgent) Run(ctx context.Context, input *adk.AgentInput,
    options ...adk.AgentRunOption) *adk.AsyncIterator[*adk.AgentEvent] {

    iter := adk.NewAsyncIterator[*adk.AgentEvent]()

    go func() {
        defer iter.Close()

        // 1. 构建消息
        messages := input.Conversation
        messages = append(messages, schema.UserMessage(input.Query))

        // 2. 调用模型
        resp, err := a.model.Generate(ctx, messages)
        if err != nil {
            iter.Send(&adk.AgentEvent{Err: err})
            return
        }

        // 3. 发送输出事件
        iter.Send(&adk.AgentEvent{
            Output: &adk.AgentOutput{
                Message: resp,
            },
        })

        // 4. 发送结束事件
        iter.Send(&adk.AgentEvent{
            Action: &adk.AgentAction{Exit: true},
        })
    }()

    return iter
}

// ============ 使用 ============
func main() {
    ctx := context.Background()

    agent := NewMyAgent(model, tools)
    events := agent.Run(ctx, &adk.AgentInput{
        Query: "你好",
    })

    for event := range events.Iter() {
        if event.Err != nil {
            fmt.Printf("错误: %v\n", event.Err)
        }
        if event.Output != nil {
            fmt.Println(event.Output.Message.Content)
        }
        if event.Action != nil && event.Action.Exit {
            fmt.Println("Agent 结束")
        }
    }
}
```

### 5.2 带状态的 Agent

```go
type StatefulAgent struct {
    *MyAgent
    mu    sync.RWMutex
    state map[string]any
}

func (a *StatefulAgent) Run(ctx context.Context, input *adk.AgentInput,
    options ...adk.AgentRunOption) *adk.AsyncIterator[*adk.AgentEvent] {

    // 合并输入状态
    a.mu.Lock()
    for k, v := range input.State {
        a.state[k] = v
    }
    a.mu.Unlock()

    return a.MyAgent.Run(ctx, input, options...)
}
```

---

## 6. Runner

### 6.1 Runner 简介

Runner 是 Agent 的执行容器，提供统一的执行接口。

```go
// 创建 Runner
runner := adk.NewRunner(ctx, adk.RunnerConfig{
    EnableStreaming: true,
    Agent:          agent,
    // 可添加回调
    //Callbacks: callbacks,
})

// Query 执行
iter := runner.Query(ctx, "用户输入")

// Invoke 执行
result := runner.Invoke(ctx, &adk.AgentInput{...})
```

### 6.2 Runner 配置

```go
adk.NewRunner(ctx, adk.RunnerConfig{
    EnableStreaming: true,   // 启用流式
    Agent:          agent,  // Agent 实例
    StreamInterval: time.Millisecond * 50,  // 流式间隔
})
```

### 6.3 完整示例

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/cloudwego/eino/adk"
    "github.com/cloudwego/eino/components/model/openai"
)

func main() {
    ctx := context.Background()

    model, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
        Model:  "gpt-4o-mini",
        APIKey: "your-key",
    })

    agent, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
        Name:        "assistant",
        Instruction: "你是一个有帮助的助手。",
        Model:       model,
    })

    // 创建 Runner
    runner := adk.NewRunner(ctx, adk.RunnerConfig{
        EnableStreaming: true,
        Agent:          agent,
    })

    // 查询
    iter := runner.Query(ctx, "你好，请介绍一下自己")

    // 处理事件
    for {
        event, ok := iter.Next()
        if !ok {
            break
        }

        if event.Err != nil {
            fmt.Printf("错误: %v\n", event.Err)
            continue
        }

        // 获取消息
        if event.Output != nil {
            msg, _, _ := adk.GetMessage(event)
            if msg != nil {
                fmt.Print(msg.Content)
            }
        }

        // 检查动作
        if event.Action != nil {
            switch {
            case event.Action.Exit:
                fmt.Println("\n[对话结束]")
            case event.Action.Interrupted != nil:
                fmt.Println("\n[等待确认...]")
            }
        }
    }
}
```

---

## 7. 最佳实践

### 7.1 工具选择

```go
// 只提供必要的工具
Tools: []tool.BaseTool{
    searchTool,   // 搜索
    calculatorTool, // 计算
    // 不要提供太多工具，AI 会困惑
}

// 工具描述要清晰
utils.InferTool(
    "get_weather",
    "获取指定城市的当前天气，包括温度、湿度、风力",
    ...
)
```

### 7.2 迭代限制

```go
// 始终设置最大迭代次数，防止死循环
react.NewAgent(ctx, &react.AgentConfig{
    MaxStep: 10,  // 10 次足够了
})
```

### 7.3 错误处理

```go
events := agent.Run(ctx, input)

for event := range events.Iter() {
    if event.Err != nil {
        // 分类处理
        switch {
        case errors.Is(event.Err, context.Canceled):
            fmt.Println("用户取消")
        case errors.Is(event.Err, context.DeadlineExceeded):
            fmt.Println("超时")
        default:
            fmt.Printf("错误: %v\n", event.Err)
        }
    }
}
```

### 7.4 流式处理

```go
// 使用 goroutine 并行处理
func chatStream(ctx context.Context, agent *adk.Agent, query string) {
    iter := agent.Run(ctx, &adk.AgentInput{Query: query})

    go func() {
        for event := range iter.Iter() {
            if event.Output != nil {
                // 处理输出
                print(event.Output.Message.Content)
            }
        }
    }()
}
```

---

## 下一步

- [../01-入门/手把手教程.md](../01-入门/手把手教程.md) - 手把手入门
- [../02-组件/教程-工具.md](../02-组件/教程-工具.md) - 工具创建
- [../02-组件/教程-内存.md](../02-组件/教程-内存.md) - 记忆管理
- [../05-进阶/示例代码.md](../05-进阶/示例代码.md) - 更多示例

---

*本教程基于 eino v0.8.5 编写。*
