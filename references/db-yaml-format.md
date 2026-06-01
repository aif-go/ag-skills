# Table YAML Definition Format

> **场景触发器**: 用户说"定义表结构""设计表""YAML 表定义""添加索引""自定义查询"时加载。

## 概述

表 YAML 是 `gen-go-db` 流水线的中间格式，定义数据库表结构的全部元信息。存放于 `repository/yaml/{TableName}.yaml`。

## 顶层结构

```yaml
table_name: TM_MEDIA_ACT       # 必填：数据库表名
columns:                       # 必填：列定义
  - { column_def }
primary_key:                   # 可选：主键列名
  - SEQ
constraints:                   # 可选：约束（唯一索引）
  - { constraint_def }
indexes:                       # 可选：普通索引
  - { index_def }
self_query_rules:              # 可选：自定义查询
  QueryName:
    { query_def }
```

## 列定义

### 必填字段

```yaml
- name: SEQ                    # 列名（全大写+下划线）
  type: int64                  # 类型
  not_null: true
  auto_increment: true
  support_update: false        # 是否允许 UPDATE
```

### 可选字段

```yaml
- name: NAME
  type: string
  length: "30"                 # 长度（非空时输出）
  not_null: true
  default: "xxx"               # 默认值（非空时输出）
  auto_increment: false
  support_update: true
  description: 姓名             # 描述（非空时输出）
  tag: ///@create              # 特殊标记
```

### 类型映射

| YAML type | Go 类型 | 额外导入 |
|-----------|---------|---------|
| `int` / `int32` / `tinyint` / `smallint` | `int` | — |
| `int64` / `bigint` | `int64` | — |
| `float` / `float32` / `double` / `float64` / `decimal` | `float64` | — |
| `string` / `varchar` / `char` / `text` | `string` | — |
| `bool` / `boolean` | `bool` | — |
| `time` / `datetime` / `timestamp` / `date` | `time.Time` | `"time"` |

### Tag 标记

| Tag 值 | 效果 |
|--------|------|
| `///@create` | 自动创建时间字段 |
| `///@update` | 自动更新时间字段 |
| `///@javaVersion` | 乐观锁字段 |

## 主键

```yaml
primary_key:
  - SEQ                     # 单主键
  # - SEQ2                  # 复合主键
```

## 索引

```yaml
indexes:
  - name: INDEX1_TM_TABLE
    columns:
      - BIZ_DATE             # 第一列 = 引导列（必须命中才能使用该索引）
      - ACTION_CD
```

索引影响：GORM 标签、`IndexLeadingCols` 变量、命名 SQL 的索引安全检查。

## 自定义查询（self_query_rules）

### 普通模式

```yaml
self_query_rules:
  NoPageQuery:
    select_fields: CARDNO,NAME          # 查询字段，* 表示全部
    page: false                          # 是否分页
    where:
      operator: AND
      conditions:
        - expr: BIZ_DATE = @BizDate     # @ 前缀 = 命名参数
        - expr: ACTION_CD = @ActionCd
```

- `select_fields: "*"` → 查询全部列，返回主结构体
- `page: true` → 生成分页方法
- `@Param` → 自动驼峰转换：`BIZ_DATE` → `BizDate`

### WHERE 条件树（支持嵌套）

```yaml
where:
  operator: OR
  conditions:
    - operator: AND
      conditions:
        - expr: BIZ_DATE = @BizDate
        - expr: ACTION_CD = @ActionCd
    - operator: AND
      conditions:
        - expr: ADDRESS = @Address
        - expr: BIZ_DATE IN @BizDateSlice    # IN 查询，参数为切片
```

支持操作符：`=` / `!=` / `>` / `<` / `>=` / `<=` / `in` / `not in` / `between`

### 动态模板模式

```yaml
QueryByCondition:
  select_fields: "*"
  page: true
  dynamic_sql: true
  sql_template: "SELECT * FROM TM_USER WHERE NAME = @Name AND AGE > @Age"
  Where_params:
    - colname: NAME
      paraname: Name
      slice: false
      type: string
    - colname: AGE
      paraname: Age
      slice: false
      type: int
```

> ⚠️ **`dynamic_sql: true + page: true` 时，`sql_template` 必须写成单行字符串**。YAML 的 `>` 折叠块会引入换行，gen-go-db 拼接 `LIMIT @Start,@End` 时也放在新行，导致生成的 Go 字符串内换行编译失败。单行用引号包裹即可。

## 人工编辑

YAML 可直接手动编辑：调整 `select_fields`、追加条件、修改分页、添加动态 SQL。注意重新运行 `gen-go-db yaml` 会覆盖——建议只编辑 YAML 后单独运行 `gen-go-db db`。

## 相关参考

- gen-go-db 命令：[[gen-go-db-cli]]
- DAO 使用指南：[[dao-usage]]
