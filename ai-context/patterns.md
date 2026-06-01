# Code Patterns

## Proto IDL

### Basic Service
```proto
syntax = "proto3";

package student;
option go_package = "myproject/api/student";

import "google/api/annotations.proto";

service StudentService {
  rpc GetStudent(GetStudentReq) returns (GetStudentResp) {
    option (google.api.http) = {
      get: "student/get"
    };
  }
}

message GetStudentReq {
  int64 id = 1;
}

message GetStudentResp {
  int64 id = 1;
  string name = 2;
}
```

### HTTP Methods
```proto
// GET
option (google.api.http) = { get: "path" };

// POST
option (google.api.http) = {
  post: "path"
  body: "*"
};

// Path parameter
option (google.api.http) = { get: "student/:Id" };
```

## aggo Commands

### Generation Flags
```
-p 插件列表（逗号分隔，默认 all）
-m 模式（仅对 kitex/hertz 有效：server|client|all）

-p go      → pb.go 接口代码
-p api     → 接口描述
-p server  → adapter 基础
-p kitex   → Kitex adapter（需 -m server|client）
-p hertz   → Hertz adapter（需 -m server|client）
-p service → service 层（业务入口，不覆盖已有文件）
```

### 服务端生成（推荐合并写法）
```bash
# 一条命令生成全部服务端代码
aggo proto -p go,api,server,kitex,hertz,service -m server -e ./idl/api ./idl/api/student/student.proto
```

### 客户端生成
```bash
# 仅 kitex/hertz 受 -m client 影响
aggo proto -p kitex,hertz -m client -e ./idl/api ./idl/api/student/student.proto
```

## Service Implementation

```go
// internal/biz/student_biz.go — 业务逻辑
type StudentBiz struct { /* 注入 repository, gateway 等 */ }

func (b *StudentBiz) GetStudent(ctx context.Context, req *pb.GetStudentReq) (*pb.GetStudentResp, error) {
    return &pb.GetStudentResp{Id: req.Id, Name: "result"}, nil
}

// internal/service/agservice_studentservice.go — 薄层入口
// 此文件不会被 aggo proto 覆盖
type StudentServiceImpl struct { Biz *StudentBiz }

func (s *StudentServiceImpl) GetStudent(ctx context.Context, req *pb.GetStudentReq) (*pb.GetStudentResp, error) {
    return s.Biz.GetStudent(ctx, req)
}
```

## Project Structure

```
project/
├── api/              # 生成的接口代码
├── cmd/server/       # 入口 + 配置
│   ├── app.yml
│   └── main.go
├── idl/api/          # proto 定义（手动编写 ✍）
├── internal/
│   ├── adpgen/       # 生成的 adapter（不可修改）
│   ├── svcgen/       # 生成的 service 代理（不可修改）
│   ├── biz/          # 业务逻辑（手动编写 ✍）
│   └── service/      # 入口薄层（手动编写 ✍）
└── third_party/      # proto 依赖
```

## Configuration

```go
// config/mymodule.go — 标准三件套
package config

const MyConfigKey = "mymodule"

type MyConfig struct {
    Host string `required:"true"`
    Port int
}

func DefaultMyConfig() MyConfig {
    return MyConfig{Port: 8080}
}

func NewMyConfig(binder ag_conf.IBinder) (*MyConfig, error) {
    cfg := DefaultMyConfig()
    if err := binder.Bind(&cfg, MyConfigKey); err != nil {
        return nil, err
    }
    return &cfg, nil
}
```

## Rules

- Proto 文件在 `idl/api/<service>/` 下手动编写
- adpgen/ svcgen/ 目录为生成代码，不可手动修改
- 业务逻辑统一写在 `internal/biz/`，service/ 只做委托
- `-p` 支持逗号分隔多值（如 `-p go,api,server`）
- `-m` 仅对 kitex/hertz 有效，控制生成 server 端还是 client 端
- `-p service` 生成的 `agservice_*.go` 不覆盖已有文件
- 生成后必须 `go mod tidy && go build ./...`
