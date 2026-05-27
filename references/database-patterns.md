# Database Patterns

## 架构概览

ag-core 数据库层基于 GORM，提供 Repository 模式、事务管理和 DAO 增强层。核心设计是通过 context 自动判断事务上下文，实现业务代码与事务管理解耦。

## 目录结构

```
contribute/agdb/                  # 数据库核心抽象
├── transaction.go                # 事务接口 + 事务传播
├── transaction_service_middleware.go  # AOP 事务中间件
├── gormdb/                       # GORM 实现
│   ├── zfx_aicgormdb.go          # fx 依赖注入模块
│   ├── db.go                     # DB 连接初始化
│   ├── config.go                 # 配置结构体
│   ├── repository.go             # Repository 模式
│   ├── namingsql_support.go      # 命名 SQL + 分页工具
│   └── page.go                   # 分页/排序
├── agdao/                        # DAO 增强层
│   ├── base_dao.go               # BaseDao 接口
│   ├── dao_opts.go               # 表名策略选项
│   └── table_info.go             # TableInfo
└── conditonwhere/                # WHERE 条件构建器
    ├── where_clause_builder.go   # WhereClauseBuilder
    └── fieldmask.go              # FieldMask + MaskWhereCondition
```

## Repository 模式（核心）

### Repository 结构

```go
type Repository struct {
    agdb.TxContextAbility[*gorm.DB]   // 泛型事务上下文能力
    db     *gorm.DB                   // 数据库连接
    DbType string                     // 数据库类型 (MYSQL/DB2)
}

func NewRepository(db *gorm.DB) *Repository
```

### 核心方法：DB(ctx) — 事务透明

`DB(ctx)` 是整个 Repository 的**关键方法**，通过 context 自动判断当前请求是否在事务中：

```go
func (r *Repository) DB(ctx context.Context) *gorm.DB {
    // 1. 从 ctx 中查找事务对象
    v := r.GetTxFromCtx(ctx)
    if v != nil {
        return v  // 在事务中 → 返回事务 DB
    }
    // 2. 不在事务中 → 返回普通 DB（带 ctx）
    return r.db.WithContext(ctx)
}
```

**业务代码只需调用 `repo.DB(ctx)`，不感知事务/非事务差异。**

### 在 Service 中使用 Repository

```go
// internal/service/agservice_studentservice.go
type StudentServiceImpl struct {
    repo *gormdb.Repository
}

func NewStudentServiceImpl(repo *gormdb.Repository) *StudentServiceImpl {
    return &StudentServiceImpl{repo: repo}
}

func (s *StudentServiceImpl) GetStudent(ctx context.Context, req *pb.GetStudentReq) (*pb.Student, error) {
    var student pb.Student
    // DB(ctx) 自动获取事务或非事务 DB
    err := s.repo.DB(ctx).
        Where("id = ?", req.Id).
        First(&student).Error
    if err != nil {
        return nil, err
    }
    return &student, nil
}

func (s *StudentServiceImpl) CreateStudent(ctx context.Context, req *pb.CreateStudentReq) (*pb.Student, error) {
    student := &pb.Student{Name: req.Name, Age: req.Age}
    err := s.repo.DB(ctx).Create(student).Error
    if err != nil {
        return nil, err
    }
    return student, nil
}
```

## 事务管理

### 事务传播行为

| 常量 | 行为描述 |
|------|----------|
| `TRANSACTION_PROPAGATION_REQUIRED` | **默认**：有事务则加入，无事务则新建 |
| `TRANSACTION_PROPAGATION_SUPPORTS` | 有事务则加入，无事务非事务执行 |
| `TRANSACTION_PROPAGATION_UNKNOWN` | 直接执行，忽略事务 |

### 编程式事务

```go
err := repo.Transaction(ctx, func(txCtx context.Context) error {
    // 在事务上下文中执行操作
    if err := repo.DB(txCtx).Create(&user).Error; err != nil {
        return err  // 自动回滚
    }
    if err := repo.DB(txCtx).Create(&order).Error; err != nil {
        return err  // 自动回滚
    }
    return nil  // 自动提交
})
```

### AOP 声明式事务（事务中间件）

框架提供 AOP 事务中间件，通过 `CallInfo` 标记方法需要事务：

```go
// internal/init.go — 注册事务标签
func init() {
    agservice.AddTag("CreateStudent", true) // 标记 CreateStudent 需要事务
}
```

事务中间件自动包装标记的方法，执行前后管理事务生命周期。

```go
// 事务中间件执行流程
func TransactionPreOpt(ctx context.Context, info *agservice.CallInfo) {
    if info.HasTag(agdb.TransactionTag) {
        // 标记需要事务
    }
}

func (p *TransactionMiddlewareProvider) Middleware(next endpoint.Endpoint) endpoint.Endpoint {
    return func(ctx context.Context, req interface{}) (interface{}, error) {
        // 1. 检查方法是否需要事务
        // 2. repo.Transaction(ctx, func(txCtx) { return next(txCtx, req) })
    }
}
```

## 数据库配置

### app.yml 配置

```yaml
data:
  db:
    user:
      driver: "mysql"                            # 数据库驱动
      dsn: "root:password@tcp(localhost:3306)/dbname?parseTime=True"
    pool:
      maxIdleConns: 10                           # 最大空闲连接
      maxOpenConns: 100                          # 最大打开连接
      connMaxLifetime: 3600                      # 连接最大生命周期(秒)
      connMaxIdleTime: 600                       # 连接最大空闲时间(秒)
    logger:
      name: "agdb"                               # Logger 名称
      debug: false                               # 是否开启调试模式
```

### 数据库初始化（fx 注入）

```go
// 框架内置的 fx 模块
var FxAicGromdbModule = fx.Module("fx_aic_gormdb",
    fx.Provide(
        NewAggormDbConfig,          // 配置绑定（前缀 data.db）
        NewDB_V2,                   // *gorm.DB 连接
        NewRepository,              // *Repository
        NewTransactionManager,      // TransactionManager
        FindGormLoggerFromAgslog,   // GORM Logger
    ),
)
```

## 分页查询

### 分页结构体

```go
type Page struct {
    PageNum  int           // 页码
    PageSize int           // 每页大小
    Orders   []OrderItem   // 排序
}
```

### 分页查询模式

```go
func (s *StudentServiceImpl) ListStudents(ctx context.Context, req *pb.ListReq) (*pb.ListResp, error) {
    var students []pb.Student
    var total int64

    db := s.repo.DB(ctx).Model(&pb.Student{})

    // 条件过滤
    if req.Name != "" {
        db = db.Where("name LIKE ?", "%"+req.Name+"%")
    }

    // 计数
    db.Count(&total)

    // 分页 + 排序
    err := db.
        Offset((req.PageNum - 1) * req.PageSize).
        Limit(req.PageSize).
        Order("id DESC").
        Find(&students).Error

    return &pb.ListResp{Total: total, Data: students}, err
}
```

## fx 依赖注入总览

### 数据库 fx 模块链

```
main.go
└── FxAgDbModule (contribute/agdb/zfx_agdb.go)
    ├── FxAicGromdbModule (gormdb/zfx_aicgormdb.go)
    │   ├── Config → NewDB_V2 → *gorm.DB
    │   ├── *gorm.DB → NewRepository → *Repository
    │   └── *Repository → NewTransactionManager
    ├── TransactionMiddlewareProvider (事务 AOP 中间件)
    └── agdao.FxNewAgGormBaseDao (DAO 层)
```

### 在 service 中注入 Repository

```go
// 在项目的 fx 模块中添加 Repository 依赖
type StudentServiceFx struct {
    fx.In
    Repo *gormdb.Repository
}

func NewStudentServiceImpl(p StudentServiceFx) *StudentServiceImpl {
    return &StudentServiceImpl{repo: p.Repo}
}
```

## WHERE 条件构建器

框架提供 `WhereClauseBuilder` 和 `FieldMask` 模式用于动态查询条件：

```go
// 条件构建（在 conditonwhere 包中）
builder := NewWhereClauseBuilder()
builder.AddCondition("name", "LIKE", "%"+name+"%")
builder.AddCondition("age", ">=", minAge)
where, params := builder.Build()
db.Where(where, params...).Find(&results)
```

## 核心原则

1. **使用 `repo.DB(ctx)` 获取 DB** — 自动处理事务上下文
2. **事务用 `repo.Transaction(ctx, fn)`** — 自动回滚/提交
3. **配置在 app.yml 的 `data.db` 段** — 不硬编码连接信息
4. **通过 fx 注入 Repository** — 不在 service 中自己创建 DB 连接
5. **Service 层访问 Repository** — 不在 Handler/Adapter 层直接操作数据库
6. **用 GORM 链式调用** — 不写原生 SQL（除特殊情况）

## 相关文件

- 项目结构：[[project-structure]]
- fx 依赖注入：参考 ag-core `contribute/agdb/zfx_agdb.go`
