# Service Clients — 连接管理

> **场景触发器**: 跨服务调用、生成 client 代码、配置下游服务连接、SD/直连切换、组织 `clients/` 层时加载。

---

## 前置步骤

```
1. 复制下游 proto → idl/api/<callee>/
2. aggo proto -p <platform> -m client -e ./idl/api ./idl/api/<callee>/<callee>.proto
   - `-p kitex` — 仅 gRPC | `-p hertz` — 仅 HTTP | `-p kitex,hertz` — 两者
3. go mod tidy && go build ./...
```

> 详见 [[code-generation]]#Client Generation

---

## 目录结构

```
clients/
├── config.go                  # ClientCfg + ClientsConfig
├── <svc>_client_kitex.go     # gRPC (Kitex) 工厂
├── <svc>_client_hertz.go     # HTTP (Hertz) 工厂
└── zfx_clients.go            # fx.Provide(NewClientsConfig, ...)
```

---

## 配置

### config.go

```go
package clients

import (
    "fmt"
    "strings"

    "github.com/aif-go/ag-core/ag/ag_conf"
)

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

func NewClientsConfig(binder ag_conf.IBinder) (*ClientsConfig, error) {
    cfg := &ClientsConfig{Services: make(map[string]ServiceClientCfg)}
    if err := binder.Bind(&cfg, "clients"); err != nil {
        return nil, err
    }
    return cfg, nil
}
```

### app.yml — SD/直连规则

```
mode 指定       → 校验对应字段，缺则报错
  sd              → sdName 必填
  direct          → directAddr 必填
mode 为空       →
  仅 sdName         → SD
  仅 directAddr     → 直连
  sdName + directAddr → 报错（冲突）
  两者皆空           → 报错（缺地址）
```

```yaml
clients:
  services:
    scorer:
      grpc:
        directAddr: localhost:9996          # 最简直连
        # sdName: scorer-grpc               # 最简 SD
        # mode: sd; sdName: scorer-grpc     # 完整 SD（推荐）
```

> key 用 camelCase（`directAddr`），map key 用下游服务名。环境切换纯 YAML。

---

## 工厂函数

### 辅助函数 — config.go

```go
func resolveUseSD(cfg *ClientCfg) (bool, error) {
    if cfg.Mode != "" {
        switch strings.ToLower(cfg.Mode) {
        case "sd":
            if cfg.SdName == "" {
                return false, fmt.Errorf("mode is 'sd' but sdName is empty")
            }
            return true, nil
        case "direct":
            if cfg.DirectAddr == "" {
                return false, fmt.Errorf("mode is 'direct' but directAddr is empty")
            }
            return false, nil
        default:
            return false, fmt.Errorf("invalid mode '%s', must be 'sd' or 'direct'", cfg.Mode)
        }
    }
    if cfg.SdName != "" && cfg.DirectAddr != "" {
        return false, fmt.Errorf("both sdName and directAddr set, add 'mode: sd' or 'mode: direct'")
    }
    if cfg.SdName == "" && cfg.DirectAddr == "" {
        return false, fmt.Errorf("neither sdName nor directAddr is configured")
    }
    return cfg.SdName != "", nil
}
```

### gRPC (Kitex)

```go
// clients/scorer_client_kitex.go
import (
    kclient "github.com/cloudwego/kitex/client"
    agclient "github.com/aif-go/ag-core/contribute/agkitex/client"
)

func NewScorerKitexClient(cfg *ClientsConfig, suite *agclient.KitexClientSuite) (scorerservice.Client, error) {
    scorer := cfg.Services["scorer"]
    grpc := &scorer.Grpc

    useSD, err := resolveUseSD(grpc)
    if err != nil { return nil, err }
    if useSD {
        return scorerservice.NewClientWithSuite(grpc.SdName, suite)
    }
    return scorerservice.NewClientWithSuite(grpc.DirectAddr, suite,
        kclient.WithHostPorts(grpc.DirectAddr),
    )
}
```

### HTTP (Hertz)

```go
// clients/scorer_client_hertz.go
import (
    agclient "github.com/aif-go/ag-core/contribute/aghertz/aghertzclient"
    hclient "github.com/cloudwego/hertz/pkg/app/client"
)

func NewScorerHertzClient(cfg *ClientsConfig, hc *hclient.Client) (scorersvc.ScorerServiceHertzClient, error) {
    scorer := cfg.Services["scorer"]
    httpCfg := &scorer.Http

    useSD, err := resolveUseSD(httpCfg)
    if err != nil { return nil, err }
    if useSD {
        return scorersvc.NewScorerServiceHertzClient(hc,
            agclient.WithSDEndpoint(httpCfg.SdName),
        ), nil
    }
    return scorersvc.NewScorerServiceHertzClient(hc,
        agclient.WithDirectEndpoint(httpCfg.DirectAddr),
    ), nil
}
```

---

## FX 注册

### clients/zfx_clients.go

```go
var FxClientModule = fx.Module("fx-client-module",
    fx.Provide(NewClientsConfig, NewScorerKitexClient, NewScorerHertzClient),
)
```

### cmd/server/main.go（缺了启动失败）

```go
import (
    hclient "github.com/aif-go/ag-core/contribute/aghertz/client"
    kclient "github.com/aif-go/ag-core/contribute/agkitex/client"
)

var mainFx = fx.Module("main",
    // …
    hclient.FxModuleAgHertzClient,       // 注入 *hclient.Client
    kclient.FxKitexClientBaseModule,     // 注入 *client.KitexClientSuite
    internal.FxInternalModule,           // 包含 FxClientModule
)
```

### internal/zfx_internal.go

```go
var FxInternalModule = fx.Module("fx-internal-module",
    config.FxAppConfigModule,
    clients.FxClientModule,
    gateway.FxGatewayModule,
    biz.FxBizModule,
    svcgen.FxServiceWithProxyModule(),
    adpgen.FxAdapterModule(),
)
```
