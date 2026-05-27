# Troubleshooting

## 代码生成问题

### aggo proto 报错：protoc 找不到

**症状**: `protoc: command not found` 或 `protoc-gen-go: program not found`

**原因**: protoc 或 protoc 插件未安装或不在 PATH 中。

**解决**:
```bash
# 检查 protoc
which protoc && protoc --version

# 安装 protoc-gen-go
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest

# 确认 GOPATH/bin 在 PATH 中
export PATH=$PATH:$(go env GOPATH)/bin
```

### aggo proto 报错：import google/api/annotations.proto not found

**症状**: `could not find import google/api/annotations.proto`

**原因**: `third_party/` 目录缺少 google api proto 文件。

**解决**:
```bash
# 检查 third_party/ 是否存在
ls third_party/google/api/annotations.proto

# 如果缺失，从 ag-core 模板复制或重新 aggo new 创建项目
```

### -m 参数不生效

**症状**: 使用 `-p go -m server` 或 `-p api -m client` 没有效果。

**原因**: `-m` 仅对 `kitex` 和 `hertz` 插件有效，对 `go/api/server/service` 无效。

**解决**: 正确用法：
```bash
# ✅ 正确：-m 配对 kitex/hertz
aggo proto -p kitex,hertz -m server ...

# ❌ 错误：-m 对 go/api/server/service 无效
aggo proto -p go,api -m server ...
```

### 生成的文件未出现

**症状**: 执行 `aggo proto` 后没有生成预期文件。

**排查步骤**:
1. 确认 proto 文件路径正确：`aggo proto -e ./idl/api ./idl/api/<svc>/<svc>.proto`
2. 确认 `-e` 参数指向 idl 根目录
3. 确认 proto 文件有 `option go_package` 声明
4. 检查 `-p` 参数是否包含了目标插件

### agservice_*.go 重复生成/覆盖

**症状**: 担心 `-p service` 会覆盖已编写的业务代码。

**说明**: `-p service` 生成的 `internal/service/agservice_*.go` **不会覆盖已有文件**。框架在生成前检测文件是否存在，存在则跳过。

## 编译问题

### go build 报错：undefined: xxx

**症状**: `undefined: agserver.RegisterStudentService` 或类似。

**原因**: 生成代码未完成或未 `go mod tidy`。

**解决**:
```bash
# 1. 确保所有插件都已生成
aggo proto -p go,api,server,kitex,hertz,service -m server -e ./idl/api ./idl/api/<svc>/<svc>.proto

# 2. 整理依赖
go mod tidy

# 3. 重新构建
go build ./...
```

### go mod tidy 报错：module not found

**症状**: `module xxx@latest found but does not contain package`

**原因**: protoc 插件版本不匹配或 go module 缓存问题。

**解决**:
```bash
# 清理缓存
go clean -modcache
go mod tidy

# 如果仍有问题，检查 go.mod 中 replace 指令
```

### import cycle not allowed

**症状**: `import cycle not allowed in test` 或编译错误。

**原因**: 不能在 `api/` 包中导入 `internal/` 包，反之亦然。

**解决**: 检查 `go_package` 选项的路径，确保生成代码包路径正确：
```proto
option go_package = "myapp/api/student"; // ✅ 在 api/ 下
```

## 运行时问题

### 端口冲突

**症状**: `bind: address already in use`。

**解决**: 修改 `cmd/server/app.yml` 中的端口：
```yaml
kitex:
  server:
    Port: 9998   # 改为未占用的端口
hertz:
  server:
    Port: 9999
```

### 服务注册失败

**症状**: 启动日志中无服务注册成功信息，或 `no available server`。

**排查步骤**:
1. 确认 Nacos 服务可用：`curl http://127.0.0.1:8848/nacos/v1/console/health`
2. 确认 `app.yml` 中 nacos 配置正确
3. 本地开发可临时关闭 nacos（不配置 nacos 段只启动 HTTP）

### gRPC 调用超时

**症状**: `context deadline exceeded` 或 RPC 调用一直超时。

**排查**:
1. 确认目标服务已启动并注册到 Nacos
2. 检查 `kitex.client.RpcTimeout` 配置是否过短
3. 检查网络连通性：`telnet <host> <port>`

### HTTP 请求 404

**症状**: curl 返回 404。

**排查**:
1. 确认 proto 中定义了正确的 HTTP 注解：
   ```proto
   option (google.api.http) = { get: "/student/get" };
   ```
2. 确认已生成 Hertz adapter：`-p hertz`
3. 确认 `-m server` 参数已设置
4. 检查 app.yml 中 `hertz.server.Port` 是否是预期端口

### 数据库连接失败

**症状**: `dial tcp: connect: connection refused` 或 `Access denied for user`。

**排查**:
1. 确认数据库服务运行：`mysql -u root -p -h localhost`
2. 检查 app.yml 中 `data.db.user.dsn` 是否正确
3. 确认数据库已创建：`CREATE DATABASE IF NOT EXISTS dbname`
4. 确认 `parseTime=True` 在 DSN 中

### 事务未生效

**症状**: 操作失败但数据未回滚。

**排查**:
1. 确认使用了 `repo.Transaction(ctx, fn)` 或事务标签已注册
2. 确认数据库引擎支持事务（InnoDB，非 MyISAM）
3. 确认在事务 fn 中使用 `txCtx` 而非原始 `ctx`：
   ```go
   repo.Transaction(ctx, func(txCtx context.Context) error {
       return repo.DB(txCtx).Create(&data).Error // ✅ 使用 txCtx
   })
   ```

## 调试技巧

### 查看生成的代码

```bash
# 列出 adpgen 目录所有生成文件
ls -R internal/adpgen/

# 查看生成的路由
grep -r "Router_" internal/adpgen/hertz/
```

### 开启 GORM 调试日志

```yaml
data:
  db:
    logger:
      debug: true   # 打印 SQL 语句
```

### 查看 aggo 执行过程

```bash
# --desc / -d 查看执行的 protoc 命令（不实际执行）
aggo proto -p go -d -e ./idl/api ./idl/api/student/student.proto
```

### 检查 proto 语法

```bash
# 用 protoc 直接检查 proto 语法
protoc --proto_path=./idl/api --proto_path=./third_party \
    --go_out=. ./idl/api/student/student.proto
```

### HTTP 接口测试

```bash
# GET 请求
curl -v http://localhost:9997/student/get/1

# POST 请求
curl -v -X POST http://localhost:9997/student/create \
    -H "Content-Type: application/json" \
    -d '{"name":"张三","age":20}'
```

## 常见错误速查

| 错误信息 | 原因 | 解决方案 |
|----------|------|----------|
| `protoc: command not found` | protoc 未安装 | `apt install protobuf-compiler` |
| `import xxx not found` | proto import 路径错误 | 检查 `-e` 和 `--proto_path` |
| `undefined: xxx` | 生成不完整 | 重新 `-p all` 生成 + `go mod tidy` |
| `address already in use` | 端口冲突 | 修改 app.yml 端口 |
| `connection refused` | 下游服务未启动 | 确认目标服务运行状态 |
| `no available server` | 服务发现找不到实例 | 检查 Nacos 注册状态 |
| `RpcTimeout exceeded` | RPC 超时 | 调整 timeout 或优化下游 |
| `cannot find package` | go module 问题 | `go mod tidy` + 检查 go.mod |
