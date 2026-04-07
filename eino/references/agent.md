# Eino Agent 详解

Agent（智能代理）是结合 LLM 理解能力和 Tool 执行能力的系统，能自主完成复杂任务。

## Agent 核心组成

### ChatModel - Agent 的大脑

负责理解用户意图、选择工具、生成参数、转化结果。

```go
import "github.com/cloudwego/eino-ext/components/model/openai"

chatModel, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
    Model:  "gpt-4",
    APIKey: os.Getenv("OPENAI_API_KEY"),
})
```

### ToolsNode - 工具管理器

管理和执行多个工具。

```go
import "github.com/cloudwego/eino/compose"

toolsNode, _ := compose.NewToolNode(ctx, &compose.ToolsNodeConfig{
    Tools: []tool.BaseTool{tool1, tool2, tool3},
})
```

## 用 Chain 构建 Agent

最简单的 Agent 实现，使用线性链式编排。

```go
import (
    "github.com/cloudwego/eino/compose"
    "github.com/cloudwego/eino/schema"
)

// 1. 准备工具
searchTool, _ := utils.InferTool("search", "搜索信息", SearchFunc)
weatherTool, _ := utils.InferTool("weather", "查询天气", WeatherFunc)

tools := []tool.BaseTool{searchTool, weatherTool}

// 2. 绑定工具到 ChatModel
toolInfos := make([]*schema.ToolInfo, 0, len(tools))
for _, t := range tools {
    info, _ := t.Info(ctx)
    toolInfos = append(toolInfos, info)
}
chatModel.BindTools(toolInfos)

// 3. 创建 ToolsNode
toolsNode, _ := compose.NewToolNode(ctx, &compose.ToolsNodeConfig{
    Tools: tools,
})

// 4. 构建 Chain
chain := compose.NewChain[[]*schema.Message, []*schema.Message]()
chain.
    AppendChatModel(chatModel).
    AppendToolsNode(toolsNode)

// 5. 编译运行
agent, _ := chain.Compile(ctx)
result, _ := agent.Invoke(ctx, []*schema.Message{
    schema.UserMessage("今天北京天气怎么样？"),
})
```

## 用 Graph 构建 ReAct Agent

ReAct（Reasoning + Acting）Agent 通过思考-行动-观察循环解决问题。

```go
// 创建 Graph
g := compose.NewGraph[map[string]any, []*schema.Message]()

// 添加节点
g.AddChatTemplateNode("template", prompt)
g.AddChatModelNode("model", chatModel)
g.AddToolsNode("tools", toolsNode)
g.AddLambdaNode("converter", converter)

// 添加分支 - 根据模型输出决定是否调用工具
branch := compose.NewBranch(func(ctx context.Context, msg *schema.Message) (string, error) {
    if len(msg.ToolCalls) > 0 {
        return "tools", nil  // 有工具调用，走工具节点
    }
    return "end", nil        // 无工具调用，直接结束
})

// 连接节点
g.AddEdge(compose.START, "template")
g.AddEdge("template", "model")
g.AddBranch("model", branch)
g.AddEdge("tools", "converter")
g.AddEdge("converter", compose.END)

// 编译
agent, _ := g.Compile(ctx, compose.WithMaxRunSteps(10))
```

### ReAct 完整示例

```go
func main() {
    ctx := context.Background()

    // 1. 创建工具
    searchTool := createSearchTool()
    calculatorTool := createCalculatorTool()

    tools := []tool.BaseTool{searchTool, calculatorTool}

    // 2. 创建 ChatModel 并绑定工具
    chatModel, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
        Model:  "gpt-4",
        APIKey: os.Getenv("OPENAI_API_KEY"),
    })

    toolInfos := getToolInfos(tools)
    chatModel.BindTools(toolInfos)

    // 3. 创建 ToolsNode
    toolsNode, _ := compose.NewToolNode(ctx, &compose.ToolsNodeConfig{
        Tools: tools,
    })

    // 4. 创建系统提示
    prompt := prompt.FromMessages(schema.FString,
        schema.SystemMessage("你是一个有帮助的助手，可以使用工具获取信息"),
        schema.MessagesPlaceholder("chat_history", true),
        schema.UserMessage("{input}"),
    )

    // 5. 构建 Graph
    g := compose.NewGraph[map[string]any, []*schema.Message]()
    g.AddChatTemplateNode("template", prompt)
    g.AddChatModelNode("model", chatModel)
    g.AddToolsNode("tools", toolsNode)

    // 6. 添加分支
    branch := compose.NewBranch(func(ctx context.Context, msg *schema.Message) (string, error) {
        if len(msg.ToolCalls) > 0 {
            return "tools", nil
        }
        return END, nil
    })
    g.AddBranch("model", branch)
    g.AddEdge("tools", "model")  // 工具结果回传给模型

    // 7. 编译运行
    agent, _ := g.Compile(ctx, compose.WithMaxRunSteps(10))

    result, _ := agent.Invoke(ctx, map[string]any{
        "input": "搜索一下 Eino 框架的信息",
        "chat_history": []*schema.Message{},
    })
}
```

## Multi-Agent 系统

多个 Agent 协同工作，每个 Agent 有特定职责。

### Host MultiAgent 模式

一个主 Agent 协调多个子 Agent。

```go
import "github.com/cloudwego/eino/flow/agent/multi_agent"

// 创建子 Agent
researchAgent := createAgent("研究员", "搜索和分析信息")
writerAgent := createAgent("作者", "撰写报告")
reviewerAgent := createAgent("审核员", "审核内容")

// 创建 Host
host := multi_agent.NewHost(ctx, &multi_agent.HostConfig{
    Agents: []*multi_agent.AgentConfig{
        {Name: "research", Agent: researchAgent},
        {Name: "writer", Agent: writerAgent},
        {Name: "reviewer", Agent: reviewerAgent},
    },
    ChatModel: chatModel,
})

// 运行
result, _ := host.Invoke(ctx, "写一份关于 Eino 的报告")
```

### Plan-Executor 模式

规划 Agent 负责任务分解，执行 Agent 负责具体执行。

```go
import "github.com/cloudwego/eino/flow/agent/multi_agent/plan_executor"

// 创建规划器
planner := createPlannerAgent()

// 创建执行器
executor := createExecutorAgent()

// 创建 Plan-Executor 系统
agent := plan_executor.NewPlanExecutor(ctx, &plan_executor.Config{
    Planner: planner,
    Executor: executor,
    ChatModel: chatModel,
})

result, _ := agent.Invoke(ctx, "帮我分析公司数据并生成报告")
```

### Reflection Agents

自反思 Agent，能够自我评估和改进。

```go
import "github.com/cloudwego/eino/flow/agent/multi_agent/reflection"

agent := reflection.NewReflectionAgent(ctx, &reflection.Config{
    MainAgent:    mainAgent,
    ReflectAgent: reflectAgent,
    MaxReflections: 3,  // 最多反思3次
})
```

## Agent 开发最佳实践

### 1. 工具设计

```go
// 工具描述要清晰
func GetWeatherTool() tool.BaseTool {
    info := &schema.ToolInfo{
        Name: "get_weather",
        Desc: "查询指定城市的实时天气信息",
        ParamsOneOf: schema.NewParamsOneOfByParams(map[string]*schema.ParameterInfo{
            "city": {
                Type:     schema.String,
                Desc:     "城市名称，例如：北京、上海",
                Required: true,
            },
        }),
    }
    return utils.NewTool(info, GetWeather)
}
```

### 2. 提示词工程

```go
// ReAct 提示词模板
reactPrompt := prompt.FromMessages(schema.FString,
    schema.SystemMessage(`你是一个智能助手，可以使用以下工具：
{tools}

请按以下格式思考：
Thought: 我需要思考什么
Action: 工具名称
Action Input: 工具参数

观察结果后继续思考，直到得到最终答案。`),
    schema.MessagesPlaceholder("history", true),
    schema.UserMessage("{input}"),
)
```

### 3. 错误处理

```go
// 在 Lambda 中添加错误处理
errorHandler := compose.InvokableLambda(func(ctx context.Context, msgs []*schema.Message) ([]*schema.Message, error) {
    for _, msg := range msgs {
        if msg.Role == schema.Tool && msg.Content == "" {
            // 工具执行失败，添加错误信息
            return append(msgs, schema.SystemMessage("工具执行失败，请重试")), nil
        }
    }
    return msgs, nil
})
```

### 4. 状态管理

```go
// 使用状态保存对话历史
type AgentState struct {
    History []*schema.Message
    Step    int
}

g := compose.NewGraph[string, string](
    compose.WithGenLocalState(func(ctx context.Context) *AgentState {
        return &AgentState{History: []*schema.Message{}}
    }),
)
```

## Agent 场景示例

### 场景1：客服 Agent

```go
// 工具：查询订单、退款、物流
tools := []tool.BaseTool{
    createOrderQueryTool(),
    createRefundTool(),
    createShippingTool(),
}

// 构建客服 Agent
customerServiceAgent := buildAgent(tools, "你是专业的客服助手")
```

### 场景2：代码助手 Agent

```go
// 工具：搜索代码、执行代码、解释代码
tools := []tool.BaseTool{
    createCodeSearchTool(),
    createCodeExecutorTool(),
    createCodeExplainerTool(),
}

codeAgent := buildAgent(tools, "你是编程助手")
```

### 场景3：数据分析 Agent

```go
// 工具：查询数据库、生成图表、导出报告
tools := []tool.BaseTool{
    createDBQueryTool(),
    createChartTool(),
    createReportTool(),
}

analysisAgent := buildAgent(tools, "你是数据分析专家")
```
