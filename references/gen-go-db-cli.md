# gen-go-db Command Reference

> **场景触发器**: 用户说"生成数据库代码""gen-go-db""生成 DAO""生成 Model"时加载。

## 命令格式

```bash
gen-go-db db -i <yaml-path> -o <output-dir> -m <module> [options]
```

从 YAML 表定义生成 Go Model + DAO + 命名 SQL 代码。

## 参数

| 参数 | 必填 | 默认值 | 说明 |
|------|:----:|--------|------|
| `-i` / `--input` | ✅ | — | YAML 文件路径或目录 |
| `-o` / `--output` | ✅ | — | 输出目录（自动拼接 `repository/model/` 和 `repository/dao/`） |
| `-m` / `--module` | ✅ | — | Go 模块名，影响 import 路径和 internal 前缀 |
| `-T` / `--table` | ❌ | 全部表 | 指定表名，逗号分隔多个 |
| `-d` / `--dbtype` | ❌ | 不指定（生成全部） | 指定数据库类型：`mysql` / `db2`。不指定时默认生成 MYSQL + DB2 两套 |

## 使用示例

```bash
# 基本用法：YAML 目录 → 全部表
gen-go-db db -i ./repository/yaml -o ./ -m myproject

# YAML 在 internal/ 下时，-m 需包含 internal 前缀以生成正确的 import 路径：
gen-go-db db -i ./internal/repository/yaml -o ./internal -m myproject/internal

# 单文件
gen-go-db db -i ./repository/yaml/TM_USER.yaml -o ./ -m myproject

# 指定表名
gen-go-db db -i ./repository/yaml -o ./ -m myproject -T TM_USER

# 生成 MYSQL + DB2 两套（推荐，不指定 -d 即可）
gen-go-db db -i ./repository/yaml -o ./ -m myproject

# 只生成 MySQL 类型
gen-go-db db -i ./repository/yaml -o ./ -m myproject -d mysql

# 只生成 DB2 类型
gen-go-db db -i ./repository/yaml -o ./ -m myproject -d db2
```

## 输出文件

**`-o` 自动拼接 `repository/model/` 和 `repository/dao/`**。例如 `-o ./internal` 输出到 `internal/repository/`。

对于表 `STUDENT`，生成 6 个文件：

```
repository/
├── model/
│   └── student_model.go              # Model 结构体 + GORM 标签
└── dao/
    ├── student_dao.go                # DAO CRUD 接口 + 实现
    ├── student_constant.go           # 命名 SQL 信息注册
    ├── student_namingsql.go          # 命名 SQL 初始化入口
    ├── mysql_student_namingsql.go    # MySQL 特定 SQL
    └── db2_student_namingsql.go      # DB2 特定 SQL
```

所有生成文件有 `DO NOT EDIT` 标记，不可手动修改。

> **⚠️ `-d` 参数限制**：`-d` 仅接受单个值（`mysql` 或 `db2`），**不支持** `-d mysql+db2` 或 `-d mysql,db2` 等多值组合写法。传入多值会导致生成的代码包含非法 Go 标识符（如 `MYSQL+DB2_Student_...`）。生成全部类型只需**不指定 `-d` 参数**即可。
>
> `-m` 同时决定生成代码的 import 前缀。若 `repository/` 在 `internal/` 下，`-m` 需包含 `/internal`，如 `-m myproject/internal`。

## 验证

**编译检查**：
✅ `go build ./...`  — 生成后必须验证，确保 `mysql_*.go` 和 `db2_*.go` 编译通过

**代码规范**：
□ `-d` 参数只用单个值或不指定，不使用 `+` / `,` 组合

## 相关参考

- YAML 定义格式：[[db-yaml-format]]
- DAO 使用指南：[[dao-usage]]
