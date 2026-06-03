# Gateway Pattern — 基础设施集成

> **场景触发器**：跨服务调用、Redis 缓存、MQ 消息、第三方 API 接入，或需要组织 biz/gateway/clients 三层架构时加载。

---

## 核心思想

**biz 定义接口，gateway 实现，clients 提供原生连接**。biz 不 import 任何外部协议包。

```
biz/gateway.go      定义接口（"我要什么"）
gateway/<svc>.go    实现接口 + 协议适配（"怎么做"）
clients/<svc>.go    创建原生 client + 连接配置
```

---

## 首先评估：Gateway 接口用 biz 自有类型还是 api pb？

不是所有场景都需要 biz 自有类型。用 biz 自有类型意味着 gateway 要做 pb↔biz 转换（~10 行/call），这个成本需要评估是否值得。

| 条件 | 接口类型 | gateway 转换 |
|------|---------|-------------|
| 同团队、下游稳定不变 | 直接引用 `api/<downstream>/` pb 类型 | 不需要 |
| 跨团队、下游独立迭代 | biz 自有类型 | gateway 做 pb↔biz 转换 |
| 下游字段多（≥5）且命名对齐 | biz 自有类型 + copier | copier 减少手写 |

**判断逻辑**：
```
这个下游服务由谁维护？
  ├── 同一个团队、同一仓库 → 直接用 pb，省转换
  └── 另一个团队、独立迭代 → biz 自有类型，隔离下游变更
```

同一项目中三种方式可共存。以下模式以「biz 自有类型 + gateway 转换」为例（最完整的写法）。

---

## biz 层——Gateway 接口规范

### 接口集中管理

```go
// biz/gateway.go — 接口定义
package biz

type ScorerGateway interface {
    ScoreStudent(ctx context.Context, params *ScoreStudentParams) (*ScoreResult, error)
}

// biz 自有类型，不依赖下游 proto
type ScoreStudentParams struct {
    Stuno string
    Name  string
}
type ScoreResult struct {
    Stuno string
    Score int32
    Level string
}
```

| Gateway 数量 | 文件 |
|-------------|------|
| 1-5 个 | `biz/gateway.go` 单文件 |
| 6+ 个 | 按域拆分 `biz/gateway_student.go`、`biz/gateway_order.go` |

**禁止项**：
- ✗ Gateway 接口 import `api/<downstream>/`（依赖下游 pb）
- ✗ Gateway 接口 import `adpgen/`（依赖生成代码）
- ✗ 随业务文件分散定义

---

## gateway 层——实现规范

### 基础模式

```go
// gateway/scorer_gateway.go
type scorerGatewayImpl struct {
    grpcClient kscorersvc.Client
    httpClient hscorersvc.ScorerServiceHertzClient
}

// 构造器直接返回 biz 接口，fx 自动识别，无需 fx.As
func NewScorerGateway(grpcClient kscorersvc.Client, httpClient hscorersvc.ScorerServiceHertzClient) biz.ScorerGateway {
    return &scorerGatewayImpl{grpcClient: grpcClient, httpClient: httpClient}
}
```

### copier 转换模式

```go
func (g *scorerGatewayImpl) ScoreStudent(ctx context.Context, params *biz.ScoreStudentParams) (*biz.ScoreResult, error) {
    var req scorerpb.ScoreStudentRequest
    copier.Copy(&req, params)       // biz 类型 → 下游 pb

    pbResp, err := g.grpcClient.ScoreStudent(ctx, &req)
    if err != nil {
        return nil, err
    }

    var result biz.ScoreResult
    copier.Copy(&result, pbResp)    // 下游 pb → biz 类型
    return &result, nil
}
```

> `copier` 选择：字段 ≤4 个手写，≥5 个且命名对齐时用 copier。注意 `copier` 静默跳过不匹配字段，字段名需与之对齐。

### 多协议轮询（gRPC + HTTP 交替）

```go
type scorerGatewayImpl struct {
    grpcClient kscorersvc.Client
    httpClient hscorersvc.ScorerServiceHertzClient
    idx        atomic.Uint64
}

func (g *scorerGatewayImpl) ScoreStudent(ctx context.Context, params *biz.ScoreStudentParams) (*biz.ScoreResult, error) {
    var req scorerpb.ScoreStudentRequest
    copier.Copy(&req, params)

    var pbResp *scorerpb.ScoreStudentResponse
    var err error
    if g.idx.Add(1)%2 == 0 {
        pbResp, err = g.grpcClient.ScoreStudent(ctx, &req)
    } else {
        pbResp, err = g.httpClient.ScoreStudent(ctx, &req)
    }
    // ...
}
```

### fx 装配

```go
// gateway/zfx_gateway.go
var FxGatewayModule = fx.Module("fx-gateway-module",
    fx.Provide(NewScorerGateway),     // 返回 biz.ScorerGateway 接口，直接注册
)
```

构造器返回接口类型，fx 将该接口注册到容器中，无需 `fx.Annotate(fx.As(...))`。

---

## 端到端流程

以调用外部评分服务（scorer）为例，完整串联 clients → gateway → biz：

```
1. 复制 callee proto → idl/api/scorer/
2. aggo proto -p kitex,hertz -m client → 生成 adpgen client 代码
3. clients/config.go + clients/zfx_clients.go + 工厂函数 → 连接层
   详见 [[service-clients]]
4. biz/gateway.go 定义接口
5. gateway/scorer_gateway.go 实现接口（注入 clients 工厂创建的 client）
6. biz/xxx_biz.go 注入 gateway 接口 → 调用
7. internal/zfx_internal.go 装配顺序: config → clients → gateway → biz → svcgen → adpgen
```

## fx 装配

在 `internal/zfx_internal.go` 中：

```go
var FxInternalModule = fx.Module("fx-internal-module",
    config.FxAppConfigModule,
    clients.FxClientModule,       // clients 连接层（详见 [[service-clients]]）
    gateway.FxGatewayModule,      // gateway 实现（返回接口）
    biz.FxBizModule,              // biz 编排 + 依赖 gateway 接口
    svcgen.FxServiceWithProxyModule(),
    adpgen.FxAdapterModule(),
)
```

---

> 完成后执行验证：[[verification#新增跨服务调用]]

## 相关文件

- Kitex 实现层：[[kitex-patterns]]
- Hertz 实现层：[[hertz-patterns]]
- 数据库（DAO）：[[dao-usage]]
- 项目结构：[[project-structure]]
