# Verification

> 按操作场景路由。每完成一项开发任务，执行对应章节的检查清单。

---

## 新增数据库表

**编译检查**：
✅ `go build ./...`  — gen-go-db 生成后

**注入完整性**：
□ `internal/repository/dao/zfx_dao.go` — `fx.Provide(NewXxxDao)` 已添加
□ `internal/zfx_internal.go` — `dao.FxDaoModule` 已包含
□ `cmd/server/main.go` — `gormdb.FxAicGromdbModule` + `agdb.FxAgDbModule` 已声明

**运行时声明**：
□ `internal/init.go` — 需要事务的 RPC 方法已 `AddTag(TransactionTag, ...)`

---

## 新增配置模块

**注入完整性**：
□ `internal/config/zfx_config.go` — `fx.Provide(NewXxxConfig)` 已添加
□ `internal/zfx_internal.go` — `config.FxAppConfigModule` 已包含

---

## 新增跨服务调用

**编译检查**：
✅ `go build ./...`  — client/gateway 注入后

**注入完整性**：
□ `clients/zfx_clients.go` — `fx.Provide(NewXxxClient)` 已添加
□ `gateway/zfx_gateway.go` — `fx.Provide(NewXxxGateway)` 已添加

**装配顺序**：
□ `internal/zfx_internal.go` — `clients.FxClientModule` → `gateway.FxGatewayModule` → `biz.FxBizModule` 顺序正确

---

## 新增 Kitex/gRPC 服务

**注入完整性**：
□ `cmd/server/main.go` — `kserver.FxKitexServerBaseModule` 已声明
□ `cmd/server/main.go` — `kclient.FxKitexClientBaseModule` 已声明（调用外部 gRPC 时）

---

## 新增 Hertz/HTTP 服务

**注入完整性**：
□ `cmd/server/main.go` — `hserver.FxAgHertzServerModule` 已声明
□ `cmd/server/main.go` — `hclient.FxModuleAgHertzClient` 已声明（调用外部 HTTP 时）

---

## gen-go-db 代码生成后

**编译检查**：
✅ `go build ./...`  — 生成后必须验证，确保 `mysql_*.go` 和 `db2_*.go` 编译通过

**代码规范**：
□ `-d` 参数只用单个值或不指定，不使用 `+` / `,` 组合

---

## 项目初始化

项目初始化或新增模块后，Read 以下文件逐项确认。

**编译检查**：
✅ `go build ./...`  — 每次装配后

**注入完整性**（`cmd/server/main.go`）：

□ `fxs.FxAgConfModule`              — 配置绑定器，必须
□ `ag_log.FxAglogMode`              — 日志，必须
□ `gormdb.FxAicGromdbModule`        — DB 连接，有数据库时
□ `agdb.FxAgDbModule`               — 事务中间件，有数据库时
□ `fxs.FxAppMode`                   — 应用生命周期，必须
□ `hserver.FxAgHertzServerModule`   — HTTP 服务，有 HTTP 时
□ `kserver.FxKitexServerBaseModule` — gRPC 服务，有 gRPC 时
□ `hclient.FxModuleAgHertzClient`   — HTTP client，调用外部 HTTP 时
□ `kclient.FxKitexClientBaseModule` — gRPC client，调用外部 gRPC 时
□ `ag_service.FxAgServiceMode`      — 服务代理，必须
□ `internal.FxInternalModule`       — 自定义组件入口，必须

**装配顺序**（`internal/zfx_internal.go`）：

□ 顺序为 `config.FxAppConfigModule` → `dao.FxDaoModule` → `clients.FxClientModule` → `gateway.FxGatewayModule` → `biz.FxBizModule` → `svcgen.FxServiceWithProxyModule()` → `adpgen.FxAdapterModule()`

**运行时声明**（`internal/init.go`）：

□ 所有自定义 metadata key 已 `RegMdKey()` 注册
□ 所有需要事务的 RPC 方法已 `AddTag(TransactionTag, ...)` 声明
