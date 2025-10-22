# 服务器 API

<cite>
**本文档中引用的文件**   
- [server.go](file://mcp/server.go)
- [protocol.go](file://mcp/protocol.go)
- [shared.go](file://mcp/shared.go)
- [tool.go](file://mcp/tool.go)
- [resource.go](file://mcp/resource.go)
- [prompt.go](file://mcp/prompt.go)
</cite>

## 目录
1. [简介](#简介)
2. [初始化与配置](#初始化与配置)
3. [ServerOptions 配置项详解](#serveroptions-配置项详解)
4. [核心方法集](#核心方法集)
5. [并发安全性](#并发安全性)
6. [实际调用示例](#实际调用示例)
7. [错误处理与 Panic 条件](#错误处理与-panic-条件)

## 简介
MCP 服务器 API 提供了一套完整的机制，用于构建和管理 MCP（Model Context Protocol）服务器实例。该 API 的核心是 `mcp.NewServer` 函数，它用于创建服务器实例，并通过 `Server` 结构体暴露一系列公开方法来管理工具、提示和资源等功能。服务器支持通过 `ServerOptions` 进行细粒度配置，包括分页大小、心跳机制、订阅处理程序等，以适应不同的运行时需求。本文档详细说明了服务器的初始化过程、参数约束、默认值配置以及关键方法的调用流程和错误处理机制。

## 初始化与配置
服务器的初始化通过调用 `mcp.NewServer` 函数完成，该函数接收一个 `*Implementation` 和一个可选的 `*ServerOptions` 作为参数。`Implementation` 结构体必须非空，否则会触发 panic。`ServerOptions` 用于配置服务器的运行时行为，如果未提供，则会使用默认值。

**Section sources**
- [server.go](file://mcp/server.go#L109-L150)

## ServerOptions 配置项详解
`ServerOptions` 结构体定义了服务器的各种配置选项，这些选项直接影响服务器的运行时行为。

- **Instructions**: 可选字符串，用于向连接的客户端提供使用说明。
- **Logger**: 指向 `*slog.Logger` 的指针，用于记录服务器活动。如果为 nil，则使用默认日志记录器。
- **InitializedHandler**: 当收到 "notifications/initialized" 请求时调用的函数。
- **PageSize**: 列表方法（如 ListTools）单次返回的最大项目数。如果为零，则默认为 `DefaultPageSize`（1000）。如果为负数，则会触发 panic。
- **KeepAlive**: 定义定期 "ping" 请求的时间间隔。如果对端未能响应心跳检查发起的 ping 请求，会话将自动关闭。
- **SubscribeHandler** 和 **UnsubscribeHandler**: 分别在客户端会话订阅或取消订阅资源时调用的函数。这两个处理程序必须同时存在或同时不存在，否则会触发 panic。
- **HasPrompts**, **HasResources**, **HasTools**: 布尔值，指示即使没有注册相应的功能，是否在初始化期间通告这些能力。
- **GetSessionID**: 提供下一个会话 ID 的函数。如果为 nil，则使用默认的随机生成 ID。

**Section sources**
- [server.go](file://mcp/server.go#L54-L99)

## 核心方法集
`Server` 结构体提供了一系列公开方法，用于动态管理服务器的功能。

### AddTool 方法
`AddTool` 方法用于向服务器添加或替换一个工具。该方法对输入和输出 Schema 有严格的类型检查：输入 Schema 必须非空且类型为 "object"；如果存在输出 Schema，其类型也必须为 "object"。如果验证失败，会触发 panic。

**Section sources**
- [server.go](file://mcp/server.go#L191-L234)

### AddPrompt 方法
`AddPrompt` 方法用于向服务器添加或替换一个提示。该方法没有额外的参数验证，直接将提示和处理程序添加到服务器中。

**Section sources**
- [server.go](file://mcp/server.go#L153-L160)

### AddResource 方法
`AddResource` 方法用于向服务器添加或替换一个资源。该方法会验证资源 URI 的有效性，如果 URI 无效或不是绝对路径（scheme 为空），则会触发 panic。

**Section sources**
- [server.go](file://mcp/server.go#L422-L431)

### RemoveTools 方法
`RemoveTools` 方法用于移除指定名称的工具。如果要移除的工具不存在，不会产生错误。

**Section sources**
- [server.go](file://mcp/server.go#L415-L418)

### Sessions 方法
`Sessions` 方法返回一个迭代器，用于遍历当前服务器会话的集合。该方法保证在迭代过程中不会观察到会话的添加或移除。

**Section sources**
- [server.go](file://mcp/server.go#L512-L517)

## 并发安全性
`Server` 结构体内部使用互斥锁（`sync.Mutex`）来保护共享状态，确保在多 goroutine 环境下的并发安全。所有对服务器状态的修改操作（如添加或移除工具、提示、资源）都是线程安全的。服务器会话的管理也通过锁机制保证了数据的一致性。

**Section sources**
- [server.go](file://mcp/server.go#L37-L51)

## 实际调用示例
以下是一个典型的服务器初始化和工具添加示例：

```go
server := mcp.NewServer(&mcp.Implementation{Name: "greeter", Version: "v0.0.1"}, nil)
mcp.AddTool(server, &mcp.Tool{Name: "greet", Description: "say hi"}, SayHi)
```

此示例创建了一个名为 "greeter" 的服务器实例，并添加了一个名为 "greet" 的工具，该工具的处理程序为 `SayHi` 函数。

**Section sources**
- [basic\main.go](file://examples/server/basic/main.go#L26-L57)

## 错误处理与 Panic 条件
服务器 API 在多种情况下会触发 panic，以确保配置的正确性和数据的完整性。例如，`NewServer` 函数会在 `Implementation` 为 nil 时 panic；`AddTool` 方法会在输入 Schema 为空或类型不为 "object" 时 panic；`AddResource` 方法会在资源 URI 无效时 panic。此外，`SubscribeHandler` 和 `UnsubscribeHandler` 必须同时存在或同时不存在，否则也会触发 panic。这些 panic 条件有助于在开发阶段尽早发现和修复配置错误。

**Section sources**
- [server.go](file://mcp/server.go#L109-L150)
- [server.go](file://mcp/server.go#L191-L234)
- [server.go](file://mcp/server.go#L422-L431)