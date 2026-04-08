# A2UI 协议 - Agent 到 UI 流式渲染

> **前置知识**：[教程-ADK.md](../04-智能体/教程-ADK.md)
>
> **学习目标**：理解 A2UI 协议，掌握将 Agent 输出实时渲染到 Web UI 的方法
>
> **相关章节**：[进阶-人机交互.md](./进阶-人机交互.md) | [教程-流编排.md](../03-编排/教程-流编排.md)

---

## 目录

1. [概述](#1-概述)
2. [协议架构](#2-协议架构)
3. [消息类型](#3-消息类型)
4. [组件系统](#4-组件系统)
5. [使用示例](#5-使用示例)
6. [前端集成](#6-前端集成)
7. [最佳实践](#7-最佳实践)

---

## 1. 概述

### 1.1 什么是 A2UI

A2UI（Agent to UI）是一种将 Agent 输出实时流式渲染到 Web UI 的协议规范：

```
┌─────────────────────────────────────────────────────────────────┐
│                      A2UI 协议架构                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────┐      JSONL/SSE      ┌─────────────────┐     │
│   │    Eino     │ ──────────────────▶ │    Web UI       │     │
│   │   Agent     │                     │   (前端渲染)     │     │
│   └─────────────┘                     └─────────────────┘     │
│                                                                  │
│   协议：a2ui v0.8                                                 │
│   传输：JSONL (JSON Lines) 或 SSE (Server-Sent Events)            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 协议特点

| 特性 | 说明 |
|------|------|
| 实时流式 | 边生成边渲染，低延迟 |
| 结构化组件 | Text/Column/Card/Row 组件系统 |
| 数据绑定 | 支持 DataKey 动态更新 |
| 中断支持 | 内置 InterruptRequest 机制 |
| 无依赖 | 纯 JSON，可对接任何前端框架 |

### 1.3 版本

当前版本：**v0.8**

---

## 2. 协议架构

### 2.1 消息信封

每个 A2UI 消息是一个 JSON 对象，顶层包含以下字段之一：

```go
type Message struct {
    BeginRendering   *BeginRenderingMsg   `json:"beginRendering,omitempty"`
    SurfaceUpdate    *SurfaceUpdateMsg    `json:"surfaceUpdate,omitempty"`
    DataModelUpdate  *DataModelUpdateMsg  `json:"dataModelUpdate,omitempty"`
    DeleteSurface    *DeleteSurfaceMsg    `json:"deleteSurface,omitempty"`
    InterruptRequest *InterruptRequestMsg `json:"interruptRequest,omitempty"`
}
```

**原则**：每个消息只包含一种类型，通过 `omitempty` 标签确保互斥。

### 2.2 传输方式

**方式一：JSONL (JSON Lines)**
```
{"beginRendering":{"surfaceId":"chat-123","root":"root-col"}}
{"surfaceUpdate":{"surfaceId":"chat-123","components":[...]}}
{"dataModelUpdate":{"surfaceId":"chat-123","contents":[...]}}
```

**方式二：SSE (Server-Sent Events)**
```
event: message
data: {"surfaceUpdate":{"surfaceId":"chat-123",...}}

event: message
data: {"dataModelUpdate":{"surfaceId":"chat-123",...}}
```

### 2.3 Surface（表面）

Surface 是 UI 的独立渲染区域：

- 每个 Surface 有唯一的 `surfaceId`
- 多个 Surface 可同时存在
- 可动态创建和销毁

---

## 3. 消息类型

### 3.1 BeginRendering - 开始渲染

通知前端开始一个渲染会话：

```go
type BeginRenderingMsg struct {
    SurfaceID string `json:"surfaceId"`  // 表面 ID
    Root      string `json:"root"`        // 根组件 ID
}
```

**示例**：
```json
{
  "beginRendering": {
    "surfaceId": "chat-abc123",
    "root": "root-col"
  }
}
```

**用途**：
- 创建新的聊天会话
- 初始化 UI 组件树

---

### 3.2 SurfaceUpdate - 更新表面

添加或更新 UI 组件：

```go
type SurfaceUpdateMsg struct {
    SurfaceID  string      `json:"surfaceId"`   // 表面 ID
    Components []Component `json:"components"`   // 组件列表
}
```

**示例**：
```json
{
  "surfaceUpdate": {
    "surfaceId": "chat-abc123",
    "components": [
      {
        "id": "root-col",
        "component": {
          "Column": {
            "children": ["msg-0-card", "msg-1-card"]
          }
        }
      },
      {
        "id": "msg-0-card",
        "component": {
          "Card": {
            "children": ["msg-0-col"]
          }
        }
      }
    ]
  }
}
```

---

### 3.3 DataModelUpdate - 更新数据模型

动态更新组件的数据绑定：

```go
type DataModelUpdateMsg struct {
    SurfaceID string        `json:"surfaceId"`  // 表面 ID
    Contents  []DataContent `json:"contents"`    // 数据内容
}

type DataContent struct {
    Key          string `json:"key"`            // 数据键
    ValueString string `json:"valueString,omitempty"` // 字符串值
}
```

**示例**：
```json
{
  "dataModelUpdate": {
    "surfaceId": "chat-abc123",
    "contents": [
      {
        "key": "chat-abc123/msg-0",
        "valueString": "这是实时生成的内容..."
      }
    ]
  }
}
```

**用途**：
- 流式文本更新
- 动态数据绑定

---

### 3.4 DeleteSurface - 删除表面

移除一个渲染表面：

```go
type DeleteSurfaceMsg struct {
    SurfaceID string `json:"surfaceId"`  // 表面 ID
}
```

**示例**：
```json
{
  "deleteSurface": {
    "surfaceId": "chat-abc123"
  }
}
```

---

### 3.5 InterruptRequest - 中断请求

当 Agent 需要人工审批时发送：

```go
type InterruptRequestMsg struct {
    InterruptID string `json:"interruptId"`  // 中断 ID
    Description  string `json:"description"`  // 中断描述
}
```

**示例**：
```json
{
  "interruptRequest": {
    "interruptId": "intr-456",
    "description": "执行转账 10000 元到账户 A，需要审批"
  }
}
```

**用途**：
- 敏感操作审批
- 人工确认

---

## 4. 组件系统

### 4.1 组件定义

```go
type Component struct {
    ID        string         `json:"id"`         // 组件 ID（唯一）
    Component ComponentValue `json:"component"`   // 组件值
}

type ComponentValue struct {
    Text   *TextComp   `json:"Text,omitempty"`
    Column *ColumnComp `json:"Column,omitempty"`
    Card   *CardComp   `json:"Card,omitempty"`
    Row    *RowComp    `json:"Row,omitempty"`
}
```

### 4.2 Text - 文本组件

显示文本内容：

```go
type TextComp struct {
    Value     string `json:"value,omitempty"`      // 静态文本值
    DataKey   string `json:"dataKey,omitempty"`   // 数据绑定键
    UsageHint string `json:"usageHint,omitempty"` // 用途提示
}
```

| 字段 | 说明 | 可选值 |
|------|------|--------|
| Value | 静态文本 | - |
| DataKey | 从数据模型读取值 | - |
| UsageHint | 样式提示 | `"caption"` / `"body"` / `"title"` |

**示例**：
```json
{
  "id": "title-1",
  "component": {
    "Text": {
      "value": "欢迎使用",
      "usageHint": "title"
    }
  }
}
```

**动态文本**：
```json
{
  "id": "content-1",
  "component": {
    "Text": {
      "dataKey": "chat-123/msg-0",
      "usageHint": "body"
    }
  }
}
```

---

### 4.3 Column - 垂直布局

垂直排列子组件：

```go
type ColumnComp struct {
    Children []string `json:"children"`  // 子组件 ID 列表
}
```

**示例**：
```json
{
  "id": "main-col",
  "component": {
    "Column": {
      "children": ["header", "content", "footer"]
    }
  }
}
```

---

### 4.4 Row - 水平布局

水平排列子组件：

```go
type RowComp struct {
    Children []string `json:"children"`  // 子组件 ID 列表
}
```

**示例**：
```json
{
  "id": "tool-bar",
  "component": {
    "Row": {
      "children": ["btn-search", "btn-export", "btn-settings"]
    }
  }
}
```

---

### 4.5 Card - 卡片容器

包装子组件为卡片：

```go
type CardComp struct {
    Children []string `json:"children"`  // 子组件 ID 列表
}
```

**示例**：
```json
{
  "id": "message-card",
  "component": {
    "Card": {
      "children": ["msg-label", "msg-content"]
    }
  }
}
```

---

### 4.6 组件组合示例

构建一个消息卡片：

```json
{
  "components": [
    {
      "id": "root-col",
      "component": {
        "Column": {
          "children": ["msg-card-0", "msg-card-1"]
        }
      }
    },
    {
      "id": "msg-card-0",
      "component": {
        "Card": {
          "children": ["msg-col-0"]
        }
      }
    },
    {
      "id": "msg-col-0",
      "component": {
        "Column": {
          "children": ["msg-role", "msg-content"]
        }
      }
    },
    {
      "id": "msg-role",
      "component": {
        "Text": {
          "value": "You",
          "usageHint": "caption"
        }
      }
    },
    {
      "id": "msg-content",
      "component": {
        "Text": {
          "dataKey": "chat-123/msg-0",
          "usageHint": "body"
        }
      }
    }
  ]
}
```

**渲染效果**：
```
┌─────────────────────────┐
│ You                      │  ← caption
│ 这是动态生成的内容...     │  ← body (流式更新)
└─────────────────────────┘
```

---

## 5. 使用示例

### 5.1 后端集成

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"

	"github.com/cloudwego/eino/adk"
	"github.com/cloudwego/eino-ext/a2ui"
	"github.com/cloudwego/eino/chatbot/openai"
)

// A2UI 流式服务示例
// 将 Agent 事件流式转换为 A2UI 消息

func main() {
	ctx := context.Background()

	// 1. 创建 Agent
	model, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey: "your-api-key",
		Model:  "gpt-4o",
	})

	agent, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
		Model:       model,
		Instruction: "你是一个有帮助的助手。",
		Tools:       []tool.BaseTool{/* tools */},
	})

	runner := adk.NewRunner(ctx, adk.RunnerConfig{Agent: agent})

	// 2. 设置 HTTP 处理
	http.HandleFunc("/stream", func(w http.ResponseWriter, r *http.Request) {
		sessionID := r.URL.Query().Get("session")
		userMsg := r.URL.Query().Get("message")

		// 设置 SSE
		w.Header().Set("Content-Type", "text/event-stream")
		w.Header().Set("Cache-Control", "no-cache")
		w.Header().Set("Connection", "keep-alive")

		// 创建事件流
		events, _ := runner.Run(ctx, []adk.Message{
			schema.UserMessage(userMsg),
		})

		// 转换为 A2UI 并发送
		lastContent, interruptID, _, err := a2ui.StreamToWriter(w, sessionID, history, events)
		if err != nil {
			log.Printf("流错误: %v", err)
		}

		// 处理中断
		if interruptID != "" {
			// 等待用户审批...
		}

		// 保存到历史
		history = append(history,
			schema.UserMessage(userMsg),
			schema.AssistantMessage(lastContent),
		)
	})

	log.Println("服务启动: http://localhost:8080")
	http.ListenAndServe(":8080", nil)
}
```

### 5.2 消息编码

```go
package main

import (
	"fmt"

	"github.com/cloudwego/eino-ext/a2ui"
)

// 手动构造 A2UI 消息

func main() {
	// 创建消息
	msg := a2ui.Message{
		BeginRendering: &a2ui.BeginRenderingMsg{
			SurfaceID: "chat-123",
			Root:      "root-col",
		},
	}

	// 编码为 JSON
	data, err := a2ui.Encode(msg)
	if err != nil {
		panic(err)
	}

	fmt.Printf("%s\n", string(data))
	// 输出: {"beginRendering":{"surfaceId":"chat-123","root":"root-col"}}
}
```

### 5.3 会话历史渲染

```go
// 渲染历史消息到 UI
func renderSessionHistory(w io.Writer, sessionID string, history []*schema.Message) error {
	return a2ui.RenderHistory(w, sessionID, history)
}
```

---

## 6. 前端集成

### 6.1 JavaScript SSE 客户端

```javascript
class A2UIClient {
    constructor(url) {
        this.url = url;
        this.surface = null;
        this.dataModel = {};
        this.components = {};
    }

    async connect(sessionId, message) {
        const response = await fetch(
            `${this.url}?session=${sessionId}&message=${encodeURIComponent(message)}`
        );

        const reader = response.body.getReader();
        const decoder = new TextDecoder();

        while (true) {
            const { done, value } = await reader.read();
            if (done) break;

            const text = decoder.decode(value);
            const lines = text.split('\n').filter(line => line.trim());

            for (const line of lines) {
                try {
                    const msg = JSON.parse(line);
                    this.handleMessage(msg);
                } catch (e) {
                    console.error('解析错误:', e);
                }
            }
        }
    }

    handleMessage(msg) {
        if (msg.beginRendering) {
            this.onBeginRendering(msg.beginRendering);
        } else if (msg.surfaceUpdate) {
            this.onSurfaceUpdate(msg.surfaceUpdate);
        } else if (msg.dataModelUpdate) {
            this.onDataModelUpdate(msg.dataModelUpdate);
        } else if (msg.deleteSurface) {
            this.onDeleteSurface(msg.deleteSurface);
        } else if (msg.interruptRequest) {
            this.onInterruptRequest(msg.interruptRequest);
        }
    }

    onBeginRendering(data) {
        console.log('开始渲染:', data);
        this.surface = {
            id: data.surfaceId,
            root: data.root
        };
    }

    onSurfaceUpdate(data) {
        console.log('更新组件:', data.components);

        // 更新组件树
        for (const comp of data.components) {
            this.components[comp.id] = comp.component;
        }

        // 渲染 UI
        this.render();
    }

    onDataModelUpdate(data) {
        // 更新数据模型
        for (const content of data.contents) {
            this.dataModel[content.key] = content.valueString;
        }

        // 重新渲染
        this.render();
    }

    onDeleteSurface(data) {
        console.log('删除表面:', data.surfaceId);
        this.surface = null;
        this.components = {};
        this.dataModel = {};
    }

    onInterruptRequest(data) {
        console.log('中断请求:', data);
        // 显示审批对话框
        if (confirm(data.description)) {
            // 批准
            this.resume(data.interruptId, true);
        } else {
            // 拒绝
            this.resume(data.interruptId, false);
        }
    }

    render() {
        if (!this.surface) return;

        // 递归渲染组件
        const root = this.components[this.surface.root];
        const element = this.renderComponent(root);

        // 挂载到 DOM
        document.getElementById('app').innerHTML = '';
        document.getElementById('app').appendChild(element);
    }

    renderComponent(comp) {
        if (comp.Column) {
            const el = document.createElement('div');
            el.className = 'a2ui-column';
            for (const childId of comp.Column.children) {
                el.appendChild(this.renderComponent(this.components[childId]));
            }
            return el;
        }

        if (comp.Row) {
            const el = document.createElement('div');
            el.className = 'a2ui-row';
            for (const childId of comp.Row.children) {
                el.appendChild(this.renderComponent(this.components[childId]));
            }
            return el;
        }

        if (comp.Card) {
            const el = document.createElement('div');
            el.className = 'a2ui-card';
            for (const childId of comp.Card.children) {
                el.appendChild(this.renderComponent(this.components[childId]));
            }
            return el;
        }

        if (comp.Text) {
            const el = document.createElement('span');
            el.className = `a2ui-text a2ui-text--${comp.Text.usageHint || 'body'}`;

            // 支持数据绑定
            if (comp.Text.dataKey) {
                el.textContent = this.dataModel[comp.Text.dataKey] || '';
            } else {
                el.textContent = comp.Text.value || '';
            }

            return el;
        }

        return document.createElement('div');
    }

    async resume(interruptId, approved) {
        await fetch(`${this.url}/resume`, {
            method: 'POST',
            body: JSON.stringify({ interruptId, approved }),
        });
    }
}

// 使用
const client = new A2UIClient('http://localhost:8080/stream');
client.connect('session-123', '你好，请介绍一下 Eino 框架');
```

### 6.2 React 集成

```tsx
import React, { useState, useEffect, useRef } from 'react';

interface A2UIMessage {
    beginRendering?: { surfaceId: string; root: string };
    surfaceUpdate?: { surfaceId: string; components: any[] };
    dataModelUpdate?: { surfaceId: string; contents: any[] };
    interruptRequest?: { interruptId: string; description: string };
}

interface Component {
    id: string;
    component: {
        Text?: { value?: string; dataKey?: string; usageHint?: string };
        Column?: { children: string[] };
        Row?: { children: string[] };
        Card?: { children: string[] };
    };
}

export function A2UIChat() {
    const [components, setComponents] = useState<Record<string, Component>>({});
    const [dataModel, setDataModel] = useState<Record<string, string>>({});
    const [pendingInterrupt, setPendingInterrupt] = useState<{ id: string; desc: string } | null>(null);

    const surfaceId = useRef('chat-' + Math.random().toString(36).slice(2));

    const connect = async (message: string) => {
        const response = await fetch(
            `http://localhost:8080/stream?session=${surfaceId.current}&message=${encodeURIComponent(message)}`
        );

        const reader = response.body?.getReader();
        if (!reader) return;

        const decoder = new TextDecoder();

        while (true) {
            const { done, value } = await reader.read();
            if (done) break;

            const text = decoder.decode(value);
            const lines = text.split('\n').filter(Boolean);

            for (const line of lines) {
                try {
                    const msg: A2UIMessage = JSON.parse(line);
                    handleMessage(msg);
                } catch (e) {
                    console.error('解析错误:', e);
                }
            }
        }
    };

    const handleMessage = (msg: A2UIMessage) => {
        if (msg.surfaceUpdate) {
            setComponents(prev => {
                const next = { ...prev };
                for (const comp of msg.surfaceUpdate!.components) {
                    next[comp.id] = comp;
                }
                return next;
            });
        }

        if (msg.dataModelUpdate) {
            setDataModel(prev => {
                const next = { ...prev };
                for (const c of msg.dataModelUpdate!.contents) {
                    next[c.key] = c.valueString;
                }
                return next;
            });
        }

        if (msg.interruptRequest) {
            setPendingInterrupt({
                id: msg.interruptRequest.interruptId,
                desc: msg.interruptRequest.description,
            });
        }
    };

    const handleInterruptResponse = async (approved: boolean) => {
        await fetch('http://localhost:8080/resume', {
            method: 'POST',
            body: JSON.stringify({
                interruptId: pendingInterrupt?.id,
                approved,
            }),
        });
        setPendingInterrupt(null);
    };

    const renderComponent = (id: string): React.ReactNode => {
        const comp = components[id];
        if (!comp) return null;

        const { Text, Column, Row, Card } = comp.component;

        if (Text) {
            const content = Text.dataKey ? dataModel[Text.dataKey] : Text.value;
            return (
                <span
                    key={id}
                    className={`a2ui-text a2ui-text--${Text.usageHint || 'body'}`}
                >
                    {content || ''}
                </span>
            );
        }

        if (Column) {
            return (
                <div key={id} className="a2ui-column">
                    {Column.children.map(cid => renderComponent(cid))}
                </div>
            );
        }

        if (Row) {
            return (
                <div key={id} className="a2ui-row">
                    {Row.children.map(cid => renderComponent(cid))}
                </div>
            );
        }

        if (Card) {
            return (
                <div key={id} className="a2ui-card">
                    {Card.children.map(cid => renderComponent(cid))}
                </div>
            );
        }

        return null;
    };

    return (
        <div className="a2ui-chat">
            <div className="a2ui-messages">
                {components['root-col'] &&
                    components['root-col'].component.Column?.children.map(id =>
                        renderComponent(id)
                    )}
            </div>

            {pendingInterrupt && (
                <div className="a2ui-interrupt">
                    <p>{pendingInterrupt.desc}</p>
                    <button onClick={() => handleInterruptResponse(true)}>批准</button>
                    <button onClick={() => handleInterruptResponse(false)}>拒绝</button>
                </div>
            )}

            <button onClick={() => connect('你好')}>发送</button>
        </div>
    );
}
```

---

## 7. 最佳实践

### 7.1 流式优化

| 实践 | 说明 |
|------|------|
| 批量更新 | 合并多个小更新为一批 |
| 截断显示 | 长内容截断，展开查看完整 |
| 虚拟滚动 | 大量消息时使用虚拟列表 |

### 7.2 错误处理

```go
// 优雅处理错误
func streamEvents(w io.Writer, events *adk.AsyncIterator[*adk.AgentEvent]) {
    for {
        event, ok := events.Next()
        if !ok {
            break
        }

        if event.Err != nil {
            // 发送错误消息
            emitToolChip(w, "error", event.Err.Error())
            break
        }

        // 处理正常事件...
    }
}
```

### 7.3 中断处理

```go
// 检测并处理中断
if event.Action != nil && event.Action.Interrupted != nil {
    // 获取根因中断
    for _, ic := range event.Action.Interrupted.InterruptContexts {
        if ic.IsRootCause {
            emitInterruptRequest(w, ic.ID, fmt.Sprintf("%v", ic.Info))
            return // 等待用户响应
        }
    }
}
```

### 7.4 性能考虑

| 场景 | 建议 |
|------|------|
| 大量组件 | 分批发送 SurfaceUpdate |
| 长文本流 | 使用 DataModelUpdate 增量更新 |
| 高频更新 | 合并相邻的数据更新 |

---

## 8. 核心概念总结

### 8.1 Surface = 会话容器

```
┌─────────────────────────────────────────────────────────────┐
│  Surface (每个会话对应一个 Surface)                          │
│                                                              │
│  Surface ID = "chat-" + sessionID                          │
│                                                              │
│  └── 组件树 (Components)                                     │
│        ├── root-col                                          │
│        │     ├── msg-0-card  (用户消息)                      │
│        │     ├── msg-1-card  (Agent 回复)                   │
│        │     └── msg-2-card  (Tool 调用)                    │
│        │           └── ...                                   │
│                                                              │
│  └── 数据模型 (DataModel)                                    │
│        ├── "chat-xxx/msg-0" → "用户输入"                     │
│        └── "chat-xxx/msg-1" → "Agent 回复内容..."  (流式)    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 8.2 完整会话流程

```
┌──────────────────────────────────────────────────────────────┐
│                     完整会话生命周期                            │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. 新建会话                                                   │
│     POST /sessions → 返回 sessionID                          │
│                                                               │
│  2. 加载历史                                                   │
│     GET /sessions/:id/render                                  │
│     → a2ui.RenderHistory()                                  │
│     → 一次性发送所有历史消息的组件树                           │
│                                                               │
│  3. 发送消息                                                   │
│     POST /sessions/:id/chat                                   │
│     → Runner.Run() → Agent 事件流                            │
│     → a2ui.StreamToWriter() → SSE                           │
│     → 增量更新组件 + DataModel                                │
│                                                               │
│  4. 中断等待审批                                              │
│     → InterruptRequest 消息                                  │
│     → 前端显示审批对话框                                       │
│                                                               │
│  5. 用户审批                                                  │
│     POST /sessions/:id/approve                               │
│     → Runner.ResumeWithParams()                             │
│     → a2ui.StreamContinue() → 继续流式                       │
│                                                               │
│  6. 会话结束/删除                                             │
│     DELETE /sessions/:id                                      │
│     → 可选发送 DeleteSurface                                  │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### 8.3 数据量优化

| 场景 | 处理方式 | 数据量 |
|------|---------|--------|
| 历史消息 | 一次性发送 | 大但只发一次 |
| 新消息 | 增量更新 | 小 |
| 流式文本 | DataModelUpdate | 极小（只更新 key-value） |
| Tool 结果 | SurfaceUpdate | 中等 |

### 8.4 SSE 传输优化

```go
// 1. Keep-Alive 保活
go func() {
    ticker := time.NewTicker(5 * time.Second)
    for { <-ticker.C {
        stream.Publish(&sse.Event{Data: []byte{}})  // 空心跳
    }}
}()

// 2. JSONL 累积发送（遇到换行符才发送）
type sseLineWriter struct { buf []byte }
func (w *sseLineWriter) Write(p []byte) {
    w.buf = append(w.buf, p...)
    if idx := bytes.IndexByte(w.buf, '\n'); idx >= 0 {
        line := w.buf[:idx]
        w.buf = w.buf[idx+1:]
        stream.Publish(&sse.Event{Data: line})
    }
}
```

### 8.5 核心 API 对照

| API | 用途 | 返回值 |
|-----|------|--------|
| `RenderHistory` | 渲染历史消息 | 一次性发送组件树 |
| `StreamToWriter` | 流式发送新消息 | lastContent, interruptID, msgIdx |
| `StreamContinue` | 中断后继续 | 同上 |
| `Encode` | 编码消息 | JSON bytes + `\n` |

### 8.6 类比

| A2UI | 前端框架类比 |
|------|-------------|
| Surface | Vue/React 的根组件实例 |
| Component | 组件树节点 |
| DataModel | 响应式状态 |
| DataKey | 数据绑定引用 |
| SurfaceUpdate | setState() / 状态更新 |
| DataModelUpdate | 细粒度数据更新 |

---

## 相关链接

- [教程-ADK.md](../04-智能体/教程-ADK.md) - ADK 完整指南
- [进阶-人机交互.md](./进阶-人机交互.md) - 人机交互机制
- [教程-流编排.md](../03-编排/教程-流编排.md) - 流式处理

---

> *本文档基于 A2UI v0.8 编写*
