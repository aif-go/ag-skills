# Agonet Patterns

> **场景触发器**: TCP 长连接服务、短连接客户端、自定义通信协议、TLS/TLCP 安全传输、EventLoop 网络框架、Pipeline 管道、LengthField 编解码时加载。

---

## FX 模块注册

`cmd/server/main.go` 中按需启用：

```go
import (
    "github.com/aif-go/ag-core/contribute/agonet"
    "github.com/aif-go/ag-core/contribute/agonet/simple"
)

var mainFx = fx.Module("main",
    fxs.FxAgConfModule,

    // 服务端：自动装配 ServerConfig + Server + WarpServer
    agonet.FxAgonetServerModule,

    // 客户端：自动装配 ClientConfig + Client
    agonet.FxAgonetClientModule,
)
```

| 模块 | 作用 | 说明 |
|------|------|------|
| `FxAgonetServerModule` | 提供 Server | 依赖 `ag_conf.IEnvironment`（需在 `FxAgConfModule` 之后） |
| `FxAgonetClientModule` | 提供 Client | 同上 |

> 用户只需通过 `fx.Provide` 提供 `agonet.EventHandler` 实现，Server 即可自动装配启动。

### ShortClient FX 注入

```go
import "time"

// 提供超时配置（group tag 模式）
fx.Provide(
    simple.FxAgonetSimpleGroupTag(func() simple.ShortClientOption {
        return func(opts *simple.ShortClientOptions) {
            opts.Timeout = 5 * time.Second
        }
    }),
)
fx.Provide(simple.NewShortClientFx)
```

---

## 配置

### 最小服务端

```yaml
addr: "tcp://:9000"
config:
  engine:
    multicore: true
    numEventLoop: 0       # 0 = NumCPU，手动指定则覆盖 multicore
```

### TLS 服务端

```yaml
addr: "tcp://:9443"
config:
  security:
    type: tls
    certsDir: /etc/certs
    tls:
      caPath: ca.crt
      authCertPath: server.crt
      authKeyPath: server.key
```

### TLCP（国密）服务端

```yaml
addr: "tcp://:9443"
config:
  security:
    type: tlcp
    certsDir: /etc/certs
    tlcp:
      caPath: sm2-ca.cer
      authCertPath: sm2-auth.cer
      authKeyPath: sm2-auth.key
      signCertPath: sm2-sign.cer
      signKeyPath: sm2-sign.key
      encCertPath: sm2-enc.cer
      encKeyPath: sm2-enc.key
```

> TLCP 使用双证书体系（签名 + 加密），auth / sign / enc 是三种不同证书，不能复用。

### 双栈（同时支持 TLS + TLCP）

```yaml
config:
  security:
    type: tls_tlcp
    cliType: tls          # 客户端独立类型（必填，否则客户端尝试 tls_tlcp 失败）
```

### 客户端配置

```yaml
config:
  engine:
    multicore: true
  security:
    cliType: tls          # 客户端 TLS 类型（可选，默认复用服务端配置）
    tls:
      caPath: ca.crt
      authCertPath: client.crt
      authKeyPath: client.key
```

### KeepAlive

```yaml
config:
  keepAlive:
    enable: true
    idle: 60              # 空闲 60s 开始探测
    interval: 12          # 探测间隔 12s
    count: 5              # 5 次失败断开
```

### 支持的传输协议

| 协议 | 格式 | 状态 |
|------|------|------|
| TCP | `tcp://:port` / `tcp4://` / `tcp6://` | 支持 |
| Unix Socket | `unix:///path` | 支持 |
| UDP | — | 不支持 |

---

## Simple 管道层（推荐）

> 除极简固定长度协议外，所有业务场景都应用 Simple 管道层。

### 三步创建

```go
import (
    "github.com/aif-go/ag-core/contribute/agonet"
    "github.com/aif-go/ag-core/contribute/agonet/simple"
)

// 1. 创建 SimpleEventHandler（桥接 agonet 事件到 Pipeline）
handler, _ := simple.NewSimpleEventHandlerWithOptions(
    simple.WithChannelInitializer(func(channel simple.Channel) error {
        // 2. 向 Pipeline 添加 Handler（按 Inbound 传播顺序）
        channel.Pipeline().AddLast(
            simple.NewLengthFieldDecoder(binary.BigEndian, 1024*1024, 0, 4, 0, 4),
            &MyBizHandler{},
        )
        return nil
    }),
)

// 3. 启动
config := agonet.DefaultServerConfig()
config.Addr = "tcp://:9000"
server, _ := agonet.NewServer(handler, &config)
server.Start()
```

### 三层架构

```
agonet EventHandler (OnOpen / OnTraffic / OnClose)
    │ 桥接
SimpleEventHandler ──────创建 Channel × Pipeline
    │
Pipeline (双向链表 head ⇄ ctx1 ⇄ ctx2 ⇄ ... ⇄ tail)
    │ Inbound: next 正向   Outbound: prev 反向
Handler 链 (Decoder → BizHandler → Encoder → headHandler)
```

### Pipeline — 双向链表

- **head**：Outbound 终点 → `HandleWrite([]byte)` 写入 TCP
- **tail**：Exception 终点 → 打印日志 + 关闭连接
- 初始状态：`head ⇄ tail`
- `AddLast(h1, h2)` 添加在 tail 之前

### 事件传播方向

| 事件 | 方法 | 方向 | Handler 接口 |
|------|------|------|-------------|
| 读 | `FireRead` | next 正向 | `InboundHandler` → `HandleRead(ctx, msg)` |
| 写 | `FireWrite` | prev 反向 | `OutboundHandler` → `HandleWrite(ctx, msg)` |
| 激活 | `FireActive` | next 正向 | `ActiveHandler` → `HandleActive(ctx)` |
| 关闭 | `FireInactive` | next 正向 | `InactiveHandler` → `HandleInactive(ctx, err)` |
| 异常 | `FireExceptionCaught` | next 正向 | `ExceptionHandler` → `HandleException(ctx, ex)` |
| 自定义 | `FireEvent` | next 正向 | `EventHandler` → `HandleEvent(ctx, event)` |

> Handler 调用 `ctx.FireRead(msg)` / `ctx.FireWrite(msg)` 继续传播，不调用则拦截。

---

## Handler 编写

### 业务 Handler 类型

```go
// apis.go 中定义的接口
Handler                     // 空接口——标记所有 Handler

ActiveHandler      → HandleActive(ctx ActiveContext)
InactiveHandler    → HandleInactive(ctx InactiveContext, ex error)
InboundHandler     → HandleRead(ctx InboundContext, message any)
OutboundHandler    → HandleWrite(ctx OutboundContext, message any)
ExceptionHandler   → HandleException(ctx ExceptionContext, ex error)
EventHandler       → HandleEvent(ctx EventContext, event any)

// 复合类型
DuplexHandler  → InboundHandler + OutboundHandler
CodecHandler   → DuplexHandler + Name()
DecoderHandler → InboundHandler + Name()
EncoderHandler → OutboundHandler + Name()
```

### 简单 Handler

```go
// 内嵌 SimpleHandler 避免实现所有接口
type BizHandler struct{ simple.SimpleHandler }

func (h *BizHandler) HandleRead(ctx simple.InboundContext, msg any) {
    data := msg.([]byte)
    resp := process(data)
    ctx.Write(resp)              // 沿 Outbound 管道写回
}
```

### 泛型 InboundHandler

```go
handler := simple.NewSimpleInboundHandler(func(ctx simple.InboundContext, frame []byte) {
    // frame 已是 []byte，无需手动类型断言
    ctx.Write(process(frame))
})
```

### 函数式 Handler

```go
// 适合简单场景，不必定义结构体
channel.Pipeline().AddLast(
    simple.ActiveHandlerFunc(func(ctx simple.ActiveContext) {
        slog.Info("连接建立", "remote", ctx.Channel().RemoteAddr())
    }),
)
```

### 解码器链示例

```
Pipeline.AddLast(
    LengthFieldDecoder  ← HandleRead: Reader → 拆帧 → []byte → FireRead
    AuthHandler         ← HandleRead: []byte → 鉴权 → FireRead
    BizHandler          ← HandleRead: []byte → 业务 → ctx.Write(resp)
    Encoder             ← HandleWrite(resp) → 编码 → FireWrite
)
// Write 沿 prev 反向传播: head ← Encoder ← BizHandler
```

---

## 客户端

### 长连接（推荐，高频通信）

```go
import (
    "github.com/aif-go/ag-core/contribute/agonet"
    "github.com/aif-go/ag-core/contribute/agonet/simple"
)

handler, _ := simple.NewSimpleEventHandlerWithOptions(
    simple.WithChannelInitializer(func(ch simple.Channel) error {
        ch.Pipeline().AddLast(
            simple.NewLengthFieldDecoder(binary.BigEndian, 1024*1024, 0, 4, 0, 4),
            &ClientBizHandler{},
        )
        return nil
    }),
)

client, _ := agonet.NewClient(handler, agonet.DefaultClientConfig())
client.Start()
defer client.Stop()

conn, _ := client.Dial("tcp", "server:9000")
// conn 保持打开，可反复调用 conn.Write()，数据流经 Pipeline
conn.Write(payload)
// ...
conn.Close()
```

### 从 Conn 获取 Channel

```go
channel, err := simple.ChannelFromConn(conn)
channel.Write(request)
channel.Close()
```

### 短连接（管理面、健康检查）

```go
// 底层长连接客户端
client, _ := agonet.NewClient(handler, agonet.DefaultClientConfig())
client.Start()

// 包装为短连接客户端（超时 5s）
shortClient, _ := simple.NewSimpleShortClient(client,
    func(opts *simple.ShortClientOptions) { opts.Timeout = 5 * time.Second },
)

// 同步请求：建连 → 发 → 收 → 断连
resp, err := shortClient.RequestSync(context.Background(), "server:9000", requestData)
```

> `RequestSync` 每次创建新连接，高频场景（>10 QPS）用长连接 + `ChannelFromConn` 复用。

### 连接模式对比

| 模式 | 方式 | 适用场景 |
|------|------|---------|
| 长连接 | `client.Dial` → 保持 Conn | 高频 RPC、实时推送、网关 |
| 短连接 | `shortClient.RequestSync` | 管理命令、健康检查、偶发请求 |
| 纯 agonet | 直接实现 `agonet.EventHandler` | 固定长度协议、无需编解码 |

---

## LengthField 编解码器

解决了 TCP 半包/粘包问题的核心编解码器：

```go
// 参数说明
simple.NewLengthFieldDecoder(
    byteOrder,        // binary.BigEndian / binary.LittleEndian
    maxFrameLength,   // 最大帧长
    lengthFieldOffset,// 长度字段偏移量（字节）
    lengthFieldLength,// 长度字段自身长度（字节），常见 2 或 4
    lengthAdjustment, // 长度调整值
    initialBytesToStrip, // 跳过前 N 字节（通常 = lengthFieldOffset + lengthFieldLength）
)
```

```go
// 示例：4 字节头（大端）+ 数据体
simple.NewLengthFieldDecoder(binary.BigEndian, 1024*1024, 0, 4, 0, 4)
// HandleRead 收到的 msg 是解码后的完整帧 []byte
```

---

## 常见陷阱

### Handler 中不要做阻塞操作

`OnTraffic`、`HandleRead` 等都运行在 EventLoop 的 goroutine 中。

```go
// ❌ 同步阻塞数据库查询
func (h *Handler) HandleRead(ctx simple.InboundContext, msg any) {
    result := heavyDBQuery(msg)       // 阻塞整个 EventLoop！
    ctx.Write(result)
}

// ✅ 用 EventLoop.Execute() 异步执行
func (h *Handler) HandleRead(ctx simple.InboundContext, msg any) {
    ctx.Channel().EventLoop().Execute(context.Background(),
        agonet.RunnableFunc(func(ctx context.Context) error {
            result := heavyDBQuery(msg)
            conn.Wake(callback)       // 异步完成后通知
            return nil
        }),
    )
}
```

### 不要直接操作 `NetConn()`

绕过 Conn 接口直接读/写底层 `net.Conn` 会导致双缓冲（`buffer` + `inboundBuffer`）状态不一致。

### 解码器必须在业务 Handler 之前

`AddLast(decoder, bizHandler)` → 正确，`bizHandler` 收到解码后的帧。
`AddLast(bizHandler, decoder)` → 错误，`bizHandler` 收到的是 `agonet.Reader`。

### headHandler 只接受 `[]byte`

出站 `Write` 数据必须最终编码为 `[]byte`，否则 `headHandler.HandleWrite` 会 panic。

```go
// ❌ ctx.Write(someStruct)  →  headHandler → panic
// ✅ ctx.FireWrite(encode(someStruct))  →  Encoder → headHandler → []byte → TCP
```

### 异常不处理会关闭连接

没有 `ExceptionHandler` 时，异常传播到 tail 会关闭连接。建议在 Handler 链末尾添加 Recovery Handler。

### Simple 层 OnOpen 的 `out []byte` 无效

`SimpleEventHandler` 接管了 `OnOpen` 始终返回 nil。连接建立时发送数据用 `HandleActive` + `ctx.Write()`。

### 不要用废弃 API `ctx.Next()`

当前 API 无返回值，传播用 `ctx.FireRead(msg)` / `ctx.FireWrite(msg)`。
