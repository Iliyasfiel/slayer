# 02 - 架构总览

## 分层视图

SLayer 的代码组织遵循一个清晰的"中心 + 周边"结构：**`slayer/core` 是领域模型与解析器中心**，`slayer/engine` 把它编排成可执行的查询流水线，`slayer/sql` 把查询转译为方言 SQL，`slayer/storage` 负责持久化，外围的 `cli` / `api` / `mcp` / `flight` / `pg_facade` / `client` 是同一份核心的不同"出入口"。

```
┌──────────────────────────────────────────────────────────────────────────┐
│                            对外接口 (Surface)                            │
│  CLI  ·  REST API  ·  MCP  ·  Flight SQL  ·  Postgres Wire  ·  Python    │
│   slayer/cli.py      slayer/api/      slayer/mcp/                          │
│   slayer/client/     slayer/flight/   slayer/pg_facade/                    │
└──────────────────────────────────────────────────────────────────────────┘
                                  ▲
                                  │  all surfaces build a
                                  │  SlayerQueryEngine (or SlayerClient)
                                  ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                            服务编排 (Service)                            │
│  slayer/inspect/  ·  slayer/memories/  ·  slayer/search/                  │
│  (render & point-lookup)  (CRUD for Memory)  (3-channel RRF search)      │
└──────────────────────────────────────────────────────────────────────────┘
                                  ▲
                                  ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                          引擎 (Engine = orchestrator)                    │
│  slayer/engine/query_engine.py  —  SlayerQueryEngine.execute()            │
│  ├─ enrich_query  →  EnrichedQuery                                        │
│  ├─ SQLGenerator.generate  →  final SQL                                   │
│  ├─ apply_session_policy  →  RLS wrapping                                 │
│  ├─ execute via SlayerSQLClient (or cache hit)                            │
│  └─ ingest / edit / schema-drift / profiling / cache / join_graph         │
└──────────────────────────────────────────────────────────────────────────┘
            ▲                       ▲                          ▲
            │                       │                          │
┌──────────────────┐  ┌────────────────────────┐  ┌──────────────────────┐
│  领域模型 (core) │  │  SQL 生成 (sql)         │  │  持久化 (storage)     │
│  enums            │  │  generator.py           │  │  StorageBackend      │
│  errors           │  │  dialects/  策略模式     │  │  ├─ YAMLStorage      │
│  models  ←─ 重点  │  │  client.py  异步驱动     │  │  └─ SQLiteStorage    │
│  query            │  │  engine_factory         │  │  + migrations v1→v7  │
│  formula  Python  │  │  reserved_keywords      │  │  + sidecar embedding │
│          AST 解析  │  │  session_policy  RLS    │  │                      │
│  refs             │  │  sql_predicate  SQL→DSL │  │                      │
│  policy           │  │  window_detect          │  │                      │
│  recommend        │  │                         │  │                      │
│  format           │  │                         │  │                      │
└──────────────────┘  └────────────────────────┘  └──────────────────────┘
            ▲                       ▲                          ▲
            │                       │                          │
┌──────────────────────────────────────────────────────────────────────────┐
│                          外部依赖与扩展                                  │
│  sqlglot  ·  sqlalchemy  ·  asyncpg/aiomysql/pyodbc/snowflake/BQ …        │
│  fastapi  ·  mcp  ·  pyarrow  ·  tantivy  ·  rank-bm25  ·  litellm       │
│  duckdb  ·  jafgen (demo)  ·  dbt-core (import)  ·  kuzu→ladybug (graph)  │
└──────────────────────────────────────────────────────────────────────────┘
```

## 端到端查询生命周期

下面是一个从"MCP 客户端发起 query"到"返回结果"的完整路径：

```
1. 请求进入
   mcp/server.py  /  api/server.py  /  client/slayer_client.py
        │  构造 SlayerQuery（or dict / list / str）
        ▼
2. 引擎入口
   engine/query_engine.py::SlayerQueryEngine.execute(query, ...)
        │  ① _normalize_input         ← 接受四种形态
        │  ② _topologically_order_queries  ← 列表型 DAG 排序
        │  ③ resolve 模型 + columns
        ▼
3. 阶段内处理（per query）
   engine/enrichment.py::enrich_query(query, all_models, ...)
        │  把 Column / ModelMeasure 名称解析成完整的 SQL 表达式
        │  处理 transform、cross-model、refinement、distinct dedup
        ▼
4. SQL 生成
   sql/generator.py::SQLGenerator.generate(enriched)
        │  委托给当前方言 SqlDialect 子类
        │  应用 SessionPolicy（RLS）
        │  应用 reserved-word quoting
        │  产出最终 SQL
        ▼
5. 缓存查询（可选）
   engine/cache.py::QueryCache
        │  hit → 直接返回 cached 响应
        │  miss → 继续
        ▼
6. 异步执行
   sql/client.py::SlayerSQLClient.execute(sql)
        │  SQLAlchemy + 方言驱动（asyncpg / aiomysql / duckdb / sqlite / …）
        ▼
7. 结果包装
   engine/query_engine.py → SlayerResponse
        │  包含 .data  ·  .sql  ·  .attributes
        ▼
8. 返回
   mcp / rest / client / flight / pg-wire
```

关键代码锚点：

- 入口：[`slayer/engine/query_engine.py`](../../slayer/engine/query_engine.py)（`SlayerQueryEngine.execute` / `execute_sync`）
- 富化：[`slayer/engine/enrichment.py`](../../slayer/engine/enrichment.py)（`enrich_query`）
- 富化结果：[`slayer/engine/enriched.py`](../../slayer/engine/enriched.py)（`EnrichedQuery` / `EnrichedMeasure` / `CrossModelMeasure`）
- 拓扑排序：[`slayer/engine/query_engine.py::_topologically_order_queries`](../../slayer/engine/query_engine.py)
- SQL 生成：[`slayer/sql/generator.py`](../../slayer/sql/generator.py)
- 异步执行：[`slayer/sql/client.py`](../../slayer/sql/client.py)

## 模块依赖关系

| 上游模块 | 依赖的下游模块 |
| --- | --- |
| `slayer.cli` | `engine`, `storage`, `search`, `memories`, `inspect`, `ingest_report` |
| `slayer.api.server` | `core`, `engine`, `mcp.server`（复用 help_seed 等）、`search`, `memories`, `storage` |
| `slayer.mcp.server` | `core`, `engine`, `inspect`（从 `slayer.inspect` 重导出 helpers），`memories`, `search`, `storage` |
| `slayer.flight` | `slayer.facade`（共享 translator、catalog），`core`, `engine`, `sql`, `storage` |
| `slayer.pg_facade` | `slayer.facade`（共享），`core`, `engine`, `sql`, `storage` |
| `slayer.client` | `core`, `engine`, `memories`（typed response） |
| `slayer.engine.query_engine` | `core`, `engine.*`（cache, join_graph, enrichment, enriched, introspect_utils, profiling, timing），`sql.*`, `memories.resolver`, `storage.base` |
| `slayer.engine.ingestion` | `core`, `engine.introspect_utils`, `memories.resolver`（用于 cascade strip），`storage.base` |
| `slayer.search.service` | `core`, `engine.profiling`（post-fusion column refresh），`memories.resolver`, `storage.base` |
| `slayer.facade.translator` | `core`, `engine`（column_expansion 通过 `_root_scope_column_ids`） |
| `slayer.dbt.converter` | `core`, `engine.ingestion`（`introspect_table_to_model`），`slayer.ingest_report` |
| `slayer.osi.converter` | 同上 + `slayer.engine.join_graph`（metric 锚点选择） |

> **循环依赖规避**：`slayer.engine.introspect_utils` 是 dependency-free 的叶子模块，承载 `_safe_get_columns`；`ingestion` 与 `schema_drift` 都从这里 import，避免 `ingestion ↔ query_engine` 的循环。

## 异步模型

> SLayer 是 **async-first** 的。所有 `StorageBackend` 方法都是 `async def`；`SlayerQueryEngine.execute()` 是 `async`；SQL 客户端按方言选择原生异步驱动或回退到 `asyncio.to_thread`。

| 类型 | 行为 |
| --- | --- |
| **Postgres** | 原生异步 `asyncpg`（连接池按 `SlayerSQLClient` 实例缓存） |
| **MySQL** | 原生异步 `aiomysql` |
| **SQLite / DuckDB / ClickHouse** | 回退到 `asyncio.to_thread` |
| **同进程 `:memory:` SQLite** | `SlayerSQLClient` 自带 `StaticPool` 跨 await 共享 |
| **生命周期** | `SlayerQueryEngine.aclose()` / `SlayerSQLClient.aclose()` 释放资源；`execute_sync` 在 `finally` 中调用 `aclose()` 防连接泄漏 |
| **同步桥** | `slayer/async_utils.py::run_sync` 在无 loop / Jupyter 已有 loop / 普通脚本三种情况下都安全 |

测试使用 `pytest-asyncio`，`asyncio_mode = "auto"`，函数可直接 `async def`。

## 数据流（写入侧）

```
外部触发                          SLayer 内部                                落盘
─────────                         ──────────                                ────
slayer ingest --ds X       →   engine/ingestion.py
                                     │  introspect_table_to_model
                                     │  ingest_datasource_idempotent
                                     ▼
                                storage.save_model
                                     │  按 v 版本号
                                     │  _migrate_and_refine_on_load
                                     │  save → 走 YAMLStorage / SQLiteStorage
                                     ▼
                                embeddings 侧车 .db
                                     │  按 (canonical_id, embedding_model_name)
                                     │  content_hash skip → litellm 调嵌入
                                     ▼
                                记忆 cascade-strip（按 entities 检查）
```

## 数据流（读取 / 搜索侧）

```
slayer search  ─►  search/service.py::SearchService.search
                       │
                       ├─ BM25Retriever        (rank-bm25 over memory tags)
                       ├─ TantivyRetriever     (in-memory full-text)
                       └─ EmbeddingRetriever   (optional, litellm cosine)
                       │
                       ▼
                  RRF 融合 (k=60) → 单一扁平 SearchResponse.results
                       │
                       ▼
                  (可选) post-fusion column sample refresh
                  (DEV-1615) via engine.profiling.ensure_column_sample_fresh
```

## 默认端口与进程边界

| 端口 | 进程/角色 | 启动 |
| --- | --- | --- |
| `5143` | REST API / MCP-HTTP (SSE) | `slayer serve` |
| `5144` | Arrow Flight SQL | `slayer flight-serve` |
| `5145` | Postgres wire-protocol facade | `slayer pg-serve` |
| — | MCP stdio | `slayer mcp`（被 Claude Code 等 stdio 调用方拉起） |

`loopback-no-token fallback`：非回环地址绑定时，Flight 与 pg-facade 都要求 `--token` 或 `SLAYER_FLIGHT_TOKEN` / `SLAYER_PG_TOKEN`，否则启动失败。

## 代码体量参考

- `slayer/` 包下 Python 源文件约 **80+** 个（不含 `tests/`、`docs/`、`examples/`）。
- 测试文件 **300+** 个（`tests/` + `tests/integration/` + `tests/dialects/` + `tests/flight/` + `tests/facade/` + `tests/pg_facade/` + `tests/perf/`）。
- 集成测试覆盖 8 个 Tier-1 数据库 + 飞行 SQL + pg-facade + 真实 Metabase e2e。

## 下一步

- 想了解 **数据模型**：继续 [03 - 核心数据模型](03-Core-Data-Models.md)。
- 想了解 **执行路径**：继续 [04 - 查询生命周期](04-Query-Lifecycle.md)。
- 想了解 **每个包负责什么**：参考 [09 - 对外接口](09-Interfaces.md) 与 [08 - BI 接入层](08-BI-Facades.md)。
