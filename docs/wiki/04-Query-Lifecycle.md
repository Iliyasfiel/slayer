# 04 - 查询生命周期

本章跟踪一个 `SlayerQuery` 从进入引擎到拿到 `SlayerResponse` 的每一步。代码路径以 `slayer/engine/query_engine.py::SlayerQueryEngine.execute` 为主线。

## 1. 输入归一化（`_normalize_input`）

引擎接受四种输入形态（DEV-1437）：

| 形态 | 含义 | 处理 |
| --- | --- | --- |
| `SlayerQuery` | 强类型查询对象 | 直接进入富化 |
| `dict` | JSON 形态的查询 | `SlayerQuery.model_validate(...)` 后进入 |
| `list[SlayerQuery \| dict]` | 多阶段 DAG | 递归归一化 → `_topologically_order_queries` |
| `str` | run-by-name | 通过 `storage.get_model(name, data_source=...)` 取回查询链模型 |

返回归一化后的内部结构 `_NormalizedInput` / `_NormalizedByName`，再由 `_normalize_by_name` 进一步处理。

## 2. 多阶段 DAG 拓扑排序

当输入是 `list[...]` 时调用 [`_topologically_order_queries`](../../slayer/engine/query_engine.py)：

- 自实现的 **Kahn 算法**（不依赖 `graphlib`）
- 引用提取：从 `source_model`（字符串兄弟引用）、`ModelExtension.source_name`、`joins[].target_model` 抽取依赖
- 校验：缺失 name / 重名 / 自指 / 根被引用 / 环
- 排序后保持 **最后一段作为 entry point / DAG 根** —— 这一点很关键：返回的结果不依赖于用户提交时的顺序
- 不可达的工具子查询：被静默 drop（不出现在最终 SQL 里，因为没人 resolve 它们）

```python
# 简化形态
def _topologically_order_queries(queries: list[SlayerQuery]) -> list[SlayerQuery]:
    # 1. 提取依赖边
    # 2. 验证 (name 唯一, 自指, 根被引用, 环)
    # 3. Kahn 排序
    # 4. 保留最后一段固定为 root
```

> **存储 vs 运行时的语义差异**：存储在 `SlayerModel.source_queries` 中的查询保持 strict-order 语义，靠 `_scope_named_queries_to_prior` + `_forbidden_sibling_refs_var` ContextVar 在 `_resolve_model_inner`/`_resolve_join_target` 中报严格顺序错。运行时 list 走拓扑排序，更宽容。

## 3. 模型解析

`_validate_and_populate_cache` 阶段把所有引用（`source_model="orders"`、跨模型 `customers.revenue:sum`、JOIN target）解析为具体的 `SlayerModel` 实例。

- 数据源路由：[`AmbiguousModelError`](../../slayer/core/errors.py) —— 多个 DS 共享模型名
- 优先级：hint > datasource_priority > 唯一匹配
- 跨模型 join 解析：根据 `ModelJoin` 声明与 join graph 走 `engine.join_graph.JoinGraph` 自动连线

## 4. 富化（`enrich_query`）

[`slayer/engine/enrichment.py::enrich_query`](../../slayer/engine/enrichment.py) 是查询编译的真正核心。它接收 `SlayerQuery + 已解析的 models` 并产出 [`EnrichedQuery`](../../slayer/engine/enriched.py)。

主要工作：

1. **维度解析**：
   - `ColumnRef`（含 `customers.regions.name` 多跳）→ 跳数路径 + leaf
   - 时间维度：`TimeDimension` 按 `TimeGranularity` 切分
2. **度量解析**：
   - 字符串 `"revenue:sum"` 提升为 `{"formula": "revenue:sum"}` 形态
   - 公式字符串走 `parse_formula`（Python AST）→ `FieldSpec` 树
   - 跨模型 `customers.revenue:sum` → `CrossModelMeasure`（DEV-1448 之后支持 user `name` 重命名）
3. **Transform 检查**：
   - `cumsum / change / time_shift / lag / lead` 等需要时间维度；若 query 没有，从 model `default_time_dimension` 取，再没有则报错
   - `change / change_pct` 在富化期**反糖**成 `hidden time_shift + 算术`
   - `rank / dense_rank / percent_rank` 是"无时间"窗口 transform；`partition_by=` 决定是否按维度分区
4. **过滤器富化**：
   - `parse_filter` 把 DSL 字符串解析为 AST
   - 标量白名单（`SCALAR_PASSTHROUGH`）过滤非法函数调用
   - `Window_function_in_filter_error` 拦截 `OVER(...)`
   - 计算型过滤（`change(revenue:sum) > 0`）抽出为 hidden field，在外层 WHERE 应用
   - `resolve_filter_columns` 严格解析 bare 名（DEV-1369）
5. **Distinct-dimension dedup**（DEV-1543）：
   - 没有 measures 但至少一个 dim/td → 自动 `GROUP BY` 出现在最外层
   - 用 `SlayerQuery.distinct_dimension_values=False` 关掉
6. **引用 dedup**：transform-wrapped inner ref 合并（DEV-1446）

输出：[`EnrichedQuery`](../../slayer/engine/enriched.py) 携带：
- `measures: list[EnrichedMeasure]`
- `cross_model_measures: list[CrossModelMeasure]`
- `dimensions / time_dimensions` 完整 SQL 表达式
- `filters` 解析后的 AST
- 所有 SQL 表达式都是裸的 sqlglot `exp.Expression` —— 至此任何模型 lookup 都已完成

## 5. SQL 生成

[`SQLGenerator.generate(enriched_query)`](../../slayer/sql/generator.py) 把 `EnrichedQuery` 翻译成最终 SQL。

### 5.1 策略模式委托

```python
class SqlDialect:  # slayer/sql/dialects/base.py
    dialect_name: ClassVar[str]
    def build_date_trunc(self, granularity, column): ...
    def build_approx_count_distinct(self, column): ...
    def rewrite_emitted_sql(self, sql: str) -> str: ...   # 默认 identity
    def decode_result_keys(self, keys: list[str]) -> list[str]: ...  # 默认 identity
    def on_correlated_emitted(self, ast): ...  # 默认 None
    # ...
```

每个 Tier-1 方言一个文件（`sqlite.py` / `postgres.py` / `mysql.py` / `clickhouse.py` / `duckdb.py` / `tsql.py` / `snowflake.py` / `bigquery.py`），Tier-2 合在 `_tier2.py`（Redshift / Trino / Presto / Databricks / Spark / Oracle）。

注册表见 [`slayer/sql/dialects/__init__.py`](../../slayer/sql/dialects/__init__.py)：
- `get_dialect(sqlglot_name)` —— 严格，失败抛 `KeyError`
- `dialect_for_ds_type(ds_type)` —— 宽容，未知时回退到 `PostgresDialect`

### 5.2 生成流程（`SQLGenerator._generate_base`）

1. **基表子查询**：从 `EnrichedQuery.source_model` 的物理表 / `sql` / `source_queries` 起步
2. **JOIN 子句**：按需附加 ModelJoin（多跳 diamond 用 `__` 路径别名消歧：`customers__regions` vs `warehouses__regions`）
3. **聚合 / GROUP BY**：
   - `has_aggregation` 或 `cross_model_measures` 或 `skip_isolated` → 强制 GROUP BY
   - dim-only dedup 触发也并入
4. **SELECT 列表**：
   - 度量别名：`revenue:sum` → `orders.revenue_sum`，`*:count` → `orders._count`（保留前导下划线避免与列名 `count` 冲突）
   - 用户显式 `name` 字段覆盖（DEV-1443 / DEV-1448）
5. **Filter 应用**：
   - 简单度量过滤 → `HAVING`
   - 维度过滤 / 计算型过滤（hidden field）→ 子查询 + 外层 `WHERE`
6. **Transform 应用**：
   - `cumsum` / `lag` / `lead` → 窗口函数
   - `time_shift` → 自连接 CTE（按 INTERVAL 偏移）
7. **CAST 包装**（DEV-1361）：
   - 非裸 `Column.sql`（函数调用 / 算术 / CASE WHEN）按 `Column.type` / `ModelMeasure.type` 套 `CAST(... AS <DataType>)`
   - 裸标识符与 `sql=None` 不动
   - `TEXT` 跳过（cosmetic 且不影响 SQLite JSON 抽取）
8. **保留字引用加引号**（DEV-1686）：
   - `SLAYER_RESERVED_KEYWORDS` 单点定义
   - `install_reserved_keywords()` 在 `dialects/__init__.py` 末尾执行，把词集合并进每个 sqlglot dialect 的 `RESERVED_KEYWORDS`
   - 字符串侧 `_parse` / `_parse_predicate` 用 `prequote_reserved_identifiers` token-quote 标符
9. **`SELECT *` 兼容**（pg-facade 模式）：见 [08 - BI 接入层 / `SELECT *` browse-mode](08-BI-Facades.md#select--browse-mode-扩展)

### 5.3 输出形态

`SQLGenerator.generate(...)` 返回：
- `final_sql: str` —— 拼好的最终 SQL
- `select_aliases: dict[str, str]` —— 别名 → 完整投影名（如 `orders.revenue_sum`）
- `attributes: ResponseAttributes` —— dimension/measure 元数据（label, format）

## 6. RLS 应用（`apply_session_policy`）

如果构造 engine 时传了 `SessionPolicy`：

1. 解析 `final_sql` AST
2. 用 `sqlglot.optimizer.scope.traverse_scope` 遍历，**只重写物理表引用**（CTE/derived 跳过；同名的物理表在 CTE body 内仍会被重写）
3. 每个物理表 → `FROM (SELECT * FROM t WHERE col = val) AS t`（保留 alias）
4. Join rule：相关 `EXISTS` 半连接（cardinality-safe），由 `slayer/sql/session_policy.py::_build_exists` 构造
5. 值通过 `exp.convert` 注入 → SQL 注入安全

## 7. 缓存查询

[`slayer/engine/cache.py::QueryCache`](../../slayer/engine/cache.py)（DEV-1587，可选）：

- `execute(query, cache=True)` 启用
- 缓存键 = `sha256(final_sql + "|" + connection_string + "|" + runtime_fingerprint)`
- 变量已替换进 SQL（不同变量 → 不同键）
- `dry_run` / `explain` 忽略缓存
- 失效：
  - **TTL**：懒加载；超期算 miss 重跑
  - **Refresh keys**：Cube 风格 `(physical_table, select_expression)`，`engine.refresh()` 主动扫描
- API：仅 Python，REST/MCP/CLI/Flight/pg-facade/`SlayerClient` 都**不**暴露

## 8. 异步执行（`SlayerSQLClient.execute`）

[`slayer/sql/client.py::SlayerSQLClient`](../../slayer/sql/client.py)：

- 缓存键：`(connection_string, runtime_fingerprint(datasource))`
- 按方言选择驱动：
  - Postgres → `asyncpg`（连接池按实例缓存）
  - MySQL → `aiomysql`
  - SQLite / DuckDB / ClickHouse → `asyncio.to_thread`
- 同进程 `:memory:` SQLite → `StaticPool` 跨 await 共享
- 返回 `Row` 列表，转换为 `pandas.DataFrame`（如调用方传入 `query_df`）

## 9. 结果包装（`SlayerResponse`）

```python
class SlayerResponse(BaseModel):
    data: list[dict[str, Any]]        # 行级结果
    sql: str | None = None            # 实际执行的 SQL（dry_run=False 时填）
    attributes: ResponseAttributes    # {dimensions, measures} → FieldMetadata(label, format)
    row_count: int
    query_runtime_ms: float
    query_warnings: list[str] = []
```

`ResponseAttributes` 把投影拆分维度与度量，键是别名（`orders.revenue_sum`），值是 `FieldMetadata(label, format)`。

## 10. Schema drift 归因（执行后兜底）

如果 DB 报错：

- `engine._attribute_drift_if_any` 调 `validate_models` 检查被触碰的 models
- 命中漂移 → 抛 `SchemaDriftError(models, to_delete, original)`
- REST 翻译为 HTTP 422 + `{"error": "schema_drift", ...}`

健康查询零开销。

## 同步入口（`execute_sync`）

[`slayer.engine.query_engine.SlayerQueryEngine.execute_sync`](../../slayer/engine/query_engine.py) 是 CLI / 笔记本 / 脚本的便捷入口：

- 内部走 `run_sync` 桥到 `execute(...)`
- `finally` 中调 `aclose()` 释放连接（修了 STORYLINE-BG-API-37 `TooManyConnectionsError`）
- 同时支持 `cache` 与 `data_source` 关键字

## 端到端伪代码

```python
# 服务端
async def handle_query(client_input):
    engine = SlayerQueryEngine(storage, policy=policy, cache_config=CacheConfig(ttl_seconds=300))
    try:
        response = await engine.execute(
            client_input,                  # SlayerQuery | dict | list | str
            variables={...},               # for {var}
            data_source="my_postgres",     # optional hint
            cache=True,                    # optional
        )
        return response.data, response.attributes
    except SchemaDriftError as e:
        return JSONResponse(status_code=422, content={"error": "schema_drift", **e.details()})
    finally:
        await engine.aclose()
```

## 关键文件清单

| 文件 | 角色 |
| --- | --- |
| `slayer/engine/query_engine.py` | 引擎主体、`SlayerQueryEngine` 类 |
| `slayer/engine/enrichment.py` | `enrich_query` |
| `slayer/engine/enriched.py` | `EnrichedQuery` / `EnrichedMeasure` / `CrossModelMeasure` |
| `slayer/engine/cache.py` | `QueryCache` / `CacheConfig` / `RefreshResult` |
| `slayer/engine/join_graph.py` | 跨模型 join 图 + 最短路径 |
| `slayer/engine/schema_drift.py` | `validate_models` / `apply_drift_deletes` |
| `slayer/engine/profiling.py` | 列 sample 缓存与刷新 |
| `slayer/sql/generator.py` | `SQLGenerator` |
| `slayer/sql/client.py` | `SlayerSQLClient` |
| `slayer/sql/session_policy.py` | `apply_session_policy` |
| `slayer/sql/dialects/*` | 14 个方言策略类 |
| `slayer/sql/reserved_keywords.py` | `SLAYER_RESERVED_KEYWORDS` |
| `slayer/sql/sql_predicate.py` | `parse_sql_predicate`（Mode A） |
| `slayer/sql/window_detect.py` | `has_window_function` + 错误信息 |
