# Proto IDL Patterns

## File Location

```
idl/api/<service-name>/<service-name>.proto
```

Example: `idl/api/student/student.proto`

## Basic Service Template

```proto
syntax = "proto3";

package student;
  option go_package = "myproject/api/student";

import "google/api/annotations.proto";

// Service definition
service StudentService {
  rpc GetStudent(GetStudentReq) returns (GetStudentResp) {
    option (google.api.http) = {
      get: "student/get"
    };
  }
  rpc CreateStudent(CreateStudentReq) returns (CreateStudentResp) {
    option (google.api.http) = {
      post: "student/create"
      body: "*"
    };
  }
  rpc DeleteStudent(DeleteStudentReq) returns (DeleteStudentResp) {
    option (google.api.http) = {
      delete: "student/:Id"
    };
  }
}

// Request messages
message GetStudentReq {
  int64 id = 1;
}

message CreateStudentReq {
  string name = 1;
  int32 age = 2;
}

message DeleteStudentReq {
  int64 id = 1;
}

// Response messages
message GetStudentResp {
  int64 id = 1;
  string name = 2;
  int32 age = 3;
}

message CreateStudentResp {
  int64 id = 1;
}

message DeleteStudentResp {
  bool success = 1;
}
```

## HTTP Annotation Rules

### GET - 查询
```proto
rpc Get(GetReq) returns (GetResp) {
  option (google.api.http) = {
    get: "resource/get"
  };
}
```

### POST - 创建/提交
```proto
rpc Create(CreateReq) returns (CreateResp) {
  option (google.api.http) = {
    post: "resource/create"
    body: "*"    // "*" = 使用整个请求体
  };
}
```

### DELETE - 删除（带路径参数）
```proto
  rpc Delete(DeleteReq) returns (DeleteResp) {
    option (google.api.http) = {
      delete: "resource/:Id"
    };
  }
}
// DeleteReq 中必须有 id 字段:
// proto `id` → Go `Id` → 路径变量 `:Id` 匹配 Go 导出名
```proto
message DeleteReq {
  int64 id = 1;
}
```

### PUT - 更新
```proto
  rpc Update(UpdateReq) returns (UpdateResp) {
    option (google.api.http) = {
      put: "resource/:Id"
      body: "*"
    };
  }
```

## Proto Required Elements

1. `syntax = "proto3";` - 必须是 proto3
2. `package <name>;` - 包名
3. `option go_package = "<module>/api/<name>";` - Go 包路径，需包含模块名
4. `import "google/api/annotations.proto";` - HTTP 注解依赖

## Data Types

| Proto Type | Go Type | Usage |
|------------|---------|-------|
| `int64` | `int64` | ID、时间戳 |
| `int32` | `int32` | 数值字段 |
| `string` | `string` | 文本字段 |
| `bool` | `bool` | 布尔值 |
| `repeated T` | `[]T` | 列表字段 |
| `bytes` | `[]byte` | 二进制数据 |

## Best Practices

- Service name 用 PascalCase：`StudentService`
- RPC method name 用 PascalCase：`GetStudent`
- Message name 用 PascalCase + Req/Resp 后缀：`GetStudentReq`
- Field name 用 snake_case：`student_id`
- 每个 service 一个 proto 文件
- HTTP 路径**不带 `/` 前缀**：`"student/get"` 而非 `"/student/get"`
- **路径变量 `:<Name>` 必须与请求消息字段的 Go 导出名一致**（proto `stuno` → Go `Stuno` → 路径 `:Stuno`），否则路由匹配时无法正确绑定
- go_package 为完整模块路径
