# Multi-Agent 多 Agent 系统进阶指南

> **前置知识**：[教程-ADK.md](../04-智能体/教程-ADK.md)
>
> **学习目标**：掌握多 Agent 协作模式，设计复杂 Agent 系统
>
> **相关章节**：[教程-基础Agent.md](../04-智能体/教程-基础Agent.md) | [进阶-人机交互.md](../05-进阶/进阶-人机交互.md)

---

## 目录

1. [概述](#1-概述)
2. [协作模式](#2-协作模式)
3. [Supervisor 模式](#3-supervisor-模式)
4. [Plan-Execute 模式](#4-plan-execute-模式)
5. [Deep Agent 深度 Agent](#5-deep-agent-深度-agent)
6. [Deer-Go 模式](#6-deer-go-模式)
7. [实际应用案例](#7-实际应用案例)
8. [架构设计最佳实践](#8-架构设计最佳实践)
9. [常见问题](#9-常见问题)

---

## 1. 概述

### 1.1 什么是 Multi-Agent System

Multi-Agent System（多 Agent 系统）是由多个智能 Agent 协同工作的系统，每个 Agent 负责特定任务：

```
┌─────────────────────────────────────────────────────────────────┐
│                 Multi-Agent System 架构                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    用户请求                                │   │
│  └────────────────────────┬─────────────────────────────────┘   │
│                           │                                       │
│                           ▼                                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Orchestrator (编排器)                         │   │
│  │              - 任务分解                                     │   │
│  │              - Agent 协调                                  │   │
│  │              - 结果聚合                                     │   │
│  └────────────────────────┬─────────────────────────────────┘   │
│                           │                                       │
│     ┌──────────────────────┼──────────────────────┐              │
│     │                      │                      │              │
│     ▼                      ▼                      ▼              │
│  ┌────────┐            ┌────────┐            ┌────────┐        │
│  │ Agent  │            │ Agent  │            │ Agent  │        │
│  │   A    │            │   B    │            │   C    │        │
│  │ (搜索) │            │ (分析) │            │ (执行) │        │
│  └────────┘            └────────┘            └────────┘        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Multi-Agent vs 单 Agent

| 特性 | 单 Agent | Multi-Agent |
|------|---------|-------------|
| 能力范围 | 单一职责 | 多职责分工 |
| 复杂度 | 低 | 中-高 |
| 可扩展性 | 受限于单一模型 | 易于扩展 |
| 协作能力 | 无 | 强 |
| 调试难度 | 低 | 中-高 |
| 适用场景 | 简单任务 | 复杂任务 |

### 1.3 适用场景

| 场景 | 说明 | 典型应用 |
|------|------|----------|
| 复杂问答 | 多角度分析 | Research Assistant |
| 代码开发 | 多人协作 | Code Assistant |
| 数据分析 | 多步骤处理 | Data Analyst |
| 对话系统 | 专业分工 | 客服系统 |
| 自动化流程 | 任务流水线 | 自动化办公 |

---

## 2. 协作模式

### 2.1 模式概览

```
┌─────────────────────────────────────────────────────────────────┐
│                    Agent 协作模式对比                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Sequential（顺序）                                               │
│  A ──▶ B ──▶ C ──▶ Result                                       │
│  线性传递，下游依赖上游结果                                       │
│                                                                  │
│  Parallel（并行）                                                │
│     ┌──▶ A ──┐                                                   │
│  In ├──▶ B ──┼──▶ Result                                        │
│     └──▶ C ──┘                                                   │
│  并行独立执行，结果合并                                           │
│                                                                  │
│  Loop（循环）                                                    │
│  A ──▶ B ──▶ C ──┐                                               │
│                  │                                               │
│                  ▼                                               │
│               [检查] ──▶ 继续/结束                                 │
│  迭代优化，直到满足条件                                           │
│                                                                  │
│  Hierarchical（层级）                                            │
│  ┌─────────────────────────────────────┐                       │
│  │ Supervisor                          │                       │
│  │   ├── A                             │                       │
│  │   ├── B                             │                       │
│  │   └── C                             │                       │
│  └─────────────────────────────────────┘                       │
│  星型结构，Supervisor 协调                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Sequential Agent（顺序执行）

**功能说明**：Agent 按顺序执行，每个 Agent 的输出作为下一个的输入

```go
package main

import (
	"context"
	"fmt"
	"strings"

	"github.com/cloudwego/eino/adk/agent"
	"github.com/cloudwego/eino/adk/agent/sequential"
	"github.com/cloudwego/eino/chatbot"
	"github.com/cloudwego/eino/chatbot/openai"
)

// Sequential Agent 示例
// 参考：Eino 示例项目 adk/intro/workflow/sequential/

func main() {
	ctx := context.Background()

	// 1. 创建各个阶段的 Agent
	searchAgent := createSearchAgent(ctx)
	analyzeAgent := createAnalyzeAgent(ctx)
	writeAgent := createWriteAgent(ctx)

	// 2. 创建顺序执行 Agent
	seqAgent := sequential.NewSequentialAgent(
		ctx,
		&sequential.Config{
			Agents: []agent.Agent{searchAgent, analyzeAgent, writeAgent},
			Name:   "research_writer",
			// OutputKey 用于在 Agent 间传递数据
			OutputKeys: []string{"search_result", "analysis", "final_report"},
		},
	)

	// 3. 执行
	result, err := seqAgent.Run(ctx, "研究 AI Agent 的最新发展")
	if err != nil {
		fmt.Printf("执行失败: %v\n", err)
		return
	}

	fmt.Printf("最终报告:\n%s\n", result)
}

// 创建搜索 Agent
func createSearchAgent(ctx context.Context) agent.Agent {
	model, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey: "your-api-key",
		Model:  "gpt-4o-mini",
	})

	prompt := `你是一个研究助手，负责搜索相关信息。
根据用户查询，搜索相关内容并返回摘要。
输出格式：{"topics": [...], "summary": "..."}`

	a, _ := agent.NewAgent(ctx, &agent.AgentConfig{
		Model:  model,
		Prompt: prompt,
	})
	return a
}

// 创建分析 Agent
func createAnalyzeAgent(ctx context.Context) agent.Agent {
	model, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey: "your-api-key",
		Model:  "gpt-4o-mini",
	})

	prompt := `你是一个分析师，负责分析搜索结果。
输入：搜索结果摘要
输出：深入分析报告

分析要点：
1. 主要发现
2. 趋势分析
3. 潜在影响`

	a, _ := agent.NewAgent(ctx, &agent.AgentConfig{
		Model:  model,
		Prompt: prompt,
	})
	return a
}

// 创建写作 Agent
func createWriteAgent(ctx context.Context) agent.Agent {
	model, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey: "your-api-key",
		Model:  "gpt-4o-mini",
	})

	prompt := `你是一个专业作家，负责将分析结果写成报告。
输入：分析报告
输出：结构化的最终报告

格式要求：
1. 标题
2. 摘要
3. 详细内容
4. 结论`

	a, _ := agent.NewAgent(ctx, &agent.AgentConfig{
		Model:  model,
		Prompt: prompt,
	})
	return a
}
```

### 2.3 Parallel Agent（并行执行）

**功能说明**：多个 Agent 同时执行，最后合并结果

```go
package main

import (
	"context"
	"fmt"
	"sync"

	"github.com/cloudwego/eino/adk/agent"
	"github.com/cloudwego/eino/adk/agent/parallel"
	"github.com/cloudwego/eino/chatbot"
	"github.com/cloudwego/eino/chatbot/openai"
)

// Parallel Agent 示例
// 参考：Eino 示例项目 adk/intro/workflow/parallel/

func main() {
	ctx := context.Background()

	// 1. 创建多个并行 Agent
	newsAgent := createTopicAgent(ctx, "最新新闻")
	techAgent := createTopicAgent(ctx, "技术发展")
	marketAgent := createTopicAgent(ctx, "市场动态")
	policyAgent := createTopicAgent(ctx, "政策法规")

	// 2. 创建并行执行 Agent
	parallelAgent := parallel.NewParallelAgent(
		ctx,
		&parallel.Config{
			Agents: []agent.Agent{
				newsAgent,
				techAgent,
				marketAgent,
				policyAgent,
			},
			Name:        "comprehensive_researcher",
			ResultMerge: mergeResults, // 结果合并函数
		},
	)

	// 3. 执行
	result, err := parallelAgent.Run(ctx, "AI 行业最新动态")
	if err != nil {
		fmt.Printf("执行失败: %v\n", err)
		return
	}

	fmt.Printf("综合报告:\n%s\n", result)
}

// 创建主题研究 Agent
func createTopicAgent(ctx context.Context, topic string) agent.Agent {
	model, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey: "your-api-key",
		Model:  "gpt-4o-mini",
	})

	prompt := fmt.Sprintf(`你是一个 %s 研究专家。
请分析并提供关于 "%s" 的详细信息。
输出格式：
## %s
### 主要发现
...
### 详细说明
...`, topic, topic, topic)

	a, _ := agent.NewAgent(ctx, &agent.AgentConfig{
		Model:  model,
		Prompt: prompt,
	})
	return a
}

// 合并结果
func mergeResults(results map[string]string) string {
	var sb strings.Builder
	sb.WriteString("# 综合研究报告\n\n")

	// 按照固定顺序合并
	order := []string{"最新新闻", "技术发展", "市场动态", "政策法规"}
	for _, topic := range order {
		if result, ok := results[topic]; ok {
			sb.WriteString(result)
			sb.WriteString("\n\n")
		}
	}

	return sb.String()
}
```

### 2.4 Loop Agent（循环反思）

**功能说明**：Agent 循环执行，每次反思改进结果

```go
package main

import (
	"context"
	"fmt"

	"github.com/cloudwego/eino/adk/agent"
	"github.com/cloudwego/eino/adk/agent/loop"
	"github.com/cloudwego/eino/chatbot"
	"github.com/cloudwego/eino/chatbot/openai"
)

// Loop Agent 示例 - 反思改进模式
// 参考：Eino 示例项目 adk/intro/workflow/loop/

func main() {
	ctx := context.Background()

	// 1. 创建主 Agent
	mainAgent := createMainAgent(ctx)

	// 2. 创建反思 Agent
	reflectAgent := createReflectAgent(ctx)

	// 3. 创建循环执行 Agent
	loopAgent := loop.NewLoopAgent(
		ctx,
		&loop.Config{
			Agent:     mainAgent,
			Reflecter: reflectAgent,
			MaxIterations: 5,
			StopCondition: func(ctx context.Context, iteration int, output string) bool {
				// 如果输出包含 "完成" 字样，停止循环
				return contains(output, "完成") || iteration >= 5
			},
		},
	)

	// 4. 执行
	result, err := loopAgent.Run(ctx, "写一篇关于 AI 的短文")
	if err != nil {
		fmt.Printf("执行失败: %v\n", err)
		return
	}

	fmt.Printf("最终输出:\n%s\n", result)
}

// 创建主 Agent
func createMainAgent(ctx context.Context) agent.Agent {
	model, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey: "your-api-key",
		Model:  "gpt-4o-mini",
	})

	prompt := `你是一个作家，负责撰写文章。
基于反馈改进你的文章。
直接输出改进后的文章内容。`

	a, _ := agent.NewAgent(ctx, &agent.AgentConfig{
		Model:  model,
		Prompt: prompt,
	})
	return a
}

// 创建反思 Agent
func createReflectAgent(ctx context.Context) agent.Agent {
	model, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey: "your-api-key",
		Model:  "gpt-4o-mini",
	})

	prompt := `你是一个评审专家，负责审查文章并提供反馈。
审查标准：
1. 内容准确性
2. 表达清晰度
3. 逻辑连贯性
4. 完整性

如果文章已经足够好，回复"完成"。
否则，回复改进建议。`

	a, _ := agent.NewAgent(ctx, &agent.AgentConfig{
		Model:  model,
		Prompt: prompt,
	})
	return a
}

func contains(s, substr string) bool {
	return len(s) >= len(substr) && (s == substr ||
		(len(s) > len(substr) && containsHelper(s, substr)))
}

func containsHelper(s, substr string) bool {
	for i := 0; i <= len(s)-len(substr); i++ {
		if s[i:i+len(substr)] == substr {
			return true
		}
	}
	return false
}
```

---

## 3. Supervisor 模式

### 3.1 Supervisor 原理

Supervisor 是一个协调者，负责将任务分配给合适的子 Agent：

```
┌─────────────────────────────────────────────────────────────────┐
│                     Supervisor 模式                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    ┌─────────────────┐                          │
│                    │   Supervisor    │                          │
│                    │   (协调者)       │                          │
│                    └────────┬────────┘                          │
│                             │                                    │
│       ┌─────────────────────┼─────────────────────┐              │
│       │                     │                     │              │
│       ▼                     ▼                     ▼              │
│  ┌─────────┐           ┌─────────┐           ┌─────────┐      │
│  │ Agent A │           │ Agent B │           │ Agent C │      │
│  │ (搜索)  │           │ (分析)  │           │ (写作)  │      │
│  └────┬────┘           └────┬────┘           └────┬────┘      │
│       │                     │                     │              │
│       └─────────────────────┼─────────────────────┘              │
│                             │                                    │
│                             ▼                                    │
│                    ┌─────────────────┐                          │
│                    │   结果聚合       │                          │
│                    └─────────────────┘                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Supervisor 实现

```go
package main

import (
	"context"
	"fmt"

	"github.com/cloudwego/eino/adk/agent"
	"github.com/cloudwego/eino/adk/supervisor"
	"github.com/cloudwego/eino/chatbot"
	"github.com/cloudwego/eino/chatbot/openai"
)

// Supervisor 示例
// 参考：Eino 示例项目 adk/human-in-the-loop/5_supervisor/

func main() {
	ctx := context.Background()

	// 1. 创建子 Agent
	searchAgent := createSubAgent(ctx, "search", "你是一个搜索专家，负责搜索相关信息。")
	mathAgent := createSubAgent(ctx, "math", "你是一个数学专家，负责计算和分析数据。")
	codeAgent := createSubAgent(ctx, "code", "你是一个编程专家，负责编写和调试代码。")
	writeAgent := createSubAgent(ctx, "write", "你是一个作家，负责撰写报告。")

	// 2. 创建 Supervisor
	sv, err := supervisor.New(
		ctx,
		&supervisor.Config{
			Name: "main_supervisor",
			Model: func(ctx context.Context) (chatbot.ChatModel, error) {
				return openai.NewChatModel(ctx, &openai.ChatModelConfig{
					APIKey: "your-api-key",
					Model:  "gpt-4o",
				})
			},
			SubAgents: []supervisor.SubAgent{
				{Name: "search", Agent: searchAgent, Description: "搜索相关信息"},
				{Name: "math", Agent: mathAgent, Description: "数学计算和分析"},
				{Name: "code", Agent: codeAgent, Description: "编程和调试"},
				{Name: "write", Agent: writeAgent, Description: "撰写报告"},
			},
			// Supervisor 的系统提示
			SystemPrompt: `你是一个智能助手协调者，负责将用户请求分配给合适的专家。

可用专家：
- search: 搜索相关信息
- math: 数学计算和分析
- code: 编程和调试
- write: 撰写报告

根据用户请求，选择合适的专家执行任务。
可以通过调用多个专家来协作完成复杂任务。`,
		},
	)
	if err != nil {
		panic(err)
	}

	// 3. 运行
	result, err := sv.Run(ctx, "请帮我计算 1 到 100 的和，并写一份报告")
	if err != nil {
		fmt.Printf("执行失败: %v\n", err)
		return
	}

	fmt.Printf("Supervisor 结果:\n%s\n", result)
}

// 创建子 Agent
func createSubAgent(ctx context.Context, name, prompt string) agent.Agent {
	model, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey: "your-api-key",
		Model:  "gpt-4o-mini",
	})

	a, _ := agent.NewAgent(ctx, &agent.AgentConfig{
		Model:  model,
		Prompt: prompt,
	})
	return a
}
```

### 3.3 任务路由策略

```go
package main

import (
	"context"
	"strings"

	"github.com/cloudwego/eino/adk/supervisor"
)

// 自定义路由策略

type RoutingStrategy struct {
	keywordRules map[string][]string // 关键词 -> Agent 映射
}

func NewRoutingStrategy() *RoutingStrategy {
	return &RoutingStrategy{
		keywordRules: map[string][]string{
			"search":  {"搜索", "查找", "查询", "search"},
			"math":   {"计算", "数学", "分析", "calculate"},
			"code":   {"代码", "编程", "程序", "code", "bug"},
			"write":  {"写", "报告", "文章", "write", "report"},
		},
	}
}

func (s *RoutingStrategy) Route(query string) []string {
	var matchedAgents []string
	query = strings.ToLower(query)

	for agentName, keywords := range s.keywordRules {
		for _, keyword := range keywords {
			if strings.Contains(query, keyword) {
				matchedAgents = append(matchedAgents, agentName)
				break
			}
		}
	}

	return matchedAgents
}

// 使用自定义路由
func customRoute(s *RoutingStrategy, query string) supervisor.SubAgent {
	agents := s.Route(query)
	if len(agents) == 0 {
		return supervisor.SubAgent{Name: "default"}
	}
	return supervisor.SubAgent{Name: agents[0]}
}
```

---

## 4. Plan-Execute 模式

### 4.1 Plan-Execute 原理

Plan-Execute 是一种"规划-执行-反思"循环模式：

```
┌─────────────────────────────────────────────────────────────────┐
│                   Plan-Execute 模式                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────┐                                                      │
│  │ Planner │  制定计划                                            │
│  └────┬────┘                                                      │
│       │                                                           │
│       ▼                                                           │
│  ┌─────────┐                                                      │
│  │ Executor│  执行计划                                            │
│  └────┬────┘                                                      │
│       │                                                           │
│       ▼                                                           │
│  ┌─────────┐                                                      │
│  │ Checker │  检查结果                                            │
│  └────┬────┘                                                      │
│       │                                                           │
│       ├── 满足 ──▶ 输出结果                                       │
│       │                                                           │
│       └── 不满足 ──▶ 重新规划                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Plan-Execute 实现

```go
package main

import (
	"context"
	"fmt"
	"strings"

	"github.com/cloudwego/eino/adk/agent"
	"github.com/cloudwego/eino/adk/plan"
	"github.com/cloudwego/eino/adk/runner"
	"github.com/cloudwego/eino/chatbot"
	"github.com/cloudwego/eino/chatbot/openai"
)

// Plan-Execute Agent 示例

func main() {
	ctx := context.Background()

	// 1. 创建 Planner（规划器）
	planner := createPlanner(ctx)

	// 2. 创建 Executor（执行器）
	executor := createExecutor(ctx)

	// 3. 创建 Checker（检查器）
	checker := createChecker(ctx)

	// 4. 创建 Plan-Execute Agent
	peAgent := plan.NewPlanExecuteAgent(
		ctx,
		&plan.Config{
			Planner:     planner,
			Executor:    executor,
			Checker:     checker,
			MaxAttempts: 3,
		},
	)

	// 5. 运行
	result, err := peAgent.Run(ctx, "帮我分析一下这家公司的财务状况")
	if err != nil {
		fmt.Printf("执行失败: %v\n", err)
		return
	}

	fmt.Printf("分析结果:\n%s\n", result)
}

// 创建 Planner
func createPlanner(ctx context.Context) agent.Agent {
	model, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey: "your-api-key",
		Model:  "gpt-4o",
	})

	prompt := `你是一个规划专家，负责将复杂任务分解为可执行的步骤。

根据用户请求，制定执行计划。
输出格式：
步骤1: [具体任务]
步骤2: [具体任务]
...

每个步骤应该是原子化的，可以独立执行。`

	a, _ := agent.NewAgent(ctx, &agent.AgentConfig{
		Model:  model,
		Prompt: prompt,
	})
	return a
}

// 创建 Executor
func createExecutor(ctx context.Context) agent.Agent {
	model, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey: "your-api-key",
		Model:  "gpt-4o-mini",
	})

	prompt := `你是一个执行专家，负责执行具体的分析任务。

根据给定的步骤，执行相应的任务。
返回执行结果和状态（成功/失败）。`

	a, _ := agent.NewAgent(ctx, &agent.AgentConfig{
		Model:  model,
		Prompt: prompt,
	})
	return a
}

// 创建 Checker
func createChecker(ctx context.Context) agent.Agent {
	model, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey: "your-api-key",
		Model:  "gpt-4o-mini",
	})

	prompt := `你是一个质量检查专家，负责验证执行结果是否满足要求。

检查以下方面：
1. 完整性 - 是否完成了所有步骤
2. 准确性 - 结果是否正确
3. 质量 - 结果是否有价值

如果结果满足要求，回复"完成"。
如果需要改进，回复需要改进的具体内容。`

	a, _ := agent.NewAgent(ctx, &agent.AgentConfig{
		Model:  model,
		Prompt: prompt,
	})
	return a
}
```

### 4.3 带 Replan 的 Plan-Execute

```go
package main

import (
	"context"
	"fmt"

	"github.com/cloudwego/eino/adk/agent"
	"github.com/cloudwego/eino/adk/plan"
)

// 带 Replan 的 Plan-Execute

type ReplanningExecutor struct {
	agent agent.Agent
}

func (e *ReplanningExecutor) Execute(ctx context.Context, plan *plan.Plan) (*plan.ExecutionResult, error) {
	result := &plan.ExecutionResult{
		CompletedSteps: []*plan.Step{},
		RemainingSteps: plan.Steps,
	}

	for i, step := range plan.Steps {
		// 执行当前步骤
		output, err := e.agent.Run(ctx, step.Task)
		if err != nil {
			// 步骤失败，标记为需要重新规划
			result.CompletedSteps = plan.Steps[:i]
			result.RemainingSteps = plan.Steps[i:]
			result.NeedsReplan = true
			result.ReplanReason = fmt.Sprintf("步骤 %d 失败: %v", i+1, err)
			return result, nil
		}

		result.CompletedSteps = append(result.CompletedSteps, &plan.Step{
			Task:  step.Task,
			Result: output,
		})
	}

	return result, nil
}

// 自定义 Plan-Execute 配置
func main() {
	ctx := context.Background()

	executor := &ReplanningExecutor{
		agent: createExecutor(ctx),
	}

	peAgent := plan.NewPlanExecuteAgent(
		ctx,
		&plan.Config{
			Planner:     createPlanner(ctx),
			Executor:    executor,
			Checker:     createChecker(ctx),
			MaxAttempts: 5,
			// 启用自动重新规划
			AutoReplan: true,
		},
	)

	result, _ := peAgent.Run(ctx, "分析市场趋势")
	fmt.Println(result)
}
```

---

## 5. Deep Agent 深度 Agent

### 5.1 Deep Agent 与 Plan-Execute 的区别

| 特性 | Plan-Execute | Deep Agent |
|------|-------------|------------|
| 问题分解 | 单层分解 | 多层递归分解 |
| 子问题处理 | 一次性完成 | 可能需要进一步分解 |
| 适用场景 | 线性任务 | 树状/层次任务 |
| 实现复杂度 | 中等 | 较高 |

### 5.2 Deep Agent 实现

```go
package main

import (
	"context"
	"fmt"
	"strings"

	"github.com/cloudwego/eino/adk/agent"
	"github.com/cloudwego/eino/adk/deep"
	"github.com/cloudwego/eino/chatbot/openai"
)

// Deep Agent 示例 - 深度问题分析

func main() {
	ctx := context.Background()

	// 1. 创建深度分析 Agent
	depthAgent := createDepthAgent(ctx)

	// 2. 创建浅层执行 Agent
	surfaceAgent := createSurfaceAgent(ctx)

	// 3. 创建 Deep Agent
	dAgent := deep.NewDeepAgent(
		ctx,
		&deep.Config{
			DepthAnalyzer: depthAgent,
			SurfaceExecutor: surfaceAgent,
			MaxDepth:     5,
			MinTaskSize:  "simple", // 任务小到此程度不再分解
		},
	)

	// 4. 执行复杂任务
	result, err := dAgent.Run(ctx, "分析全球经济形势")
	if err != nil {
		fmt.Printf("执行失败: %v\n", err)
		return
	}

	fmt.Printf("深度分析结果:\n%s\n", result)
}

// 创建深度分析 Agent
func createDepthAgent(ctx context.Context) agent.Agent {
	model, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey: "your-api-key",
		Model:  "gpt-4o",
	})

	prompt := `你是一个深度分析专家，负责将复杂问题分解为子问题。

分析问题结构：
1. 识别核心问题
2. 分解为多个子问题
3. 确定子问题之间的关系

如果问题足够简单，可以直接回答，回复"[直接回答] ..."
否则，回复：
[分解]
- 子问题1
- 子问题2
- ...`

	a, _ := agent.NewAgent(ctx, &agent.AgentConfig{
		Model:  model,
		Prompt: prompt,
	})
	return a
}

// 创建浅层执行 Agent
func createSurfaceAgent(ctx context.Context) agent.Agent {
	model, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey: "your-api-key",
		Model:  "gpt-4o-mini",
	})

	prompt := `你是一个执行专家，负责回答具体的子问题。

直接回答问题，不需要进一步分解。`

	a, _ := agent.NewAgent(ctx, &agent.AgentConfig{
		Model:  model,
		Prompt: prompt,
	})
	return a
}

// 深度分析处理
func handleDepthAnalysis(ctx context.Context, analysis string) []string {
	if strings.HasPrefix(analysis, "[直接回答]") {
		return nil // 不需要进一步分解
	}

	if strings.HasPrefix(analysis, "[分解]") {
		lines := strings.Split(analysis, "\n")[1:]
		var subProblems []string
		for _, line := range lines {
			if strings.HasPrefix(line, "-") {
				subProblems = append(subProblems, strings.TrimSpace(line[1:]))
			}
		}
		return subProblems
	}

	return nil
}
```

---

## 6. Deer-Go 模式

### 6.1 Deer-Go 简介

Deer-Go 是一种专门为复杂推理任务设计的多 Agent 模式：

```
┌─────────────────────────────────────────────────────────────────┐
│                     Deer-Go 模式                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐                                               │
│  │   Deer      │  导航者                                        │
│  │ (导航协调)  │  - 理解任务                                    │
│  └──────┬──────┘  - 规划路径                                    │
│         │        - 协调执行                                     │
│         │                                                        │
│         ├────────────────────────────────┐                     │
│         │                                │                      │
│         ▼                                ▼                      │
│  ┌─────────────┐                  ┌─────────────┐              │
│  │  Router     │                  │  Executor   │              │
│  │  (路由器)    │                  │  (执行器)    │              │
│  └──────┬──────┘                  └─────────────┘              │
│         │                                                        │
│         ▼                                                        │
│  ┌─────────────┐                                                │
│  │  Worker     │  工作者                                        │
│  │  (专业Agent)│  - 搜索Agent                                   │
│  │             │  - 分析Agent                                   │
│  │             │  - 写作Agent                                   │
│  └─────────────┘                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 Deer-Go 实现

```go
package main

import (
	"context"
	"fmt"
	"strings"

	"github.com/cloudwego/eino/adk/agent"
	"github.com/cloudwego/eino/adk/deer"
	"github.com/cloudwego/eino/chatbot/openai"
)

// Deer-Go 模式实现

func main() {
	ctx := context.Background()

	// 1. 创建 Deer（导航者）
	deer := createDeer(ctx)

	// 2. 创建 Router（路由器）
	router := createRouter(ctx)

	// 3. 创建 Workers（专业 Agent）
	workers := map[string]agent.Agent{
		"search": createSearchWorker(ctx),
		"analyze": createAnalyzeWorker(ctx),
		"write":   createWriteWorker(ctx),
		"code":    createCodeWorker(ctx),
	}

	// 4. 创建 Deer-Go 系统
	dg := deer.NewDeerGo(
		ctx,
		&deer.Config{
			Deer:    deer,
			Router:  router,
			Workers: workers,
			MaxSteps: 20,
		},
	)

	// 5. 执行复杂任务
	result, err := dg.Run(ctx, "研究并分析 AI 在医疗领域的应用")
	if err != nil {
		fmt.Printf("执行失败: %v\n", err)
		return
	}

	fmt.Printf("Deer-Go 结果:\n%s\n", result)
}

// 创建 Deer（导航者）
func createDeer(ctx context.Context) agent.Agent {
	model, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey: "your-api-key",
		Model:  "gpt-4o",
	})

	prompt := `你是一个智能导航专家，负责理解复杂任务并规划执行路径。

分析任务并确定：
1. 需要哪些专家参与
2. 任务的执行顺序
3. 如何协调各专家的工作

输出格式：
计划：
1. [专家类型]: [具体任务]
2. [专家类型]: [具体任务]
...`

	a, _ := agent.NewAgent(ctx, &agent.AgentConfig{
		Model:  model,
		Prompt: prompt,
	})
	return a
}

// 创建 Router（路由器）
func createRouter(ctx context.Context) agent.Agent {
	model, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey: "your-api-key",
		Model:  "gpt-4o-mini",
	})

	prompt := `你是一个任务路由器，负责将任务分配给最合适的专家。

可用专家：
- search: 搜索专家，负责查找信息
- analyze: 分析专家，负责分析数据
- write: 写作专家，负责撰写内容
- code: 编程专家，负责编写代码

根据任务内容，选择最合适的专家。
输出：专家名称`

	a, _ := agent.NewAgent(ctx, &agent.AgentConfig{
		Model:  model,
		Prompt: prompt,
	})
	return a
}

// 创建各个 Worker
func createSearchWorker(ctx context.Context) agent.Agent {
	model, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey: "your-api-key",
		Model:  "gpt-4o-mini",
	})

	a, _ := agent.NewAgent(ctx, &agent.AgentConfig{
		Model:  model,
		Prompt: "你是一个搜索专家，使用工具搜索相关信息并返回结果。",
		Tools:  []agent.Tool{/* search tool */},
	})
	return a
}

func createAnalyzeWorker(ctx context.Context) agent.Agent {
	model, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey: "your-api-key",
		Model:  "gpt-4o-mini",
	})

	a, _ := agent.NewAgent(ctx, &agent.AgentConfig{
		Model:  model,
		Prompt: "你是一个分析专家，负责分析数据并提供洞察。",
	})
	return a
}

func createWriteWorker(ctx context.Context) agent.Agent {
	model, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey: "your-api-key",
		Model:  "gpt-4o-mini",
	})

	a, _ := agent.NewAgent(ctx, &agent.AgentConfig{
		Model:  model,
		Prompt: "你是一个写作专家，负责撰写高质量的内容。",
	})
	return a
}

func createCodeWorker(ctx context.Context) agent.Agent {
	model, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey: "your-api-key",
		Model:  "gpt-4o-mini",
	})

	a, _ := agent.NewAgent(ctx, &agent.AgentConfig{
		Model:  model,
		Prompt: "你是一个编程专家，负责编写和调试代码。",
		Tools:  []agent.Tool{/* code execution tool */},
	})
	return a
}

// 路由解析
func parseWorkerName(output string) string {
	output = strings.TrimSpace(output)
	for _, name := range []string{"search", "analyze", "write", "code"} {
		if strings.Contains(strings.ToLower(output), name) {
			return name
		}
	}
	return "analyze" // 默认
}
```

---

## 7. 实际应用案例

### 7.1 Research Assistant（研究助手）

```go
package main

import (
	"context"
	"fmt"

	"github.com/cloudwego/eino/adk/agent"
	"github.com/cloudwego/eino/adk/supervisor"
)

// Research Assistant - 多 Agent 协作研究系统

type ResearchAssistant struct {
	supervisor *supervisor.Supervisor
}

func NewResearchAssistant(ctx context.Context) (*ResearchAssistant, error) {
	// 创建子 Agent
	searchAgent := createAgent(ctx, "search", "你负责搜索相关学术文献。")
	paperAgent := createAgent(ctx, "paper", "你负责阅读和总结论文。")
	analysisAgent := createAgent(ctx, "analysis", "你负责分析研究趋势。")
	synthesizeAgent := createAgent(ctx, "synthesize", "你负责综合分析结果。")

	// 创建 Supervisor
	sv, err := supervisor.New(ctx, &supervisor.Config{
		Name: "research_assistant",
		Model: func(ctx context.Context) (chatbot.ChatModel, error) {
			return openai.NewChatModel(ctx, &openai.ChatModelConfig{
				APIKey: "your-api-key",
				Model:  "gpt-4o",
			})
		},
		SubAgents: []supervisor.SubAgent{
			{Name: "search", Agent: searchAgent, Description: "搜索学术文献"},
			{Name: "paper", Agent: paperAgent, Description: "阅读和总结论文"},
			{Name: "analysis", Agent: analysisAgent, Description: "分析研究趋势"},
			{Name: "synthesize", Agent: synthesizeAgent, Description: "综合分析结果"},
		},
	})
	if err != nil {
		return nil, err
	}

	return &ResearchAssistant{supervisor: sv}, nil
}

func (ra *ResearchAssistant) Research(ctx context.Context, topic string) (string, error) {
	return ra.supervisor.Run(ctx, fmt.Sprintf(
		"请对 '%s' 主题进行深入研究，包括：\n1. 搜索相关学术文献\n2. 阅读关键论文\n3. 分析研究趋势\n4. 综合写成研究报告", topic))
}

// 创建 Agent 辅助函数
func createAgent(ctx context.Context, name, prompt string) agent.Agent {
	model, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey: "your-api-key",
		Model:  "gpt-4o-mini",
	})
	a, _ := agent.NewAgent(ctx, &agent.AgentConfig{
		Model:  model,
		Prompt: prompt,
	})
	return a
}
```

### 7.2 Code Assistant（代码助手）

```go
package main

import (
	"context"
	"fmt"

	"github.com/cloudwego/eino/adk/agent"
	"github.com/cloudwego/eino/adk/plan"
)

// Code Assistant - 代码开发和调试系统

func buildCodeAssistant(ctx context.Context) agent.Agent {
	// 创建代码生成 Agent
	codeGen := createAgent(ctx, "你是一个代码生成专家。")

	// 创建代码审查 Agent
	codeReview := createAgent(ctx, "你是一个代码审查专家，负责发现潜在问题。")

	// 创建测试生成 Agent
	testGen := createAgent(ctx, "你是一个测试专家，负责编写单元测试。")

	// 创建 Plan-Execute Agent
	codeAgent := plan.NewPlanExecuteAgent(ctx, &plan.Config{
		Planner:     codeGen,
		Executor:    codeReview,
		Checker:     testGen,
		MaxAttempts: 3,
	})

	return codeAgent
}

func (c *CodeAssistant) GenerateCode(ctx context.Context, requirement string) (string, error) {
	return c.agent.Run(ctx, fmt.Sprintf(
		"根据以下需求生成代码：\n%s\n\n要求：\n1. 代码规范\n2. 包含单元测试\n3. 错误处理完善", requirement))
}
```

---

## 8. 架构设计最佳实践

### 8.1 Agent 职责划分

| 原则 | 说明 | 示例 |
|------|------|------|
| 单一职责 | 每个 Agent 只负责一个领域 | SearchAgent 只搜索 |
| 清晰边界 | Agent 间接口明确 | 固定的输入输出格式 |
| 最小知识 | Agent 只知道必要信息 | 不暴露其他 Agent 细节 |
| 可复用性 | Agent 可独立使用 | SearchAgent 可单独运行 |

### 8.2 通信协议设计

```go
// 标准 Agent 通信协议

type AgentMessage struct {
	Type      MessageType  `json:"type"`
	From      string       `json:"from"`
	To        string       `json:"to"`
	Content   string       `json:"content"`
	Metadata  map[string]any `json:"metadata"`
	Timestamp time.Time    `json:"timestamp"`
}

type MessageType string

const (
	MsgTypeRequest   MessageType = "request"
	MsgTypeResponse  MessageType = "response"
	MsgTypeDelegate  MessageType = "delegate"
	MsgTypeResult    MessageType = "result"
	MsgTypeError     MessageType = "error"
)
```

### 8.3 错误处理策略

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/cloudwego/eino/adk/agent"
)

// 多级错误处理

type ErrorHandler struct {
	retryLimits map[string]int
	fallbacks   map[string]agent.Agent
}

func NewErrorHandler() *ErrorHandler {
	return &ErrorHandler{
		retryLimits: map[string]int{
			"search": 3,
			"code":   2,
			"write":  1,
		},
		fallbacks: make(map[string]agent.Agent),
	}
}

func (h *ErrorHandler) HandleError(ctx context.Context, agentName string, err error) error {
	// 检查重试次数
	if retryCount, ok := ctx.Value("retry_count").(int); ok {
		if retryCount >= h.retryLimits[agentName] {
			// 使用 fallback
			if fallback, ok := h.fallbacks[agentName]; ok {
				return fallback.Run(ctx, "处理之前失败的任务")
			}
			return fmt.Errorf("agent %s 失败，已达最大重试次数", agentName)
		}
	}

	// 返回错误继续重试
	return err
}

func (h *ErrorHandler) SetFallback(agentName string, fallback agent.Agent) {
	h.fallbacks[agentName] = fallback
}
```

---

## 9. 常见问题

### Q1: Agent 之间如何传递复杂数据？

**解决方案**：
- 使用结构化的输出格式（如 JSON）
- 定义标准的消息协议
- 使用共享的 Context

```go
type TaskResult struct {
	AgentID  string
	Output   string
	Metadata map[string]any
	Success  bool
}
```

### Q2: 如何避免 Agent 循环调用？

**解决方案**：
- 设置最大调用深度
- 使用有向无环图（DAG）控制调用关系
- 维护调用图检测循环

### Q3: 如何调试 Multi-Agent 系统？

**解决方案**：
1. 添加详细的日志
2. 使用 Callback 记录调用链
3. 可视化 Agent 调用图
4. 独立测试每个 Agent

### Q4: Supervisor 如何选择合适的子 Agent？

**解决方案**：
- 基于规则的路由（关键词匹配）
- 基于 LLM 的路由（让 LLM 决策）
- 混合路由（先规则后 LLM）

### Q5: 如何保证 Multi-Agent 的稳定性？

**解决方案**：
1. 添加超时机制
2. 实现错误重试
3. 设置降级策略
4. 监控关键指标

---

## 相关链接

- [教程-ADK.md](../04-智能体/教程-ADK.md) - ADK 完整指南
- [进阶-人机交互.md](../05-进阶/进阶-人机交互.md) - 人机交互机制
- [教程-回调.md](../02-组件/教程-回调.md) - 回调机制

---

> *本文档基于 Eino v0.x 编写*
