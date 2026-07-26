# 13 - 术语表

> 来源：项目内 `docs/concepts/terminology.md` + `slayer/core/` 中分散的定义，本表侧重代码层面出现频次高的概念。

## A

- **Additive merge** —— `ingest_datasource_idempotent` 的核心策略：新列/新表/新 join 追加，已有 metadata 不覆盖。详见 [07 - 引擎内部](07-Engine-Internals.md#ingestionpy--auto-ingestion)。
- **AggregatedMeasureRef** —— `parse_formula` 输出的 AST 节点，对应 `revenue:sum` 这种 `column:agg` 形式。
- **Aggregation（查询时）** —— 通过 `:` 语法在 query 时刻选择而非在 model 定义中固化（`revenue:sum` / `*:count` / `price:weighted_avg(weight=quantity)`）。
- **Allowed aggregations** —— `Column.allowed_aggregations` 白名单，PK 列强制只允许 `count` / `count_distinct`。
- **Auto-Ingestion** —— 反射 DB schema + FK 关系 → 生成 `SlayerModel` 的过程（`ingest_datasource_idempotent`）。
- **Async-first** —— 引擎、SQL 客户端、storage 后端都是 async；`execute_sync` 桥接到 sync 上下文。

## B

- **Bare same-model derived ref** —— `Column.sql` 中的裸标识符引用同 model 的另一个派生列（DEV-1410），在 `_expand_derived_refs` 中被带括号内联。
- **Browse mode** —— pg-facade 模式下 `SELECT * FROM <model>` 自动展开为每个非隐藏列。
- **Build step** —— SLayer 不需要；与 dbt Semantic Layer 关键差异。

## C

- **Canonical entity form** —— `<ds>` / `<ds>.<model>` / `<ds>.<model>.<leaf>` / `memory:<id>`，搜索 / 记忆 / ingest 共同使用。
- **Cascade-on-delete** —— DEV-1428：删除 model / datasource / memory 时自动 strip 其它 memory 里的 dangling 引用。
- **Cascading type comparison** —— `validate_models` 的级联：列删除带动派生列 / 度量 / join 全部 drop。
- **Cypher filter** —— `cypher_filter` 入参，可用 openCypher `MATCH ... RETURN ... AS id` 对所有三个搜索 channel 做 hard allowlist（DEV-1464 / DEV-1532）。
- **Chained CTE** —— SQL 中 `WITH x AS (...), y AS (SELECT ... FROM x)` 形式。Session policy 重写时 chained CTE 不失败。

## D

- **DataType** —— `TEXT / INT / DOUBLE / BOOLEAN / DATE / TIMESTAMP`，与 sqlglot 字节对齐。
- **Datasource priority** —— 多个 DS 共享 model 名时的消歧顺序；通过 `set_datasource_priority` / CLI / REST 设置。
- **Datasource-scoped storage** —— 模型按 `(data_source, name)` 复合主键存储（DEV-1330 / v4）。
- **DAG** —— 多阶段 query 中 `source_queries` 或运行时 list 形成的有向无环图，Kahn 算法排序。
- **Dev-* ticket** —— MotleyAI 内部 Linear 工单编号，源码注释中频繁出现。
- **Diamond join** —— `orders → customers → regions` 与 `orders → warehouses → regions` 同表不同路径。SLayer 用 `__` 路径别名消歧。
- **Distinct dimension values** —— DEV-1543：dim-only query 自动 `GROUP BY` 去重，可用 `distinct_dimension_values=False` 关掉。
- **Dim-only query** —— query 没有 measures 但至少一个 dimension / time_dimension。

## E

- **EnrichedQuery** —— `enrich_query` 输出的强类型对象，含完整 sqlglot 表达式，不再查 storage。
- **`extra="forbid"`** —— Pydantic v2 配置：禁止未知字段。`SlayerQuery` / `SessionPolicy` 都用。
- **Extras（Poetry）** —— `motley-slayer[postgres,mysql,...]` 的可选依赖分组。

## F

- **F1/2/... migrations** —— 见 [06 - 存储后端](06-Storage-Backends.md)。
- **Facet (Dev-1486 shared layer)** —— `slayer/facade/` —— Flight SQL 与 pg-facade 共享的翻译/目录层。
- **Fingerprint** —— `StorageBackend.graph_fingerprint() -> str`，搜索图缓存失效用。
- **`first` / `last`** —— `FIRST_VALUE(x ORDER BY ...)` 窗口形式的特殊聚合，分别 ASC / DESC。
- **Forced filter** —— `SessionPolicy` 把物理表重写为带谓词的 subquery 的过程。
- **Frozen + forbid** —— `SessionPolicy` / `ColumnFilterRule` 的 Pydantic 配置：不可变 + 禁未知字段。

## G

- **Graph pre-filter** —— `cypher_filter` 路径（DEV-1464），三个 channel 共同的 hard allowlist。
- **GROUP BY (dim-only)** —— 见 Distinct dimension values。
- **Gunning for two DB round-trips** —— Flight SQL Path A 概念：1 次 `LIMIT 0` 拿 schema + 1 次全量取数。

## H

- **Hidden columns / models** —— 渲染 search 索引 / 列表时排除，但不阻止用户主动 inspect（escape hatch）。
- **`has_column` probe** —— RLS 中通过 `_safe_get_columns` 检测物理表是否含某列（case-insensitive）。
- **Hand-rolled Kahn's algorithm** —— `_topologically_order_queries` 不依赖 `graphlib`。

## I

- **Idempotent ingestion** —— 见 Additive merge。
- **In-memory SQLite** —— `sqlite:///:memory:` 跨 await 共享（每 `SlayerSQLClient` 自带 `StaticPool`）。
- **Inspect (single-entity)** —— DEV-1588 新工具，统一取代 `inspect_model` / 单 `inspect` 调用。`entity_type` 必填消歧 3-part canonical collision。
- **`install_reserved_keywords()`** —— 在 `slayer/sql/dialects/__init__.py` 末尾调一次，把 `SLAYER_RESERVED_KEYWORDS` 并入每个 sqlglot dialect 的 `RESERVED_KEYWORDS`。

## J

- **Jaffle Shop** —— 内置 demo dataset，由 `duckdb + jafgen` 生成。
- **Join graph** —— `slayer/engine/join_graph.py::JoinGraph`，LEFT = directed edge，INNER = undirected via storage symmetry。
- **`json_extract` 函数形式** —— SQLite 中强制使用（不归一为 `->` 算子），避免 JSON-quoted 字符串问题。

## K

- **Kahn's algorithm** —— 见 Hand-rolled Kahn's algorithm。
- **`kind` discriminator** —— `SessionPolicy.data_filters` 的 Pydantic discriminator，区分 `ColumnFilterRule` / `JoinFilterRule`。

## L

- **Lazy sample profiling** —— 不在 ingest 时跑（避免每列全表扫描），首次 inspect 触发，DEV-1516 起 search 也触发。
- **`log10` / `log2` literal preservation** —— 用户写法原样 emit，不归一为 `LOG(B, x)`。
- **Loopback-no-token fallback** —— Flight / pg-facade 在 `127.0.0.1` 上不需要 token；非 loopback 必填。

## M

- **Mandatory block backstop** —— 含 `JoinFilterRule` 的策略必须同时含 `on_unapplicable="block"` 的 `ColumnFilterRule`。
- **Memory id** —— DEV-1428 起为非空 string，禁字符集 `:`/`/`/`?`/`#`/`\`/空白/控制。
- **Mode A (SQL)** —— 见 [03 - 核心数据模型](03-Core-Data-Models.md#引用模式-mode-a--mode-b)。
- **Mode B (DSL)** —— 同上。
- **`meta`** —— 模型/列/度量/join 上的用户自定义 JSON metadata，可经 MCP / REST / CLI 编辑。

## N

- **`*:count`** —— `count(*)` 的冒号语法；emit 时 alias 为 `orders._count`（保留前导下划线避免与列名 `count` 冲突）。
- **Naive kind filter** —— `cypher_filter` 在未装 advanced_search 时仅支持 `MATCH (n:Label) RETURN n.id AS id`（DEV-1532）。

## O

- **Open Semantic Interchange (OSI)** —— 见 [11 - 导入器](11-Importers.md)。
- **`on_unapplicable`** —— `ColumnFilterRule` / `JoinFilterRule` 字段：`block` 强制表必须能被改写 / `pass` 放过未确认存在的表。

## P

- **Pareto frontier** —— `recommend_root_model` 在找不到根时返回的"覆盖率"边界（dominates：strictly-larger reach set 或 equal reach set with no longer and ≥1 shorter path）。
- **`partition_by=`** —— rank-family transform 的可选 kwarg，按维度分区；不传则 rank 跨整个结果集。
- **Path-based table alias** —— `__` 分隔的 join path 别名；`customers__regions.name` vs `warehouses__regions.name` 区分 diamond join。
- **Payload via Starlette/FastAPI** —— MCP-SSE / streamable HTTP 模式。
- **`Power:count` mapping** —— `*` 是 "all rows"，`count` 只是普通聚合。
- **Probe query** —— `SELECT 1` / `SELECT NULL WHERE 1=0` / `version()` / `current_database()` 等白名单。
- **Pydantic before-validator** —— `Column.type` / `ModelMeasure.type` 上的 lenient mapping，把 `"string"` → `"TEXT"`、伪类型 → `None`。

## Q

- **Query cache** —— DEV-1587；仅 Python API；TTL + refresh keys。
- **Query-time aggregation** —— 见 Aggregation（查询时）。

## R

- **Rank family** —— `rank` / `dense_rank` / `percent_rank` / `ntile`，默认无 `PARTITION BY`（跨整个结果集 rank）。
- **RDS / runtime fingerprint** —— `engine_factory._runtime_fingerprint(datasource)` 用于 SQL 客户端缓存键。
- **Reserved keywords** —— `SLAYER_RESERVED_KEYWORDS` 单点定义。
- **`refresh_keys`** —— Cube 风格 `(table, expr)` 配置，TTL 之外的 stale 检测。
- **RLS** —— Row-Level Security，由 `SessionPolicy` 实现。
- **Round-trip (Flight)** —— 见 Gunning for two DB round-trips。
- **RRF** —— Reciprocal Rank Fusion，三个 channel 融合公式 `Σ 1/(k + rank)`，`k=60`。
- **Run-by-name** —— `engine.execute(str, ...)` 取回查询链模型。

## S

- **Sample cache** —— `Column.sampled` / `sampled_values` / `distinct_count`。
- **`SCALAR_PASSTHROUGH`** —— Mode B 标量函数白名单，单点定义。
- **Schema-drift** —— 持久化列/类型/join 与 live schema 的差异；`validate_models` 检测。
- **Schema versioning** —— `SlayerModel.v7` / `SlayerQuery.v3` / `DatasourceConfig.v1`，`@model_validator(mode="before")` 链式迁移。
- **Search (lenient)** —— DEV-1428 起，未解析的 `entities` / `query` token 变 warning 而非抛错。
- **Session policy** —— 见 RLS。
- **Sidecar embedding store** —— `<base>/embeddings.db` (YAML) / 主 db (SQLite)。
- **SLAYER_RESERVED_KEYWORDS** —— 见 Reserved keywords。
- **Source modes** —— `sql_table` / `sql` / `source_queries` 三种 `SlayerModel` 互斥源模式。
- **Stages as DAG** —— 多阶段 `source_queries` 不限于链，可以是任意 DAG。

## T

- **Tier 1 / Tier 2 dialects** —— 1 = 完整集成测试；2 = 仅单元测试覆盖。Tier 1: SQLite / Postgres / DuckDB / MySQL / ClickHouse / SQL Server / BigQuery / Snowflake。Tier 2: Redshift / Trino / Presto / Databricks / Spark / Oracle。
- **TimeShift via self-join CTE** —— 时间偏移走 INTERVAL 偏移的子查询，calendar-based gap-safe。
- **Token-quote** —— 字符串侧给 `__` 段两侧的保留字加引号（与 AST 加引号并行）。
- **Topological order** —— 见 Hand-rolled Kahn's algorithm。
- **Transform** —— `cumsum` / `change` / `change_pct` / `time_shift` / `lag` / `lead` / `rank` / `dense_rank` / `percent_rank` / `ntile` / `first` / `last` / `consecutive_periods`。

## U

- **Unified columns** —— v2 起 `Column` 统一了旧的 `dimensions` + `measures` 列表。
- **Untiered / unbounded cache** —— QueryCache 无 LRU；管理靠 `evict` / `clear_cache`。

## V

- **Validation chain** —— `migrate(obj, kind)` → v1→v2→...→vN 顺序调各 `vN_migration.py`。
- **Versions (model)** —— 见 Schema versioning。

## W

- **WEEK vs WEEK_SUNDAY** —— DEV-1572：Monday-anchored vs Sunday-anchored 周切。
- **Window function in filter** —— DEV-1369 起禁止 filter / formula / Column.sql 中含 `OVER (...)`；用 rank-family transform。
- **`WHERE <agg> <cmp> <literal>`** —— Metabase 风格 HAVING；DEV-1486 shared 翻译器处理为 `SlayerQuery` filter。

## X / Y / Z

- **`{var}` placeholders** —— DSL filter 中的变量占位符，运行时按 `variables=` 替换；未定义抛错。
- **YamlStorage vs SQLiteStorage** —— 见 [06 - 存储后端](06-Storage-Backends.md)。
