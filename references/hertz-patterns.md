# Hertz (HTTP) Patterns

## 架构概览

Hertz 适配层负责 HTTP 通信。代码生成器从 proto 文件的 `google.api.http` 注解生成 3 个文件（server + client + fx），通过 `ServerConfigurator` 统一装配路由和中间件。

## 生成文件结构

```
internal/adpgen/hertz/<service>/
├── aghertz_<service>_server.go     # HTTP 路由 + Handler
├── aghertz_<service>_client.go     # HTTP 客户端
└── aghertz_<service>_fx.go         # fx 模块：提供路由 + 注册

internal/adpgen/adpinit/
└── zfx_aghertz_<svc>_<service>_adpinit.go  # init() 自动注册
```

## Server 端

### 路由注册模式

代码生成器为每个 `google.api.http` 注解生成路由定义：

```go
// aghertz_<service>_server.go (生成代码)
func Router_Student_CreateStudent_0_POST_Hertz(s student.StudentService) *hserver.Route {
    return &hserver.Route{
        HttpMethod:   "POST",
        RelativePath: "/student/create",
        Handlers:     append(make([]app.HandlerFunc, 0),
            _Student_CreateStudent_0_HTTP_Handler(s),
        ),
    }
}

func _Student_CreateStudent_0_HTTP_Handler(s student.StudentService) app.HandlerFunc {
    return func(ctx context.Context, c *app.RequestContext) {
        var in CreateStudentReq
        // 自动绑定请求体
        if err := c.BindByContentType(&in); err != nil {
            c.String(consts.StatusBadRequest, err.Error())
            return
        }
        resp, err := s.CreateStudent(ctx, &in)
        if err != nil {
            c.String(consts.StatusInternalServerError, err.Error())
            return
        }
        c.JSON(consts.StatusOK, resp)
    }
}
```

### ServerConfigurator 装配器

`ServerConfigurator` 是路由和中间件的统一装配器，在 `InitHertzServer()` 中按顺序初始化：

```
InitHertzServer()
├── ApplyServerOptions()   # 应用服务器选项（pprof、H2C 等）
├── ApplyMiddleware()      # 注册全局中间件
└── ApplyRoute()           # 注册所有路由
```

```go
type ServerConfigurator struct {
    Server   *server.Hertz       // Hertz 实例
    Opts     []*ServerOption     // 服务器选项
    Routes   []*Route            // 路由集合
    Mws      []Middleware        // 全局中间件
    MwsHFunc []app.HandlerFunc   // 全局中间件（原生 HandlerFunc）
}

// 路由通过 fx group "aghertz_route" 自动收集
func (m *ServerConfigurator) ApplyRoute() error {
    for _, route := range m.Routes {
        m.Server.Handle(route.HttpMethod, route.RelativePath, route.Handlers...)
    }
}

// 全局中间件注册
func (m *ServerConfigurator) ApplyMiddleware() error {
    for _, mw := range m.Mws {
        m.Server.Use(app.HandlerFunc(mw))
    }
    for _, hf := range m.MwsHFunc {
        m.Server.Use(hf)
    }
}
```

### 添加自定义 HTTP 中间件

通过 fx group 注入，不修改生成代码：

#### 方式1: 通过 fx group 注入

```go
// 在项目的 middleware/hertz/auth.go 中编写
func NewAuthMiddleware() app.HandlerFunc {
    return func(ctx context.Context, c *app.RequestContext) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.AbortWithStatus(consts.StatusUnauthorized)
            return
        }
        // 验证 token
        ctx = context.WithValue(ctx, "user_id", userId)
        c.Next(ctx)
    }
}

// 在 fx 模块中注册为 Option
func FxAuthMiddleware() fx.Option {
    return fx.Provide(
        hserver.NewFxServerMiddlewareProvider(NewAuthMiddleware),
    )
}
```

#### 方式2: 使用 `NewFxServerRouteProvider` 包装路由

```go
// 在生成的 fx 文件中自动生成
var FxStudent_hertz_RegProvider = fx.Provide(
    hserver.NewFxServerRouteProvider(
        Router_Student_CreateStudent_0_POST_Hertz,
        Router_Student_GetStudent_0_GET_Hertz,
    ),
)
```

### 服务端配置

```yaml
hertz:
  server:
    Host: "0.0.0.0"            # 监听地址
    Port: 9997                 # HTTP 端口
    ServiceName: "myapp-http"  # 注册中心服务名
    Pprof: true                # 开启 pprof
    KeepAlive: 60s             # 连接保活时间
```

### 特殊选项

| 选项 | 用途 | 配置方式 |
|------|------|----------|
| `pprof` | 性能分析 | `server_options_suites.go` → `WithPprof(path)` |
| `H2C` | HTTP/2 Cleartext | `WithH2C()` |
| `Registry` | 服务注册 | `Registry Suite` → 配置 `server.WithRegistry()` |

## Client 端

### HTTP 客户端调用模式

生成的 client 使用 `HertzBaseClient` 封装：

```go
// aghertz_<service>_client.go (生成代码)
type StudentServiceClient struct {
    baseClient *aghertzclient.HertzBaseClient
}

func (c *StudentServiceClient) GetStudent(ctx context.Context, req *GetStudentReq) (*Student, error) {
    var resp Student
    err := c.baseClient.DoRequest(ctx, &aghertzclient.RequestParam{
        Method: "GET",
        Path:   "/student/get",
    }, req, &resp)
    return &resp, err
}
```

### 客户端配置

```yaml
hertz:
  client:
    Timeout: 30s
    Discovery:
      enable: true
      type: "nacos"
```

### 客户端中间件优先级

Hertz 客户端也支持优先级中间件：

| 优先级常量 | 值 | 典型用途 |
|------------|-----|----------|
| `ClientMiddlewarePriorityHighest` | 0 | 元数据注入 |
| `ClientMiddlewarePriorityHigh` | 1000 | 认证签名 |
| `ClientMiddlewarePriorityNormal` | 2000 | 超时控制、重试 |
| `ClientMiddlewarePriorityLow` | 3000 | 日志记录 |
| `ClientMiddlewarePriorityLowest` | 4000 | 错误处理 |

```go
type PrioritizedClientMiddleware interface {
    GetOrder() int
    GetMiddleware() client.Middleware
}

// 应用时自动排序
func SortAndApplyMiddleware(c *client.Client, prioritizedMws []PrioritizedClientMiddleware) {
    sort.Sort(ByClientPriority(prioritizedMws))
    for _, pmw := range prioritizedMws {
        c.Use(pmw.GetMiddleware())
    }
}
```

## 请求处理模式

### POST 请求（绑定请求体）

```go
func handler(ctx context.Context, c *app.RequestContext) {
    var req CreateReq
    if err := c.BindByContentType(&req); err != nil { /* ... */ }
    resp, err := service.Create(ctx, &req)
    c.JSON(consts.StatusOK, resp)
}
```

### GET 请求（绑定查询参数 + 路径参数）

```go
func handler(ctx context.Context, c *app.RequestContext) {
    var req GetReq
    c.BindQuery(&req)    // 查询参数 ?Id=1
    c.BindPath(&req)     // 路径参数 /student/:Id
    resp, err := service.Get(ctx, &req)
    c.JSON(consts.StatusOK, resp)
}
```

### 错误响应

```go
// 绑定错误
c.String(consts.StatusBadRequest, err.Error())

// 业务错误
c.String(consts.StatusInternalServerError, err.Error())

// 成功响应
c.JSON(consts.StatusOK, resp)

// 空响应
c.Status(consts.StatusNoContent)
```

## HTTP 注解到路由映射

proto 文件中的 `google.api.http` 注解决定了生成的 HTTP 路由：

| proto 注解 | 生成的路由 | 请求绑定方式 |
|------------|-----------|-------------|
| `get: "path"` | GET path | Query 参数 |
| `post: "path" body: "*"` | POST path | Body (JSON/Form) |
| `put: "path/:Id" body: "*"` | PUT path/:Id | Path + Body |
| `delete: "path/:Id"` | DELETE path/:Id | Path 参数 |
| `additional_bindings: [...]` | 多个路由 | 对应各自绑定方式 |

## FX 依赖注入模式

### 关键 fx Group

| Group | 用途 |
|-------|------|
| `"aghertz_route"` | 路由集合，自动收集所有 `*Route` |
| `"aghertz_middleware"` | 全局中间件集合 |
| `"aghertz_server_config_options"` | 服务器 config.Option |
| `"aghertz_server_options"` | 服务器 ServerOption |
| `"ag_servers"` | Server 实例集合，被 App 统一管理 |

### fx 模块装配

```go
var FxAgHertzServerModule = fx.Module("fx_aghertz_server",
    ahregistry.FxHertzRegistyModule,     // 注册中心
    fx.Provide(
        NewHertzServerProperties,         // 配置
        NewHertzServer,                   // *server.Hertz
        FxNewServerConfigurator,          // ServerConfigurator
        NewAGHertzServer,                 // AgHertzServer 包装
    ),
    fx.Invoke(func(sc *ServerConfigurator) error {
        return sc.InitHertzServer()       // 初始化
    }),
)
```

## 核心原则

1. **路由由 aggo 生成** — HTTP 路由定义在 `aghertz_<service>_server.go` 中，不可手动修改
2. **中间件通过 fx group 注入** — 不修改生成的 fx 文件
3. **配置在 app.yml** — 前缀 `hertz.server` / `hertz.client`
4. **Handler 自动绑定请求** — `BindByContentType` (POST) 或 `BindQuery` (GET)
5. **每个 RPC 对应一个 HTTP 路由** — 通过 `google.api.http` 注解定义

> **项目级组织**：客户端创建和配置应在 `internal/clients/` 中统一管理（工厂模式 + ClientsConfig + SD/直连切换）。详见 [[gateway-patterns]]#clients 层。

## 完整请求生命周期

```
请求 → 全局中间件 → 路由匹配 → Handler → Bind → Service → Resp → JSON编码 → 响应
```

Hertz Handler 调用链：
1. 解析 HTTP 请求（方法、路径、Header、Body）
2. 绑定请求参数到 protobuf 消息（`BindByContentType` / `BindQuery` / `BindPath`）
3. 调用 Service 接口方法（`s.CreateStudent(ctx, &in)`）
4. 返回 JSON 响应（`c.JSON(consts.StatusOK, resp)`）
5. 错误处理（`c.String(statusCode, errMsg)`）

## 相关文件

- 代码生成：[[code-generation]]
- 项目结构：[[project-structure]]
- Proto IDL Patterns：[[proto-idl-patterns]]
- 配置文件：参考 `cmd/server/app.yml` 的 hertz 配置段
