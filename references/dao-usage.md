# Generated DAO Usage Guide

> **场景触发器**: 用户说"使用 DAO""数据操作""InsertOne""FindByPrimaryKey""命名 SQL""FindByCondition""分页查询"时加载。

## 接口概览

每张表生成一个 DAO 接口，包含完整的单表 CRUD 操作：

```go
type IStudentDao interface {
    InsertOne(ctx context.Context, entity *model.Student) (int64, error)
    InsertOneIgnoreZeroValCols(ctx context.Context, entity *model.Student) (int64, error)
    UpdateByPrimaryKey(ctx context.Context, entity *model.Student) (int64, error)
    UpdateByPrimaryKeyIngoreZeroValCols(ctx context.Context, entity *model.Student) (int64, error)
    FindByPrimaryKey(ctx context.Context, id model.StudentPrimaryKey) (*model.Student, error)
    FindByStruct(ctx context.Context, entity *model.Student) ([]*model.Student, error)
    FindByCustomerRule(ctx context.Context, namingInfo *gormdb.NameingSqlArgInfo, args any) (any, error)
    FindByCondition(ctx context.Context, cond *conditonwhere.WhereClauseBuilder,
        order *gormdb.OrderBuilder, page *gormdb.Page) ([]*model.Student, *gormdb.PageResult, error)
    FindFirstOneByCondition(ctx context.Context, cond *conditonwhere.WhereClauseBuilder,
        order *gormdb.OrderBuilder) (*model.Student, error)
}
```

> 注：`UpdateByPrimaryKeyIngoreZeroValCols` 方法名中 `Ingore` 为生成代码自身的拼写（少一个 `n`），非文档错误。

## 快速开始

### 插入

```go
// 全字段插入（零值也会写入）
affected, err := dao.InsertOne(ctx, &model.Student{Name: "张三", Age: 18})

// 自动剔除零值列（主键、索引列不受影响）
affected, err = dao.InsertOneIgnoreZeroValCols(ctx, &model.Student{Name: "张三", Age: 18})
```

### 更新

```go
// 根据主键更新（需传入主键值，否则报错 "primary key is required"）
dao.UpdateByPrimaryKey(ctx, &model.Student{Id: 1, Name: "李四"})

// 忽略零值更新
dao.UpdateByPrimaryKeyIngoreZeroValCols(ctx, &model.Student{Id: 1, Name: "李四"})
```

### 按主键查询

```go
student, err := dao.FindByPrimaryKey(ctx, model.StudentPrimaryKey(1))
// 未找到返回 (nil, nil)，非 ErrRecordNotFound
```

## 查询

### 查询方式选择

```
查询需求 →
├─ 简单等值查询（固定 WHERE 条件）
│   └─ FindByStruct  — 零代码，自动拼接索引字段
│      ✅ 运行时索引安全检查
│      ❌ 无分页，不支持 LIKE / IN / BETWEEN
│
├─ 自定义查询（默认推荐）
│   └─ FindByCustomerRule  — 命名 SQL + FieldMask 动态 WHERE
│      ✅ LIKE / IN / BETWEEN 全覆盖 + 分页内置
│      ✅ 运行时索引安全检查（ValidateLeadingCol）
│      ✅ 跨数据库 SQL（MYSQL/DB2 独立模板）
│      ⚠️ 需编写 YAML + Arg/Res 类型 + 命名 SQL
│
└─ 轻量动态查询（无需预定义 YAML）
    └─ FindByCondition  — WhereClauseBuilder 链式构建
       ⚠️ 无索引安全校验，不支持 LIKE
       ⚠️ 依赖 GORM 生成 SQL，无跨库精确控制
```

| 能力 | FindByStruct | FindByCustomerRule | FindByCondition |
|------|:---:|:---:|:---:|
| 等值查询 | ✅ | ✅ | ✅ |
| 动态条件 | ❌ | ✅ (FieldMask) | ✅ (链式) |
| LIKE 模糊查询 | ❌ | ✅ | ❌ |
| 分页 | ❌ | ✅ | ✅ |
| 索引安全检查 | ✅ | ✅ | ❌ |
| 跨数据库 SQL | ❌ | ✅ MYSQL/DB2 | ❌ |
| 开发成本 | 零 | 中 | 低 |

> **AI 默认推荐**：自定义查询优先用 `FindByCustomerRule`。运行时索引校验 + 跨数据库兼容，生产环境更安全。

### FindByStruct — 简单等值查询

根据实体中非零值的索引列/主键列查询。先检查主键 → 再检查索引列 → 附加 AND 条件。**运行时索引安全校验**。

```go
students, _ := dao.FindByStruct(ctx, &model.Student{Age: 18})
```

### FindByCustomerRule — 命名 SQL（推荐默认）

YAML 中 `self_query_rules` 定义的查询。支持非分页和分页两种模式。

**非分页**：

```go
// 1. 必须初始化 FieldMask，否则 WithXxx() 空指针
arg := &model.StudentFindByAgeArg{
    FieldMask: conditonwhere.NewFieldMask(),
}
arg.WithAge(18).WithStuno("S001")

// 2. 执行
result, _ := dao.FindByCustomerRule(ctx, dao.FindByAgeNamingInfo, arg)
list := result.([]*model.StudentFindByAgeRes) // 类型断言
```

**分页**：

```go
arg := &model.StudentFindByAgeWithPageArg{
    FieldMask: conditonwhere.NewFieldMask(),
    Page:      gormdb.Page{PageNum: int64(1), PageSize: int64(20)},
}
arg.WithAge(18)

result, _ := dao.FindByCustomerRule(ctx, dao.FindByAgeWithPageNamingInfo, arg)
pageRes := result.(*model.StudentFindByAgeWithPagePageRes)
// pageRes.ResultList   — 数据列表
// pageRes.PageResult   — CurrentPage, PageSize, TotalCount, TotalPage
```

> ⚠️ **常见错误**：忘记 `FieldMask: conditonwhere.NewFieldMask()` → `WithXxx()` 空指针 panic。

### 完整流程：新增一个自定义查询

**步骤 1：编辑 YAML**

在 `repository/yaml/STUDENT.yaml` 的 `self_query_rules` 下新增：

```yaml
self_query_rules:
  FindByAgeWithPage:
    select_fields: '*'
    page: true
    where:
      operator: AND
      conditions:
        - expr: AGE = @Age
```

- `select_fields: '*'` → 返回全列，或指定列名逗号分隔
- `page: true` → 生成嵌入 `db.Page` 的 Arg + 嵌入 `db.PageResult` 的 Result
- `@Age` → 参数名，自动驼峰转为 Go 字段 `Age`

**步骤 2：重新生成**

```bash
gen-go-db db -i ./internal/repository/yaml/STUDENT.yaml -o ./internal -m myproject/internal
```

自动更新 model（Arg/Res 类型）、namingsql（MYSQL/DB2 SQL 模板）、constant（NamingInfo 注册）、dao（switch-case 分支）。

**步骤 3：biz 层调用**

```go
arg := &model.StudentFindByAgeWithPageArg{
    FieldMask: conditonwhere.NewFieldMask(),
    Page:      gormdb.Page{PageNum: int64(req.PageNum), PageSize: int64(req.PageSize)},
}
arg.WithAge(int(req.Age))

result, _ := b.studentDao.FindByCustomerRule(ctx, dao.FindByAgeWithPageNamingInfo, arg)
pageRes := result.(*model.StudentFindByAgeWithPagePageRes)

// 转换 model → proto
var data []*pb.Student
for _, m := range pageRes.ResultList {
    var s pb.Student
    copier.Copy(&s, m)
    data = append(data, &s)
}
return &pb.ListResp{
    TotalCount: pageRes.TotalCount,
    TotalPage:  int32(pageRes.TotalPage),
    Data:       data,
}, nil
```

### FindByCondition — WhereClauseBuilder 动态查询

无需预定义 YAML，适合条件灵活变化的轻量场景。

```go
cond := conditonwhere.NewWhereClauseBuilder()
if req.Name != "" {
    cond.Eq("NAME", req.Name)
}
if req.Age > 0 {
    cond.Gt("AGE", req.Age)
}

order := gormdb.NewOrderBuilder().Desc("ID")
page := &gormdb.Page{PageNum: int64(1), PageSize: int64(20)}

list, pageResult, err := dao.FindByCondition(ctx, cond, order, page)
```

> ⚠️ `FindByCondition` 不含索引安全检查，不支持 LIKE，需自行确保 WHERE 字段有索引。

## 初始化与 fx 注入

### 步骤 1：创建 DAO 模块

在 `internal/repository/dao/zfx_dao.go` 中（不被 gen-go-db 覆盖）：

```go
package dao

import "go.uber.org/fx"

var FxDaoModule = fx.Module("fx_dao",
    fx.Provide(
        NewStudentDao,
    ),
)
```

### 步骤 2：注册到 internal

在 `internal/zfx_internal.go` 中：

```go
var FxInternalModule = fx.Module("fx-internal-module",
    config.FxAppConfigModule,
    svcgen.FxServiceWithProxyModule(),
    adpgen.FxAdapterModule(),
    dao.FxDaoModule,  // ← 新增
)
```

### 步骤 3：app 装配

```go
app := fx.New(
    gormdb.FxAicGromdbModule,  // *gorm.DB → Repository
    agdb.FxAgDbModule,         // BaseDao + 事务中间件
    internal.FxInternalModule,  // 内部聚合（含 DAO、Config、Service...）
)
```

### 在 Service/Biz 中注入

```go
type StudentBiz struct {
    studentDao dao.IStudentDao  // 面向接口
}

func NewStudentBiz(studentDao dao.IStudentDao) *StudentBiz {
    return &StudentBiz{studentDao: studentDao}
}
```

> ⚠️ 必须滚动到文件末尾执行 ## 验证 的检查清单，确认所有注入点无遗漏。

## 事务

### 声明式 — AddTag（推荐）

通过 `AddTag` 标记 RPC 方法，框架 AOP 中间件自动管理事务生命周期。biz 层代码无需感知。

**基本用法**：

```go
// internal/init.go
import (
    "your-project/internal/svcgen"
    "gitlab.allinfinance.com/aifgo/ag-core/contribute/agdb"
)

func init() {
    svcgen.StudentServiceCreateStudentCallInfo.AddTag(agdb.TransactionTag, true)
}
```

**指定传播方式**：

```go
// 有事务则加入，无则非事务执行
svcgen.StudentServiceGetStudentCallInfo.AddTag(
    agdb.TransactionTag, agdb.TRANSACTION_PROPAGATION_SUPPORTS)
```

**传播方式**：

| 值 | 行为 |
|------|------|
| `true` / `TRANSACTION_PROPAGATION_REQUIRED` | 有则加入，无则新建 |
| `TRANSACTION_PROPAGATION_SUPPORTS` | 有则加入，无则非事务 |

> ⚠️ DAO 接口不暴露 `Transaction()` 方法，声明式是 biz 层的标准事务途径。

> ⚠️ 必须滚动到文件末尾执行 ## 验证 的检查清单，确认所有注入点无遗漏。

## 注意事项

### 零值列处理

| 类型 | 零值 | 后果 |
|------|------|------|
| `int` / `int64` | `0` | 插入/更新时可能被误过滤 |
| `string` | `""` | 空字符串被 Ignore 版本剔除 |
| `time.Time` | `time.Time{}` | 零值时间被剔除 |

故意写零值 → 用 `InsertOne` / `UpdateByPrimaryKey`（非 Ignore 版本）。

### 索引安全检查

`FindByStruct` 和命名 SQL 都有索引安全检查。触发 `"query not use any index"` 表示 WHERE 条件不含任何索引引导列。

### 生成代码不可编辑

所有 `*_dao.go`、`*_model.go`、`*_namingsql.go` 有 `DO NOT EDIT` 标记。修改方式：改 YAML → 重新 `gen-go-db db`。

### 批量操作

DAO 不提供批量接口，使用 GORM：

```go
db := repository.DB(ctx)
db.CreateInBatches(students, 100)
db.Where("ID IN ?", ids).Find(&list)
```

## 验证

**编译检查**：
✅ `go build ./...`  — gen-go-db 生成后

**注入完整性**：
□ `internal/repository/dao/zfx_dao.go` — `fx.Provide(NewXxxDao)` 已添加
□ `internal/zfx_internal.go` — `dao.FxDaoModule` 已包含

**运行时声明**：
□ `internal/init.go` — 需要事务的 RPC 方法已 `AddTag(TransactionTag, ...)`

## 相关参考

- YAML 定义格式：[[db-yaml-format]]
- gen-go-db 命令：[[gen-go-db-cli]]
