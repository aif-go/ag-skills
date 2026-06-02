# AI Workflows

## 1. New Project

```
1. Ask project name and template repo
2. aggo new -r <template-url> -b <branch> <project-name>
3. cd <project-name>
4. Verify: go build ./...
```

## 2. Define API

```
1. Create proto: idl/api/<service>/<service>.proto
2. Follow patterns: [proto-idl-patterns.md](../references/proto-idl-patterns.md)
3. Define gRPC service + HTTP annotations
4. Show proto, get approval
```

## 3. Generate Code

```
1. aggo proto -p go,api,server,kitex,hertz,service -m server -e ./idl/api ./idl/api/<svc>/<svc>.proto
2. go mod tidy && go build ./...
```

`-p` 支持逗号分隔多值，默认 `all`。`-m server` 仅对 kitex/hertz 生效。

> `internal/service/agservice_*.go` 首次生成后不覆盖，后续 `-p service` 安全。

## 4. Generate Client Code

When calling other microservices:

```
1. Copy callee's .proto to idl/api/<service>/
2. aggo proto -p kitex,hertz -m client -e ./idl/api ./idl/api/<svc>/<svc>.proto
3. go mod tidy && go build ./...
```

> `-m client` 仅对 kitex/hertz 有效。go/api/server/service 插件不受影响。

## 5. Implement Business Logic

```
1. Write business logic in internal/biz/<service>_biz.go
2. Wire thin layer in internal/service/agservice_<service>.go (delegate to biz)
3. Use generated client code for RPC calls (via Gateway pattern)
4. Access database through repository layer
5. Write tests
6. go build ./...
```

## 6. Build & Run

```
1. cd cmd/server
2. go build
3. ./server
4. Test: curl localhost:<port>/<path>
```

## 7. Define Database Table

```
1. Write YAML definition: repository/yaml/<TABLE>.yaml
2. Follow: [db-yaml-format.md](../references/db-yaml-format.md)
3. Define columns, indexes, self_query_rules
```

## 8. Generate DAO Code

```
1. gen-go-db db -i ./internal/repository/yaml/<TABLE>.yaml -o ./internal -m <module/internal>
2. go mod tidy && go build ./...
```

Auto-generates: model, dao, namingsql (MYSQL+DB2), constant.

## Decision Matrix

| Request | Workflow |
|---------|----------|
| Create project | 1 |
| Add API service | 1 + 2 + 3 + 5 |
| Add database table | 7 + 8 + 5 |
| Call external service | 4 + 6 + 5 |
| Regenerate code | 2 + 3 |
| Add protobuf field | 2 + 3 + 5 |
