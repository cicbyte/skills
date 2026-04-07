# Eino 流式处理详解

Eino 提供完善的流式处理能力，支持流式输入、流式输出和流转换。

## StreamReader 基础

StreamReader 是 Eino 的流式数据抽象。

### 创建流

```go
import "github.com/cloudwego/eino/schema"

// 创建管道流
sr, sw := schema.Pipe[string](bufferSize)

// 发送数据
sw.Send("chunk1", nil)
sw.Send("chunk2", nil)
sw.Close()  // 发送结束

// 接收数据
for {
    chunk, err := sr.Recv()
    if err == io.EOF {
        break
    }
    if err != nil {
        // 处理错误
        break
    }
    fmt.Println(chunk)
}
```

### 流的类型

```go
// 只读流
type StreamReader[T any] interface {
    Recv() (T, error)
    Close() error
}

// 只写流
type StreamWriter[T any] interface {
    Send(data T, err error) error
    Close() error
}
```

## ChatModel 流式调用

### 流式输出

```go
// Stream 模式
stream, err := chatModel.Stream(ctx, messages)
if err != nil {
    return err
}
defer stream.Close()

for {
    chunk, err := stream.Recv()
    if err == io.EOF {
        break
    }
    if err != nil {
        return err
    }

    // 处理每个 chunk
    fmt.Print(chunk.Content)
}
```

### 流式和非流式转换

```go
// 非流式转流式
sr, sw := schema.Pipe[*schema.Message](0)
go func() {
    defer sw.Close()
    msg, _ := chatModel.Generate(ctx, messages)
    sw.Send(msg, nil)
}()

// 流式转非流式（自动拼接）
messages, err := schema.StreamReaderToSlice(stream)
```

## 编排中的流处理

Eino 编排框架自动处理流的拼接、合并、复制。

### 自动流拼接

当下游节点只接受非流式输入时，Eino 自动拼接流。

```go
// ChatModel 输出流 -> Lambda 只接受非流
// Eino 自动拼接所有 chunk 后传给 Lambda

g := compose.NewGraph[string, string]()
g.AddChatModelNode("model", chatModel)  // 输出流
g.AddLambdaNode("process", lambda)      // 只接受非流
g.AddEdge("model", "process")  // 自动拼接
```

### 自动流合并

当多个流汇聚到一个节点时，Eino 自动合并。

```go
// 并行节点产生多个流
parallel := compose.NewParallel()
parallel.AddLambda("p1", streamableLambda1)
parallel.AddLambda("p2", streamableLambda2)

chain.AppendParallel(parallel)
// Eino 自动合并 p1 和 p2 的流
```

### 自动流复制

当一个流传入多个下游节点时，Eino 自动复制。

```go
g.AddChatModelNode("model", chatModel)
g.AddEdge("model", "node1")  // 复制流到 node1
g.AddEdge("model", "node2")  // 复制流到 node2
// Eino 自动复制流到多个节点
```

## 四种运行模式

编译后的 Runnable 支持四种流式运行模式。

### 1. Invoke - 非流输入 → 非流输出

```go
runnable, _ := chain.Compile(ctx)

result, err := runnable.Invoke(ctx, input)
```

### 2. Stream - 非流输入 → 流输出

```go
stream, err := runnable.Stream(ctx, input)
if err != nil {
    return err
}
defer stream.Close()

for {
    chunk, err := stream.Recv()
    if err == io.EOF {
        break
    }
    // 处理 chunk
}
```

### 3. Collect - 流输入 → 非流输出

```go
// 创建输入流
sr, sw := schema.Pipe[InputType](bufferSize)
go func() {
    defer sw.Close()
    sw.Send(input1, nil)
    sw.Send(input2, nil)
}()

// 收集流并返回非流输出
result, err := runnable.Collect(ctx, sr)
```

### 4. Transform - 流输入 → 流输出

```go
// 创建输入流
inputSR, sw := schema.Pipe[InputType](bufferSize)
go func() {
    defer sw.Close()
    for _, item := range inputs {
        sw.Send(item, nil)
    }
}()

// 流式转换
outputSR, err := runnable.Transform(ctx, inputSR)
defer outputSR.Close()

for {
    chunk, err := outputSR.Recv()
    if err == io.EOF {
        break
    }
    // 处理转换后的 chunk
}
```

## 流式 Lambda

创建支持流式处理的 Lambda 节点。

### Streamable Lambda

```go
// 输入非流，输出流
lambda := compose.StreamableLambda(func(ctx context.Context, input string) (*schema.StreamReader[string], error) {
    sr, sw := schema.Pipe[string](10)
    go func() {
        defer sw.Close()
        // 产生流式输出
        words := strings.Split(input, " ")
        for _, word := range words {
            sw.Send(word+" ", nil)
        }
    }()
    return sr, nil
})
```

### Transformable Lambda

```go
// 输入流，输出流（逐块转换）
lambda := compose.TransformableLambda(func(ctx context.Context, input *schema.StreamReader[string]) (*schema.StreamReader[string], error) {
    sr, sw := schema.Pipe[string](0)
    go func() {
        defer sw.Close()
        for {
            chunk, err := input.Recv()
            if err == io.EOF {
                break
            }
            if err != nil {
                sw.Send("", err)
                break
            }
            // 转换每个 chunk
            sw.Send(strings.ToUpper(chunk), nil)
        }
    }()
    return sr, nil
})
```

### 流式 Agent 示例

```go
// 创建流式 Agent
g := compose.NewGraph[map[string]any, *schema.Message]()

// ChatModel 输出流
g.AddChatModelNode("model", chatModel)

// 流式 Lambda 处理
g.AddLambdaNode("processor", compose.StreamableLambda(
    func(ctx context.Context, msg *schema.Message) (*schema.StreamReader[string], error) {
        sr, sw := schema.Pipe[string](100)
        go func() {
            defer sw.Close()
            // 流式处理消息内容
            for _, char := range msg.Content {
                sw.Send(string(char), nil)
            }
        }()
        return sr, nil
    },
))

g.AddEdge("model", "processor")
g.AddEdge("processor", compose.END)

agent, _ := g.Compile(ctx)

// 流式输出
stream, _ := agent.Stream(ctx, input)
defer stream.Close()

for {
    chunk, err := stream.Recv()
    if err == io.EOF {
        break
    }
    fmt.Print(chunk)  // 逐字符输出
}
```

## 流式处理最佳实践

### 1. 总是关闭流

```go
stream, err := chatModel.Stream(ctx, messages)
if err != nil {
    return err
}
defer stream.Close()  // 确保关闭
```

### 2. 正确处理 EOF

```go
for {
    chunk, err := stream.Recv()
    if errors.Is(err, io.EOF) {
        // 正常结束
        break
    }
    if err != nil {
        // 错误
        return err
    }
    // 处理 chunk
}
```

### 3. 流式输出时考虑缓冲

```go
// 根据场景设置合适的缓冲区大小
sr, sw := schema.Pipe[*schema.Message](100)  // 缓冲100条消息
```

### 4. 使用 goroutine 处理流

```go
go func() {
    defer stream.Close()
    for {
        chunk, err := stream.Recv()
        if err == io.EOF {
            break
        }
        if err != nil {
            log.Printf("stream error: %v", err)
            break
        }
        // 异步处理 chunk
        processChunk(chunk)
    }
}()
```

## 流式 Callback

```go
import "github.com/cloudwego/eino/callbacks"

handler := callbacks.NewHandlerBuilder().
    OnStartWithStreamInputFn(func(ctx context.Context, info *callbacks.RunInfo, input *schema.StreamReader[callbacks.CallbackInput]) context.Context {
        // 处理流式输入
        return ctx
    }).
    OnEndWithStreamOutputFn(func(ctx context.Context, info *callbacks.RunInfo, output *schema.StreamReader[callbacks.CallbackOutput]) context.Context {
        // 处理流式输出
        return ctx
    }).
    Build()

// 使用
runnable.Invoke(ctx, input, compose.WithCallbacks(handler))
```
