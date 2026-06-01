# AI Instructions for ag-core

## File Priority

1. `workflows.md` - Task patterns
2. `tools.md` - aggo commands
3. `patterns.md` - Code patterns
4. [SKILL.md](../SKILL.md) - Detailed patterns

## Rules

### Proto-First
- ALWAYS create `.proto` spec before code
- Proto files in `idl/api/<service>/`
- Follow gRPC + HTTP annotation rules

### Tool Usage
- Use aggo commands in terminal, NOT manual code generation
- `aggo new` for new projects
- `aggo proto` for code generation
- `-p` supports comma-separated multi-value: `-p go,api,server,kitex,hertz,service`
- `-m` only affects kitex/hertz (server|client), invalid for go/api/server/service
- Always run post-generation: `go mod tidy` → `go build ./...`
- `-p service` generates `agservice_*.go` once, will not overwrite

### Implementation
- 业务逻辑写在 `internal/service/agservice_*.go`
- 生成代码（adpgen/ svcgen/）不可手动修改
- 微服务调用使用生成的 client 代码

### Documentation
- ALWAYS generate README.md for new services
- Document proto interfaces and endpoints

### ag-core Conventions
- Context propagation through all layers
- Error handling with ag-core error types
- Config via `cmd/server/app.yml`
- Protobuf IDL: `syntax = "proto3"` with http annotations

## Decision Tree

```
User Request →
├─ New Project?   → aggo new → define proto → generate → implement → build
├─ New API?       → write .proto → aggo proto → implement service → build
├─ Regenerate?    → aggo proto -p <target> → go mod tidy → go build
├─ Modify API?    → edit .proto → aggo proto → update service logic
└─ Client Call?   → get callee .proto → aggo proto -p kitex/ertz -m client
```

## Detailed Patterns

For complete implementation patterns, refer to [SKILL.md](../SKILL.md):

- Proto IDL → [proto-idl-patterns.md](../references/proto-idl-patterns.md)
- Code Generation → [code-generation.md](../references/code-generation.md)
- Project Structure → [project-structure.md](../references/project-structure.md)
- Kitex → [kitex-patterns.md](../references/kitex-patterns.md)
- Hertz → [hertz-patterns.md](../references/hertz-patterns.md)
- Database → [db-yaml-format.md](../references/db-yaml-format.md) | [gen-go-db-cli.md](../references/gen-go-db-cli.md) | [dao-usage.md](../references/dao-usage.md)
- Configuration → [ag-conf-patterns.md](../references/ag-conf-patterns.md)

## Avoid

- 手动修改 adpgen/ svcgen/ 目录的生成代码
- 跳过 post-generation 步骤（mod tidy、build verify）
- 在非 service/ 层写业务逻辑
- 忘记定义 proto 的 http annotation 就生成 HTTP 服务
- 拼接字符串构造 aggo 命令（必须用独立参数）
