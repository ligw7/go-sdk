# 客户端 API

<cite>
**本文档中引用的文件**  
- [client.go](file://mcp/client.go)
- [shared.go](file://mcp/shared.go)
- [protocol.go](file://mcp/protocol.go)
- [transport.go](file://mcp/transport.go)
- [features.go](file://mcp/features.go)
</cite>

## 目录
1. [初始化客户端](#初始化客户端)
2. [ClientOptions 配置项](#clientoptions-配置项)
3. [Client 结构体方法](#client-结构体方法)
4. [ClientSession 会话方法](#clientsession-会话方法)
5. [并发安全性](#并发安全性)
6. [实际调用示例](#实际调用示例)

## 初始化客户端

`mcp.NewClient` 函数用于创建一个新的 MCP 客户端实例。该函数接受两个参数：`*Implementation` 和 `*ClientOptions`。`Implementation` 参数必须非空，否则会触发 panic。`ClientOptions` 参数可选，用于配置客户端的行为。

**Section sources**
- [client.go](file://mcp/client.go#L39-L53)

## ClientOptions 配置项

`ClientOptions` 结构体定义了客户端的行为配置。主要配置项包括：
- `CreateMessageHandler`：处理来自服务器的 sampling/createMessage 请求。
- `ElicitationHandler`：处理来自服务器的 elicitation/create 请求。
- `ToolListChangedHandler`、`PromptListChangedHandler`、`ResourceListChangedHandler`：处理来自服务器的通知。
- `KeepAlive`：定义定期“ping”请求的时间间隔。如果对端未能响应 ping 请求，会话将自动关闭。

**Section sources**
- [client.go](file://mcp/client.go#L56-L78)

## Client 结构体方法

### Connect 方法
`Connect` 方法用于连接到 MCP 服务器，创建一个新的会话。该方法接受一个 `Transport` 参数，用于创建与服务器的双向连接。

**Section sources**
- [client.go](file://mcp/client.go#L135-L170)

### AddRoots 方法
`AddRoots` 方法用于向客户端添加根目录，替换具有相同 URI 的任何现有根目录，并通知任何连接的服务器。

**Section sources**
- [client.go](file://mcp/client.go#L239-L246)

## ClientSession 会话方法

### CallTool 方法
`CallTool` 方法用于调用服务器上的工具。参数 `CallToolParams` 包含工具名称和参数。返回值为 `*CallToolResult` 和错误。

**Section sources**
- [client.go](file://mcp/client.go#L582-L591)

### ReadResource 方法
`ReadResource` 方法用于请求服务器读取资源并返回其内容。参数 `ReadResourceParams` 包含资源的 URI。

**Section sources**
- [client.go](file://mcp/client.go#L609-L611)

### Subscribe 方法
`Subscribe` 方法用于发送 "resources/subscribe" 请求到服务器，请求在指定资源更改时收到通知。

**Section sources**
- [client.go](file://mcp/client.go#L619-L622)

## 并发安全性

客户端和会话方法是线程安全的，可以在多个 goroutine 中并发调用。内部使用互斥锁保护共享状态。

**Section sources**
- [client.go](file://mcp/client.go#L22-L30)

## 实际调用示例

以下示例展示了如何使用 `mcp.NewClient` 和 `Client.Connect` 方法连接到 MCP 服务器，并调用工具：

```go
client := mcp.NewClient(&mcp.Implementation{Name: "mcp-client", Version: "v1.0.0"}, nil)
transport := &mcp.CommandTransport{Command: exec.Command("myserver")}
session, err := client.Connect(ctx, transport, nil)
if err != nil {
    log.Fatal(err)
}
defer session.Close()

params := &mcp.CallToolParams{
    Name:      "greet",
    Arguments: map[string]any{"name": "you"},
}
res, err := session.CallTool(ctx, params)
if err != nil {
    log.Fatalf("CallTool failed: %v", err)
}
```

**Section sources**
- [mcp.go](file://mcp/mcp.go#L0-L88)