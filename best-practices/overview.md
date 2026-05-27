# Best Practices

## 配置管理

### app.yml 结构规范

```yaml
server:
  name: "my-service"                     # 服务名称，必须唯一

kitex:
  server:
    Port: 9996
    ServiceName: ${server.name}-grpc     # 使用变量引用
  client:
    RpcTimeout: 30s

hertz:
  server:
    Port: 9997
    ServiceName: ${server.name}-http
    Pprof: false                         # 生产环境关闭

data:
  db:
    user:
      driver: "mysql"
      dsn: "user:pass@tcp(host:3306)/db?parseTime=True"

aglog:
  topHandler: [zap1]
  zap:
    logs:
      zap1:
        log_level: info                  # 生产用 info，开发用 debug
        encoding: console
```

### 配置原则

- 使用 `${}` 占位符引用其他配置项，减少重复
- 敏感信息（密码、秘钥）使用 `{cipher}xxx` 加密格式
- 环境差异通过不同的配置文件管理（非代码内判断）
- 本地开发配置提交 `.yml.sample` 而非实际配置

## 日志规范

### 日志级别使用

| 级别 | 使用场景 |
|------|----------|
| `debug` | 开发调试信息 |
| `info` | 关键业务节点（请求开始/结束、状态变更） |
| `warn` | 可恢复的异常（重试成功、降级触发） |
| `error` | 不可恢复的错误（数据库连接失败、RPC 超时） |

### 日志最佳实践

```go
// ✅ 结构化日志
logger.Info("student created", "id", student.Id, "name", student.Name)

// ❌ 字符串拼接日志
logger.Info(fmt.Sprintf("student %d created", student.Id))

// ✅ 错误日志带完整上下文
logger.Error("failed to query student", "id", req.Id, "error", err)

// ❌ 错误日志无上下文
logger.Error(err.Error())
```

## 错误处理

### 错误传播模式

```go
func (s *ServiceImpl) GetStudent(ctx context.Context, req *pb.GetReq) (*pb.Student, error) {
    var student pb.Student
    err := s.repo.DB(ctx).Where("id = ?", req.Id).First(&student).Error
    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, status.Error(codes.NotFound, "student not found")
        }
        return nil, status.Errorf(codes.Internal, "query failed: %v", err)
    }
    return &student, nil
}
```

### 错误处理原则

- 使用 `status.Error/Errorf` 返回 gRPC 标准错误码
- 业务错误用自定义错误类型，非业务错误用标准错误
- 永远不要在错误信息中暴露内部细节（SQL、堆栈）
- HTTP handler 中统一处理错误响应格式

## 项目结构规范

### Service 层组织

```
internal/
├── service/
│   └── agservice_<service>.go       # 业务实现（一个 service 一个文件）
├── repository/                       # 自定义数据访问（可选）
│   └── <entity>_repo.go
├── middleware/                        # 自定义中间件（可选）
│   ├── auth.go
│   └── logging.go
└── model/                            # 自定义模型（可选）
    └── <entity>.go
```

### 文件职责单一

- 一个文件一个 service 实现，不合并多个 service
- 数据库操作封装在 Repository 中，不在 service 中写 SQL
- 中间件独立文件，不写在 service 中

## 性能优化

### 数据库

- 使用索引加速查询，Proto 定义的 ID 字段必须建索引
- 批量操作用 `CreateInBatches` 而非逐条 `Create`
- 大数据量查询必须分页，默认 pageSize 不超过 100
- 避免 `SELECT *`，用 `Select()` 指定需要的字段

```go
// ✅ 指定字段
db.Select("id", "name").Find(&students)

// ❌ 全字段查询
db.Find(&students)
```

### RPC 调用

- 设置合理的超时时间（默认 30s，查询类 5s）
- 避免循环中逐条调用 RPC，考虑批量接口
- 使用连接池复用连接

### 缓存

- 高频读取、低频更新的数据使用 Redis 缓存
- 缓存 Key 带业务前缀：`student:detail:{id}`
- 设置合理的过期时间，防止内存溢出

## 安全规范

### 输入校验

```go
// 在 service 方法入口做参数校验
func (s *ServiceImpl) CreateStudent(ctx context.Context, req *pb.CreateReq) (*pb.Student, error) {
    if req.Name == "" {
        return nil, status.Error(codes.InvalidArgument, "name is required")
    }
    if req.Age < 0 || req.Age > 150 {
        return nil, status.Error(codes.InvalidArgument, "invalid age")
    }
    // 业务逻辑
}
```

### 安全原则

- 所有外部输入必须校验（长度、格式、范围）
- SQL 查询使用参数化，防止注入
- 敏感数据不在日志中打印
- proto 文件不定义密码明文字段

## 生成的代码管理

### 修改规则快查

| 目录 | 能否修改 | 说明 |
|------|----------|------|
| `idl/api/` | ✅ | proto 手动编写 |
| `api/` | ❌ | `-p go` / `-p api` 生成 |
| `internal/adpgen/` | ❌ | `-p server/kitex/hertz` 生成 |
| `internal/svcgen/` | ❌ | `-p service` 生成 |
| `internal/service/agservice_*.go` | ✅ | 首次生成后不覆盖 |

### 生成操作安全

- 修改 proto 后需重新 `aggo proto`
- `adpgen/` 和 `svcgen/` 重新生成是安全的（会覆盖）
- `internal/service/agservice_*.go` 重新生成不会覆盖已有代码
- 重新生成后必须 `go mod tidy && go build ./...`

## 多服务开发

### 服务间调用规范

1. 将 callee 的 proto 文件复制到 `idl/api/<callee>/`
2. 用 `aggo proto -p kitex,hertz -m client` 生成客户端代码
3. 在 service 中通过构造函数注入客户端
4. 设置合理的超时和重试策略
5. 处理下游服务不可用的降级逻辑

## 开发流程最佳实践

1. **Proto-First**: 先定义 proto，再生成代码，最后实现逻辑
2. **一条命令生成**: 用 `-p go,api,server,kitex,hertz,service` 合并生成
3. **生成后验证**: `go mod tidy && go build ./...` 每次生成后必做
4. **业务逻辑在 Service**: 不修改 adpgen/svcgen，不在 main.go 写逻辑
5. **配置外部化**: 所有配置在 app.yml，环境差异通过文件切换
