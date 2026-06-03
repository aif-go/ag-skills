# Project Structure

## Standard Layout

```
<project>/
├── api/                          # 生成的接口代码（不可手动修改）
│   └── <service>/
│       ├── <service>.pb.go                    # protobuf 消息结构体
│       └── agserver_<service>_interface.go    # service 接口定义
│
├── cmd/
│   └── server/
│       ├── app.yml               # 服务配置
│       ├── main.go               # 入口
│       └── server                # 编译产物
│
├── idl/
│   └── api/                      # Proto 接口定义（手动编写 ✍）
│       └── <service>/
│           └── <service>.proto
│
├── internal/
│   ├── adpgen/                   # 生成的 adapter 层（不可修改）
│   │   ├── zfx_adapter.go
│   │   ├── adpinit/              # adapter 初始化
│   │   │   ├── zfx_adapter_init.go
│   │   │   ├── zfx_aghertz_*_adpinit.go
│   │   │   └── zfx_agkitex_*_adpinit.go
│   │   ├── hertz/                # Hertz (HTTP) adapter
│   │   │   └── <service>/
│   │   │       ├── aghertz_<service>_client.go
│   │   │       ├── aghertz_<service>_fx.go
│   │   │       └── aghertz_<service>_server.go
│   │   └── kitex/                # Kitex (gRPC) adapter
│   │       └── <service>/
│   │           ├── agkitex_<service>.go
│   │           ├── agkitex_<service>_agclient.go
│   │           ├── agkitex_<service>_client.go
│   │           ├── agkitex_<service>_fx.go
│   │           └── agkitex_<service>_server.go
│   │
│   ├── svcgen/                   # 生成的 service 代理（不可修改）
│   │   ├── zfx_service.go
│   │   ├── zfx_agservice_proxy_<service>.go
│   │   └── agservice_<service>_proxy.go
│   │
│   ├── service/                  # 入口薄层（委托 biz，手动编写 ✍）
│   │   └── agservice_<service>.go   ⭐ 不被覆盖
│   │
│   ├── biz/                       # 业务逻辑层（手动编写 ✍）
│   │   ├── <service>_biz.go       # 业务实现 + 编排
│   │   └── zfx_biz.go             # fx 模块注册
│   │
│   ├── repository/               # 数据访问层
│   │   ├── yaml/                   # 表 YAML 定义（手动编写 ✍）
│   │   │   └── <Table>.yaml
│   │   ├── model/                  # Model 结构体（gen-go-db 生成）
│   │   │   └── <table>_model.go
│   │   ├── dao/                    # DAO 接口 + 实现（gen-go-db 生成）
│   │   │   ├── <table>_dao.go      # CRUD 接口 + 实现
│   │   │   ├── <table>_constant.go # 命名 SQL 注册
│   │   │   ├── <table>_namingsql.go
│   │   │   ├── mysql_<table>_namingsql.go
│   │   │   └── db2_<table>_namingsql.go
│   │   └── dao/zfx_dao.go          # DAO fx 模块（手动编写 ✍）
│   │
│   ├── config/                    # 配置层（手动编写 ✍）
│   │   ├── <module>_config.go
│   │   └── zfx_config.go
│   ├── init.go
│   └── zfx_internal.go
│
├── third_party/                  # Proto 第三方依赖
├── go.mod
└── go.sum
```

## File Responsibilities

| 层 | 目录 | 职责 | 可否修改 |
|----|------|------|----------|
| **接口定义** | `idl/api/` | Proto 文件（gRPC + HTTP 定义） | ✅ 手动编写 |
| **接口代码** | `api/` | pb.go + interface.go | ❌ aggo 生成 |
| **Adapter** | `internal/adpgen/` | Kitex/Hertz 协议适配 | ❌ aggo 生成 |
| **Service 代理** | `internal/svcgen/` | 依赖注入代理 | ❌ aggo 生成 |
| **业务逻辑** | `internal/biz/` | 业务实现、编排、Gateway 接口定义 | ✅ 手动编写 |
| **入口** | `internal/service/` | 薄层，参数适配后委托 biz | ✅ 手动编写 |
| **配置** | `internal/config/` | 配置结构体 + fx 模块 | ✅ 手动编写 |
| **表定义** | `internal/repository/yaml/` | YAML 表结构定义 | ✅ 手动编写 |
| **Model** | `internal/repository/model/` | GORM Model 结构体 | ❌ gen-go-db 生成 |
| **DAO** | `internal/repository/dao/` | CRUD 接口 + 实现 | ❌ gen-go-db 生成 |
| **DAO fx** | `internal/repository/dao/zfx_dao.go` | DAO fx 模块注册 | ✅ 手动编写 |
| **入口** | `cmd/server/` | main.go + 配置 | ✅ 手动编写 |

## Key Rules

1. **Proto 隔离**：每个服务一个 proto 文件，放在 `idl/api/<service>/`
2. **生成不覆盖**：`agservice_*.go` 首次生成后不会重复覆盖，业务代码安全
3. **生成重跑安全**：`adpgen/` 和 `svcgen/` 可以安全重新生成（会覆盖）
4. **配置在 cmd**：配置文件统一放 `cmd/server/app.yml`
5. **依赖在 go.mod**：所有依赖通过 go module 管理

---

## 进阶：跨服务调用时的结构扩展

当需要调用其他微服务或接入基础设施时，在 `internal/` 下扩展：

```
internal/
├── gateway/    # 实现 biz 定义的 Gateway 接口（协议适配）
├── clients/    # 创建原生 client + 连接配置 + fx 注册
```

> 详见 [[gateway-patterns]]。

## 验证

> 项目初始化或新增模块后，执行 [[verification#项目初始化]]。
