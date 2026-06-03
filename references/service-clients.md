# Service Clients — 连接管理

> **场景触发器**: 跨服务调用、生成 client 代码、配置下游服务连接、SD/直连切换、组织 `clients/` 层时加载。

---

## 目录结构

```
clients/
├── config.go              # ClientCfg + ClientsConfig 三件套
├── <svc>_client.go        # 每个下游服务一个工厂函数
├── <svc>_client_http.go   # HTTP 版本
└── zfx_clients.go         # fx.Provide(NewClientsConfig, ...)
```

---

## 配置（ClientCfg 三件套）

### config.go

```go
// clients/config.go
package clients

import "gitlab.allinfinance.com/aifgo/ag-core/ag/ag_conf"

type ClientCfg struct {
    Mode       string // sd | direct
    SdName     string
    DirectAddr string
}

type ServiceClientCfg struct {
    Grpc ClientCfg
    Http ClientCfg
}

type ClientsConfig struct {
    Services map[string]ServiceClientCfg
}

func DefaultClientsConfig() *ClientsConfig {
    return &ClientsConfig{
        Services: make(map[string]ServiceClientCfg),
    }
}

func NewClientsConfig(binder ag_conf.IBinder) (*ClientsConfig, error) {
    cfg := DefaultClientsConfig()
    if err := binder.Bind(&cfg, "clients"); err != nil {
        return nil, err
    }
    return cfg, nil
}
```

### app.yml

```yaml
clients:
  services:
    scorer:
      grpc:
        mode: direct
        directAddr: localhost:9996
      http:
        mode: direct
        directAddr: http://localhost:9997
```

---

## 工厂函数

### gRPC (Kitex)

```go
// clients/scorer_client.go
func NewScorerKitexClient(cfg *ClientsConfig, suite *kitex.ClientSuite) (scorerservice.Client, error) {
    scorer := cfg.Services["scorer"]
    name := scorer.Grpc.SdName
    if name == "" { name = scorer.Grpc.DirectAddr }
    return scorerservice.NewClientWithSuite(name, suite,
        kitex.WithHostPorts(scorer.Grpc.DirectAddr),
    )
}
```

### HTTP (Hertz)

```go
// clients/scorer_client_http.go
func NewScorerHertzClient(cfg *ClientsConfig, hc *hclient.Client) scorersvc.ScorerServiceHertzClient {
    scorer := cfg.Services["scorer"]
    endpoint := scorer.Http.DirectAddr
    if scorer.Http.Mode == "sd" { endpoint = scorer.Http.SdName }
    return scorersvc.NewScorerServiceHertzClient(hc,
        agclient.WithEndpoint(endpoint),
        agclient.WithSD(scorer.Http.Mode == "sd"),
    )
}
```

> **关键**：`NewClientWithSuite` + `WithHostPorts` 统一 SD 和直连模式，suite（中间件）始终生效。`WithHostPorts` 优先于 SD 解析——直连时直接使用指定地址。

---

## fx 注册

```go
// clients/zfx_clients.go
var FxClientModule = fx.Module("fx-client-module",
    fx.Provide(
        NewClientsConfig,
        NewScorerKitexClient,
        NewScorerHertzClient,
    ),
)
```

在 `internal/zfx_internal.go` 中：

```go
var FxInternalModule = fx.Module("fx-internal-module",
    config.FxAppConfigModule,
    clients.FxClientModule,     // ← clients 层
    gateway.FxGatewayModule,
    biz.FxBizModule,
    svcgen.FxServiceWithProxyModule(),
    adpgen.FxAdapterModule(),
)
```

---

## SD / 直连切换

| 模式 | `mode` | 使用值 | 场景 |
|------|--------|--------|------|
| 直连 | `direct` | `DirectAddr` | 本地开发、测试 |
| 服务发现 | `sd` | `SdName` | 生产环境（通过 Nacos 解析） |

```yaml
# 开发环境
clients:
  services:
    scorer:
      grpc:
        mode: direct
        directAddr: localhost:9996

# 生产环境
clients:
  services:
    scorer:
      grpc:
        mode: sd
        sdName: scorer-grpc
```

环境切换纯 YAML 层面完成，代码无需改动。

---

## 实施细节

| 规范 | 说明 |
|------|------|
| YAML key 用 camelCase | `directAddr`，不用 `direct_addr`（`_` 会与环境变量冲突） |
| map key 用下游服务名 | 对应 `ClientsConfig.Services[<name>]` |
| 工厂返回 adpgen 生成的接口 | 如 `scorerservice.Client`，不额外包装 |
| 生成代码放 `idl/api/<callee>/` | 复制下游 proto → `aggo proto -p kitex,hertz -m client` |

---

> 完成后执行验证：[[verification#新增跨服务调用]]

## 相关文件

- Gateway 模式：[[gateway-patterns]]
- Kitex 实现层：[[kitex-patterns]]
- Hertz 实现层：[[hertz-patterns]]
- 项目结构：[[project-structure]]
