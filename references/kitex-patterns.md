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
      type: "agnacos"              # agnacos（推荐）或 nacos
      nacos:
        group: DEFAULT_GROUP
        cluster: DEFAULT

nacos:
  naming:
    serveraddr: "192.168.1.1:8848"
```

## 服务注册与发现

**注册**：声明 `kserver.FxKitexServerBaseModule` 后，服务启动时自动向 Nacos 注册。注册名由 `kitex.server.ServiceName` 控制。

**发现**：声明 `kclient.FxKitexClientBaseModule` 后，通过 Resolver 从 Nacos 查找下游服务实例。`type: agnacos` 兼容 Spring gRPC 的 `gRPC_port` metadata。

> 前置条件：`nacos.naming.serveraddr` 已配置 + `agnacos.FxNacosNamingMode` 已声明。详见 [[nacos-patterns]]。

## 流式传输

默认 `transportType: grpc` 已同时支持 unary 和 streaming，无需额外配置。

### 服务端实现

```go
// internal/biz/student_biz.go
func (b *StudentBiz) ListStudentsStream(
    req *student.ListStudentsReq,
    stream student.StudentService_ListStudentsServer,
) error {
    for batch := range b.fetchBatches(req.Age) {
        if err := stream.Send(&student.ListStudentsResp{Students: batch}); err != nil {
            return err
        }
    }
    return nil
}
```

注册方式与普通 RPC 相同，不需要改动 `adpgen/`：

```go
fx.Provide(
    akxserver.NewFxAgKitexServiceRegistry(func() *akxserver.AgKitexServiceRegistry {
        return akxserver.NewAgKitexServiceRegistry(
            studentservice.NewServiceInfo(),
            &StudentServiceImpl{},
        )
    }),
)
```

### 客户端调用

```go
stream, err := client.ListStudents(ctx, &student.ListStudentsReq{Age: 0})
for {
    resp, err := stream.Recv()
    if err == io.EOF { break }
    // 处理 resp.Students
}
```

### 关键限制

流式方法**不走 ag-service proxy 中间件链**，以下功能无效：

- 声明式事务（`AddTag(agdb.TransactionTag, true)`）
- ag-service 全局中间件
- CallInfo 增强

流式方法内需要事务时，在 biz 中手动管理。

---

## 核心原则

1. **不修改 adpgen/ 目录代码** — Kitex adapter 全部由 aggo 生成
2. **中间件通过 fx 注入** — 不修改生成的 fx 文件
3. **client 创建在 `clients/` 中统一管理** — 不自己构造 kitex client
4. **配置在 app.yml** — 前缀 `kitex.server` / `kitex.client`
5. **服务发现用 Nacos** — 详见 [[nacos-patterns]]

> 完成后执行验证：[[verification#新增 Kitex-gRPC 服务]]

## 相关文件

- 代码生成：[[code-generation]]
- 项目结构：[[project-structure]]
- Gateway 模式：[[gateway-patterns]]
- Nacos 服务发现：[[nacos-patterns]]
