# Code Patterns

## Proto IDL

```proto
syntax = "proto3";

package student;
option go_package = "myproject/api/student";

import "google/api/annotations.proto";

service StudentService {
  rpc GetStudent(GetStudentReq) returns (GetStudentResp) {
    option (google.api.http) = { get: "student/get" };
  }
  rpc CreateStudent(CreateStudentReq) returns (CreateStudentResp) {
    option (google.api.http) = {
      post: "student/create"
      body: "*"
    };
  }
}

// GET:   option (google.api.http) = { get: "path" };
// POST:  option (google.api.http) = { post: "path" body: "*" };
// Path:  option (google.api.http) = { get: "student/:Id" };
```

## Service + Biz

```go
// internal/biz/student_biz.go — 业务逻辑 + 编排
type StudentBiz struct {
    studentDao dao.IStudentDao
}

func NewStudentBiz(studentDao dao.IStudentDao) *StudentBiz {
    return &StudentBiz{studentDao: studentDao}
}

func (b *StudentBiz) GetStudent(ctx context.Context, req *pb.GetStudentReq) (*pb.GetStudentResp, error) {
    stu, _ := b.studentDao.FindByStruct(ctx, &model.Student{Stuno: req.Stuno})
    // ...
}


// internal/service/agservice_studentservice.go — 薄层入口（不被 aggo proto 覆盖）
type StudentServiceImpl struct {
    Biz *biz.StudentBiz
}

func NewStudentServiceImpl(biz *biz.StudentBiz) *StudentServiceImpl {
    return &StudentServiceImpl{Biz: biz}
}

func (s *StudentServiceImpl) GetStudent(ctx context.Context, req *pb.GetStudentReq) (*pb.GetStudentResp, error) {
    return s.Biz.GetStudent(ctx, req)
}
```

## Configuration

```go
// internal/config/module_config.go — 标准三件套
const XxxKey = "xxx"

type XxxConfig struct {
    Host string `required:"true"`
    Port int
}

func DefaultXxxConfig() XxxConfig { return XxxConfig{Port: 8080} }

func NewXxxConfig(binder ag_conf.IBinder) (*XxxConfig, error) {
    cfg := DefaultXxxConfig()
    if err := binder.Bind(&cfg, XxxKey); err != nil {
        return nil, err
    }
    return &cfg, nil
}
```

## Fx Module Placement

### 框架模块（只在 main.go）

```go
// cmd/server/main.go — 框架级模块只在这里
var mainFx = fx.Module("main",
    fxs.FxAgConfModule,           // ag_conf.IBinder
    gormdb.FxAicGromdbModule,     // *gorm.DB → Repository
    agdb.FxAgDbModule,            // BaseDao + 事务中间件
    // ... 协议模块 ...
    internal.FxInternalModule,    // 所有 internal/ 子模块
)
```

> ⚠️ `gormdb.FxAicGromdbModule`、`agdb.FxAgDbModule` 是框架级模块，**禁止在 internal/ 任何子模块中引用**。

### 项目模块构造器注册位置

| 构造器类型 | 注册文件 | 格式 |
|-----------|---------|------|
| `NewXxxDao` | `zfx_dao.go` | `fx.Provide(NewXxxDao, NewYyyDao)` |
| `NewXxxConfig` | `zfx_config.go` | `fx.Provide(NewXxxConfig)` |
| `NewXxxBiz` | `zfx_biz.go` | `fx.Provide(NewXxxBiz)` |

新增时在已有 `fx.Provide()` 中追加参数，不新建 `fx.Provide()` 调用。

### 组装链

```go
// internal/zfx_internal.go — 汇总所有 internal/ 子模块
var FxInternalModule = fx.Module("fx-internal-module",
    config.FxAppConfigModule,
    dao.FxDaoModule,
    biz.FxBizModule,
    svcgen.FxServiceWithProxyModule(),
    adpgen.FxAdapterModule(),
)
```

## Rules

### 代码生成
- Proto 文件在 `idl/api/<service>/` 下手动编写
- `-p` 支持逗号分隔：`-p go,api,server,kitex,hertz,service`
- `-m` 仅对 kitex/hertz 有效（server|client）
- 生成后必须 `go mod tidy && go build ./...`
- adpgen/ svcgen/ 为生成代码，不可手动修改
- `-p service` 生成的 `agservice_*.go` 不覆盖已有文件

### 分层
- 业务逻辑写在 `internal/biz/`，service/ 只做薄层委托
- 微服务调用遵循 Gateway 模式

### Fx 模块
- DAO 构造器声明在 `zfx_dao.go`，禁止散落在 biz/ 中
- 框架模块（gormdb/agdb）只在 main.go，禁止引入 internal/ 子模块
- 各层 zfx_*.go 按约定文件名注册构造器
