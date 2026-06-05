# Kafka Consumer Patterns

> **场景触发器**: Kafka 消费者、消费组、Consumer Group、Consumer Server、Multi-Handler 时加载。

---

## 目录约定

```
internal/kafka/              ← 基础设施（接口 + 生命周期），不依赖 biz
├── consumer.go
├── composite.go              # TopicHandler 接口 + 路由
├── producer.go
└── zfx_kafka.go              # AsKafkaHandler（导出）+ FxKafkaModule

internal/kafkahandler/        ← 业务适配（实现 TopicHandler），依赖 kafka + biz
├── handler_<业务>.go
└── zfx_handler.go            # FxHandlerModule
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

### 构造函数 + 集中校验

`NewKafkaConsumerServer` 返回 `(*KafkaConsumerServer, error)`，在容器启动阶段完成所有校验，失败阻止启动：

```go
// internal/kafka/consumer.go
import (
    "context"
    "fmt"
    "time"

    "github.com/IBM/sarama"
)

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

func NewKafkaConsumerServer(client sarama.Client, handler *CompositeHandler, config *ConsumerConfig) (*KafkaConsumerServer, error) {
    if len(config.Groups) == 0 {
        return nil, fmt.Errorf("kafka consumer: no groups configured")
    }
    for _, cfg := range config.Groups {
        if len(cfg.Topics) == 0 {
            return nil, fmt.Errorf("kafka consumer: group %q has empty topics", cfg.GroupID)
        }
        actual := resolveTopics(cfg.Topics, config.Routes)
        if len(actual) == 0 {
            return nil, fmt.Errorf("kafka consumer: group %q has no valid topics after route resolution", cfg.GroupID)
        }
        for _, key := range cfg.Topics {
            if !handler.HasHandler(key) {
                return nil, fmt.Errorf("kafka consumer: key %q declared in group %q but no handler registered", key, cfg.GroupID)
            }
        }
    }

    // ⑤: 多 key 映射同一 topic
    seen := make(map[string]string)
    for k, v := range config.Routes {
        if prev, ok := seen[v]; ok {
            slog.Warn("kafka consumer: duplicate route target", "topic", v, "keys", []string{prev, k})
        }
        seen[v] = k
    }

    // ⑥: routes 残留 key（无 group 引用）
    refd := make(map[string]bool)
    for _, cfg := range config.Groups {
        for _, k := range cfg.Topics {
            refd[k] = true
        }
    }
    for k := range config.Routes {
        if !refd[k] {
            slog.Warn("kafka consumer: route key not referenced by any group", "key", k)
        }
    }
    // ⑦: handler 注册了但无 group 声明
    for _, key := range handler.RegisteredKeys() {
        if !refd[key] {
            slog.Warn("kafka consumer: handler registered but not declared in any group", "key", key)
        }
    }

    ctx, cancel := context.WithCancel(context.Background())
    return &KafkaConsumerServer{
        client:  client,
        handler: handler,
        groups:  make(map[string]sarama.ConsumerGroup),
        ctx:     ctx,
        cancel:  cancel,
        config:  config,
    }, nil
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
        group, err := sarama.NewConsumerGroupFromClient(cfg.GroupID, s.client)
        if err != nil {
            return fmt.Errorf("kafka consumer: create group %q: %w", cfg.GroupID, err)
        }
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

`AsKafkaHandler` 导出为公共函数。基础设施模块（`kafka.FxKafkaModule`）不注册任何 handler，handler 在独立的 `kafkahandler.FxHandlerModule` 中通过 `AsKafkaHandler` 注入：

```go
// internal/kafka/zfx_kafka.go
func AsKafkaHandler(fn any) any {                     // 导出为公共函数
    return fx.Annotate(fn,
        fx.As(new(TopicHandler)),
        fx.ResultTags(`group:"kafka_handlers"`),
    )
}

var FxKafkaModule = fx.Module("fx-kafka-module",
    fx.Provide(
        NewConsumerConfig,
        NewCompositeHandler,
        NewKafkaConsumerServer,
        fx.Annotate(
            kafkaServerWrapper,
            fx.ResultTags(`group:"ag_servers"`),
        ),
    ),
)

// internal/kafkahandler/zfx_handler.go
var FxHandlerModule = fx.Module("fx-kafka-handler",
    fx.Provide(
        kafka.AsKafkaHandler(NewStudentHandler),
        kafka.AsKafkaHandler(NewOrderHandler),
    ),
)

// internal/zfx_internal.go
var FxInternalModule = fx.Module("fx-internal-module",
    // ...
    kafka.FxKafkaModule,
    kafkahandler.FxHandlerModule,
)
```

> `kafka` 包不 import `kafkahandler`，通过 fx `group:"kafka_handlers"` 在运行时连接，无编译期循环依赖。

func kafkaServerWrapper(s *KafkaConsumerServer) ag_server.Server {
    return s
}
```

---

## Multi-Handler 路由

### TopicHandler 接口

Handler 通过 `TopicKey()` 自声明业务标识，`CompositeHandler` 在构造时自动注册：

```go
// internal/kafka/composite.go
import (
    "context"
    "fmt"

    "github.com/IBM/sarama"
    "go.uber.org/fx"
)

type TopicHandler interface {
    TopicKey() string       // 业务标识，如 "student-event"
    Handle(ctx context.Context, msg *sarama.ConsumerMessage) error
}
```

### CompositeHandler

通过 `fx.In` 收集所有 handler，与 `ConsumerConfig` 一并注入：

```go
type CompositeHandlerParams struct {
    fx.In
    Handlers []TopicHandler `group:"kafka_handlers"`
    Config   *ConsumerConfig
}

type CompositeHandler struct {
    handlers       map[string]TopicHandler  // 业务标识 → handler
    reverseRoutes  map[string]string          // 实际 topic → 业务标识
}

func NewCompositeHandler(p CompositeHandlerParams) *CompositeHandler {
    reverse := make(map[string]string)
    for k, v := range p.Config.Routes {
        reverse[v] = k
    }

    h := &CompositeHandler{
        handlers:       make(map[string]TopicHandler),
        reverseRoutes:  reverse,
    }
    for _, handler := range p.Handlers {
        key := handler.TopicKey()
        if _, exists := h.handlers[key]; exists {
            slog.Warn("kafka consumer: duplicate handler key", "key", key)
        }
        h.handlers[key] = handler
    }
    return h
}

func (h *CompositeHandler) HasHandler(key string) bool {
    _, ok := h.handlers[key]
    return ok
}

func (h *CompositeHandler) RegisteredKeys() []string {
    var keys []string
    for k := range h.handlers {
        keys = append(keys, k)
    }
    return keys
}

func (h *CompositeHandler) Setup(_ sarama.ConsumerGroupSession) error {
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

Handler 放在 `internal/kafkahandler/` 包，注入 `biz` 处理业务逻辑。`Event` 结构体定义在 `biz` 层（领域层），handler 只做适配（反序列化 → 委托 biz）：

```go
// internal/kafkahandler/student.go
import (
    "context"
    "encoding/json"
    "log/slog"

    "hello/internal/biz"

    "github.com/IBM/sarama"
)

type StudentHandler struct {
    biz *biz.StudentBiz
}

func NewStudentHandler(b *biz.StudentBiz) *StudentHandler {
    return &StudentHandler{biz: b}
}

func (h *StudentHandler) TopicKey() string {
    return "student-event"
}

func (h *StudentHandler) Handle(ctx context.Context, msg *sarama.ConsumerMessage) error {
    var event biz.StudentCreatedEvent
    if err := json.Unmarshal(msg.Value, &event); err != nil {
        slog.ErrorContext(ctx, "unmarshal student event failed", "err", err)
        return err
    }
    return h.biz.OnStudentCreated(ctx, &event)
}
```

```go
// internal/biz/student_biz.go
type StudentCreatedEvent struct {
    Id    int64  `json:"id"`
    Stuno string `json:"stuno"`
    Name  string `json:"name"`
}

func (b *StudentBiz) OnStudentCreated(ctx context.Context, e *StudentCreatedEvent) error {
    slog.InfoContext(ctx, "学生已创建: "+e.Stuno+" - "+e.Name)
    return nil
}
```

### 序列化

producer 和 consumer 协商一致即可，常用 JSON。不强制统一格式，由业务团队约定。

---

## 校验体系

### NewKafkaConsumerServer — 启动防火墙（返回 error 阻止启动）

| # | 校验项 | 失败 |
|---|--------|------|
| ① | `Groups` 为空 | `return nil, error` |
| ② | 任一 group 的 `Topics` 为空 | `return nil, error` |
| ③ | `resolveTopics()` 结果为空（所有 key 全解析失败） | `return nil, error` |
| ④ | declared key 无对应 handler | `return nil, error` |
| ⑤ | 多 key 映射同一 topic | warn |
| ⑥ | routes 残留 key（无 group 引用） | warn |
| ⑦ | handler 注册了但无 group 声明 | warn |

### ConsumeClaim — 运行时兜底

| # | 场景 | 处理 |
|---|------|------|
| ⑧ | 收到未知 topic | drain + `return error`（阻塞进度） |

---

## 最佳实践

| ✅ 正确 | ❌ 错误 |
|------|------|
| 消费者不耗时操作 | ConsumeClaim 中同步调用远程接口 |
| 无论成败都 `MarkMessage`（offset 提交是连续水位） | 失败不 ack 期望重试（后续成功 ack 会覆盖） |
| Handler 通过 `TopicKey()` 自声明业务标识 + `fx.As` 注入 | 代码中硬编码 topic 字符串 |
| handler 委托 biz 处理业务逻辑，Event 定义在 biz 层 | handler 中直接写业务代码 |
| handler 放在 `kafkahandler` 包，依赖 `kafka` + `biz` | handler 放在 `kafka` 包，导致基础设施依赖业务 |
| 环境差异通过 `routes` 映射切换，代码不变 | 环境切换时改 handler 中的 topic 常量 |
| 关键消息处理失败记录到 DB/死信队列 | 仅依赖日志记录失败 |

---

## 验证

**注入完整性**：
□ consumer 实现 `ag_server.Server` 并通过 wrapper 归入 `group:"ag_servers"`
□ `internal/zfx_internal.go` 含 `kafka.FxKafkaModule` + `kafkahandler.FxHandlerModule`
□ `app.yml` 中 `routes` 映射覆盖所有业务标识
□ 所有 handler 通过 `kafka.AsKafkaHandler()` 在 `kafkahandler` 包中注入
□ 每组 `Topics` 中的 key 都有对应 handler
□ handler 注入 `biz` 处理业务逻辑，不直接写业务代码

**启动校验**：
□ `groups` 为空 → 启动报错
□ `topics` 为空 → 启动报错
□ 所有 key 不在 routes → 报错
□ 多 key 映射同一 topic → warn
□ handler 缺失 → 报错

**分层检查**：
□ `biz` 包不 import `kafka` 或 `kafkahandler`
□ `kafka` 包不 import `kafkahandler` 或 `biz`
□ `kafkahandler` 包 import `kafka` + `biz`
