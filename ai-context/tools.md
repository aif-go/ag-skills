# aggo CLI Tools

## Prerequisites

```bash
# Check aggo
which aggo && aggo --version
```

## new - Create Project

```bash
aggo new -r <template-git-url> -b <branch> <project-name>
```

Example:
```bash
aggo new -r http://github.com/aif-go/ag-layout-demo.git -b base agaidevdemo
cd agaidevdemo
go build ./...
```

## proto - Generate Code

### Syntax
```bash
aggo proto -p <plugins> -m <models> -e <idl-root> <proto-file>
```

### Plugins (-p)

支持逗号分隔多值：`-p go,api,server`。默认 `all` 生成全部。

| Flag | Output | Description |
|------|--------|-------------|
| `-p go` | `api/<svc>/<svc>.pb.go` | protobuf 消息代码 |
| `-p api` | `api/<svc>/agserver_*_interface.go` | 接口定义 |
| `-p server` | `internal/adpgen/zfx_adapter.go` | adapter 基础 |
| `-p kitex` | `internal/adpgen/kitex/` | Kitex 适配器 |
| `-p hertz` | `internal/adpgen/hertz/` | Hertz 适配器 |
| `-p service` | `internal/service/agservice_*.go` | service 入口模板 |
| `-p openapi` | - | OpenAPI 文档 |

### Models (-m) — 仅对 kitex/hertz 有效

`-m` 选项控制 kitex/hertz 生成**服务端**还是**客户端**代码。

**对 go/api/server/service 插件无效**，这些插件不受 `-m` 影响。

| Flag | Description | 适用插件 |
|------|-------------|----------|
| `-m server` | 生成服务端代码 | kitex, hertz |
| `-m client` | 生成客户端代码 | kitex, hertz |
| `-m all` | 同时生成服务端+客户端（默认） | kitex, hertz |
| (无效果) | - | go, api, server, service |

### Full Server Generation（推荐：合并为一条命令）

```bash
# 一条命令生成所有服务端代码（等效于逐个执行 -p）
aggo proto -p go,api,server,kitex,hertz,service -m server -e ./idl/api ./idl/api/<svc>/<svc>.proto
```

也可逐条执行（效果相同）：
```bash
aggo proto -p go,api,server,service -e ./idl/api ./idl/api/<svc>/<svc>.proto
aggo proto -p kitex,hertz -m server -e ./idl/api ./idl/api/<svc>/<svc>.proto
```

> 注意：`internal/service/agservice_*.go` 首次生成后不会重复覆盖，后续 `-p service` 安全。

### Client Generation（调用其他服务）

```bash
# 需要 -m client，且仅 kitex/hertz 受此参数影响
aggo proto -p kitex,hertz -m client -e ./idl/api ./idl/api/<svc>/<svc>.proto
```

### 各插件生成文件清单

| -p | -m | 生成文件 |
|----|-----|---------|
| `go` | (无效) | `api/<svc>/<svc>.pb.go` |
| `api` | (无效) | `api/<svc>/agserver_*_interface.go` |
| `server` | (无效) | `internal/adpgen/zfx_adapter.go`, `adpinit/zfx_adapter_init.go` |
| `service` | (无效) | `internal/svcgen/*.go`, `internal/service/agservice_*.go` ⚠️ 不覆盖 |
| `kitex` | `server` | `internal/adpgen/kitex/<svc>/agkitex_*_server.go`, `_fx.go`, `adpinit/` |
| `kitex` | `client` | `internal/adpgen/kitex/<svc>/agkitex_*_client.go`, `_agclient.go` |
| `hertz` | `server` | `internal/adpgen/hertz/<svc>/aghertz_*_server.go`, `_fx.go`, `adpinit/` |
| `hertz` | `client` | `internal/adpgen/hertz/<svc>/aghertz_*_client.go` |

## Post-Generation Checklist

After EVERY aggo proto generation:
1. `go mod tidy` - resolve dependencies
2. `go build ./...` - confirm code compiles
3. Verify `internal/service/agservice_*.go` exists for business logic entry
