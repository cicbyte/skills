# Eino 组件详解

## ChatModel

ChatModel 是 Eino 对对话大模型的统一抽象接口。

### 支持的模型实现

```go
// OpenAI (GPT-3.5/GPT-4)
import "github.com/cloudwego/eino-ext/components/model/openai"

chatModel, err := openai.NewChatModel(ctx, &openai.ChatModelConfig{
    Model:   "gpt-4o",
    APIKey:  os.Getenv("OPENAI_API_KEY"),
    BaseURL: "https://api.openai.com/v1",
})

// Ollama (本地开源模型)
import "github.com/cloudwego/eino-ext/components/model/ollama"

chatModel, err := ollama.NewChatModel(ctx, &ollama.ChatModelConfig{
    BaseURL: "http://localhost:11434",
    Model:   "llama2",
})

// ARK (火山引擎/豆包)
import "github.com/cloudwego/eino-ext/components/model/ark"

chatModel, err := ark.NewChatModel(ctx, &ark.ChatModelConfig{
    Model:  "ep-2024xxx",
    APIKey: os.Getenv("ARK_API_KEY"),
})
```

### 运行模式

```go
// 同步模式 - 返回完整消息
message, err := chatModel.Generate(ctx, messages)

// 流式模式 - 返回消息流
stream, err := chatModel.Stream(ctx, messages)
for {
    chunk, err := stream.Recv()
    if err == io.EOF {
        break
    }
    fmt.Println(chunk.Content)
}
stream.Close()
```

### 绑定工具

```go
// 获取工具信息
toolInfos := []*schema.ToolInfo{info1, info2}

// 绑定到 ChatModel
err := chatModel.BindTools(toolInfos)

// 强制使用工具（Agent 必须调用）
err := chatModel.BindForcedTools(toolInfos)
```

## Tool

Tool 是 Agent 的执行器，提供了具体的功能实现。

### 创建 Tool 的三种方式

#### 方式一：InferTool（推荐）

```go
import "github.com/cloudwego/eino/components/tool/utils"

// 参数结构体
type WeatherParams struct {
    City string `json:"city" jsonschema:"description=城市名称"`
}

// 处理函数
func GetWeather(ctx context.Context, params *WeatherParams) (string, error) {
    return fmt.Sprintf("%s 的天气是晴天", params.City), nil
}

// 创建 Tool
weatherTool, err := utils.InferTool(
    "get_weather",
    "查询城市天气",
    GetWeather,
)
```

#### 方式二：NewTool

```go
info := &schema.ToolInfo{
    Name: "search",
    Desc: "搜索信息",
    ParamsOneOf: schema.NewParamsOneOfByParams(map[string]*schema.ParameterInfo{
        "query": {
            Type:     schema.String,
            Desc:     "搜索关键词",
            Required: true,
        },
    }),
}

searchTool := utils.NewTool(info, func(ctx context.Context, params map[string]any) (string, error) {
    return "搜索结果", nil
})
```

#### 方式三：实现 Tool 接口

```go
type CustomTool struct{}

func (t *CustomTool) Info(ctx context.Context) (*schema.ToolInfo, error) {
    return &schema.ToolInfo{
        Name: "custom_tool",
        Desc: "自定义工具",
    }, nil
}

func (t *CustomTool) InvokableRun(ctx context.Context, args string, opts ...tool.Option) (string, error) {
    return "执行结果", nil
}
```

### 官方工具

```go
// DuckDuckGo 搜索
import "github.com/cloudwego/eino-ext/components/tool/duckduckgo"
searchTool, _ := duckduckgo.NewTool(ctx, &duckduckgo.Config{})

// Wikipedia 查询
import "github.com/cloudwego/eino-ext/components/tool/wikipedia"
wikiTool, _ := wikipedia.NewTool(ctx, &wikipedia.Config{})
```

## ChatTemplate

ChatTemplate 提供模板化功能来构建消息。

### 模板格式

```go
import "github.com/cloudwego/eino/components/prompt"
import "github.com/cloudwego/eino/schema"

// FString 格式（推荐）
template := prompt.FromMessages(schema.FString,
    schema.SystemMessage("你是{role}，用{style}风格回答"),
    schema.MessagesPlaceholder("history", false), // 可选的对话历史
    schema.UserMessage("问题: {question}"),
)

// Jinja2 格式
template := prompt.FromMessages(schema.Jinja2,
    schema.SystemMessage("你是{{role}}"),
    schema.UserMessage("{{question}}"),
)

// Go Template 格式
template := prompt.FromMessages(schema.GoTemplate,
    schema.SystemMessage("你是{{.Role}}"),
    schema.UserMessage("{{.Question}}"),
)
```

### 使用模板

```go
messages, err := template.Format(ctx, map[string]any{
    "role":     "AI助手",
    "style":    "专业",
    "question": "什么是Eino？",
    "history":  []*schema.Message{...}, // 对话历史
})
```

## Document Loader

文档加载器用于读取和解析各种文档。

```go
// 本地文件
import "github.com/cloudwego/eino-ext/components/document_loader/localfile"
loader, _ := localfile.NewLoader(ctx, &localfile.Config{
    Paths: []string{"./docs/*.md"},
})

// Web URL
import "github.com/cloudwego/eino-ext/components/document_loader/weburl"
loader, _ := weburl.NewLoader(ctx, &weburl.Config{
    URLs: []string{"https://example.com"},
})
```

## Embedding

文本嵌入向量模型。

```go
// OpenAI Embedding
import "github.com/cloudwego/eino-ext/components/embedding/openai"
embedder, _ := openai.NewEmbedder(ctx, &openai.Config{
    Model:  "text-embedding-3-small",
    APIKey: os.Getenv("OPENAI_API_KEY"),
})

// 生成向量
vec, err := embedder.EmbedStrings(ctx, []string{"Hello world"})
```

## Indexer & Retriever

向量存储和检索。

```go
// Redis
import "github.com/cloudwego/eino-ext/components/indexer/redis"
indexer, _ := redis.NewIndexer(ctx, &redis.Config{
    Addr:     "localhost:6379",
    Password: "",
    Index:    "my_docs",
})

// 检索
import "github.com/cloudwego/eino-ext/components/retriever/redis"
retriever, _ := redis.NewRetriever(ctx, redis.NewConfig())
docs, err := retriever.Retrieve(ctx, "查询问题")
```
