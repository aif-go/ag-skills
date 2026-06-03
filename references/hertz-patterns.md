# Hertz (HTTP) Patterns

Hertz 适配层负责 HTTP 通信。代码生成器从 proto 的 `google.api.http` 注解生成路由 + client + fx 文件，通过 `ServerConfigurator` 自动装配。

## 自定义 HTTP 中间件

在 `internal/middleware/hertz/` 下创建文件，通过 fx 注入：

```go
// internal/middleware/hertz/auth.go
package hertz

import (
    "context"
    "github.com/cloudwego/hertz/pkg/app"
    hserver "gitlab.allinfinance.com/aifgo/ag-core/contribute/aghertz/server"
)

func NewAuthMiddleware() app.HandlerFunc {
    return func(ctx context.Context, c *app.RequestContext) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.AbortWithStatus(consts.StatusUnauthorized)
            return
        }
        ctx = context.WithValue(ctx, "user_id", userId)
        c.Next(ctx)
    }
}

func FxAuthMiddleware() fx.Option {
    return fx.Provide(
        hserver.NewFxServerMiddlewareProvider(NewAuthMiddleware),
    )
}
```

**fx 注入**：在 `internal/zfx_internal.go` 中通过 `fx.Provide` 注册 `FxAuthMiddleware`。

## 请求处理速查

### POST（绑定请求体）

```go
var req CreateReq
c.BindByContentType(&req)
resp, err := service.Create(ctx, &req)
c.JSON(consts.StatusOK, resp)
```

### GET（绑定查询参数 + 路径参数）

```go
var req GetReq
c.BindQuery(&req)    // ?Id=1
c.BindPath(&req)     // /student/:Id
resp, err := service.Get(ctx, &req)
c.JSON(consts.StatusOK, resp)
```

### 错误响应

```go
c.String(consts.StatusBadRequest, err.Error())          // 400
c.String(consts.StatusInternalServerError, err.Error())  // 500
c.JSON(consts.StatusOK, resp)                            // 200
c.Status(consts.StatusNoContent)                         // 204
```

## HTTP 注解 → 路由映射

proto 的 `google.api.http` 注解决定生成的路由：

| proto 注解 | 路由 | 绑定方式 |
|------------|------|----------|
| `get: "path"` | GET path | Query |
| `post: "path" body: "*"` | POST path | Body |
| `put: "path/:Id" body: "*"` | PUT path/:Id | Path + Body |
| `delete: "path/:Id"` | DELETE path/:Id | Path |
| `additional_bindings: [...]` | 多个路由 | 对应方式 |

## 配置

`cmd/server/app.yml`：

```yaml
hertz:
  server:
    Host: "0.0.0.0"                 # 默认 "0.0.0.0"
    Port: 9997                       # HTTP 端口，默认 7000
    AdaptivePort: true              # 端口冲突时自动查找，默认 false
    EnableIPRange: "0:255"           # 多网卡时指定注册IP段，默认空
    ServiceName: ${server.name}-http
    KeepAlive: true                 # 默认 true
    KeepAliveTimeout: 60s           # 默认 60s
    Pprof: false                    # 生产关闭，默认 false

  client:
    KeepAlive: false                # 默认 false
    DialTimeout: 1000               # 连接超时(ms)，默认 1000
    MaxConnsPerHost: 512            # 每host最大连接数，默认 512
    MaxIdleConnDuration: 10000     # 空闲保活(ms)，默认 10000
```

## 核心原则

1. **路由由 aggo 生成** — 不可手动修改 `aghertz_<service>_server.go`
2. **中间件通过 fx 注入** — 不修改生成的 fx 文件
3. **每个 RPC 对应一个 HTTP 路由** — 通过 `google.api.http` 注解定义
4. **client 创建在 `clients/` 中统一管理** — 详见 [[gateway-patterns]]

> 完成后执行验证：[[verification#新增 Hertz-HTTP 服务]]

## 相关文件

- 代码生成：[[code-generation]]
- 项目结构：[[project-structure]]
- Proto IDL Patterns：[[proto-idl-patterns]]
- Gateway 模式：[[gateway-patterns]]
