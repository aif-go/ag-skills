# Kitex (gRPC) Patterns

## 架构概览

Kitex 适配层负责 gRPC 通信。代码生成器从 proto 文件生成 4 个文件（core + server + client + fx），通过 `fx.Module` 自动装配到应用中。

## 生成文件结构

```
internal/adpgen/kitex/<service>/
├── agkitex_<service>.go            # 核心：ServiceInfo + Args/Result + Handler
├── agkitex_<service>_server.go     # 服务端：Register 函数
├── agkitex_<service>_client.go     # 客户端：Client 接口 + NewClient
├── agkitex_<service>_agclient.go   # 客户端：Suite 方式创建
└── agkitex_<service>_fx.go         # fx 模块声明

internal/adpgen/adpinit/
└── zfx_agkitex_<svc>_<service>_adpinit.go  # init() 自动注册
```

## Server 端

### 服务注册模式

每个 service 生成一个 `Register` 函数，通过 `AgKitexServiceRegistry` 注册到 Kitex server：

```go
// agkitex_<service>_server.go (生成代码，不可修改)
func Register_StudentService_KitexServer(
    handler service.StudentService,
    opts ...kserver.RegisterOption,
) *kserver.AgKitexServiceRegistry {
    return &kserver.AgKitexServiceRegistry{
        ServiceInfo: serviceInfo(),
        Handler:     newStudentServiceHandler(handler),
        Opts:        opts,
    }
}
```

### Adapter 架构

框架使用 **Suite 模式**，通过 `KitexServerSuiteBuilder` 组装配置：

```
KitexServerSuiteBuilder
├── Properties (Host/Port/ServiceName)
├── Registry (服务注册中心)
├── Middlewares (普通中间件)
├── PrioritizedMiddlewares (带优先级的中间件)
└── BuildServerSuite() → KitexServerSuite → server.WithSuite()
```

```go
// AgKitexServer 实现框架统一的 Server 接口
type AgKitexServer struct {
    KitexServer server.Server
}

func (s *AgKitexServer) Start(ctx context.Context) error {
    return s.KitexServer.Run()
}

func (s *AgKitexServer) Stop(ctx context.Context) error {
    return s.KitexServer.Stop()
}
```

### 中间件优先级系统

Kitex server 支持带优先级的中间件，按 `GetOrder()` 升序执行：

| 优先级常量 | 值 | 典型用途 |
|------------|-----|----------|
| `ServerMiddlewarePriorityHighest` | 0 | 元数据提取、日志 |
| `ServerMiddlewarePriorityHigh` | 1000 | 认证鉴权 |
| `ServerMiddlewarePriorityNormal` | 2000 | 限流、监控 |
| `ServerMiddlewarePriorityLow` | 3000 | 日志记录 |
| `ServerMiddlewarePriorityLowest` | 4000 | 错误处理 |

```go
// 添加自定义中间件的接口
type PrioritizedServerMiddleware interface {
    GetOrder() int
    GetMiddleware() endpoint.Middleware
}
```

### 添加自定义拦截器

通过 fx group `"fx_kitex_server_middleware"` 注入自定义中间件：

```go
// 在项目中创建 middleware/kitex/auth.go
func NewAuthMiddleware() endpoint.Middleware {
    return func(next endpoint.Endpoint) endpoint.Endpoint {
        return func(ctx context.Context, req, resp interface{}) error {
            // 从 metadata 提取 token
            // 验证 token
            return next(ctx, req, resp)
        }
    }
}
```

在 fx 模块中通过 `NewFxServerMiddlewareProvider` 注入：

```go
func NewFxAuthMiddleware() kserver.FxServerMiddlewareOption {
    return kserver.NewFxServerMiddlewareProvider(NewAuthMiddleware)
}
```

### 服务端配置

`cmd/server/app.yml` 中配置 Kitex server：

```yaml
kitex:
  server:
    Host: "0.0.0.0"          # 监听地址
    Port: 9996                # gRPC 端口
    ServiceName: "myapp-grpc" # 注册中心服务名
    Grpc:
      enable: true
      MaxConnectionIdle: 360  # 最大空闲连接时间(秒)
```

## Client 端

### 客户端调用模式

生成的 client 提供两种创建方式：

```go
// 方式1: 基础Client（通过目标服务名创建）
client := NewClient("target-service-grpc")

// 方式2: Suite方式（注入中间件等配置）
suite := BuildClientSuite()
client := NewClientWithSuite("target-service-grpc", suite)

// 调用 RPC 方法
resp, err := client.GetStudent(ctx, &GetStudentReq{Id: 1})
```

### 客户端配置

```yaml
kitex:
  client:
    RpcTimeout: 30s                # RPC 超时
    TransportType: "grpc"          # 传输协议
    Resolver:
      enable: true
      type: "agnacos"              # 服务发现
    Conn:
      GRPCConnPoolSize: 0          # 连接池大小（0=自动：GOMAXPROCS*3/2）
```

### 客户端中间件优先级

| 优先级常量 | 值 | 典型用途 |
|------------|-----|----------|
| `ClientMiddlewarePriorityHighest` | 0 | 元数据注入 |
| `ClientMiddlewarePriorityHigh` | 1000 | 认证、签名 |
| `ClientMiddlewarePriorityNormal` | 2000 | 超时控制 |
| `ClientMiddlewarePriorityLow` | 3000 | 日志 |
| `ClientMiddlewarePriorityLowest` | 4000 | 错误处理、重试 |

## 元数据传递

ag-core 提供元数据透传机制，在 gRPC 调用链中传播跟踪信息：

```go
// Server 端：从 gRPC metadata 提取自定义元数据
// 通过 NewAgKitexServerAgMetadataHTTP2HandlerOption() 注册

// Client 端：将自定义元数据注入 gRPC metadata
// 通过 NewAgKitexClientAgMetadataHTTP2HandlerOption() 注册
```

## FX 依赖注入模式

### Server 端 fx 模块

```go
// 框架级模块 (已内置)
var FxKitexServerBaseModule = fx.Module("fx_kitex_server_base",
    agkitexReg.FxKitexRegistyModule,   // 注册中心(Nacos)
    fx.Provide(
        NewKitexServerProperties,        // 配置绑定
        FxNewKitexServerSuiteBuilder,    // SuiteBuilder
        FxBuilderKitexServerSuite,
    ),
)

// 生成的 adpinit（init() 自动注册）
// internal/adpgen/adpinit/zfx_agkitex_<svc>_<service>_adpinit.go
func init() {
    adpgen.AddFxAdapterOpt(
        func() fx.Option {
            return fx.Module("...",
                fx.Provide(Register_StudentService_KitexServer),
            )
        },
    )
}
```

### 关键 fx Group

| Group | 用途 |
|-------|------|
| `"fx_kitex_server_middleware"` | 注入自定义 server 中间件 |
| `"fx_kitex_client_middleware"` | 注入自定义 client 中间件 |
| `"ag_servers"` | Server 实例集合，被 App 统一管理生命周期 |

## 服务发现

Kitex 默认使用 Nacos 作为注册中心和配置中心：

- **Server 端**: 启动时自动注册到 Nacos
- **Client 端**: 通过 NacosResolver 发现服务实例

```yaml
nacos:
  config/naming:
    serveraddr: "127.0.0.1:8848"
    namespace: "test"
```

## 核心原则

1. **不修改 adpgen/ 目录代码** — Kitex adapter 全部由 aggo 生成
2. **中间件通过 fx group 注入** — 不直接修改生成的 fx 文件
3. **使用生成的 client 对象** — 不自己构造 kitex client
4. **配置统一在 app.yml** — kitex server/client 配置前缀分别为 `kitex.server` / `kitex.client`
5. **元数据透传使用框架机制** — 不在业务代码中手动处理 metadata

> **项目级组织**：客户端创建和配置应在 `internal/clients/` 中统一管理（工厂模式 + ClientsConfig + SD/直连切换）。详见 [[gateway-patterns]]#clients 层。

## 相关文件

- 代码生成：[[code-generation]]
- 项目结构：[[project-structure]]
- 配置文件：参考 `cmd/server/app.yml` 的 kitex 配置段
