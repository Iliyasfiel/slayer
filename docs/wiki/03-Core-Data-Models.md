# 03 - 核心数据模型

所有领域模型都是 **Pydantic v2**（`BaseModel`），位于 `slayer/core/`。这一层定义"系统能表达什么"，是其它所有层的基础。

## 主要类一览

| 类 | 文件 | 作用 |
| --- | --- | --- |
| `DatasourceConfig` | [models.py](../../slayer/core/models.py) | 一个数据库连接（含类型、host、credentials、`${ENV}` 引用） |
| `SlayerModel` | 同上 | 一个语义模型（对应物理表 / SQL 子查询 / 多阶段查询） |
| `Column` | 同上 | 模型上的一个可被 group-by / 聚合 / 过滤的字段 |
| `ModelMeasure` | 同上 | 一条命名公式（库），可在 query 中以裸名引用 |
| `ModelJoin` | 同上 | 模型之间的 LEFT/INNER JOIN 声明 |
| `Aggregation` | 同上 | 用户自定义的非内置聚合 |
| `SlayerQuery` | [query.py](../../slayer/core/query.py) | 用户查询描述（measures / dimensions / time_dimensions / filters / order / limit） |
| `ColumnRef` | 同上 | 维度引用（`customers.regions.name` 这种多跳会自动 split） |
| `TimeDimension` | 同上 | 时间维度引用（带 granularity / date_range） |
| `ModelExtension` | 同上 | 运行时给模型追加 columns/measures/joins（不写盘） |
| `DataType` | [enums.py](../../slayer/core/enums.py) | `TEXT / INT / DOUBLE / BOOLEAN / DATE / TIMESTAMP`，与 sqlglot 字节对齐 |
| `TimeGranularity` | 同上 | `SECOND / MINUTE / HOUR / DAY / WEEK / WEEK_SUNDAY / MONTH / QUARTER / YEAR` |
| `JoinType` | 同上 | `LEFT` / `INNER` |
| `BUILTIN_AGGREGATIONS` | 同上 | sum/avg/min/max/count/count_distinct/median/percentile/stddev_samp/pop/var_samp/pop/corr/covar_samp/pop/first/last/weighted_avg/count_distinct_approx |
| `SessionPolicy` | [policy.py](../../slayer/core/policy.py) | RLS 会话策略（列规则 + 跨表 join 规则） |
| `ColumnFilterRule` / `JoinFilterRule` | 同上 | RLS 规则的两种类型 |
| `NumberFormat` / `NumberFormatType` | [format.py](../../slayer/core/format.py) | 智能数字格式化（货币/百分比） |
| `RootModelRecommendation` 等 | [recommend.py](../../slayer/core/recommend.py) | `slayer recommend-root-model` 的返回类型 |
| `SlayerError` 及子类 | [errors.py](../../slayer/core/errors.py) | 异常体系 |

## 引用模式（Mode A / Mode B）

源码注释中频繁出现的"两种引用模式"是 SLayer 设计上的核心约定（**DEV-1369**）：

| 模式 | 适用字段 | 解析器 | 允许写法 |
| --- | --- | --- | --- |
| **Mode A (SQL)** | `Column.sql`、`Column.filter`、`SlayerModel.filters` | **sqlglot** 解析 | 任意 SQL 表达式（`json_extract`、`coalesce`、`CASE WHEN`），bare 名引用本表，`__` 分隔的 join path（`customers__regions.name`） |
| **Mode B (DSL)** | `ModelMeasure.formula`、`SlayerQuery.measures` / `filters`、所有其它查询字段 | **Python AST** 解析 | 仅 `Column` / `ModelMeasure` 引用；`.` 分隔的 join path；冒号聚合语法（`revenue:sum`、`*:count`）；transform 调用；算术与布尔 |

> **唯一例外**：`Column.name` 接受 `__`，因为 `_query_as_model` 会把跨模型列拍平成 `stores__name` 这样的虚拟模型列名。

> 单点真相：[`docs/concepts/references.md`](../../concepts/references.md)。

DSL 模式（Mode B）有一个统一的标量白名单 [`SCALAR_PASSTHROUGH`](../../slayer/core/formula.py)，所有 Mode B 入口共享：

- NULL：`coalesce`, `nullif`, `ifnull`
- 数学：`round`, `abs`, `ceil`/`ceiling`, `floor`, `power`/`pow`, `sqrt`, `exp`, `ln`, `log`, `log10`, `log2`, `mod`, `sign`, `trunc`
- 标量大小：`greatest`, `least`
- 字符串：`lower`, `upper`, `trim`, `ltrim`, `rtrim`, `replace`, `substr`/`substring`, `instr`, `length`, `concat`

> 扩展时：**只往 `SCALAR_PASSTHROUGH` 加**，不要新建平行白名单（CLAUDE.md 明确点出）。

## 关键模型详解

### `SlayerModel`

```python
class SlayerModel(BaseModel):
    version: int = 7                      # 当前 schema 版本
    name: str                             # 必填，需匹配 ^[a-zA-Z_][a-zA-Z0-9_]*$，不含 __/./:
    description: str | None = None
    data_source: str | None = None        # 表底模型必填
    sql_table: str | None = None          # 物理表
    sql: str | None = None                # 显式 SQL 子查询
    source_queries: list[SlayerQuery] | None = None   # 查询链 / DAG
    columns: list[Column] = []            # 统一列（v2 起 dimensions/measures 合并）
    measures: list[ModelMeasure] = []     # 命名公式库
    aggregations: list[Aggregation] = []  # 自定义聚合
    joins: list[ModelJoin] = []           # LEFT/INNER 声明
    filters: list[str] = []               # 总是应用的 WHERE（Mode A SQL）
    default_time_dimension: str | None = None
    query_variables: dict[str, Any] = {}  # 给 {var} 占位符提供默认值
    meta: dict[str, Any] = {}             # 用户自定义元数据
```

**三种 source 模式互斥**：

1. `sql_table` —— 物理表
2. `sql` —— 显式 SQL 子查询（DEV-1330 起 data_source 必填）
3. `source_queries` —— 查询链/多阶段 DAG（内部 Stage 必须有 name，name 唯一、不能自指）

> **重要**：表底模型必须有非空 `data_source`（v4 校验）；查询链模型在 save 时由 `engine._validate_and_populate_cache` 自动填入。

### `Column`

```python
class Column(BaseModel):
    name: str                # 不含 . 与 :
    sql: str | None = None   # 默认等于 name
    type: DataType | None = None
    primary_key: bool = False
    description: str | None = None
    label: str | None = None
    hidden: bool = False
    format: NumberFormat | None = None     # 智能输出格式化
    allowed_aggregations: list[str] | None = None   # 白名单
    filter: str | None = None              # 聚合时 CASE WHEN（Mode A）
    sampled: str | None = None             # 旧式 sample 文本（v6）
    sampled_values: list[str] | None = None  # 类别列前 50 频次值（v7）
    distinct_count: int | None = None      # 真实基数（≤50 时填）
    meta: dict[str, Any] = {}
```

**默认聚合规则**：按 `DataType` 在 `DEFAULT_AGGREGATIONS_BY_TYPE` 中查表；PK 列强制只允许 `count` / `count_distinct`。详细规则见 `slayer/core/enums.py`。

### `ModelMeasure`

```python
class ModelMeasure(BaseModel):
    name: str
    formula: str            # Mode B DSL
    label: str | None = None
    description: str | None = None
    type: DataType | None = None   # 结果类型，可选
    meta: dict[str, Any] = {}
```

> 当 `type` 设置且不是 `TEXT` 时，SQL 生成器会用 `CAST(... AS <type>)` 包裹外层聚合（见 `SQLGenerator._wrap_cast_for_type`）。

### `ModelJoin`

```python
class ModelJoin(BaseModel):
    name: str | None = None                # 可选显示名
    target_model: str                      # 目标模型名（必填）
    type: JoinType = JoinType.LEFT
    on_sql: str | None = None              # ON 条件（Mode A SQL 解析）
    relationship: str | None = None        # 语义关系（OSI: many-to-one / one-to-one）
    description: str | None = None
    meta: dict[str, Any] = {}
```

### `SlayerQuery`

```python
class SlayerQuery(BaseModel):
    version: int = 3
    name: str | None = None                # 链式 stage 名（首段之外都需要）
    source_model: str | SlayerModel | ModelExtension
    measures: list[MeasureSpec] = []       # 字符串简写或 {formula, name, label}
    dimensions: list[str | ColumnRef] = []
    time_dimensions: list[TimeDimension] = []
    filters: list[str] = []                # Mode B DSL，含 {var} 占位符
    order: list[OrderBy] = []
    limit: int | None = None
    offset: int | None = None
    variables: dict[str, Any] = {}         # 给 {var} 提供值
    whole_periods_only: bool = False       # 时间窗口时排掉不完整周期
    distinct_dimension_values: bool = True # DEV-1543: dim-only 自动 dedup
    query_variables: dict[str, Any] = {}   # DEV-1336: 给 source_queries 的占位符
    row_limit: int | None = None
    row_offset: int | None = None
    meta: dict[str, Any] = {}
    model_config = ConfigDict(extra="forbid")
```

> **唯一查询模式**：`query_nested` / `POST /query {"queries": [...]}` / `engine.execute(query=[...])` 都是**多 stage DAG** 形式；引擎会按 Kahn 算法重排，最后一条作为 entry point / DAG 根。

#### 关键构造期校验（DEV-1369 / DEV-1543）

- `filters` 中出现 `OVER (...)` 窗口函数 → `ValueError` 指向 `WINDOW_IN_FILTER_ERROR`
- `distinct_dimension_values=True` 时 `measures` 非空 + `dimensions` 与 `time_dimensions` 都空 → `DistinctDimensionValuesError`
- `source_queries` 中非最后一段必须 `name` 非空 + 无重名 + 无自指 + 根不被任何其它段引用

#### 变量占位符

- 写法：`filters=["status = '{status_val}'", "amount > {min_amount}"]`
- 解析器：`substitute_variables`（`slayer/core/query.py`）
- 值必须是 str / int / float；未定义会抛错；`{{` / `}}` 转义字面量
- 优先级：runtime kwarg > stage `query_variables` > outer query `query_variables` > model defaults

### `SessionPolicy`（行级安全）

```python
class SessionPolicy(BaseModel):
    version: Literal[1] = 1
    data_filters: tuple[ColumnFilterRule | JoinFilterRule, ...] = ()

    class ColumnFilterRule(BaseModel):
        column: str
        value: Scalar | tuple  # scalar → =, list/tuple 非空 → IN
        on_unapplicable: Literal["block", "pass"] = "block"
        name: str | None = None

    class JoinFilterRule(BaseModel):
        target_table: str
        join_path: tuple[str, ...]   # 每条 "from.from = to.to"
        column: str
        value: Scalar | tuple
        on_unapplicable: Literal["block", "pass"] = "block"
        name: str | None = None
```

- 不可变：`frozen=True` + `extra="forbid"`；`SessionPolicy.version=Literal[1]` 失败闭（未知版本报错）。
- 行为：
  - 任何带 `JoinFilterRule` 的策略必须同时含至少一个 `on_unapplicable="block"` 的 `ColumnFilterRule`（mandatory backstop）。
  - `value` 空列表被拒绝（degenerate `IN` 不允许）。
  - 应用点：`engine._execute_pipeline` 与 `get_column_types` 都在 `SQLGenerator.generate` 之后立即调用 `apply_session_policy`。
  - 实现细节见 [`slayer/sql/session_policy.py`](../../slayer/sql/session_policy.py)：纯 sqlglot transform，每个物理表 `FROM t` → `FROM (SELECT * FROM t WHERE ...) AS t`（CTE/derived 跳过；ClickHouse ≥ 25.4 才支持相关子查询否则失败闭）。

## 数字格式化（`format.py`）

- `NumberFormatType`：`CURRENCY` / `PERCENTAGE` / `DECIMAL` / `INTEGER` / `AUTO`
- `NumberFormat(pattern="USD", decimals=2, locale="en_US")` 描述具体格式
- 输出端 `format_number` 按 locale 渲染，附带 column 元数据时 `inspect_model` 自动套用

## `enums.py` 中的关键枚举

```python
class DataType(StrEnum):
    TEXT = "TEXT"
    INT = "INT"
    DOUBLE = "DOUBLE"
    BOOLEAN = "BOOLEAN"
    DATE = "DATE"
    TIMESTAMP = "TIMESTAMP"

# 旧式 type 字符串 → 新值（dev 输入友好）
_LEGACY_DATATYPE_ALIASES = {
    "string": "TEXT", "number": "DOUBLE", "integer": "INT",
    "time": "TIMESTAMP", "date": "DATE", "boolean": "BOOLEAN",
    # 旧聚合伪类型被丢弃
    "count": None, "count_distinct": None, "sum": None, ...
}
```

> 旧 `string` / `number` 等 agent 输入会在 `Column.type` / `ModelMeasure.type` 的 lenient `before`-validator 中自动映射为新枚举值。

## `errors.py` 中的异常体系

```
SlayerError
├── AmbiguousModelError            # 多个 DS 同名
├── EntityResolutionError          # 实体解析失败
├── MemoryNotFoundError            # 找不到 memory
├── SchemaDriftError               # 执行时检测到 schema 漂移
├── ForcedFilterError              # RLS 应用失败
├── DistinctDimensionValuesError   # dim-only + 空 dim
├── ColumnCycleError               # 派生列循环
├── RecommendedRootModelError
└── ...
```

大部分异常同时继承 `ValueError`，保证历史 `except ValueError` 调用点仍能捕获。

## `formula.py` 中的解析器

`slayer/core/formula.py` 是 Mode B 解析核心：

- `parse_formula(formula_str) -> FieldSpec` 把 `"change(cumsum(revenue:sum))"` 解析为嵌套 `TransformField` 树
- `parse_filter(formula_str) -> list[Predicate]` 处理 `column <op> literal` 形式
- 内置 transform 集合见模块顶部 `TIME_TRANSFORMS` / `TIMELESS_TRANSFORMS` / `RANK_FAMILY_TRANSFORMS`
- 标量函数白名单见 `SCALAR_PASSTHROUGH`（见前文）
- 聚合写法（`revenue:sum`）由 `refs.split_agg_suffix` 拆分

## `refs.py` 中的工具

集中放跨模块共享的正则和辅助：

- `AGG_REF_RE` —— 聚合引用的正则
- `IDENT_OR_PATH_RE` —— `customers.regions.name` 形式的多跳路径
- `split_agg_suffix("revenue:sum") -> ("revenue", "sum")`
- `canonical_agg_name(...)` —— 归一化聚合名（处理 `count_distinct_approx` / `approx_count_distinct` 别名）

## `recommend.py`

`slayer recommend-root-model` 的返回类型：

- `RootModelRecommendation(root_model, paths, reachable, coverage, message, warnings)`
- `ItemPath(item, hops, ...)` —— 单个 item 到达 root 的多跳路径
- `CandidateCoverage(...)` —— 不可达时的帕累托前沿候选
- `render_recommendation_markdown(rec)` —— 给 MCP / CLI 用的 markdown 渲染

## Pydantic 与迁移

- 三个核心类型各自带 `version`：`SlayerModel.v7`、`SlayerQuery.v3`、`DatasourceConfig.v1`
- 所有 load 路径都通过 `@model_validator(mode="before")` 钩子调用 `slayer/storage/migrations.py::migrate` 链式升级
- 写入永远 emit 当前版本
- 见 [`slayer/storage/migrations.py`](../../slayer/storage/migrations.py) 与各 `vN_migration.py`

## 下一步

- 想看这些类如何被 `enrich_query` 消费成 `EnrichedQuery`：[04 - 查询生命周期](04-Query-Lifecycle.md)
- 想看 Pydantic 模型如何在存储层保存：[06 - 存储后端](06-Storage-Backends.md)
