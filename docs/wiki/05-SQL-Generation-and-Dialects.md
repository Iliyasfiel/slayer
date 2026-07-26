# 05 - SQL 生成与方言

## `SQLGenerator` 概览

[`slayer/sql/generator.py::SQLGenerator`](../../slayer/sql/generator.py) 接收 `EnrichedQuery`，输出最终 SQL。其设计原则：

1. **只接受 `EnrichedQuery`**：所有"模型 / 字段"查找都已在 `enrich_query` 阶段完成；生成器不查 storage。
2. **AST 化构造**：使用 sqlglot 的 `exp.*` 节点而非字符串拼接；只在必须时回退到字符串（如复杂 `time_shift` 自连接）。
3. **委托方言**：所有"此方言特有"的逻辑都在 `SqlDialect` 子类上。
4. **`CAST` 包装 + 保留字引用**统一在最终 emit 之前处理。

## 主流程（`SQLGenerator._generate_base`）

```text
EnrichedQuery
   │
   ├─ source_model 物理化（CTE chain / sql subquery / sql_table）
   │
   ├─ join 子句（按 enriched 中收集的 join targets，diamond 走 `__` 路径别名）
   │
   ├─ 决定 needs_group_by：
   │     dim_only_dedup  OR  has_aggregation  OR  cross_model_measures  OR  skip_isolated
   │
   ├─ SELECT 列表：按 enriched.measures / dimensions / time_dimensions 投影
   │     命名:  revenue:sum      → orders.revenue_sum
   │            *:count          → orders._count
   │            formula + name   → orders.<user_name>          (DEV-1443)
   │            cross + name     → orders.<hops>.<user_name>   (DEV-1448)
   │
   ├─ filter 应用：
   │     简单度量过滤 → HAVING
   │     维度过滤 / 计算型过滤 → inner subquery + outer WHERE
   │
   └─ transform / order / limit
```

## 关键工具与函数

| 名称 | 位置 | 作用 |
| --- | --- | --- |
| `_wrap_cast_for_type` | `generator.py` | 按 `DataType` 套 `CAST`（裸 Column / `TEXT` 跳过） |
| `_build_stat_agg` | `generator.py` | 构造 `stddev_samp/pop`、`var_samp/pop`、`corr/covar_*` 表达式 |
| `_build_percentile` | `generator.py` | 方言感知的 percentile / median 构造 |
| `_build_time_shift_subquery` | `generator.py` | 自连接 CTE（按 INTERVAL 偏移，calendar-based） |
| `_expand_select_star` | `facade/translator.py` | pg-facade 模式下 `SELECT *` 展开为每个非隐藏列 |
| `prequote_reserved_identifiers` | `sql/reserved_keywords.py` | token-quote `__` 段两侧的保留字 |
| `_rewrite_log_aliases` | `generator.py` | `log10(x)` / `log2(x)` 保留原样（不归一为 `LOG(B, x)`） |

## 策略模式：`SqlDialect`

[`slayer/sql/dialects/base.py::SqlDialect`](../../slayer/sql/dialects/base.py) 是抽象基类。**所有方言特定行为都以重写方法的形式存在**，调用点通过 `dialect.build_xxx(...)` 调用。

```python
class SqlDialect:
    dialect_name: ClassVar[str] = "base"
    # 日期截断
    def build_date_trunc(self, granularity: TimeGranularity, column: exp.Expression) -> exp.Expression: ...
    # 近似 distinct
    def build_approx_count_distinct(self, column: exp.Expression) -> exp.Expression: ...
    # 写出后处理（BigQuery 用于 dotted-alias → triple-underscore 改名）
    def rewrite_emitted_sql(self, sql: str) -> str:
        return sql
    # 结果键解码（BigQuery 反向）
    def decode_result_keys(self, keys: list[str]) -> list[str]:
        return keys
    # ClickHouse RLS 相关子查询的 on-emit 回调
    def on_correlated_emitted(self, ast: exp.Expression) -> Callable | None: ...
```

每个 Tier-1 方言覆盖需要差异化的部分（如 `count_distinct_approx` / `median` / `percentile` 的方言专属实现）。

### 各方言特殊点速查

| 方言 | 关键差异 | 代码 |
| --- | --- | --- |
| SQLite | median/percentile/stddev/var/corr/covar 通过 Python 聚合 UDF；`ln/log10/log2/exp/sqrt/pow/power` 标量 UDF；JSON 抽取强制 `json_extract(...)` 函数形式 | [`dialects/sqlite.py`](../../slayer/sql/dialects/sqlite.py) |
| Postgres | 原生 `PERCENTILE_CONT WITHIN GROUP`、`STDDEV_SAMP/POP`、`VAR_SAMP/POP`、`CORR/COVAR_*` | [`dialects/postgres.py`](../../slayer/sql/dialects/postgres.py) |
| DuckDB | sqlglot 转译为 `QUANTILE_CONT`；其余同 Postgres | [`dialects/duckdb.py`](../../slayer/sql/dialects/duckdb.py) |
| MySQL | median/percentile → `NotImplementedError`；`corr/covar_*` 用方差分解公式 | [`dialects/mysql.py`](../../slayer/sql/dialects/mysql.py) |
| ClickHouse | `quantile(p)(x)` 形式；相关子查询需 ≥ 25.4（RLS 检查时强制） | [`dialects/clickhouse.py`](../../slayer/sql/dialects/clickhouse.py) |
| T-SQL (SQL Server) | `STDEV/STDEVP/VAR/VARP` 别名；median/percentile → `NotImplementedError`；`DATETRUNC` + `DATEADD`；类型别名 `mssql/sqlserver/tsql` | [`dialects/tsql.py`](../../slayer/sql/dialects/tsql.py) |
| Snowflake | `MEDIAN/PERCENTILE_CONT/STDDEV/VAR/CORR/COVAR` 原生；`LOG10` 原生；`LOG2` 回退到 `LOG(2, x)`；key-pair/OAuth/SSO/MFA 认证；连接器在 `engine_factory` 注册 `creator=` 桥 | [`dialects/snowflake.py`](../../slayer/sql/dialects/snowflake.py) |
| BigQuery | dotted-alias `__` 改 `___`（output schema 限制）；缺 FK 元数据，手写 join；`APPROX_COUNT_DISTINCT` 原生 | [`dialects/bigquery.py`](../../slayer/sql/dialects/bigquery.py) |
| Tier-2（Redshift/Trino/Presto/Databricks/Spark/Oracle） | 代码覆盖但无 live 验证 | [`dialects/_tier2.py`](../../slayer/sql/dialects/_tier2.py) |

## 保留字处理（DEV-1686）

`slayer/sql/reserved_keywords.py` 是单点真相：

```python
SLAYER_RESERVED_KEYWORDS: set[str] = {
    "grant", "order", "user", "group", "select", "from", "where",
    "join", "on", "and", "or", "not", "null", ...
}
```

- **类型化**：`date` / `name` / `value` / `count` 等"类型-似"非保留词**故意**不收录，避免给常见列加引号
- **两条加引号路径**：
  1. `install_reserved_keywords()`（在 `slayer/sql/dialects/__init__.py` 末尾调用）把集合并进每个注册方言的 `Generator.RESERVED_KEYWORDS`，使 sqlglot 自身的 `identifier_sql` 给 AST 节点加引号
  2. `prequote_reserved_identifiers(sql, dialect)` 在字符串侧 token-quote 标符（处理 AST 加引号够不到的场景：度量 `filter_sql`、qualifier 形式的 `WHERE`、first/last ranked 子查询、`Column.sql` 预解析）

## 窗口函数检测

[`slayer/sql/window_detect.py`](../../slayer/sql/window_detect.py)：

- `has_window_function(formula_str) -> bool` —— 用 sqlglot parse 后查 `exp.Window`
- 命中 → 抛 `WINDOW_IN_FILTER_ERROR`（DEV-1369：filter、formula、Column.sql 三处都拦）
- 推荐改用 `rank(<measure>) <= N` / `dense_rank` / `percent_rank` / `ntile(n=<N>)`

## `count_distinct_approx`（DEV-1595）

[`SqlDialect.build_approx_count_distinct`](../../slayer/sql/dialects/)：方言感知

| 方言 | 发出 |
| --- | --- |
| DuckDB / Spark / Databricks | `APPROX_COUNT_DISTINCT(x)` |
| ClickHouse | `uniq(x)` |
| BigQuery / Snowflake / T-SQL / Oracle | `APPROX_COUNT_DISTINCT(x)` |
| Trino / Presto | `APPROX_DISTINCT(x)` |
| Redshift | `APPROXIMATE COUNT(DISTINCT x)` |
| Postgres / SQLite / MySQL | **回退**到 `COUNT(DISTINCT x)`（这些方言无原生近似） |

别名 `approx_count_distinct` / `countdistinctapprox` 同样识别。

## BigQuery dotted-alias mangling（DEV-1565 补充）

BigQuery 的 output schema 必须是 `[A-Za-z_][A-Za-z0-9_]*`（无点号）。SLayer 的统一 `<model>.<column>` 别名（`orders._count`、`orders.products.category`）会破例，因此：

- **写入侧**：[`BigqueryDialect.rewrite_emitted_sql`](../../slayer/sql/dialects/bigquery.py) 把 SQL 文本中 `<backticked ident containing only word chars and dots>` 里的 `.` 替换为 `___`（三下划线，避免与跨模型列拍平的 `__` 冲突）
- **读出侧**：`BigqueryDialect.decode_result_keys` 把 `orders___revenue_sum` 解回 `orders.revenue_sum`
- **约束**：`Column.sql` 中的 BigQuery FQ 表路径必须每段独立 backtick —— `` `project`.`dataset`.`table` ``；单段 `` `my_dataset.my_table` `` 会被误改

## SQLite 亲和性类型拓宽（DEV-1538）

[`slayer/sql/sqlite_introspect.py::probe_sqlite_integer_column`](../../slayer/sql/sqlite_introspect.py)：
- SQLite 的声明类型是 affinity hint，不是约束
- 每列做 `LIMIT 100_001` 采样：`typeof()` / `ROUND(col) <> col` / BLOB 计数 / Python `float()` coerce
- 仅在确证为整数证据时返回 `INT`；否则 `DOUBLE`（REAL/非整/可强转）或 `TEXT`（BLOB/不可 coerce/饱和）

接入点：
1. `ingest_datasource` / `introspect_table_to_model` 首次落盘
2. `_additive_merge_existing` 幂等 re-ingest 时拓宽
3. `refine_dict_with_live_schema` 让 DEV-1361 的 DOUBLE→INT 缩窄在 SQLite 上也可靠

## `SessionPolicy` 接入 SQL

详见 [`slayer/sql/session_policy.py`](../../slayer/sql/session_policy.py)。关键点：

- 应用时机：在 `SQLGenerator.generate(...)` 之后、`_execute_pipeline` 之前
- 实现：纯 sqlglot transform（不解析字符串）
- `From` 子句处理：`FROM t` → `FROM (SELECT * FROM t WHERE col = val) AS t`（保留 alias，谓词 unqualified，值 `exp.convert` 防注入）
- Scope-aware：CTE / derived 跳过；同名的物理表在 CTE body 内仍被重写；chained CTE 不会失败
- 非 SELECT 根 → 失败闭
- Join rule：相关 `EXISTS` 半连接，构造在 `_build_exists`，每跳别名 `_rls_j{i}`，基源 `_rls_src`
- 组合语义：被 join rule 命中的表**只**受 join rule 约束，column rule 跳过它
- 表身份匹配：bare 目标 → any-schema by name；qualified → schema/catalog 必须匹配；case-insensitive
- ClickHouse 特殊：相关子查询需 server ≥ 25.4，由 `_preflight_clickhouse_correlated` 探活 + 缓存
- 强制 backstop：任何含 `JoinFilterRule` 的策略必须同时含 `on_unapplicable="block"` 的 `ColumnFilterRule`

## `sql_predicate.py` 中的 SQL → DSL 转换

[`slayer/sql/sql_predicate.py::parse_sql_predicate`](../../slayer/sql/sql_predicate.py)：
- Mode A 输入（`Column.filter` 表达式、`ModelJoin.on_sql`、`SlayerModel.filters`）用 sqlglot parse
- 把结构化 AST 转回 Mode B 字符串喂给 `_validate_sql_predicate` / 后续的过滤器富化
- 保留字预引号在 parse 之前应用（保证 sqlglot 能识别）

## 示例

### 例 1：纯度量查询

输入 query：
```json
{
  "source_model": "orders",
  "measures": ["*:count", "revenue:sum"],
  "dimensions": ["status"]
}
```

生成的 SQL（Postgres，简化为说明）：
```sql
SELECT
  "orders"."status" AS "orders.status",
  COUNT(*) AS "orders._count",
  SUM("orders"."revenue") AS "orders.revenue_sum"
FROM "public"."orders" AS "orders"
GROUP BY "orders"."status"
```

### 例 2：跨模型度量 + 变换

输入：
```json
{
  "source_model": "orders",
  "measures": [
    "customers.score:avg",
    {"formula": "change_pct(revenue:sum)", "name": "mom_growth"}
  ],
  "time_dimensions": [{"dimension": "created_at", "granularity": "month"}]
}
```

SQL 形态（双层 CTE：跨模型度量走 inner CTE，时间变化在 outer）：
```sql
WITH
  cm_customers AS (
    SELECT
      DATE_TRUNC('month', "orders"."created_at") AS "orders.created_at_month",
      AVG("orders__customers"."score") AS "_cm_customers__orders__customers_score_avg"
    FROM "public"."orders" AS "orders"
    LEFT JOIN "public"."customers" AS "orders__customers" ON ...
    GROUP BY 1
  )
SELECT
  "orders.created_at_month",
  "_cm_customers__orders__customers_score_avg" AS "orders.customers.score_avg",
  (SUM(...) - LAG(SUM(...)) OVER (...)) / NULLIF(LAG(SUM(...)) OVER (...), 0) AS "orders.mom_growth"
FROM cm_customers ...
```

> 实际 SQL 由 SQLGenerator 在 `EnrichedQuery` 上构建，并通过 `SlayerQueryEngine._execute_pipeline` 执行；以上为简化示例。

## 下一步

- 看引擎与缓存协同：[07 - 引擎内部](07-Engine-Internals.md)
- 看 BI facade 如何复用：[08 - BI 接入层](08-BI-Facades.md)
