# AgConf Configuration Patterns

> **场景触发器**: 当用户说"添加配置""读取配置""定义配置结构体""app.yml"时加载本文。
> 本文是 AI 编写 ag-core 配置代码的核心参考。

---

## 快速开始 — 三件套模式

**标准模式**：结构体 + 默认值 + 构造函数。代码写默认值，无配置文件也能跑。

### 第 1 步：配置文件

```yaml
# cmd/server/app.yml
app:
  name: my-service
  server:
    port: 8080
  datasource:
    url: jdbc:mysql://localhost:3306/db
    username: root
```

### 第 2 步：配置结构体（三件套）

```go
// config/app.go
package config

import "gitlab.allinfinance.com/aifgo/ag-core/ag/ag_conf"

const AppConfigKey = "app"

type AppConfig struct {
    Name       string
    Server     ServerConfig
    Datasource DatasourceConfig
}

type ServerConfig struct {
    Port int
}

type DatasourceConfig struct {
    Url      string
    Username string
}

func DefaultAppConfig() AppConfig {
    return AppConfig{
        Name: "unnamed",
        Server: ServerConfig{
            Port: 8080,
        },
        Datasource: DatasourceConfig{
            Username: "root",
        },
    }
}

func NewAppConfig(binder ag_conf.IBinder) (*AppConfig, error) {
    cfg := DefaultAppConfig()
    err := binder.Bind(&cfg, AppConfigKey)
    if err != nil {
        return nil, err
    }
    return &cfg, nil
}
```

### 第 3 步：fx 模块化

采用 fx 依赖注入，由框架模块自动提供 `IBinder`：

```go
// internal/config/zfx_config.go
package config

import "go.uber.org/fx"

var FxAppConfigModule = fx.Module("fx-app-conf-module",
    fx.Provide(
        NewAppConfig,     // fx 自动解析 IBinder 参数
    ),
)
```

在 main.go 中只需声明框架 conf 模块（提供 IBinder）和自定义 config 模块：

```go
// cmd/server/main.go
var mainFx = fx.Module("main",
    fxs.FxAgConfModule,       // 框架层：提供 ag_conf.IBinder
    // ... 其他模块
    internal.FxInternalModule, // 内部层：含 config.FxAppConfigModule
)
```

`NewAppConfig(binder ag_conf.IBinder)` — fx 自动注入 binder，无需手动创建 env。

---

## 结构体定义方式

### 3 种写法

| 写法 | 适用场景 |
|------|----------|
| `Name string` | **最推荐**，字段名 = YAML key |
| `Name string \`value:"${host}"\`` | YAML key 与字段名不一致 |
| `Name string \`value:"${host:8080}"\`` | key 不一致，且需要非零默认值 |

### 必填字段

```go
type Config struct {
    Host     string `required:"true"`
    Password string `value:"${password}" required:"true"`
}
```

加 `required:"true"` 后，Bind 找不到值会返回 error。不加则静默用零值。

---

## 获取配置值

### 方式 A：Bind 到结构体（推荐，>3 字段）

```go
type DBConfig struct {
    Host     string
    Port     int
    Username string
    Password string `required:"true"`
}

func DefaultDBConfig() DBConfig {
    return DBConfig{
        Port: 3306,
    }
}

func NewDBConfig(binder ag_conf.IBinder) (*DBConfig, error) {
    cfg := DefaultDBConfig()
    if err := binder.Bind(&cfg, "datasource"); err != nil {
        return nil, err
    }
    return &cfg, nil
}
```

### 方式 B：GetProperty 直接读（仅初始化时用，避免高频调用）

```go
host := env.GetProperty("datasource.host")
port := env.GetPropertyDefault("datasource.port", "3306")
pwd, err := env.GetRequiredProperty("datasource.password")
```

> ⚠️ `GetProperty` 高频下有性能问题。**只在启动初始化时调用一次**，不要在每个请求中反复读取。运行时取配置值应通过 Bind 后的结构体字段访问。

---

## 项目中的目录与 fx 组装

### 目录约定

```
project/
├── cmd/server/
│   ├── main.go                 # fx 模块总组装
│   └── app.yml                 # 运行时配置
└── internal/
    ├── config/                  # 所有配置代码集中在此
    │   ├── xxx_config.go       # 结构体 + Default + 构造函数
    │   └── zfx_config.go       # fx 模块 Provide
    └── zfx_internal.go          # 内部模块总入口，组装 config 子模块
```

每加一个配置模块就在 `internal/config/` 下新增对应的结构体文件和 fx 注册。

### fx 组装链

```
main.go
  ├── fxs.FxAgConfModule          ← 框架层：提供 ag_conf.IBinder
  └── internal.FxInternalModule   ← 内部总入口
        └── config.FxAppConfigModule  ← 注入自定义配置
```

`fxs.FxAgConfModule` 是框架自带的模块，**无需手动创建 env 或 binder**，注册后 IBinder 自动可用。

### 自定义配置模块示例

```go
// internal/config/hzw_config.go
package config

import "gitlab.allinfinance.com/aifgo/ag-core/ag/ag_conf"

const HzwKey = "hzw"

type HzwConfig struct {
    Org  string `required:"true"`
    Type string `required:"true"`
}

func DefaultHzwConfig() HzwConfig {
    return HzwConfig{Org: "ORG000", Type: "default"}
}

func NewHzwConfig(binder ag_conf.IBinder) (*HzwConfig, error) {
    cfg := DefaultHzwConfig()
    if err := binder.Bind(&cfg, HzwKey); err != nil {
        return nil, err
    }
    return &cfg, nil
}
```

```go
// internal/config/zfx_config.go
package config

import "go.uber.org/fx"

var FxAppConfigModule = fx.Module("fx-app-conf-module",
    fx.Provide(NewHzwConfig),
)
```

```go
// internal/zfx_internal.go
var FxInternalModule = fx.Module("fx-internal-module",
    config.FxAppConfigModule,       // 配置最先加载
    svcgen.FxServiceWithProxyModule(),
    adpgen.FxAdapterModule(),
)
```

### 消费配置

任何组件通过构造函数参数注入即可：

```go
func NewSomeComponent(cfg *HzwConfig) *SomeComponent { ... }
```

fx 会自动将 `*HzwConfig` 注入。

---

## 支持的类型

| YAML | Go | 示例 |
|------|-----|------|
| 字符串 | `string` | `name: "foo"` |
| 整数 | `int` | `port: 8080` |
| 浮点 | `float64` | `rate: 0.75` |
| 布尔 | `bool` | `enabled: true` |
| 数组 | `[]string` | `tags: ["a","b"]` |
| Map | `map[string]string` | `key: value` |
| 嵌套结构体 | `struct` | 自动递归 |

---

## 配置文件

### 格式

YAML（`.yml`）为推荐。也支持 `.json`、`.properties`、`.toml`。

### 路径

默认加载可执行文件同目录的 `app.yml`。

### YAML 写法

```yaml
# 占位符
app:
  name: my-service
  fullName: ${app.name}-v1

# 加密
db:
  password: "{cipher}ZW5jb2RlZHBhc3M="
```

---

## 配置优先级

```
-D 命令行 > 环境变量 > Nacos 远程 > app.yml 本地
```

### 命令行传参

```bash
./myapp -Ddb.host=10.0.0.1 -Dserver.port=9090
```

---

## AI 生成规则（禁止项）

| 禁止 | 正确做法 |
|------|----------|
| ✗ 用全局变量存配置 | ✓ 通过 fx 构造函数注入 `ag_conf.IBinder` |
| ✗ 省略 Default 函数 | ✓ 始终提供 `DefaultXxxConfig()` 返回合理默认值 |
| ✗ 必填字段不加 `required:"true"` | ✓ 加上标签，启动时校验 |
| ✗ 手动解析 YAML/JSON | ✓ 用 `binder.Bind(&cfg, key)` |
| ✗ 写死路径、端口 | ✓ 从配置读取 |

---

## FAQ

### Q: 改了 app.yml，为什么没生效？

本地文件修改后**必须重启**。

### Q: Bind 时字段找不到值，为什么没报错？

正常行为。`required:"true"` 才触发校验，否则静默用零值。

### Q: 多个同 key 配置，用哪个？

优先级：**-D > 环境变量 > Nacos > app.yml**。高覆盖低。

### Q: 怎么调试配置加载？

```go
slog.SetLogLoggerLevel(slog.LevelDebug)
// 输出每个 key 的查找过程
```
