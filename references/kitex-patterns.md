# Kitex (gRPC) Patterns

Kitex 适配层负责 gRPC 通信。代码生成器从 proto 生成 server + client + fx 文件，通过 `fx.Module` 自动装配。

## 自定义拦截器

在 `internal/middleware/kitex/` 下创建文件，通过 fx 注入：

```go
// internal/middleware/kitex/auth.go
package kitex

import (
    "context"
    "github.com/cloudwego/kitex/pkg/endpoint"
    kserver "gitlab.allinfinance.com/aifgo/ag-core/contribute/agkitex/server"
)

func NewAuthMiddleware() endpoint.Middleware {
    return func(next endpoint.Endpoint) endpoint.Endpoint {
        return func(ctx context.Context, req, resp interface{}) error {
            // 从 metadata 提取 token，验证
            return next(ctx, req, resp)
        }
    }
}

func FxAuthMiddleware() kserver.FxServerMiddlewareOption {
    return kserver.NewFxServerMiddlewareProvider(NewAuthMiddleware)
}
```

**fx 注入**：在 `internal/zfx_internal.go` 中通过 `fx.Provide` 注册 `FxAuthMiddleware`。

**优先级**（按 `GetOrder()` 升序执行）：

| 优先级 | 值 | 典型用途 |
|--------|-----|----------|
| Highest | 0 | 元数据提取 |
| High | 1000 | 认证鉴权 |
| Normal | 2000 | 限流、监控 |
| Low | 3000 | 日志 |
| Lowest | 4000 | 错误处理 |

## 使用生成的 Client

生成的 client 位于 `internal/adpgen/kitex/<service>/`，通过 `clients/` 工厂模式统一管理。详见 [[gateway-patterns]]#clients 层。

```go
// 通过 fx 注入到 biz
type StudentBiz struct {
    userClient userservice.Client
}
```

## 配置

`cmd/server/app.yml`：

```yaml
kitex:
  server:
    Host: "0.0.0.0"                 # 监听地址，默认 ""
    Port: 9996                       # 监听端口，默认 7000
    AdaptivePort: true              # 端口冲突时自动查找，默认 false
    EnableIPRange: "0:255"           # 多网卡时指定注册IP段，默认空
    ServiceName: ${server.name}-grpc
    Grpc:
      enable: true                  # 默认 true
      MaxConnectionIdle: 20        # 最大空闲连接(秒)，默认 0

  client:
    RpcTimeout: 30s                 # RPC 超时，默认 30s
    TransportType: "grpc"
    Conn:
      GRPCConnPoolSize: 2           # 连接池大小，默认 GOMAXPROCS*3/2
    Resolver:
      enable: true                  # 服务发现，默认 true
      type: "agnacos"

nacos:
  config/naming:
    serveraddr: "127.0.0.1:8848"
```

## 核心原则

1. **不修改 adpgen/ 目录代码** — Kitex adapter 全部由 aggo 生成
2. **中间件通过 fx 注入** — 不修改生成的 fx 文件
3. **client 创建在 `clients/` 中统一管理** — 不自己构造 kitex client
4. **配置在 app.yml** — 前缀 `kitex.server` / `kitex.client`

## 验证

**注入完整性**：
□ `cmd/server/main.go` — `kserver.FxKitexServerBaseModule` 已声明
□ `cmd/server/main.go` — `kclient.FxKitexClientBaseModule` 已声明（调用外部 gRPC 时）

## 相关文件

- 代码生成：[[code-generation]]
- 项目结构：[[project-structure]]
- Gateway 模式：[[gateway-patterns]]
