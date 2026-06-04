# Kafka Patterns

> **场景触发器**: Kafka 消息、生产者、消费者、agsarama 配置时加载。

> ag-core 提供 `agsarama`（基于 IBM Sarama 的 Kafka 客户端封装）。覆盖配置绑定、FX 模块注册、sarama.Client 注入。生产者/消费者直接使用 sarama API。

---

## 快速开始

### FX 初始化

```go
import "gitlab.allinfinance.com/aifgo/ag-core/contribute/agsarama"

var mainFx = fx.Module("main",
    agsarama.FxAgsaramaModule,   // 自动从 app.yml 读取配置，提供 sarama.Client
    // ...
)
```

> 如果 `agsarama` 包不在 go module 缓存中，先 `go get gitlab.allinfinance.com/aifgo/ag-core/contribute/agsarama@latest`。

### 注入 sarama.Client

通过 `agsarama.FxResult` 注入：

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
| `clientID` | string | `""` | 客户端标识，用于监控 |
| `version` | string | `""` | Kafka 版本，如 `"2.8.0"` |
| `channelBufferSize` | int | `256` | 通道缓冲区大小 |
| `net.dialTimeout` | int | `30000` | 连接超时（ms） |
| `net.readTimeout` | int | `30000` | 读超时（ms） |
| `net.writeTimeout` | int | `30000` | 写超时（ms） |
| `net.keepAlive` | int | `0` | KeepAlive 时间（ms），0=关闭 |
| `net.sasl.enable` | bool | `false` | 启用 SASL 认证 |
| `net.sasl.mechanism` | string | `""` | 认证机制：`plain` / `scram-sha-256` / `scram-sha-512` |
| `net.sasl.user` | string | `""` | SASL 用户名 |
| `net.sasl.password` | string | `""` | SASL 密码 |
| `producer.requiredAcks` | string | `"wait_for_local"` | ACK 策略：`no_response` / `wait_for_local` / `wait_for_all` |
| `producer.compression` | string | `"none"` | 压缩：`none` / `gzip` / `snappy` / `lz4` / `zstd` |
| `producer.partitioner` | string | `"hash"` | 分区策略 |
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

### 同步发送

> ⚠️ 使用 SyncProducer 时，必须在 app.yml 中将 `producer.return.successes` 设为 `true`（默认 false），否则运行时报错。

```go
producer, _ := sarama.NewSyncProducerFromClient(s.client)
defer producer.Close()

partition, offset, err := producer.SendMessage(&sarama.ProducerMessage{
    Topic: "orders",
    Key:   sarama.StringEncoder(orderID),
    Value: sarama.ByteEncoder(jsonBytes),
})
```

### 异步发送（高吞吐）

```go
producer, _ := sarama.NewAsyncProducerFromClient(s.client)
defer producer.Close()

producer.Input() <- &sarama.ProducerMessage{Topic: "orders", Value: sarama.ByteEncoder(data)}

select {
case success := <-producer.Successes():
    slog.Info("sent", "partition", success.Partition, "offset", success.Offset)
case err := <-producer.Errors():
    return err.Err
}
```

---

## 消费者

### 目录约定

```
internal/kafka/
├── consumer.go             # 生命周期管理器（实现 ag_server.Server）
├── composite.go            # 多 handler 路由
├── handler_<topic>.go      # 每个 topic 一个 handler 文件
└── zfx_kafka.go            # fx 模块（注册到 group:"ag_servers"）
```
> 生产者放 biz 层即可，不需要独立 producer.go。

### 生命周期管理器

消费者实现 `ag_server.Server` 接口，归入 `group:"ag_servers"` 跟随 App 统一启动/停止：

```go
// internal/kafka/consumer.go
type KafkaConsumerServer struct {
    group   sarama.ConsumerGroup
    topics  []string
    handler sarama.ConsumerGroupHandler
}

func NewKafkaConsumerServer(client sarama.Client, handler sarama.ConsumerGroupHandler) *KafkaConsumerServer {
    group, _ := sarama.NewConsumerGroupFromClient("app-consumer", client)
    return &KafkaConsumerServer{
        group:   group,
        topics:  topicsFromConfig(),
        handler: handler,
    }
}

func (s *KafkaConsumerServer) Start(ctx context.Context) error {
    go func() {
        for {
            if err := s.group.Consume(ctx, s.topics, s.handler); err != nil {
                slog.ErrorContext(ctx, "consumer error", "err", err)
            }
            if ctx.Err() != nil { return }
        }
    }()
    return nil
}

func (s *KafkaConsumerServer) Stop(ctx context.Context) error {
    return s.group.Close()
}
```

```go
// internal/kafka/zfx_kafka.go
var FxKafkaModule = fx.Module("fx-kafka-module",
    fx.Provide(
        NewCompositeHandler,
        NewStudentHandler,
        NewKafkaConsumerServer,
        fx.Annotate(
            kafkaServerWrapper,
            fx.ResultTags(`group:"ag_servers"`),
        ),
    ),
    fx.Invoke(registerHandlers),
)

func kafkaServerWrapper(s *KafkaConsumerServer) ag_server.Server {
    return s
}

func registerHandlers(composite *CompositeHandler, h *StudentHandler) {
    composite.Register("student-created", h)
}
```

### 多 Handler 注册

不同 topic 路由到不同 handler：

```go
// internal/kafka/composite.go
type TopicHandler interface {
    Handle(ctx context.Context, msg *sarama.ConsumerMessage) error
}

type CompositeHandler struct {
    handlers map[string]TopicHandler
}

func NewCompositeHandler() *CompositeHandler {
    return &CompositeHandler{handlers: make(map[string]TopicHandler)}
}

func (h *CompositeHandler) Register(topic string, handler TopicHandler) {
    h.handlers[topic] = handler
}

func (h *CompositeHandler) Setup(s sarama.ConsumerGroupSession) error   { return nil }
func (h *CompositeHandler) Cleanup(s sarama.ConsumerGroupSession) error { return nil }

func (h *CompositeHandler) ConsumeClaim(sess sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
    handler, ok := h.handlers[claim.Topic()]
    if !ok {
        for range claim.Messages() {}  // drain
        return nil
    }
    for msg := range claim.Messages() {
        if err := handler.Handle(sess.Context(), msg); err != nil {
            continue  // 失败不 ack → 下次重试
        }
        sess.MarkMessage(msg, "")
    }
    return nil
}
```

Handler 通过 `fx.Invoke(registerHandlers)` 注册：

```go
// internal/kafka/zfx_kafka.go（见上方）
func registerHandlers(composite *CompositeHandler, h *StudentHandler) {
    composite.Register("student-created", h)
}
```

### 单个 Handler 示例

```go
// internal/kafka/handler_order.go
type OrderHandler struct {
    orderService *service.OrderServiceImpl
}

func (h *OrderHandler) Handle(ctx context.Context, msg *sarama.ConsumerMessage) error {
    var event OrderEvent
    if err := json.Unmarshal(msg.Value, &event); err != nil {
        return err
    }
    return h.orderService.OnOrderCreated(ctx, &event)
}
```

### 序列化

producer 和 consumer 协商一致即可，常用 JSON。跨服务通信可用 protobuf。不强制统一格式，由业务团队约定。

---

## 最佳实践

| ✅ 正确 | ❌ 错误 |
|------|------|
| 通过 `agsarama.FxAgsaramaModule` 注入 `sarama.Client` | 手动 `sarama.NewClient(brokers, cfg)` |
| 消费者不耗时操作 | ConsumeClaim 中同步调用远程接口 |
| 处理成功 `MarkMessage`，失败不 ack | 无论成败都 ack |
| 生产者复用 client，每次创建 producer | 每次创建新的 sarama.Client |
| `MarkMessage` 后消息不会重复消费 | 未 MarkMessage 就认为处理完成 |

---

> **高级用法**（事务消息、自定义分区器、Lag 监控、Schema Registry）不在 ag-core 封装范围内，使用 Sarama 原生 API 实现。

## 验证

**注入完整性**：
□ `cmd/server/main.go` — `agsarama.FxAgsaramaModule` 已声明
□ app.yml 含 `agsarama` 配置段
