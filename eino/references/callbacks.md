# Eino 回调机制详解

Eino 的回调（Callback）机制提供切面注入能力，用于日志记录、追踪、指标统计等横切面关注点。

## Callback Handler

CallbackHandler 是回调的核心接口。

### 五种回调时机

```go
import "github.com/cloudwego/eino/callbacks"

type CallbackHandler interface {
    // 1. 开始执行（非流式输入）
    OnStart(ctx context.Context, info *RunInfo, input CallbackInput) context.Context

    // 2. 结束执行（非流式输出）
    OnEnd(ctx context.Context, info *RunInfo, output CallbackOutput) context.Context

    // 3. 执行出错
    OnError(ctx context.Context, info *RunInfo, err error) context.Context

    // 4. 开始执行（流式输入）
    OnStartWithStreamInput(ctx context.Context, info *RunInfo, input *StreamReader[CallbackInput]) context.Context

    // 5. 结束执行（流式输出）
    OnEndWithStreamOutput(ctx context.Context, info *RunInfo, output *StreamReader[CallbackOutput]) context.Context
}
```

### RunInfo 信息

```go
type RunInfo struct {
    Name      string      // 节点名称
    Type      string      // 节点类型
    Component Component   // 组件类型
    TraceID   string      // 追踪ID
    Parent    *RunInfo    // 父节点信息
}
```

## 创建 Callback Handler

### 使用 Builder（推荐）

```go
handler := callbacks.NewHandlerBuilder().
    OnStartFn(func(ctx context.Context, info *callbacks.RunInfo, input callbacks.CallbackInput) context.Context {
        log.Printf("[%s] Started: %+v", info.Name, input)
        return ctx
    }).
    OnEndFn(func(ctx context.Context, info *callbacks.RunInfo, output callbacks.CallbackOutput) context.Context {
        log.Printf("[%s] Ended: %+v", info.Name, output)
        return ctx
    }).
    OnErrorFn(func(ctx context.Context, info *callbacks.RunInfo, err error) context.Context {
        log.Printf("[%s] Error: %v", info.Name, err)
        return ctx
    }).
    Build()
```

### 实现 Handler 接口

```go
type LoggerCallbacks struct{}

func (l *LoggerCallbacks) OnStart(ctx context.Context, info *callbacks.RunInfo, input callbacks.CallbackInput) context.Context {
    log.Printf("Start: %s (%s)", info.Name, info.Type)
    return ctx
}

func (l *LoggerCallbacks) OnEnd(ctx context.Context, info *callbacks.RunInfo, output callbacks.CallbackOutput) context.Context {
    log.Printf("End: %s", info.Name)
    return ctx
}

func (l *LoggerCallbacks) OnError(ctx context.Context, info *callbacks.RunInfo, err error) context.Context {
    log.Printf("Error: %s - %v", info.Name, err)
    return ctx
}

func (l *LoggerCallbacks) OnStartWithStreamInput(ctx context.Context, info *callbacks.RunInfo, input *schema.StreamReader[callbacks.CallbackInput]) context.Context {
    return ctx
}

func (l *LoggerCallbacks) OnEndWithStreamOutput(ctx context.Context, info *callbacks.RunInfo, output *schema.StreamReader[callbacks.CallbackOutput]) context.Context {
    return ctx
}
```

## 使用 Callback

### 全局 Callback

```go
// 添加全局处理器，影响所有调用
callbacks.AppendGlobalHandlers(&LoggerCallbacks{})

// 之后的所有调用都会触发回调
runnable.Invoke(ctx, input)
```

### Runnable 级别

```go
// 为特定 Runnable 添加 Callback
handler := callbacks.NewHandlerBuilder().
    OnStartFn(logStart).
    OnEndFn(logEnd).
    Build()

runnable.Invoke(ctx, input, compose.WithCallbacks(handler))
```

### 节点级别

```go
// 只为特定节点添加 Callback
runnable.Invoke(ctx, input,
    compose.WithCallbacks(handler).
        DesignateNode("node_name"),  // 只应用到这个节点
)
```

### 组件类型级别

```go
// 只为 ChatModel 组件添加 Callback
runnable.Invoke(ctx, input,
    compose.WithCallbacks(handler).
        DesignateComponent(compose.ComponentChatModel),
)
```

### 多个 Callback

```go
// 组合多个 Handler
runnable.Invoke(ctx, input,
    compose.WithCallbacks(loggerHandler),
    compose.WithCallbacks(metricsHandler),
    compose.WithCallbacks(tracingHandler),
)
```

## 常见 Callback 场景

### 1. 日志记录

```go
type LoggingHandler struct {
    Logger *zap.Logger
}

func (h *LoggingHandler) OnStart(ctx context.Context, info *callbacks.RunInfo, input callbacks.CallbackInput) context.Context {
    h.Logger.Info("invoke started",
        zap.String("node", info.Name),
        zap.String("type", string(info.Component)),
    )
    return ctx
}

func (h *LoggingHandler) OnEnd(ctx context.Context, info *callbacks.RunInfo, output callbacks.CallbackOutput) context.Context {
    h.Logger.Info("invoke ended",
        zap.String("node", info.Name),
    )
    return ctx
}
```

### 2. 指标统计

```go
type MetricsHandler struct {
    Counter prometheus.Counter
    Histogram prometheus.Histogram
}

func (h *MetricsHandler) OnStart(ctx context.Context, info *callbacks.RunInfo, input callbacks.CallbackInput) context.Context {
    start := time.Now()
    context.WithValue(ctx, "start_time", start)
    return ctx
}

func (h *MetricsHandler) OnEnd(ctx context.Context, info *callbacks.RunInfo, output callbacks.CallbackOutput) context.Context {
    start := ctx.Value("start_time").(time.Time)
    duration := time.Since(start)

    h.Counter.Inc()
    h.Histogram.Observe(duration.Seconds())
    return ctx
}
```

### 3. 分布式追踪

```go
type TracingHandler struct {
    Tracer trace.Tracer
}

func (h *TracingHandler) OnStart(ctx context.Context, info *callbacks.RunInfo, input callbacks.CallbackInput) context.Context {
    ctx, span := h.Tracer.Start(ctx, info.Name,
        trace.WithAttributes(
            attribute.String("node.type", string(info.Type)),
            attribute.String("node.component", string(info.Component)),
        ),
    )
    context.WithValue(ctx, "span", span)
    return ctx
}

func (h *TracingHandler) OnEnd(ctx context.Context, info *callbacks.RunInfo, output callbacks.CallbackOutput) context.Context {
    span := ctx.Value("span").(trace.Span)
    span.End()
    return ctx
}

func (h *TracingHandler) OnError(ctx context.Context, info *callbacks.RunInfo, err error) context.Context {
    span := ctx.Value("span").(trace.Span)
    span.RecordError(err)
    span.SetStatus(codes.Error, err.Error())
    return ctx
}
```

### 4. 流式输出处理

```go
type StreamOutputHandler struct {
    Writer io.Writer
}

func (h *StreamOutputHandler) OnEndWithStreamOutput(ctx context.Context, info *callbacks.RunInfo, output *schema.StreamReader[callbacks.CallbackOutput]) context.Context {
    go func() {
        defer output.Close()
        for {
            chunk, err := output.Recv()
            if err == io.EOF {
                break
            }
            if err != nil {
                continue
            }
            // 实时处理流式输出
            fmt.Fprintf(h.Writer, "%s", chunk)
        }
    }()
    return ctx
}
```

### 5. 成本追踪

```go
type CostTracker struct {
    ModelCosts map[string]float64  // 模型单价
}

func (h *CostTracker) OnEnd(ctx context.Context, info *callbacks.RunInfo, output callbacks.CallbackOutput) context.Context {
    if info.Component == compose.ComponentChatModel {
        // 计算成本
        tokens := h.estimateTokens(output)
        cost := h.calculateCost(info.Name, tokens)
        logCost(info.Name, cost)
    }
    return ctx
}
```

## Callback 链

多个 Callback 按添加顺序执行。

```go
// 执行顺序：logger -> validator -> transformer
runnable.Invoke(ctx, input,
    compose.WithCallbacks(loggerHandler),
    compose.WithCallbacks(validatorHandler),
    compose.WithCallbacks(transformerHandler),
)
```

## 切面注入

Eino 会自动为不支持回调的组件注入切面。

```go
// 自定义组件不支持 Callback
type CustomModel struct{}

func (m *CustomModel) Generate(ctx context.Context, msgs []*schema.Message, opts ...model.Option) (*schema.Message, error) {
    // 不支持 Callback
}

// Eino 自动注入切面后，Callback 仍然会生效
chain.AppendChatModel(&CustomModel{})
```

## 最佳实践

### 1. 命名你的 Handler

```go
// 为 Handler 添加类型标识
type MyHandler struct{}

func (h *MyHandler) Type() string {
    return "MyHandler"
}
```

### 2. 避免 Panic

```go
func (h *Handler) OnStart(ctx context.Context, info *callbacks.RunInfo, input callbacks.CallbackInput) context.Context {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("callback panic: %v", r)
        }
    }()
    // 回调逻辑
    return ctx
}
```

### 3. 传递 Context

```go
func (h *Handler) OnStart(ctx context.Context, info *callbacks.RunInfo, input callbacks.CallbackInput) context.Context {
    // 向 Context 添加值，供后续回调使用
    ctx = context.WithValue(ctx, "request_id", uuid.New().String())
    return ctx
}
```

### 4. 条件触发

```go
func (h *Handler) OnEnd(ctx context.Context, info *callbacks.RunInfo, output callbacks.CallbackOutput) context.Context {
    // 只处理特定节点
    if info.Name == "important_node" {
        // 特殊处理
    }
    return ctx
}
```

## 官方 Callback 实现

Eino 提供了一些官方的 Callback 实现。

### Langfuse 集成

```go
import "github.com/cloudwego/eino-ext/components/callback/langfuse"

handler, _ := langfuse.NewHandler(&langfuse.Config{
    PublicKey: os.Getenv("LANGFUSE_PUBLIC_KEY"),
    SecretKey: os.Getenv("LANGFUSE_SECRET_KEY"),
})

runnable.Invoke(ctx, input, compose.WithCallbacks(handler))
```

### APMPlus 集成

```go
import "github.com/cloudwego/eino-ext/components/callback/apmplus"

handler := apmplus.NewHandler()

runnable.Invoke(ctx, input, compose.WithCallbacks(handler))
```
