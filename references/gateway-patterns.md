# Gateway Pattern — Cross-Service Call Patterns

> **场景触发器**: 跨服务调用、跨团队协作、需要解耦下游服务依赖时加载本文。
> **注意**: 这是进阶模式，单一服务项目不需要。仅在首次跨服务调用时引入。

---

## 何时使用

| 场景 | 推荐方式 |
|------|---------|
| 同一团队、下游服务稳定 | 继续在 `internal/service/` 处理 |
| 跨团队、下游频繁迭代 | Gateway 模式 |
| 调用方和被调方各自独立迭代 | Gateway 模式 |

**核心判断**: 如果你的 `biz/` 要 import 下游服务的 `api/<service>/` 或 `adpgen/kitex/<service>/`，且下游服务由其他团队维护 → 用 Gateway。

---

## 架构概览

```
biz/gateway.go     ← 定义接口（biz 说"我要什么"）
    ↑ implements
gateway/           ← 实现接口（gateway 说"我怎么拿到"）
    ↑ uses
clients/           ← 创建原始 client（注入 host、port、配置）
    ↑ uses
adpgen/            ← 生成的 client 代码
```

**关键原则**: biz 不 import 下游 `api/` 或 `adpgen/`。下游 proto 变更只影响 gateway 实现层一个文件。

---

## 分层职责

| 层 | 目录 | 职责 | import 谁 |
|----|------|------|----------|
| **接口定义** | `biz/gateway.go` | 定义业务语义接口和自有类型 | 无（纯 biz） |
| **协议适配** | `gateway/` | 实现接口，做类型转换 | adpgen + api/<svc> |
| **连接管理** | `clients/` | 创建 client、注入配置 | adpgen |
| **生成代码** | `adpgen/` | aggo 生成，不可修改 | — |

---

## 实现方式（三选一，可共存）

### 方式 A: 透明 pb（最简单）

适用: 同团队、下游稳定、字段 <5 个。

biz 直接 import 下游 api pb 类型，不定义 Gateway 接口。当前 `agaidevdemo` 的实现即为这种方式。

```go
// biz/student_biz.go
import scorerapi "agaidevdemo/api/scorer"

func (b *StudentBiz) CreateStudent(...) {
    scoreResp, _ := b.ScorerClient.ScoreStudent(ctx, &scorerapi.ScoreStudentRequest{
        Stuno: req.Stuno, Name: req.Name,
    })
}
```

### 方式 B: Gateway + 手写转换（最解耦）

适用: 跨团队、下游频繁迭代、字段 ≤10 个。

biz 定义自有类型和 Gateway 接口，gateway 手写 pb ↔ biz 转换。

**Step 1: biz 定义接口和类型**

```go
// biz/scorer_gateway.go
package biz

import "context"

type ScoreStudentParams struct {
    Name string
    Age  int32
}

type ScoreResult struct {
    Score int32
    Level string
}

type ScorerGateway interface {
    ScoreStudent(ctx context.Context, params *ScoreStudentParams) (*ScoreResult, error)
}
```

**Step 2: gateway 实现转换**

```go
// gateway/scorer_gateway.go
package gateway

import (
    "your-project/internal/biz"
    scorerpb "callee-project/api/scorer"
    scorerclient "callee-project/internal/adpgen/kitex/scorerservice"
)

type ScorerGatewayImpl struct {
    client scorerclient.Client
}

func NewScorerGateway(client scorerclient.Client) *ScorerGatewayImpl {
    return &ScorerGatewayImpl{client: client}
}

func (g *ScorerGatewayImpl) ScoreStudent(ctx context.Context, params *biz.ScoreStudentParams) (*biz.ScoreResult, error) {
    resp, err := g.client.ScoreStudent(ctx, &scorerpb.ScoreStudentRequest{
        Name: params.Name,
        Age:  params.Age,
    })
    if err != nil {
        return nil, err
    }
    return &biz.ScoreResult{Score: resp.Score, Level: resp.Level}, nil
}
```

**Step 3: fx 注册**

```go
// gateway/zfx_gateway.go
var FxGatewayModule = fx.Module("fx-gateway-module",
    fx.Provide(
        NewScorerGateway,
        fx.Annotate(NewScorerGateway, fx.As(new(biz.ScorerGateway))),
    ),
)
```

`fx.As(new(biz.ScorerGateway))` — 提供具体类型 `*ScorerGatewayImpl`，但以接口类型 `biz.ScorerGateway` 注入给消费者。

**Step 4: biz 依赖接口而非具体类型**

```go
// biz/student_biz.go
type StudentBiz struct {
    scorerGateway ScorerGateway  // 注入接口，不是 *ScorerGatewayImpl
}

func NewStudentBiz(scorerGateway ScorerGateway) *StudentBiz {
    return &StudentBiz{scorerGateway: scorerGateway}
}
```

### 方式 C: Gateway + copier（少写代码）

适用: 跨团队、字段 >5 且 **pb 字段名与 biz 类型字段名一致**。

```go
// gateway/scorer_gateway.go
import "github.com/jinzhu/copier"

func (g *ScorerGatewayImpl) ScoreStudent(ctx context.Context, params *biz.ScoreStudentParams) (*biz.ScoreResult, error) {
    var pbReq scorerpb.ScoreStudentRequest
    copier.Copy(&pbReq, params)

    pbResp, err := g.client.ScoreStudent(ctx, &pbReq)
    if err != nil {
        return nil, err
    }

    var result biz.ScoreResult
    copier.Copy(&result, pbResp)
    return &result, nil
}
```

> ⚠️ copier 风险：字段名不一致时**静默跳过**，不报错。运行期数据丢失。字段数少或命名差异大时用手写转换。

### 选型矩阵

| 维度 | 透明 pb | Gateway + 手写 | Gateway + copier |
|------|--------|---------------|------------------|
| biz 解耦 | ❌ | ✅ | ✅ |
| 下游变更影响 | biz 层 | 仅 gateway 1 文件 | 仅 gateway 1 文件 |
| 代码量 | 最少 | 中等 | 少 |
| 字段不匹配 | 编译期报错 | 编译期报错 | 运行时静默丢失 |
| 适用场景 | 同团队稳定服务 | 跨团队高迭代 | 跨团队多字段且命名一致 |

---

## 完整项目结构

```
internal/
├── config/          # 配置
├── service/         # 薄层入口
├── biz/
│   ├── <svc>_biz.go
│   ├── gateway.go   # ≤5 个接口放一个文件，>5 按域拆分
│   └── zfx_biz.go
├── gateway/
│   ├── <svc>_gateway.go
│   └── zfx_gateway.go
├── clients/
│   ├── <svc>_client.go
│   └── zfx_clients.go
├── adpgen/          # 生成代码
├── svcgen/          # 生成代码
└── zfx_internal.go
```

## fx 组装顺序

```go
// internal/zfx_internal.go
var FxInternalModule = fx.Module("fx-internal-module",
    config.FxAppConfigModule,
    clients.FxClientModule,     // 1. client 连接
    gateway.FxGatewayModule,    // 2. gateway（依赖 clients）
    biz.FxBizModule,            // 3. biz（依赖 gateway 接口）
    svcgen.FxServiceModule,
    adpgen.FxAdapterModule,
)
```

---

## 何时不用 Gateway

- 单一服务、无 RPC 调用 → 保持 `internal/service/` 即可
- 同团队维护的服务 → 透明 pb 足够
- proto 极少变更 → 解耦收益小于成本

**渐进式引入**: 不需要一开始就建 Gateway 层。当第一次出现跨团队 RPC 调用时，再把对应的调用方迁移到 Gateway 模式。
