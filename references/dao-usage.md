# Generated DAO Usage Guide

> **场景触发器**: 用户说"使用 DAO""数据操作""InsertOne""FindByPrimaryKey""命名 SQL""FindByCondition""分页查询"时加载。

## 接口概览

每张表生成一个 DAO 接口，包含完整的单表 CRUD 操作：

```go
type IStudentDao interface {
    InsertOne(ctx, *model.Student) (int64, error)
    InsertOneIgnoreZeroValCols(ctx, *model.Student) (int64, error)

    UpdateByPrimaryKey(ctx, *model.Student) (int64, error)
    UpdateByPrimaryKeyIngoreZeroValCols(ctx, *model.Student) (int64, error)

    FindByPrimaryKey(ctx, id model.StudentPrimaryKey) (*model.Student, error)
    FindByStruct(ctx, *model.Student) ([]*model.Student, error)
    FindByCustomerRule(ctx, namingInfo *gormdb.NameingSqlArgInfo, args any) (any, error)
    FindByCondition(ctx, cond *conditonwhere.WhereClauseBuilder,
        order *gormdb.OrderBuilder, page *gormdb.Page) ([]*model.Student, *gormdb.PageResult, error)
    FindFirstOneByCondition(ctx, cond *conditonwhere.WhereClauseBuilder,
        order *gormdb.OrderBuilder) (*model.Student, error)
}
```

## 查询方式选择

DAO 提供三种查询方式，按需选择：

```
查询需求 →
├─ 简单等值查询（固定 WHERE 条件）
│   └─ FindByStruct  — 零代码，自动拼接索引字段
│      ✅ 运行时索引安全检查
│      ❌ 无分页
│      ❌ 不支持 LIKE / IN / BETWEEN
│
├─ 自定义查询（推荐默认）
│   └─ FindByCustomerRule  — 命名 SQL + FieldMask 动态 WHERE
│      ✅ LIKE / IN / BETWEEN / JOIN 全覆盖
│      ✅ 运行时索引安全检查（ValidateLeadingCol）
│      ✅ 分页内置（嵌入 db.Page 即可）
│      ✅ 跨数据库 SQL（MYSQL/DB2 独立模板）
│      ⚠️ 需编写 YAML + Arg/Res 类型 + 命名 SQL
│
└─ 轻量动态查询（无需预定义 YAML）
    └─ FindByCondition  — WhereClauseBuilder 链式构建
       ⚠️ 无索引安全校验
       ⚠️ 不支持 LIKE（ConditionLike 未开放）
       ⚠️ 依赖 GORM 生成 SQL，无跨库精确控制
```

| 能力 | FindByStruct | FindByCustomerRule | FindByCondition |
|------|:---:|:---:|:---:|
| 等值查询 | ✅ | ✅ | ✅ |
| 动态条件 | ❌ | ✅ (FieldMask) | ✅ (链式) |
| LIKE 模糊查询 | ❌ | ✅ | ❌ |
| 分页 | ❌ | ✅ (内置) | ✅ (内置) |
| 索引安全检查 | ✅ | ✅ ValidateLeadingCol | ❌ |
| 跨数据库 SQL | ❌ | ✅ MYSQL/DB2 | ❌ |
| 开发成本 | 零 | 中 | 低 |

> **AI 默认推荐**：自定义查询优先用 `FindByCustomerRule`。它能确保运行时索引校验和跨数据库兼容，对生产环境更安全。

```go
// 全字段插入（零值也会写入）
affected, _ := dao.InsertOne(ctx, &model.Student{Name: "张三", Age: 18})

// 自动剔除零值列（主键、索引列不受影响）
affected, _ := dao.InsertOneIgnoreZeroValCols(ctx, &model.Student{Name: "张三", Age: 18})
```

## 更新

```go
// 根据主键更新（需传入主键值，否则报错 "primary key is required"）
affected, _ := dao.UpdateByPrimaryKey(ctx, &model.Student{Id: 1, Name: "李四"})

// 忽略零值更新
affected, _ := dao.UpdateByPrimaryKeyIngoreZeroValCols(ctx, &model.Student{Id: 1, Name: "李四"})
```

## 按主键查询

```go
student, err := dao.FindByPrimaryKey(ctx, model.StudentPrimaryKey(1))
// 未找到返回 (nil, nil)，非 ErrRecordNotFound
```

## 按实体查询（FindByStruct）

```go
// 根据实体中非零值的索引列/主键列查询
students, _ := dao.FindByStruct(ctx, &model.Student{Age: 18})
```

查询逻辑：先检查主键 → 再检查索引列（取第一个命中索引的） → 其他非零普通列附加为 AND 条件。**编译时索引安全检查**，确保查询必然走索引，否则返回 `"query not use any index"`。

## 命名 SQL（FindByCustomerRule）

即 YAML 中 `self_query_rules` 定义的自定义查询。

```go
// 1. 必须初始化 FieldMask，否则 WithXxx() 空指针
arg := &model.StudentFindByAgeArg{
    FieldMask: conditonwhere.NewFieldMask(),
}
arg.WithAge(18).WithStuno("S001")

// 2. 执行命名 SQL
result, err := dao.FindByCustomerRule(ctx, dao.FindByAgeNamingInfo, arg)
list := result.([]*model.StudentFindByAgeRes)
```

### 分页命名查询

```go
arg := &model.StudentFindByAgeWithPageArg{
    FieldMask: conditonwhere.NewFieldMask(),
    Page:      gormdb.Page{PageNum: 1, PageSize: 20},
}
arg.WithAge(18)

result, _ := dao.FindByCustomerRule(ctx, dao.FindByAgeWithPageNamingInfo, arg)
pageRes := result.(*model.StudentFindByAgeWithPagePageRes)
// pageRes.ResultList   — 数据列表
// pageRes.PageResult   — 分页信息（CurrentPage, PageSize, TotalCount, TotalPage）
```

## 条件构建器（FindByCondition）

适用于查询条件灵活多变的场景：

```go
import "gitlab.allinfinance.com/aifgo/ag-core/contribute/agdb/conditonwhere"

cond := conditonwhere.NewWhereClauseBuilder()
if req.Name != "" {
    cond.And("NAME = ?", req.Name)
}
if req.Age > 0 {
    cond.And("AGE > ?", req.Age)
}

order := gormdb.NewOrderBuilder().Desc("ID")
page := &gormdb.Page{PageNum: 1, PageSize: 20}

list, pageResult, err := dao.FindByCondition(ctx, cond, order, page)
```

> ⚠️ `FindByCondition` 不含索引安全检查，需自行确保 WHERE 字段有索引。

## 初始化与 fx 注入

### 第一步：创建 DAO 模块（手动）

在 `internal/repository/dao/` 下创建 `zfx_dao.go`（不被 gen-go-db 覆盖）：

```go
// internal/repository/dao/zfx_dao.go
package dao

import "go.uber.org/fx"

var FxDaoModule = fx.Module("fx_dao",
    fx.Provide(
        NewStudentDao,
        // 后续新增表只需加一行：
        // NewTeacherDao,
        // NewCourseDao,
    ),
)
```

### 第二步：注册到内部模块

在 `internal/zfx_internal.go` 中：

```go
import "your-project/internal/repository/dao"

var FxInternalModule = fx.Module("fx-internal-module",
    config.FxAppConfigModule,
    svcgen.FxServiceWithProxyModule(),
    adpgen.FxAdapterModule(),
    dao.FxDaoModule,  // ← 新增
)
```

### 第三步：app 装配

```go
import (
    "gitlab.allinfinance.com/aifgo/ag-core/contribute/agdb/gormdb"
    agdb "gitlab.allinfinance.com/aifgo/ag-core/contribute/agdb"
)

app := fx.New(
    gormdb.FxAicGromdbModule,   // *gorm.DB → Repository
    agdb.FxAgDbModule,          // BaseDao + 事务中间件
    internal.FxInternalModule,   // 内部聚合（含 DAO、Config、Service...）
)
```

**优势**：
- `NewStudentDao` 依赖的 `*Repository` 和 `BaseDao` 由 fx 自动注入
- 新增表只改 `zfx_dao.go` 一行，不动入口
- 遵循 `config.FxAppConfigModule` 同层聚合模式

### 在 Service 中注入

```go
type StudentServiceImpl struct {
    studentDao dao.IStudentDao  // 面向接口
}

func NewStudentServiceImpl(studentDao dao.IStudentDao) *StudentServiceImpl {
    return &StudentServiceImpl{studentDao: studentDao}
}

func (s *StudentServiceImpl) GetStudent(ctx context.Context, req *pb.GetReq) (*pb.Student, error) {
    stu, _ := s.studentDao.FindByPrimaryKey(ctx, model.StudentPrimaryKey(req.Id))
    if stu == nil {
        return nil, status.Errorf(codes.NotFound, "学生不存在")
    }
    return convertToPB(stu), nil
}
```

## 事务

```go
// 自动事务（REQUIRED 传播）
err := repository.Transaction(ctx, func(txCtx context.Context) error {
    dao.InsertOne(txCtx, &model.Student{Name: "张三"})
    dao.InsertOne(txCtx, &model.Student{Name: "李四"})
    return nil // commit; return error → rollback
})
```

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

## 相关参考

- YAML 定义格式：[[db-yaml-format]]
- gen-go-db 命令：[[gen-go-db-cli]]
