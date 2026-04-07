---
name: eino
description: Eino 是字节跳动开源的 Go 语言大模型应用开发框架。用于：构建 LLM 应用、开发 AI Agent、使用 Chain/Graph/Workflow 编排、集成各类 ChatModel、创建和使用 Tool、实现流式处理和回调。当用户需要使用 Go 语言开发 AI 应用、询问 Eino 框架使用、或者需要编写基于 Eino 的代码时使用此 skill。
---

# Eino 开发指南

Eino [ˈaino] 是字节跳动开源的 Go 语言大模型应用开发框架，提供组件抽象、编排能力、流式处理和完整的开发工具链。

## 快速参考

### 核心概念

| 概念 | 说明 |
|------|------|
| **ChatModel** | 对话大模型抽象，支持 OpenAI/Ollama/ARK 等 |
| **Tool** | 工具抽象，Agent 的执行器 |
| **ChatTemplate** | 消息模板，支持 FString/Jinja2/GoTemplate |
| **Chain** | 简单链式编排，只能向前推进 |
| **Graph** | 有向图编排，支持分支、状态管理 |
| **Workflow** | 字段级数据映射的 DAG 编排 |
| **Agent** | 智能代理，ReAct/Multi-Agent 模式 |

### 最简 LLM 应用

```go
import (
    "context"
    "github.com/cloudwego/eino-ext/components/model/openai"
    "github.com/cloudwego/eino/schema"
)

// 创建 ChatModel
chatModel, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
    Model:  "gpt-4o",
    APIKey: os.Getenv("OPENAI_API_KEY"),
})

// 生成消息
message, _ := chatModel.Generate(ctx, []*schema.Message{
    schema.SystemMessage("你是一个有帮助的助手"),
    schema.UserMessage("你好！"),
})
```

### 创建 Agent

```go
import (
    "github.com/cloudwego/eino/compose"
    "github.com/cloudwego/eino/components/tool/utils"
)

// 创建 Tool
searchTool, _ := utils.InferTool("search", "搜索信息", SearchFunc)

// 绑定工具到 ChatModel
chatModel.BindTools([]*schema.ToolInfo{toolInfo})

// 构建Chain
chain := compose.NewChain[[]*schema.Message, []*schema.Message]()
chain.AppendChatModel(chatModel).
    AppendToolsNode(toolsNode)

agent, _ := chain.Compile(ctx)
```

## 编排模式选择

**使用 Chain：**
- 简单的线性处理流程
- 模板 → 模型 → 输出
- 不需要条件分支

**使用 Graph：**
- 需要条件分支（Branch）
- 需要状态共享（State）
- 复杂的多节点编排
- Agent 场景

**使用 Workflow：**
- 需要字段级数据映射
- 结构化输入输出

## 详细文档

对于具体组件和高级功能的详细信息，请查阅：

- [组件详解](references/components.md) - ChatModel、Tool、PromptTemplate 等
- [编排详解](references/orchestration.md) - Chain、Graph、Workflow 使用
- [Agent 详解](references/agent.md) - ReAct、Multi-Agent 实现
- [流式处理](references/streaming.md) - 流式输入输出处理
- [回调机制](references/callbacks.md) - 切面注入和监控

## 常用命令

```bash
# 安装 Eino
go get github.com/cloudwego/eino@latest
go get github.com/cloudwego/eino-ext@latest

# 查看官方示例
git clone https://github.com/cloudwego/eino-examples
```

## 官方资源

- GitHub: https://github.com/cloudwego/eino
- 官方文档: https://www.cloudwego.io/zh/docs/eino/
- API Reference: https://pkg.go.dev/github.com/cloudwego/eino
