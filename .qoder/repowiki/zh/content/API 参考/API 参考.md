# API 参考

<cite>
**本文档中引用的文件**   
- [mcp.go](file://mcp/mcp.go)
- [server.go](file://mcp/server.go)
- [client.go](file://mcp/client.go)
- [tool.go](file://mcp/tool.go)
- [resource.go](file://mcp/resource.go)
- [prompt.go](file://mcp/prompt.go)
- [event.go](file://mcp/event.go)
- [protocol.go](file://mcp/protocol.go)
- [transport.go](file://mcp/transport.go)
- [shared.go](file://mcp/shared.go)
</cite>

## 目录
1. [简介](#简介)
2. [服务器与客户端初始化](#服务器与客户端初始化)
3. [核心数据模型](#核心数据模型)
4. [服务器功能](#服务器功能)
5. [客户端功能](#客户端功能)
6. [事件与通知](#事件与通知)
7. [传输机制](#传输机制)
8. [并发安全性](#并发安全性)

## 简介
本API参考文档系统性地记录了Go MCP SDK的所有公开接口。文档覆盖了服务器和客户端的初始化、数据模型、功能方法、事件机制和传输选项。所有API都遵循Go语言标准注释格式，明确标注参数类型、返回值、错误条件和并发安全性。每个API都提供了简短的调用示例。

## 服务器与客户端初始化

### mcp.NewServer
创建一个新的MCP服务器实例。服务器用于暴露MCP功能并处理一个或多个MCP会话。

**参数约束与默认值**:
- `impl`: 实现信息，必须不为nil
- `options`: 服务器选项，可为nil

当`options`为nil时，使用以下默认值：
- `PageSize`: 默认为`DefaultPageSize`（1000）
- `GetSessionID`: 使用随机文本生成器
- `Logger`: 使用丢弃日志处理器

如果`SubscribeHandler`被设置，则`UnsubscribeHandler`也必须被设置，反之亦然。

**调用示例**:
```go
server := mcp.NewServer(&mcp.Implementation{Name: "greeter"}, nil)
```

**Section sources**
- [server.go](file://mcp/server.go#L109-L150)

### mcp.NewClient
创建一个新的MCP客户端实例。客户端用于连接到MCP服务器。

**参数约束与默认值**:
- `impl`: 实现信息，必须不为nil
- `opts`: 客户端选项，可为nil

当`opts`为nil时，所有选项使用零值。

**调用示例**:
```go
client := mcp.NewClient(&mcp.Implementation{Name: "mcp-client"}, nil)
```

**Section sources**
- [client.go](file://mcp/client.go#L39-L53)

### ServerOptions
服务器选项结构体，用于配置服务器行为。

**字段语义**:
- `Instructions`: 发送给连接客户端的可选说明
- `Logger`: 用于记录服务器活动的日志记录器
- `InitializedHandler`: 当收到"notifications/initialized"时调用
- `PageSize`: 列表方法（如ListTools）单页返回的最大项目数
- `KeepAlive`: 定期"ping"请求的间隔时间
- `SubscribeHandler`: 客户端会话订阅资源时调用的函数
- `UnsubscribeHandler`: 客户端会话取消订阅资源时调用的函数
- `HasPrompts`: 是否在初始化期间广告提示功能
- `HasResources`: 是否在初始化期间广告资源功能
- `HasTools`: 是否在初始化期间广告工具功能
- `GetSessionID`: 提供下一个会话ID的函数

**并发安全性**: 此结构体在服务器创建后被视为不可变。

**Section sources**
- [server.go](file://mcp/server.go#L54-L99)

### ClientOptions
客户端选项结构体，用于配置客户端行为。

**字段语义**:
- `CreateMessageHandler`: 处理sampling/createMessage请求
- `ElicitationHandler`: 处理elicitation/create请求
- `ToolListChangedHandler`: 处理来自服务器的工具列表更改通知
- `PromptListChangedHandler`: 处理来自服务器的提示列表更改通知
- `ResourceListChangedHandler`: 处理来自服务器的资源列表更改通知
- `ResourceUpdatedHandler`: 处理来自服务器的资源更新通知
- `LoggingMessageHandler`: 处理来自服务器的日志消息通知
- `ProgressNotificationHandler`: 处理来自服务器的进度通知
- `KeepAlive`: 定期"ping"请求的间隔时间

**并发安全性**: 此结构体在客户端创建后被视为不可变。

**Section sources**
- [client.go](file://mcp/client.go#L56-L78)

## 核心数据模型

### Tool
工具数据模型，表示可调用的功能。

**字段语义**:
- `Meta`: 通用元数据字段
- `Annotations`: 工具的可选附加信息
- `Description`: 工具的人类可读描述
- `InputSchema`: 定义工具预期参数的JSON Schema对象
- `Name`: 程序化或逻辑使用的名称
- `OutputSchema`: 定义工具输出结构的JSON Schema对象
- `Title`: 优化为人类可读的UI显示名称

**并发安全性**: 此结构体在添加到服务器后不应被修改。

**Section sources**
- [protocol.go](file://mcp/protocol.go#L900-L950)

### Resource
资源数据模型，表示可访问的数据资源。

**字段语义**:
- `Meta`: 通用元数据字段
- `Annotations`: 客户端的可选注释
- `Description`: 资源的描述
- `MIMEType`: 资源的MIME类型
- `Name`: 程序化或逻辑使用的名称
- `Size`: 原始资源内容的大小（字节）
- `Title`: 优化为人类可读的UI显示名称
- `URI`: 资源的URI

**并发安全性**: 此结构体在添加到服务器后不应被修改。

**Section sources**
- [protocol.go](file://mcp/protocol.go#L753-L784)

### Prompt
提示数据模型，表示可获取的提示信息。

**字段语义**:
- `Meta`: 通用元数据字段
- `Arguments`: 用于模板化提示的参数列表
- `Description`: 提示提供的可选描述
- `Name`: 程序化或逻辑使用的名称
- `Title`: 优化为人类可读的UI显示名称

**并发安全性**: 此结构体在添加到服务器后不应被修改。

**Section sources**
- [protocol.go](file://mcp/protocol.go#L661-L675)

## 服务器功能

### Server 结构体
服务器实例，用于暴露服务器端MCP功能。

**字段语义**:
- `impl`: 创建时固定的实现信息
- `opts`: 服务器选项
- `prompts`: 提示功能集合
- `tools`: 工具功能集合
- `resources`: 资源功能集合
- `sessions`: 当前服务器会话列表

**并发安全性**: 所有方法都是并发安全的。

**Section sources**
- [server.go](file://mcp/server.go#L37-L51)

### Server.AddTool
将工具添加到服务器，或替换同名的现有工具。

**参数约束**:
- `t`: 工具定义，输入模式必须非nil且类型为"object"
- `h`: 工具处理器

**错误条件**:
- 如果输入模式为nil，会panic
- 如果输入或输出模式类型不是"object"，会panic

**调用示例**:
```go
server.AddTool(&mcp.Tool{Name: "greet"}, func(ctx context.Context, req *mcp.CallToolRequest) (*mcp.CallToolResult, error) {
    return &mcp.CallToolResult{Content: []mcp.Content{&mcp.TextContent{Text: "Hello"}}}, nil
})
```

**Section sources**
- [server.go](file://mcp/server.go#L191-L234)

### Server.AddPrompt
将提示添加到服务器，或替换同名的现有提示。

**调用示例**:
```go
server.AddPrompt(&mcp.Prompt{Name: "welcome"}, func(ctx context.Context, req *mcp.GetPromptRequest) (*mcp.GetPromptResult, error) {
    return &mcp.GetPromptResult{Content: "Welcome!"}, nil
})
```

**Section sources**
- [server.go](file://mcp/server.go#L153-L160)

### Server.AddResource
将资源添加到服务器，或替换同URI的现有资源。

**参数约束**:
- `r`: 资源定义，URI必须有效且为绝对URI
- `h`: 资源处理器

**错误条件**:
- 如果资源URI无效，会panic

**调用示例**:
```go
server.AddResource(&mcp.Resource{URI: "file:///data.txt"}, func(ctx context.Context, req *mcp.ReadResourceRequest) (*mcp.ReadResourceResult, error) {
    return &mcp.ReadResourceResult{Contents: []*mcp.ResourceContents{{URI: req.Params.URI, Text: "content"}}}, nil
})
```

**Section sources**
- [server.go](file://mcp/server.go#L422-L431)

### Server.AddResourceTemplate
将资源模板添加到服务器，或替换同URI模板的现有模板。

**参数约束**:
- `t`: 资源模板定义，URI模板必须有效且为绝对URI
- `h`: 资源处理器

**错误条件**:
- 如果URI模板无效，会panic

**调用示例**:
```go
server.AddResourceTemplate(&mcp.ResourceTemplate{URITemplate: "file:///data/{id}.txt"}, fileResourceHandler("/data"))
```

**Section sources**
- [server.go](file://mcp/server.go#L442-L453)

## 客户端功能

### Client 结构体
客户端实例，用于连接到MCP服务器。

**字段语义**:
- `impl`: 实现信息
- `opts`: 客户端选项
- `roots`: 根目录集合
- `sessions`: 当前客户端会话列表

**并发安全性**: 所有方法都是并发安全的。

**Section sources**
- [client.go](file://mcp/client.go#L22-L30)

### Client.AddRoots
将根目录添加到客户端，替换同URI的现有根目录，并通知任何连接的服务器。

**调用示例**:
```go
client.AddRoots(&mcp.Root{URI: "file:///project"})
```

**Section sources**
- [client.go](file://mcp/client.go#L239-L246)

### Client.AddSendingMiddleware
包装当前发送方法处理器，使用提供的中间件。

**调用示例**:
```go
client.AddSendingMiddleware(tracingMiddleware, metricsMiddleware)
```

**Section sources**
- [client.go](file://mcp/client.go#L470-L474)

### Client.AddReceivingMiddleware
包装当前接收方法处理器，使用提供的中间件。

**调用示例**:
```go
client.AddReceivingMiddleware(authMiddleware, loggingMiddleware)
```

**Section sources**
- [client.go](file://mcp/client.go#L485-L489)

## 事件与通知

### Event
服务器发送事件，遵循服务器发送事件规范。

**字段语义**:
- `Name`: "event"字段
- `ID`: "id"字段
- `Data`: "data"字段

**并发安全性**: 此结构体是不可变的。

**Section sources**
- [event.go](file://mcp/event.go#L30-L34)

### MemoryEventStore
基于内存的事件存储，用于SSE流。

**方法**:
- `Open`: 打开新流
- `Append`: 追加数据到指定流
- `After`: 返回指定索引后的数据迭代器
- `SessionClosed`: 通知存储指定会话已结束

**并发安全性**: 所有方法都是并发安全的。

**Section sources**
- [event.go](file://mcp/event.go#L300-L350)

## 传输机制

### Transport 接口
用于在MCP客户端和服务器之间创建双向连接的传输机制。

**方法**:
- `Connect`: 返回逻辑JSON-RPC连接

**并发安全性**: `Connect`方法应只调用一次。

**Section sources**
- [transport.go](file://mcp/transport.go#L20-L25)

### StdioTransport
通过stdin/stdout使用换行分隔JSON进行通信的传输。

**调用示例**:
```go
server.Run(ctx, &mcp.StdioTransport{})
```

**Section sources**
- [transport.go](file://mcp/transport.go#L70-L75)

### InMemoryTransport
通过内存网络连接使用换行分隔JSON进行通信的传输。

**调用示例**:
```go
clientTransport, serverTransport := mcp.NewInMemoryTransports()
```

**Section sources**
- [transport.go](file://mcp/transport.go#L90-L105)

## 并发安全性
SDK中的所有公共API都是并发安全的。服务器和客户端实例可以安全地从多个goroutine中调用。会话方法也是并发安全的，可以同时从多个goroutine中调用。数据模型结构体在添加到服务器或客户端后不应被修改，以确保一致性。