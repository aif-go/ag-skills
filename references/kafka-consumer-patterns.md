# Kafka Consumer Patterns

> **场景触发器**: Kafka 消费者、消费组、Consumer Group、Consumer Server、Multi-Handler 时加载。

---

## 目录约定

```
internal/kafka/
├── consumer.go             # 生命周期管理器（实现 ag_server.Server）
├── composite.go            # 多 handler 路由 + topic 映射
├── handler_<业务>.go       # 每个业务标识一个 handler 文件
└── zfx_kafka.go            # fx 模块（注册到 group:"ag_servers"）
```

---

## 生命周期管理器

消费者实现 `ag_server.Server` 接口，归入 `group:"ag_servers"` 跟随 App 统一启动/停止。

**核心设计**：自行管理 context（不依赖 App 传入），Stop 时先 cancel 再 close group。

### 配置

`app.yml` 中 `routes` 定义业务标识→实际 topic 的映射。`topics` 中的值可以是 route key 或直接 topic 名：

```yaml
kafka:
  consumer:
    routes:                               # 业务标识 → 实际 topic
      student-event: student-created      # 生产
      # student-event: dev-student         # 开发
    groups:
      - groupID: "order-group"
        topics: ["student-event", "order-paid"]  # route key / 直接 topic
```

```go
// internal/kafka/consumer.go
type KafkaConsumerServer struct {
    client  sarama.Client
    handler sarama.ConsumerGroupHandler
    groups  map[string]sarama.ConsumerGroup

    ctx     context.Context
    cancel  context.CancelFunc
    config  *ConsumerConfig
}

type ConsumerConfig struct {
    Routes map[string]string          // 业务标识 → 实际 topic
    Groups []ConsumerGroupConfig
}
type ConsumerGroupConfig struct {
    GroupID string
    Topics  []string                   // route key 或直接 topic 名
}

func NewKafkaConsumerServer(client sarama.Client, handler *CompositeHandler, config *ConsumerConfig) *KafkaConsumerServer {
    ctx, cancel := context.WithCancel(context.Background())
    return &KafkaConsumerServer{
        client:  client,
        handler: handler,
        groups:  make(map[string]sarama.ConsumerGroup),
        ctx:     ctx,
        cancel:  cancel,
        config:  config,
    }
}

// resolveTopics 将 topics 中的 route key 解析为实际 topic
func resolveTopics(topics []string, routes map[string]string) []string {
    var result []string
    for _, t := range topics {
        if actual, ok := routes[t]; ok {
            result = append(result, actual)
        } else {
            result = append(result, t)
        }
    }
    return result
}

func (s *KafkaConsumerServer) Start(_ context.Context) error {
    for _, cfg := range s.config.Groups {
        actualTopics := resolveTopics(cfg.Topics, s.config.Routes)
        group, _ := sarama.NewConsumerGroupFromClient(cfg.GroupID, s.client)
        s.groups[cfg.GroupID] = group
        go s.consumeGroup(cfg.GroupID, group, actualTopics)
    }
    return nil
}

func (s *KafkaConsumerServer) consumeGroup(groupID string, group sarama.ConsumerGroup, topics []string) {
    defer func() {
        if err := group.Close(); err != nil {
            slog.Error("consumer group close failed", "group", groupID, "err", err)
        }
    }()
    for {
        select {
        case <-s.ctx.Done():
            return
        default:
            err := group.Consume(context.Background(), topics, s.handler)
            if err != nil {
                if err == sarama.ErrClosedConsumerGroup {
                    return
                }
                slog.Error("consumer error, retry", "group", groupID, "err", err)
                time.Sleep(time.Second)
            }
        }
    }
}

func (s *KafkaConsumerServer) Stop(_ context.Context) error {
    s.cancel()
    for _, group := range s.groups {
        group.Close()
    }
    return nil
}
```

### fx 注册

```go
// internal/kafka/zfx_kafka.go
var FxKafkaModule = fx.Module("fx-kafka-module",
    fx.Provide(
        NewConsumerConfig,          // *ConsumerConfig — 从 app.yml 绑定
        NewCompositeHandler,        // 依赖 ConsumerConfig.Routes
        NewStudentHandler,
        NewKafkaConsumerServer,
        fx.Annotate(
            kafkaServerWrapper,
            fx.ResultTags(`group:"ag_servers"`),
        ),
    ),
    fx.Invoke(registerHandlers),
)

func NewCompositeHandler(cfg *ConsumerConfig) *CompositeHandler {
    var expected []string
    for _, g := range cfg.Groups {
        expected = append(expected, g.Topics...)
    }
    return newCompositeHandler(expected, cfg.Routes)
}

func kafkaServerWrapper(s *KafkaConsumerServer) ag_server.Server {
    return s
}

func registerHandlers(composite *CompositeHandler, h *StudentHandler) {
    composite.Register("student-event", h)
}
```

---

## Multi-Handler 路由

### CompositeHandler

Handler 用**业务标识**注册（不是 topic 名）。ConsumeClaim 通过 `reverseRoutes` 将实际 topic 反向解析为标识，再查找 handler：

```go
// internal/kafka/composite.go
type TopicHandler interface {
    Handle(ctx context.Context, msg *sarama.ConsumerMessage) error
}

type CompositeHandler struct {
    handlers       map[string]TopicHandler  // 业务标识 → handler
    reverseRoutes  map[string]string          // 实际 topic → 业务标识
    expectedTopics []string                   // 所有声明的标识
}

func newCompositeHandler(expectedTopics []string, routes map[string]string) *CompositeHandler {
    reverse := make(map[string]string)
    for k, v := range routes {
        reverse[v] = k
    }
    return &CompositeHandler{
        handlers:       make(map[string]TopicHandler),
        reverseRoutes:  reverse,
        expectedTopics: expectedTopics,
    }
}

func (h *CompositeHandler) Register(key string, handler TopicHandler) {
    h.handlers[key] = handler
}

func (h *CompositeHandler) Setup(_ sarama.ConsumerGroupSession) error {
    for _, key := range h.expectedTopics {
        if _, ok := h.handlers[key]; !ok {
            return fmt.Errorf("no handler registered for key: %s", key)
        }
    }
    return nil
}

func (h *CompositeHandler) Cleanup(sarama.ConsumerGroupSession) error { return nil }

func (h *CompositeHandler) ConsumeClaim(sess sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
    key := claim.Topic()
    if k := h.reverseRoutes[key]; k != "" {
        key = k
    }
    handler, ok := h.handlers[key]
    if !ok {
        for range claim.Messages() {}
        return fmt.Errorf("no handler for topic: %s", claim.Topic())
    }
    for msg := range claim.Messages() {
        if err := handler.Handle(sess.Context(), msg); err != nil {
            slog.Error("handle failed", "topic", msg.Topic, "err", err)
        }
        sess.MarkMessage(msg, "")
    }
    return nil
}
```

---

## Handler 示例

```go
// internal/kafka/handler_student.go
type StudentHandler struct{}

func NewStudentHandler() *StudentHandler {
    return &StudentHandler{}
}

func (h *StudentHandler) Handle(ctx context.Context, msg *sarama.ConsumerMessage) error {
    var event StudentEvent
    if err := json.Unmarshal(msg.Value, &event); err != nil {
        return err
    }
    slog.InfoContext(ctx, "学生已创建: "+event.Stuno+" - "+event.Name)
    return nil
}
```

### 序列化

producer 和 consumer 协商一致即可，常用 JSON。不强制统一格式，由业务团队约定。

---

## 最佳实践

| ✅ 正确 | ❌ 错误 |
|------|------|
| 消费者不耗时操作 | ConsumeClaim 中同步调用远程接口 |
| 无论成败都 `MarkMessage`（offset 提交是连续水位） | 失败不 ack 期望重试（后续成功 ack 会覆盖） |
| Handler 用业务标识注册（`"student-event"`），topic 名在 YAML 管理 | 代码中硬编码 topic 字符串 |
| 环境差异通过 `routes` 映射切换，代码不变 | 环境切换时改 handler 中的 topic 常量 |

---

## 验证

**注入完整性**：
□ consumer 实现 `ag_server.Server` 并通过 wrapper 归入 `group:"ag_servers"`
□ `internal/zfx_internal.go` 含 Kafka 模块
□ `app.yml` 中 `routes` 映射覆盖所有业务标识
