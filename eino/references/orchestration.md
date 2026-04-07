# Eino 编排详解

Eino 提供三种编排方式：Chain、Graph、Workflow，适用于不同的场景。

## Chain 编排

Chain 是最简单的编排方式，适合线性处理流程。

### 基本用法

```go
import "github.com/cloudwego/eino/compose"

// 创建 Chain，指定输入输出类型
chain := compose.NewChain[map[string]any, *schema.Message]()

// 添加节点
chain.AppendChatTemplate(template).
    AppendChatModel(model).
    AppendLambda(converter)

// 编译
runnable, err := chain.Compile(ctx)

// 运行
result, err := runnable.Invoke(ctx, map[string]any{
    "query": "用户问题",
})
```

### 添加 Lambda 节点

```go
// 可调用 Lambda
lambda := compose.InvokableLambda(func(ctx context.Context, input string) (string, error) {
    return "处理后的: " + input, nil
})
chain.AppendLambda(lambda)

// 可流式 Lambda
lambda := compose.StreamableLambda(func(ctx context.Context, input string) (*schema.StreamReader[string], error) {
    sr, sw := schema.Pipe[string](10)
    go func() {
        defer sw.Close()
        sw.Send("chunk1", nil)
        sw.Send("chunk2", nil)
    }()
    return sr, nil
})
```

### 并行处理

```go
parallel := compose.NewParallel()
parallel.
    AddLambda("task1", lambda1).
    AddLambda("task2", lambda2).
    AddChatModel("model", model)

chain.AppendParallel(parallel)
```

### 条件分支

```go
// 分支条件函数
branchCond := func(ctx context.Context, input map[string]any) (string, error) {
    if input["type"] == "A" {
        return "branchA", nil
    }
    return "branchB", nil
}

// 创建分支
branch := compose.NewChainBranch(branchCond).
    AddLambda("branchA", lambdaA).
    AddLambda("branchB", lambdaB)

chain.AppendBranch(branch)
```

## Graph 编排

Graph 是最强大的编排方式，支持复杂的数据流和状态管理。

### 基本用法

```go
// 创建 Graph
g := compose.NewGraph[map[string]any, *schema.Message]()

// 添加节点
g.AddChatTemplateNode("template", template)
g.AddChatModelNode("model", model)
g.AddLambdaNode("converter", lambda)

// 连接节点
g.AddEdge(compose.START, "template")
g.AddEdge("template", "model")
g.AddEdge("model", "converter")
g.AddEdge("converter", compose.END)

// 编译
runnable, err := g.Compile(ctx)
```

### 条件分支

```go
// 分支条件
branch := compose.NewBranch(func(ctx context.Context, input *schema.Message) (string, error) {
    if input.ToolCalls != nil {
        return "tools", nil  // 走工具节点
    }
    return "end", nil        // 直接结束
})

// 添加分支
g.AddBranch("model", branch)
g.AddEdge("tools", "converter")
g.AddEdge("converter", END)
```

### 状态管理

```go
// 定义状态类型
type AgentState struct {
    Messages   []*schema.Message
    ToolCallID string
}

// 创建带状态的 Graph
g := compose.NewGraph[string, string](
    compose.WithGenLocalState(func(ctx context.Context) *AgentState {
        return &AgentState{}
    }),
)

// 添加状态处理器
g.AddLambdaNode("node1", lambda,
    compose.WithStatePreHandler(func(ctx context.Context, in string, state *AgentState) (string, error) {
        state.Messages = append(state.Messages, schema.UserMessage(in))
        return in, nil
    }),
    compose.WithStatePostHandler(func(ctx context.Context, out string, state *AgentState) (string, error) {
        // 从状态读取数据
        return out + state.ToolCallID, nil
    }),
)

// 在 Lambda 内部处理状态
func processWithState(ctx context.Context, input string) (string, error) {
    var result string
    err := compose.ProcessState[*AgentState](ctx, func(ctx context.Context, state *AgentState) error {
        state.ToolCallID = "123"
        result = "处理完成"
        return nil
    })
    return result, err
}
```

### 运行模式

```go
// 四种运行模式
runnable, _ := g.Compile(ctx)

// 1. Invoke - 输入非流，输出非流
result, _ := runnable.Invoke(ctx, input)

// 2. Stream - 输入非流，输出流
stream, _ := runnable.Stream(ctx, input)

// 3. Collect - 输入流，输出非流
sr, _ := schema.Pipe[map[string]any](1)
sr.Send(input, nil)
sr.Close()
result, _ := runnable.Collect(ctx, sr)

// 4. Transform - 输入流，输出流
inputSR, _ := schema.Pipe[map[string]any](1)
inputSR.Send(input, nil)
inputSR.Close()
outputSR, _ := runnable.Transform(ctx, inputSR)
```

## Workflow 编排

Workflow 支持字段级数据映射，适合结构化输入输出。

### 基本用法

```go
// 创建 Workflow
wf := compose.NewWorkflow[InputType, OutputType]()

// 添加节点
wf.AddChatModelNode("model", model)
wf.AddLambdaNode("process", lambda)

// 添加输入映射
wf.AddInput(compose.START,
    compose.MapFields("Field1", "ModelInput"),
    compose.MapFields("Field2", "ModelInput2"),
)

// 节点间映射
wf.AddLambdaNode("l1", lambda1).
    AddInput("model", compose.MapFields("Content", "Input"))

wf.AddLambdaNode("l2", lambda2).
    AddInput("model", compose.MapFields("Role", "Role"))

// 多输入汇聚
wf.AddLambdaNode("l3", lambda3).
    AddInput("l1", compose.MapFields("Output", "Query")).
    AddInput("l2", compose.MapFields("Output", "Meta"))

// 结束节点
wf.AddEnd("l3")

// 编译运行
runnable, _ := wf.Compile(ctx)
result, _ := runnable.Invoke(ctx, input)
```

## 编排模式选择

| 场景 | 推荐编排 | 原因 |
|------|---------|------|
| 简单 LLM 调用 | Chain | 线性流程，简单直接 |
| 带 Tool 的 Agent | Graph | 需要条件分支判断是否调用工具 |
| 多轮对话 | Graph | 需要状态管理保存历史 |
| 并行处理多个任务 | Chain + Parallel | 简单的并行执行 |
| 复杂业务流程 | Graph | 完整的控制流能力 |
| 结构化数据转换 | Workflow | 字段级精确映射 |

## 编译选项

```go
// 最大运行步数（防止死循环）
runnable, err := g.Compile(ctx, compose.WithMaxRunSteps(100))

// 调试模式
runnable, err := g.Compile(ctx, compose.WithDebug(true))

// 自定义中间件
runnable, err := g.Compile(ctx, compose.WithMiddleware(myMiddleware))
```
