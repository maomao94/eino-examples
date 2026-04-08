# 02-组件

本章节讲解 Eino 框架的基础组件。

---

## 文件列表

| 文件 | 说明 |
|------|------|
| [教程-模型.md](教程-模型.md) | ChatModel 的配置、调用、工具绑定 |
| [教程-工具.md](教程-工具.md) | 工具创建、中间件、内置工具 |
| [教程-内存.md](教程-内存.md) | 内存、文件、Redis、滑动窗口 |
| [教程-组件.md](教程-组件.md) | Model、Tool、Retriever、Prompt 详解 |

---

## 学习内容

### 模型 (Model)

- ChatModel 接口定义
- 流式调用
- 工具绑定
- 多模型支持

### 工具 (Tool)

- BaseTool 接口
- InvokableTool vs RunnableTool
- 工具中间件
- JSON 修复、错误移除

### 检索器 (Retriever)

- MultiQuery 多查询
- Router 路由检索
- RAG 完整示例

### 提示词 (Prompt)

- ChatPrompt 模板
- 变量替换
- 动态模板

---

## 下一步

继续学习 [03-编排](../03-编排/README.md)
