# 07 - 引擎内部

[`slayer/engine/`](../../slayer/engine/) 是 SLayer 的"业务编排层"：它把 `core/` 的领域模型、`sql/` 的 SQL 生成、`storage/` 的持久化粘合成可执行的工作流。本章介绍主要子模块。

## `query_engine.py` —— `SlayerQueryEngine`

[`SlayerQueryEngine`](../../slayer/engine/query_engine.py) 是核心类。

### 构造

```python
SlayerQueryEngine(
    storage: StorageBackend,
    *,
    policy: SessionPolicy | None = None,
    cache_config: CacheConfig | None = None,
)
```

- `policy` —— 不可变 RLS 策略（DEV-1578/1627）
- `cache_config` —— TTL + refresh keys（DEV-1587）。**setter 会清空缓存**。

### 入口

| 方法 | 说明 |
| --- | --- |
| `async def execute(query, *, variables=None, data_source=None, cache=True, dry_run=False, explain=False)` | async 主入口，接受 `SlayerQuery \| dict \| list \| str` |
| `def execute_sync(query, *, variables=None, data_source=None, cache=True, dry_run=False, explain=False)` | 同步桥接（CLI / 笔记本 / 脚本），`finally` 中 `aclose()` 释放连接 |
| `def recommend_root_model(items, *, data_source=None, root_hint=None) -> RootModelRecommendation` | 同步版 + `_sync` 后缀 |
| `def save_model(model)` / `delete_model` / `edit_model_*` | 运行时编辑入口，写回存储后端 |
| `async def apply_drift_deletes(deletes)` | 一键清漂移条目（CLI 触发） |
| `async def refresh() -> RefreshResult` | refresh-key 风格缓存失效 |
| `def clear_cache()` / `cache_size` | 缓存管理 |
| `async def aclose()` | 关闭所有缓存的 SQL 客户端 |

### 内部流水线（`execute` → `_execute_pipeline`）

```python
async def _execute_pipeline(self, normalized, *, dry_run, explain, cache, variables, data_source):
    prepared = self._prepare_pipeline(normalized, variables=variables, data_source=data_source)
    sql = prepared.final_sql
    if dry_run:    return SlayerResponse(sql=sql, data=[], ...)
    if explain:    return SlayerResponse(sql=_explain_sql(sql, ...), data=[], ...)
    if cache:
        cached = self._cache.get(sql_key)
        if cached: return deep_copy(cached)
    rows = await self._sql_client.execute(sql)
    return SlayerResponse(data=rows, sql=sql, attributes=prepared.attributes, ...)
```

### `_prepare_pipeline`

无 DB 副作用：resolve → enrich → generate → attributes → 缓存键计算。共享给 `execute` / `evict`（只算 key）/ `refresh` re-exec。

## `cache.py` —— QueryCache

[`slayer/engine/cache.py`](../../slayer/engine/cache.py)（DEV-1587，**仅 Python 内部 API**）：

### 数据类

```python
class CacheConfig(BaseModel):
    ttl_seconds: int = 0          # 0 = 不缓存
    refresh_keys: list[tuple[str, str]] = []   # (table, expr)

class _CacheEntry(BaseModel):
    sql: str
    connection_key: str
    response: SlayerResponse
    created_at: float             # time.monotonic
    applicable_refresh_keys: list[tuple[str, str]]
    refresh_key_baselines: dict[tuple[str, str], Any]
```

### 关键行为

- 缓存键 = `sha256(final_sql + "|" + connection_string + "|" + runtime_fingerprint)`
- 变量已替换进 SQL（不同变量 → 不同键）
- **TTL**：懒加载；`age > ttl_seconds` → miss；可注入 `clock`（默认 `time.monotonic`）便于测试
- **Refresh keys**（Cube 风格）：
  - 用户表达式**不**自动包 `MAX(...)`（`COUNT(*)` 形式可捕获删除）
  - 配置项 table 不指定的字段是通配（`"orders"` 匹配 `orders` / `public.orders` / `db.public.orders`）
  - 解析 final SQL 减掉 CTE/derived 别名后取剩余 `exp.Table` 与配置求交
- **`refresh()`** 流程：
  1. snapshot 所有 entry
  2. 按 `(datasource, table)` 一次批扫
  3. 每个 entry：TTL 超期 → re-exec；applicable 失败 → 保持原样；任何 applicable 值变化 → re-exec；否则 unchanged
  4. re-exec 重新走 `_prepare_pipeline`、重置 `created_at`、identity-guarded commit（避免并发 clobber）
- **写时合同**：miss 时 baseline 扫描失败 → 向上传播；`refresh()` 时失败 → 保持旧 entry 继续 on-failure
- **并发**：`asyncio.Lock` 只保护 dict ops，DB await 都在锁外
- **无 LRU**：管理靠 `evict` / `clear_cache`

### 不暴露

REST / MCP / CLI / Flight / pg-facade / `SlayerClient` 都不暴露此缓存。

## `ingestion.py` —— Auto-Ingestion

[`slayer/engine/ingestion.py`](../../slayer/engine/ingestion.py)：

### `introspect_table_to_model(table_name, ds, schema_name=None) -> SlayerModel`

- 通过 SQLAlchemy `Inspector` 反射列类型 + PK
- 检测 FK，按 FK 走 BFS 构建 LEFT JOIN 元数据
- SQLite 额外走 [`sqlite_introspect.probe_sqlite_integer_column`](../../slayer/sql/sqlite_introspect.py) 验证 INT
- 输出 v7 `SlayerModel`

### `ingest_datasource(ds_name, schema_name=None) -> list[SlayerModel]`

- 列出所有表
- 对每张表调 `introspect_table_to_model`
- 返回 list

### `ingest_datasource_idempotent(ds_name, schema_name=None) -> IdempotentIngestResult`

- 关键属性：**幂等**（DEV-1356）
- 与已有 models 做 additive merge：
  - 新列 / 新表 / 新 join → 追加
  - 已有列的 `description / label / format / meta / allowed_aggregations` → **永不覆盖**
- 跳过 `sql`-mode 与查询链 model
- 返回 `(additions, to_delete, errors)`：`to_delete` 是 `validate_models` 的输出，类型漂移也走同一次调用

### `ingest_all_datasources_idempotent()` —— `slayer --ingest-on-startup` 的核心

- 对 `storage.list_datasources()` 中每个 DS 跑上面那个
- 继续 on failure：单 DS 错误写到 stderr，不阻止服务启动
- 每个 DS 的 ingest 顺带**重新嵌入**该 DS 下所有 memory（DEV-1416）

### FK 限制

- ClickHouse / BigQuery 都不通过 `Inspector` 暴露 FK 元数据 → 必须手写 join
- Snowflake 的 declarative FK 不强制但 Inspector 能读 → 自动发现 join 像 Postgres/MySQL

## `schema_drift.py` —— Drift Detection

[`slayer/engine/schema_drift.py`](../../slayer/engine/schema_drift.py)（DEV-1356）：

### `validate_models(data_source=None) -> list[EditModelDelete | WholeModelDelete]`

- 只读（不写盘）
- 比对持久化列 / 类型 / join 与 live introspection
- 类型比较用**粗桶**（`number` / `string` / `boolean` / `temporal`）：INTEGER↔FLOAT 与 DATE↔TIMESTAMP 折叠
- PK drop 不级联
- cascade walking 限定在父 DS 内
- 接入 MCP `validate_models` / REST `POST /validate-models` / CLI `slayer validate-models`

### `SchemaDriftError`

- 触发点：`engine.execute` 抛 DBAPI 错时调用 `validate_models` 归因
- 健康查询零开销
- REST 翻译为 HTTP 422 + `{"error": "schema_drift", "models": [...], "to_delete": [...], "original": "..."}`

### `apply_drift_deletes(deletes) -> ApplyDriftResult`

- 按 entry 通过 `edit_model_remove` / `delete_model_by_name` 实施
- per-entry 失败捕获，继续
- **只**通过 `slayer validate-models --force-clean [--yes]` 触发 —— 不暴露给 MCP/REST

## `profiling.py` —— Sample-Value Cache

[`slayer/engine/profiling.py`](../../slayer/engine/profiling.py)：

### `ensure_column_sample_fresh(model, column, engine, storage)`

- DEV-1516 起为类别列 + 数值/时间列都做缓存补全（DEV-1615）
- 来源：
  - `inspect_model` 类别循环
  - `SearchService` post-fusion column-hit hook
  - `slayer search refresh-samples` 主动刷
  - `edit_model` 改 column → 该列；改 `filters/sql/source_queries` → 全列
  - `inspect_model` 缓存未命中（best-effort write-back）
  - `search()` 内任意 stale column hit（DEV-1516）
- 共享 helper —— **single source of truth**

### `refresh_table_backed_model_sampled` / `refresh_all_table_backed_sampled`

- 整模型或全 DS 刷新
- 失败 best-effort，不中断整体

## `join_graph.py` —— JoinGraph

[`slayer/engine/join_graph.py`](../../slayer/engine/join_graph.py)：

```python
class JoinGraph:
    def reachable_from(self, root: str) -> set[str]: ...
    def shortest_path(self, src: str, dst: str) -> list[str] | None: ...
```

- LEFT = directed edge，INNER = 通过存储对称的无向边
- 用于：
  - `recommend_root_model`（DEV-1626）：最小跳数根 + 字典序 tiebreak
  - OSI converter：metric 锚点选择（`min_hops_root`）
  - schema_drift 的级联扩展

## `enrichment.py` & `enriched.py`

`enrichment.py` 中的 `enrich_query` 是查询编译的中心算子。详见 [04 - 查询生命周期](04-Query-Lifecycle.md)。

`enriched.py` 定义输出模型：

- `EnrichedQuery` —— 携带完整 SQL 表达式
- `EnrichedMeasure` —— 单模型聚合或算术
- `CrossModelMeasure` —— 跨模型聚合（含 user `name` 重命名支持，DEV-1448）
- `public_projection_aliases(enriched) -> dict[str, str]` —— 投影别名到完整名（`orders.customers.region`）

## `column_expansion.py` & `column_dependency.py`

- `column_expansion` —— 把 `Column.sql` 中的 bare 派生引用（指向同模型其它 derived column）内联为带括号的子表达式（DEV-1410）
- `column_dependency` —— 解析 derived column 的依赖图，cycle 检测（`ColumnCycleError`）

## `introspect_utils.py`

- dependency-free 的叶子模块
- `_safe_get_columns(ds, table) -> list[(name, type)] | None` —— SQLAlchemy `Inspector` 容错版本
- `_FLOAT_LIKE_INFO_SCHEMA_TYPES` —— 用于类型粗桶
- `ingestion` / `schema_drift` 都从这里 import，避免 `ingestion ↔ query_engine` 循环

## `timing.py`

- 简单的计时 / trace 工具
- 集成到 `engine.execute` 用于 `query_runtime_ms`

## `cache.py` 的 refresh-key 匹配规则

`exp.Table` 归一化为 `(catalog, db, name)` 三元组（dialect folding，quoted 保留），与配置的 `(table, expr)` 求交：

- 配置 `"orders"` → 匹配 `orders` / `public.orders` / `db.public.orders`
- 配置 `"public.orders"` → 只匹配 `db = public`

写入侧 SQL: `SELECT (<expr0>) AS "slayer_rk_0", ... FROM <table>` —— 一次批扫一张表。

## 引擎与 Python 客户端

[`slayer/client/slayer_client.py::SlayerClient`](../../slayer/client/slayer_client.py) 是统一入口（DEV-1437）：

- 远程模式：`url=...`（HTTP）
- 本地模式：`storage=...`（绕过 HTTP）
- 所有查询方法（`query / query_sync / sql / sql_sync / explain / explain_sync / query_df`）接受同样的 `SlayerQuery | dict | list | str` 输入
- `policy=` 仅本地模式支持；远程模式需要在服务端配置

## 下一步

- 看 SQL 端：`slayer/sql/` 各文件 → [05 - SQL 生成与方言](05-SQL-Generation-and-Dialects.md)
- 看 AI 相关：`slayer/memories/` / `slayer/search/` / `slayer/embeddings/` → [10 - AI 记忆与搜索](10-AI-Memory-and-Search.md)
