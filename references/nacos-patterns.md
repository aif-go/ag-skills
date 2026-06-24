# Nacos Patterns

> **场景触发器**: 服务注册、服务发现、Nacos 配置、远程配置中心、`agnacos.FxNacosNamingMode`、SD 模式切换时加载。

---

## FX 模块注册

`cmd/server/main.go` 中按需启用，注册在 `fxs.FxAgConfModule` 之后：

```go
import "github.com/aif-go/ag-core/contribute/agnacos"

var mainFx = fx.Module("main",
    fxs.FxAgConfModule,

    agnacos.FxNacosConfigMode,              // ① 配置中心客户端
    agnacos.FxEnableNacosRemoteConfigMode,  // ② 启用远程配置注入
    agnacos.FxNacosNamingMode,              // ③ 服务发现客户端

    // ... 其他模块
)
```

| 模块 | 作用 | 何时启用 |
|------|------|---------|
| `FxNacosConfigMode` | 创建 ConfigClient | 从 Nacos 拉取远程配置 |
| `FxEnableNacosRemoteConfigMode` | 注入远程配置到 ag_conf | 配合 ① 使用 |
| `FxNacosNamingMode` | 创建 NamingClient | 服务发现/注册 |

> 只做服务发现不需要配置中心：只注册 `FxNacosNamingMode`。

---

## app.yml 配置

### 服务发现与注册

配置前缀：`nacos.naming`

```yaml
nacos:
  naming:
    enable: true
    serveraddr: 192.168.1.1:8848,192.168.1.2:8848    # Nacos 地址，逗号分隔，必填
    namespace: aic-dev                                  # 命名空间 ID
    username: nacos                                     # 用户名
    password: nacos                                     # 密码
    schema: http                                        # 默认 http
    contextpath: /nacos                                 # 默认 /nacos
    loglevel: error                                     # SDK 日志级别，默认 error
```

### 远程配置中心

配置前缀：`nacos.config`

```yaml
nacos:
  config:
    enable: true
    serveraddr: 192.168.1.1:8848
    namespace: aic-dev
    username: nacos
    password: nacos
    dataids:
      - dataid: app-config.yaml
        group: DEFAULT_GROUP                    # 默认 DEFAULT_GROUP
        type: yaml                              # yaml / properties / json
        autorefresh: true                       # 监听变更，默认 true
      - dataid: db-config.yaml
        group: DEFAULT_GROUP
        type: yaml
        autorefresh: true
```

### 同时启用（推荐用占位符复用连接信息）

```yaml
nacos:
  config:
    serveraddr: 192.168.1.1:8848
    namespace: aic-dev
    username: nacos
    password: nacos
    dataids:
      - dataid: app-config.yaml
        group: DEFAULT_GROUP
        type: yaml
        autorefresh: true

  naming:
    serveraddr: ${nacos.config.serveraddr}    # ← 引用 config 的值
    namespace: ${nacos.config.namespace}
    username: ${nacos.config.username}
    password: ${nacos.config.password}
```

---

## 服务注册与发现

`FxNacosNamingMode` 提供 `INamingClient`，供 agkitex 和 aghertz 使用。

**服务注册**：`kserver.FxKitexServerBaseModule` 和 `hserver.FxAgHertzServerModule` 自动处理。只需配置 `nacos.naming` 和对应协议的服务名即可：

- gRPC 配置：`kitex.server.ServiceName`（详见 [[kitex-patterns]]）
- HTTP 配置：`hertz.server.ServiceName`（详见 [[hertz-patterns]]）

**服务发现**：在客户端配置中启用 Resolver/Discovery：

- gRPC：`kitex.client.Resolver.type: agnacos`（详见 [[kitex-patterns]]）
- HTTP：`hertz.client.Discovery.type: nacos`（详见 [[hertz-patterns]]）

---

## 远程配置中心

### 优先级

**系统环境变量 > Nacos 远程配置 > 本地 app.yml**。环境变量可覆盖 Nacos 配置。

### 热更新

`AutoRefresh: true`（默认）时，Nacos 配置变更后**实时生效**，通过 `Bind()` 绑定的结构体自动刷新。

### 验证

启动日志：

```
"nacos config"                    → 每个 DataID 加载成功
"create kitex nacos registry"     → Kitex 服务注册成功
"nacos remote config is disable"  → 配置中心未启用（Enable=false）
```

---

## 本地开发绕过

本地开发不需要 Nacos 时，注释掉 `agnacos` 相关模块即可：

```go
// agnacos.FxNacosConfigMode,
// agnacos.FxEnableNacosRemoteConfigMode,
// agnacos.FxNacosNamingMode,
```

服务仍会以**直连模式**启动，不会向 Nacos 注册或拉取配置。

---

## 验证

**注入完整性**：
□ `cmd/server/main.go` — `agnacos.FxNacosNamingMode` 已声明（需要服务发现时）
□ `cmd/server/main.go` — `agnacos.FxNacosConfigMode` + `FxEnableNacosRemoteConfigMode` 已声明（需要远程配置时）

**配置检查**：
□ `nacos.naming.serveraddr` 已配置
□ 生产环境 `kitex.client.Resolver.enable: true` 且 `type: agnacos`
□ 本地开发无 Nacos 时，agnacos 模块已注释

## 相关文件

- Kitex 模式：[[kitex-patterns]]
- Service Clients：[[service-clients]]
- 项目结构：[[project-structure]]
