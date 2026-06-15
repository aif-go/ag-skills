# Redis Patterns

> **场景触发器**: Redis 缓存、Redis 配置、缓存操作、agredis 配置时加载。

---

## 快速开始

### FX 初始化

```go
import "github.com/aif-go/ag-core/contribute/agredis"

var mainFx = fx.Module("main",
    agredis.FxAgRedisServerMode,   // 自动从 app.yml 读取配置初始化
    // ...
)
```

### 注入 AgRedisClient

```go
import "github.com/aif-go/ag-core/contribute/agredis"

type StudentBiz struct {
    redis agredis.AgRedisClient
}

func NewStudentBiz(cli agredis.AgRedisClient) *StudentBiz {
    return &StudentBiz{redis: cli}
}
```

---

## 配置

配置前缀：`agredis`

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `type` | string | — | `universal`（推荐）或 `rw` |
| `config.addrs` | `[]string` | — | Redis 地址列表 |
| `config.db` | int | `0` | 数据库编号 |
| `config.username` | string | `""` | 用户名 |
| `config.password` | string | `""` | 密码 |
| `config.poolSize` | int | `10` | 连接池大小 |
| `config.dialTimeout` | int | `5000` | 连接超时（ms） |
| `config.readTimeout` | int | `3000` | 读超时（ms） |
| `config.writeTimeout` | int | `3000` | 写超时（ms） |
| `config.maxRetries` | int | `0` | 最大重试次数 |
| `config.connMaxIdleTime` | int | `300000` | 连接最大空闲时间（ms） |

> **键名映射**：YAML key 按 Go 字段名做 EqualFold 大小写不敏感匹配。若配置不生效，对比 key 拼写与字段名是否一致。

### 单机模式（最常用）

```yaml
agredis:
  type: universal
  config:
    addrs:
      - 127.0.0.1:6379
    db: 0
    poolSize: 10
```

### 集群模式

```yaml
agredis:
  type: universal
  config:
    addrs:
      - node1:6379
      - node2:6379
      - node3:6379
```

### Sentinel 模式

```yaml
agredis:
  type: universal
  config:
    addrs:
      - sentinel1:26379
      - sentinel2:26379
    masterName: mymaster
    password: "mypass"
```

### 读写分离模式

```yaml
agredis:
  type: rw
  config:
    addrs:
      - master:6379
  replicas:
    - addrs:
      - slave1:6379
    - addrs:
      - slave2:6379
```

---

## 基本操作

### String

```go
cli.Set(ctx, "key", "value", 10*time.Minute)       // 写，带 TTL
val, _ := cli.Get(ctx, "key").Result()               // 读
cli.Incr(ctx, "counter")                             // 自增
cli.SetNX(ctx, "lock:order", "1", 10*time.Second)    // 分布式锁
```

### Hash

```go
cli.HSet(ctx, "user:1", "name", "Alice", "age", 30)
name, _ := cli.HGet(ctx, "user:1", "name").Result()
all, _ := cli.HGetAll(ctx, "user:1").Result()
cli.HDel(ctx, "user:1", "age")
```

### List

```go
cli.LPush(ctx, "queue:tasks", taskJSON)
task, _ := cli.RPop(ctx, "queue:tasks").Result()
length, _ := cli.LLen(ctx, "queue:tasks").Result()
```

### Set / ZSet

```go
cli.SAdd(ctx, "tags", "golang", "redis")
members, _ := cli.SMembers(ctx, "tags").Result()

cli.ZAdd(ctx, "leaderboard", redis.Z{Score: 100, Member: "player1"})
rank, _ := cli.ZRank(ctx, "leaderboard", "player1").Result()
```

### 通用

```go
cli.Del(ctx, "key1", "key2")
exists, _ := cli.Exists(ctx, "key").Result()
cli.Expire(ctx, "key", 24*time.Hour)
ttl, _ := cli.TTL(ctx, "key").Result()
```

---

## 最佳实践

| ✅ 正确 | ❌ 错误 |
|------|------|
| 依赖接口 `AgRedisClient` | 手动创建 redis.Client |
| Key 带业务前缀 `student:detail:{id}` | 裸 key `detail` |
| 高频读取用缓存 + 合理 TTL | 不设 TTL 导致内存溢出 |
| `Universal` 模式优先（自动适配单机/集群/Sentinel） | 写死单机模式 |

---

## 验证

**注入完整性**：
□ `cmd/server/main.go` — `agredis.FxAgRedisServerMode` 已声明
□ app.yml 含 `agredis` 配置段
