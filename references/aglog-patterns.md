# AgLog Patterns

> **场景触发器**: 日志配置、日志级别、结构化日志、aglog 配置、日志文件轮转时加载。

---

## 快速开始

### FX 初始化

```go
import (
    "gitlab.allinfinance.com/aifgo/ag-core/ag/ag_log"
    "go.uber.org/fx"
)

func main() {
    fxapp := fx.New(
        ag_log.FxAglogMode,     // 自动从 app.yml 读取配置初始化
    )
    fxapp.Run()
}
```

### 代码中使用

```go
import "gitlab.allinfinance.com/aifgo/ag-core/ag/ag_log/agslog"

// 获取顶层 Logger
logger := agslog.GetSlog()

// 推荐：始终用 InfoContext 系列，自动携带 ctx 中的 trace 信息
logger.InfoContext(ctx, "订单创建", "order_id", id, "amount", amount)
logger.ErrorContext(ctx, "数据库错误", "err", err)
logger.DebugContext(ctx, "调试信息")
logger.WarnContext(ctx, "警告", "detail", msg)

// 带分组
logger.WithGroup("request").
    With("method", "POST").
    InfoContext(ctx, "收到请求")
```

> `isDefault: true`（默认）时，标准库 `slog.Info()` 也走 ag-log 管道。

---

## 模块专属日志

不同模块写不同文件，按名称获取独立 Logger：

```yaml
aglog:
  zap:
    logs:
      trade:
        log_level: info
        log_file_name: "logs/trade.log"
      system:
        log_level: info
        log_file_name: "logs/system.log"
```

```go
tradeLog := agslog.GetSlogByName("trade")
tradeLog.InfoContext(ctx, "订单创建", "order_id", "O001")

systemLog := agslog.GetSlogByName("system")
systemLog.InfoContext(ctx, "心跳正常", "node", "192.168.1.1")
```

同一个名称始终拿到同一个 Logger 实例。

---

## 级别与格式

| 级别 | 说明 |
|------|------|
| `debug` | 调试信息，仅开发 |
| `info` | 普通信息，默认级别 |
| `warn` | 警告，不一定是错误 |
| `error` | 错误，需关注 |

| 格式 | `console` | `encoding` | 说明 |
|------|:---:|------|------|
| 控制台彩色 | `true` | — | 人类可读，带颜色 |
| JSON | `false` | `json` | 机器解析，生产推荐 |
| 文件输出 | — | — | 指定 `log_file_name` 路径 |

---

## 配置

```yaml
aglog:
  isDefault: true              # 替换 slog 全局默认，默认 true
  topHandler:                  # 顶层 handler 名称列表
    - "default"
  zap:
    logs:
      default:
        log_level: debug       # debug / info / warn / error
        stdout: true           # 输出到 stdout
        console: true          # 控制台彩色格式
        log_file_name: "logs/app.log"   # 日志文件路径
        max_size: 512          # 单文件最大 MB，默认 100
        max_backups: 30        # 保留备份数，0=不限
        max_age: 7             # 保留天数，0=不限
        compress: true         # 压缩归档
```

### 环境切换

| 维度 | 开发 | 生产 |
|------|------|------|
| 级别 | `debug` | `info` |
| 格式 | `console: true` | `console: false` |
| 输出 | stdout + console | file only |
| 轮转 | 不轮转 | `max_size`/`max_backups`/`max_age` |

---

## 生产最佳实践

| ✅ 正确 | ❌ 错误 |
|------|------|
| `logger.InfoContext(ctx, "创建", "id", id)` | `logger.InfoContext(ctx, fmt.Sprintf("创建 id=%d", id))` |
| 错误日志带上下文 `"err", err` | 只打印 `err.Error()` |
| 高频循环中打 `Debug` | 循环中打 `Info` |
| 生产用 `info` 级别 + JSON 格式 | 生产用 `debug` |
| `GetSlogByName` 分模块日志 | 全局单一 Logger |

---

## 验证

**注入完整性**：
□ `cmd/server/main.go` — `ag_log.FxAglogMode` 已声明
