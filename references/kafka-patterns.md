# Kafka Patterns

> **场景触发器**: Kafka 消息、生产者、消费者、agsarama 配置时加载。

> ag-core 提供 `agsarama`（基于 IBM Sarama 的 Kafka 客户端封装）。覆盖配置绑定、FX 模块注册、sarama.Client 注入。生产者/消费者直接使用 sarama API。

---

## 目录约定

```
internal/kafka/              ← 基础设施（接口 + 生命周期）
├── producer.go
├── consumer.go
├── composite.go
└── zfx_kafka.go             # AsKafkaHandler（导出）+ producer lifecycle

internal/kafkahandler/        ← 业务适配（handler 实现，详见 kafka-consumer-patterns）
├── handler_<业务>.go
└── zfx_handler.go
```

---

## 快速开始

### FX 初始化

```go
import "github.com/aif-go/ag-core/contribute/agsarama"

var mainFx = fx.Module("main",
    agsarama.FxAgsaramaModule,   // 自动从 app.yml 读取配置，提供 sarama.Client
    // ...
)
```

> 如果 `agsarama` 包不在 go module 缓存中，先 `go get github.com/aif-go/ag-core/contribute/agsarama@latest`。

### 注入 sarama.Client

通过 `agsarama.FxResult` 注入。`FxResult` 含 3 个出参字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `Config` | `*agsarama.Config` | agsarama 友好配置（YAML 绑定结果） |
| `SaramaConfig` | `*sarama.Config` | 转换后的 Sarama 原生配置 |
| `Client` | `sarama.Client` | Sarama 客户端（最常用） |

> 多数场景只需 `result.Client`；需自建 producer/consumer 时可用 `result.SaramaConfig` 直接 `sarama.NewSyncProducer(brokers, cfg)` / `sarama.NewConsumerGroup(...)`。

```go
type OrderService struct {
    client sarama.Client
}

func NewOrderService(result agsarama.FxResult) *OrderService {
    return &OrderService{client: result.Client}
}
```

---

## 配置

配置前缀：`agsarama`

### 字段说明

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `brokers` | `[]string` | — | Kafka broker 地址列表 |
| `clientID` | string | `"agsarama"` | 客户端标识，用于监控 |
| `version` | string | `""` | Kafka 版本，如 `"2.8.0"` |
| `channelBufferSize` | int | `256` | 通道缓冲区大小 |
| `net.dialTimeout` | int | `30000` | 连接超时（ms） |
| `net.readTimeout` | int | `30000` | 读超时（ms） |
| `net.writeTimeout` | int | `30000` | 写超时（ms） |
| `net.keepAlive` | int | `0` | KeepAlive 时间（ms），0=关闭 |
| `net.sasl.enable` | bool | `false` | 启用 SASL 认证 |
| `net.sasl.mechanism` | string | `""` | 认证机制：`plain` / `scram-sha-256` / `scram-sha-512` / `oauth` / `gssapi` |
| `net.sasl.user` | string | `""` | SASL 用户名 |
| `net.sasl.password` | string | `""` | SASL 密码 |
| `producer.requiredAcks` | string | `"wait_for_local"` | ACK 策略：`no_response` / `wait_for_local` / `wait_for_all` |
| `producer.compression` | string | `"none"` | 压缩：`none` / `gzip` / `snappy` / `lz4` / `zstd` |
| `producer.partitioner` | string | `"hash"` | 分区策略：`hash` / `manual` / `random` / `roundrobin` |
| `producer.maxMessageBytes` | int | `1048576` | 最大消息字节数（默认 1MB） |
| `producer.timeout` | int | `10000` | 发送超时（ms） |
| `producer.idempotent` | bool | `false` | 幂等生产者，防重复 |
| `producer.retry.max` | int | `3` | 生产者重试次数 |
| `producer.retry.backoff` | int | `100` | 重试间隔（ms） |
| `producer.return.successes` | bool | `false` | 返回成功确认通道（SyncProducer 设为 `true`） |
| `producer.return.errors` | bool | `true` | 返回错误通道 |
| `consumer.group.session.timeout` | int | `10000` | 会话超时（ms） |
| `consumer.group.heartbeat.interval` | int | `3000` | 心跳间隔（ms），建议 session.timeout/3 |
| `consumer.group.rebalance.timeout` | int | `60000` | 重平衡超时（ms） |
| `consumer.offsets.autoCommit.enable` | bool | `true` | 自动提交 offset |
| `consumer.offsets.autoCommit.interval` | int | `1000` | 自动提交间隔（ms） |
| `consumer.offsets.initial` | int | `-1` | 初始 offset：`-1`=最新，`-2`=最早 |
| `consumer.isolationLevel` | string | `"read_uncommitted"` | 隔离级别：`read_uncommitted` / `read_committed` |

> **键名映射**：YAML key 按 Go 字段名做 EqualFold 大小写不敏感匹配（如 `sid` 等于 `SId`）。若绑定失败，字段保持零值不报错。遇到配置不生效时，对比 app.yml 的键名与字段名拼写是否一致。

> **partitioner 取值**：`hash`（默认，按 key hash 分区，未设 key 则随机）/ `manual`（按 `message.PartitionKey` 手动指定）/ `random` / `roundrobin`。**大小写不敏感**（`manual`/`Manual`/`MANUAL` 均可），**无效值返回 error，不再静默降级**。

### 最小配置

```yaml
agsarama:
  brokers:
    - 127.0.0.1:9092
```

### 带认证的生产环境

```yaml
agsarama:
  brokers:
    - kafka-1:9092
    - kafka-2:9092
    - kafka-3:9092
  clientID: my-app
  version: "2.8.0"

  net:
    sasl:
      enable: true
      mechanism: scram-sha-256
      user: myuser
      password: "{cipher}xxxxx"    # 加密密码

  producer:
    requiredAcks: wait_for_all
    compression: snappy
    idempotent: true
    return:
      successes: true         # SyncProducer 必需

  consumer:
    group:
      session:
        timeout: 30000
    offsets:
      autoCommit:
        enable: true
      initial: -1
```

---

## 生产者

producer 由 fx 管理生命周期（构造 → 注入 → 关闭）。biz 层直接注入 `sarama.SyncProducer` 或 `sarama.AsyncProducer`，无需手动创建/关闭。

### 构造器

```go
// internal/kafka/producer.go
func NewSyncProducer(client sarama.Client) (sarama.SyncProducer, error) {
    return sarama.NewSyncProducerFromClient(client)
}

func NewAsyncProducer(client sarama.Client) (sarama.AsyncProducer, error) {
    return sarama.NewAsyncProducerFromClient(client)
}
```

### fx 注册 + 生命周期

```go
// internal/kafka/zfx_kafka.go
var FxKafkaModule = fx.Module("fx-kafka-module",
    fx.Provide(
        NewSyncProducer,
        NewAsyncProducer,
        // consumer providers 详见 [[kafka-consumer-patterns]]
    ),
    fx.Invoke(syncProducerLifecycle),
    fx.Invoke(asyncProducerLifecycle),
)

func syncProducerLifecycle(lc fx.Lifecycle, producer sarama.SyncProducer) {
    lc.Append(fx.Hook{
        OnStop: func(ctx context.Context) error { return producer.Close() },
    })
}

func asyncProducerLifecycle(lc fx.Lifecycle, producer sarama.AsyncProducer) {
    go func() { for range producer.Successes() {} }()
    go func() {
        for e := range producer.Errors() {
            slog.Error("async producer error", "err", e)
        }
    }()
    lc.Append(fx.Hook{
        OnStop: func(ctx context.Context) error { return producer.Close() },
    })
}
```

### 同步发送

> ⚠️ `SyncProducer` 要求 `producer.return.successes: true`（默认 false），否则运行时报错。

```go
type OrderBiz struct {
    producer sarama.SyncProducer   // 直接注入，无需 create/close
}

func (b *OrderBiz) CreateOrder(ctx context.Context, order *Order) error {
    data, _ := json.Marshal(order)
    _, _, err := b.producer.SendMessage(&sarama.ProducerMessage{
        Topic: "orders",
        Key:   sarama.StringEncoder(order.ID),
        Value: sarama.ByteEncoder(data),
    })
    if err != nil {
        return err
    }
    slog.InfoContext(ctx, "order produced", "id", order.ID)
    return nil
}
```

### 异步发送（高吞吐）

```go
type OrderBiz struct {
    producer sarama.AsyncProducer   // 直接注入，无需 create/close
}

func (b *OrderBiz) CreateOrder(ctx context.Context, order *Order) error {
    data, _ := json.Marshal(order)
    b.producer.Input() <- &sarama.ProducerMessage{
        Topic: "orders",
        Key:   sarama.StringEncoder(order.ID),
        Value: sarama.ByteEncoder(data),
    }
    return nil
}
```

### 轻量替代：注入 Client + 临时 producer

简单或低频发送场景，也可只注入 `sarama.Client`，每次发送时从 client 创建 producer 并及时 `Close()`：

```go
type OrderService struct {
    client sarama.Client   // 注入 agsarama.FxResult.Client
}

func (s *OrderService) Publish(ctx context.Context, topic string, value []byte) error {
    producer, err := sarama.NewSyncProducerFromClient(s.client)
    if err != nil {
        return err
    }
    defer producer.Close()
    _, _, err = producer.SendMessage(&sarama.ProducerMessage{
        Topic: topic,
        Value: sarama.ByteEncoder(value),
    })
    return err
}
```

> ⚠️ 必须复用同一个注入的 `sarama.Client`（连接开销大），仅每次新建轻量 producer——**不要每次 `sarama.NewClient`**。
>
> **选型权衡**：高频 / 固定 topic → 用上面的 fx 托管 SyncProducer/AsyncProducer 单例（免去每次创建开销）；低频 / 简单 / topic 多变 → 用本方式 client 临时创建，代码更直接。

---

## 消费者

消费者实现 `ag_server.Server`，归入 `group:"ag_servers"` 跟随 App 统一管理。详见 [[kafka-consumer-patterns]]。

## 最佳实践

| ✅ 正确 | ❌ 错误 |
|------|------|
| 通过 `agsarama.FxAgsaramaModule` 注入 `sarama.Client` | 手动 `sarama.NewClient(brokers, cfg)` |
| 高频固定 topic：biz 注入 fx 托管的 `sarama.SyncProducer`/`AsyncProducer` 单例；低频简单场景：注入 `Client` 临时创建 producer | 每次发送都 `sarama.NewClient` 重建客户端 |
| producer 创建失败阻止启动（`NewSyncProducer` 返回 error） | 创建失败 log 后继续，消息丢失 |
| AsyncProducer 的 Successes/Errors 通道持续 drain | 不 drain 导致 goroutine 泄漏 |

---

## 验证

**注入完整性**：
□ `cmd/server/main.go` — `agsarama.FxAgsaramaModule` 已声明
□ app.yml 含 `agsarama` 配置段
□ `producer.return.successes: true`（使用 SyncProducer 时）
□ consumer 已注册到 `group:"ag_servers"`（详见 [[kafka-consumer-patterns]]）

**生命周期**：
□ producer fx hook 已注册 `lc.Append(fx.Hook{OnStop: ...})`
□ AsyncProducer 的 Successes/Errors 通道已 drain
