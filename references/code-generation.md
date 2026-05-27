# Code Generation

## Command Syntax

```bash
aggo proto -p <plugins> -m <models> -e <idl-root> <proto-file>
```

| Flag | Required | Description |
|------|----------|-------------|
| `-p` | No (默认 all) | 插件列表，逗号分隔：`go,api,server,kitex,hertz,service,openapi` |
| `-m` | No (默认 all) | 模式，**仅对 kitex/hertz 有效**：`server`, `client`, `all` |
| `-e` | Yes | 外部 proto 路径，通常 `./idl/api` |
| `<file>` | Yes | Proto 文件路径 |

> `--desc` / `-d` 开启调试日志（仅显示执行的 protoc 命令，不实际执行）

## Plugin Reference

### -p go — Protobuf 消息代码
不受 `-m` 影响。
```bash
aggo proto -p go -e ./idl/api ./idl/api/student/student.proto
```
**Output**: `api/student/student.pb.go`

### -p api — 接口描述
不受 `-m` 影响。
```bash
aggo proto -p api -e ./idl/api ./idl/api/student/student.proto
```
**Output**: `api/student/agserver_student_interface.go`

### -p server — Adapter 基础
不受 `-m` 影响。
```bash
aggo proto -p server -e ./idl/api ./idl/api/student/student.proto
```
**Output**:
- `internal/adpgen/zfx_adapter.go`
- `internal/adpgen/adpinit/zfx_adapter_init.go`

### -p kitex — Kitex (gRPC) Adapter
**受 `-m` 影响**，控制生成 server 端或 client 端。

**-m server:**
```bash
aggo proto -p kitex -m server -e ./idl/api ./idl/api/student/student.proto
```
Output:
- `internal/adpgen/kitex/studentservice/agkitex_studentservice_server.go`
- `internal/adpgen/kitex/studentservice/agkitex_studentservice_fx.go`
- `internal/adpgen/adpinit/zfx_agkitex_student_studentservice_adpinit.go`

**-m client:**
```bash
aggo proto -p kitex -m client -e ./idl/api ./idl/api/student/student.proto
```
Output:
- `internal/adpgen/kitex/studentservice/agkitex_studentservice.go`
- `internal/adpgen/kitex/studentservice/agkitex_studentservice_client.go`
- `internal/adpgen/kitex/studentservice/agkitex_studentservice_agclient.go`

### -p hertz — Hertz (HTTP) Adapter
**受 `-m` 影响**，控制生成 server 端或 client 端。

**-m server:**
```bash
aggo proto -p hertz -m server -e ./idl/api ./idl/api/student/student.proto
```
Output:
- `internal/adpgen/hertz/studentservice/aghertz_studentservice_server.go`
- `internal/adpgen/hertz/studentservice/aghertz_studentservice_fx.go`
- `internal/adpgen/adpinit/zfx_aghertz_student_studentservice_adpinit.go`

**-m client:**
```bash
aggo proto -p hertz -m client -e ./idl/api ./idl/api/student/student.proto
```
Output:
- `internal/adpgen/hertz/studentservice/aghertz_studentservice_client.go`

### -p service — 业务逻辑入口
不受 `-m` 影响。⭐ 输出的 `agservice_*.go` **不重复覆盖**。
```bash
aggo proto -p service -e ./idl/api ./idl/api/student/student.proto
```
**Output**:
- `internal/svcgen/zfx_service.go`
- `internal/svcgen/zfx_agservice_proxy_student.go`
- `internal/svcgen/agservice_studentservice_proxy.go`
- `internal/service/agservice_studentservice.go` ⭐ 仅首次生成

## Full Server Generation Pipeline

推荐使用逗号分隔合并为一条命令：

```bash
aggo proto -p go,api,server,kitex,hertz,service -m server -e ./idl/api ./idl/api/<svc>/<svc>.proto
```

等效于逐条：
```bash
# 不受 -m 影响的插件
aggo proto -p go,api,server,service -e ./idl/api ./idl/api/<svc>/<svc>.proto
# 受 -m 影响的插件
aggo proto -p kitex,hertz -m server -e ./idl/api ./idl/api/<svc>/<svc>.proto
```

## Client Generation

```bash
aggo proto -p kitex,hertz -m client -e ./idl/api ./idl/api/<svc>/<svc>.proto
```

> `-m client` 仅对 kitex/hertz 有效。

## Post-Generation

```bash
go mod tidy
go build ./...
```

## File Modification Rules

| Directory | Can Modify | Notes |
|-----------|------------|-------|
| `idl/api/` | ✅ | Proto 手动编写 |
| `api/` | ❌ | 由 `-p go` / `-p api` 生成 |
| `internal/adpgen/` | ❌ | 由 `-p server/kitex/hertz` 生成 |
| `internal/svcgen/` | ❌ | 由 `-p service` 生成 |
| `internal/service/agservice_*.go` | ✅ | `-p service` 仅首次生成，后续不覆盖 |
