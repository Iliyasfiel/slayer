# 01 - 项目概览

## 一句话定位

**SLayer = 轻量级、Agent 优先的语义层**。在数据库之上架一层"度量 + 维度 + 过滤"的领域模型，接收结构化查询描述，生成并执行目标方言的 SQL，跨 Postgres / MySQL / ClickHouse / SQLite / DuckDB / SQL Server / BigQuery / Snowflake 等 8+ 数据库返回统一形状的结果。

源码中的开篇说明（`slayer/__init__.py`）：

> *"SLayer — a lightweight semantic layer for AI agents."*

## 核心能力矩阵

| 能力 | 说明 | 入口 |
| --- | --- | --- |
| **结构化查询** | 用 `measures` / `dimensions` / `time_dimensions` / `filters` 描述意图，引擎生成 SQL | `SlayerQuery`（`slayer/core/query.py`） |
| **查询时聚合** | 同一个 `Column` 在不同查询里可选 `:sum` / `:avg` / `:count` / `:count_distinct` / `:weighted_avg(...)` 等 | `BUILTIN_AGGREGATIONS`（`slayer/core/enums.py`） |
| **可组合时间变换** | `cumsum` / `change` / `change_pct` / `time_shift` / `lag` / `lead` / `rank` / `dense_rank` / `ntile` 等窗口函数 | `ALL_TRANSFORMS`（`slayer/core/formula.py`） |
| **跨模型度量** | 写法 `customers.revenue:sum`、`customers.regions.name` 多跳；引擎按 join graph 自动解 | `engine/enrichment.py`、`engine/join_graph.py` |
| **多阶段 DAG 查询** | 列表形式的 `query=[...]`，可引用彼此 stage 名称，引擎用 Kahn 算法自动拓扑排序 | `_topologically_order_queries`（`slayer/engine/query_engine.py`） |
| **运行时编辑模型** | 通过 CLI / REST / MCP 直接增删改 model、column、measure、join，无 build 步骤 | `engine.edit_model_*`（`slayer/engine/query_engine.py`） |
| **自动建模（Auto-Ingestion）** | 接 DB 反射 schema、识别外键、生成 models 与 LEFT JOIN | `ingest_datasource_idempotent`（`slayer/engine/ingestion.py`） |
| **Schema drift 检测** | 每次执行落库失败时归因到具体列/表漂移，可读也可一键清理 | `validate_models` / `apply_drift_deletes`（`slayer/engine/schema_drift.py`） |
| **AI 记忆 + 语义搜索** | 三通道（BM25 / Tantivy / Embeddings）RRF 融合的统一 `search` 工具 | `slayer/search/service.py` |
| **行级安全（RLS）** | `SessionPolicy` 在引擎层透明加 `WHERE` 谓词（列规则 / 跨表 join 规则） | `slayer/core/policy.py` + `slayer/sql/session_policy.py` |
| **查询缓存** | 可选 in-memory 缓存，TTL + Cube 风格 refresh_keys | `slayer/engine/cache.py` |
| **多方言 SQL 编译** | 策略模式：每种方言一个 `SqlDialect` 子类 | `slayer/sql/dialects/` |
| **多接口** | CLI / REST / MCP（stdio + SSE）/ Python client / Flight SQL（JDBC 兼容）/ Postgres wire protocol | `slayer/{cli,api,mcp,client,flight,pg_facade}/` |
| **可插拔存储** | `YAMLStorage`（默认，git-friendly）/ `SQLiteStorage`（embedded） | `slayer/storage/` |
| **导入器** | `slayer import-dbt`（dbt Semantic Layer YAML）、`slayer import-osi`（Open Semantic Interchange） | `slayer/dbt/`、`slayer/osi/` |

## 典型使用场景

1. **Agent self-serve analytics**：Agent 通过 MCP 调用 `models_summary` → `inspect_model` → `query`，无需写 SQL。
2. **嵌入应用分析**：应用内嵌 `SlayerClient`，把"想看什么"翻译成 SQL，自带行级安全。
3. **BI 工具接入**：把 BI 工具的 PostgreSQL 连接器指向 `slayer pg-serve`，把 JDBC 客户端指向 `slayer flight-serve`，即可看到所有模型当作"表"。
4. **ETL 后的语义层**：把现有 dbt Semantic Layer 项目 `import-dbt` 进来，或把 OSI 配置 `import-osi` 进来。

## 技术栈一览

| 类别 | 选型 | 用途 |
| --- | --- | --- |
| 语言 | Python ≥ 3.11 | 全栈 |
| 数据模型 | Pydantic v2 | 全部领域模型 + 校验 + 迁移 hook |
| SQL AST | sqlglot ≥ 30 | 解析 + 方言 + AST 级 SQL 构造 |
| DB 驱动 | SQLAlchemy 2.0 + 各类方言驱动 | 跨数据库访问 |
| 异步 I/O | asyncio + `asyncpg` / `aiomysql` + `asyncio.to_thread` | 异步优先的引擎 |
| HTTP | FastAPI + uvicorn | REST API |
| 协议 | mcp（Model Context Protocol） | MCP stdio / SSE 传输 |
| 序列化 | pyarrow | Flight SQL / pg-facade rows |
| 本地搜索 | tantivy ≥ 0.26 + rank-bm25 | 全文 + 词频倒排 |
| 嵌入 | litellm（多 provider）+ numpy | 语义搜索（可选） |
| 图查询 | ladybug（LadybugDB，原 KuzuDB） | `cypher_filter` 可选后端 |
| 测试 | pytest ≥ 9 + pytest-asyncio + pytest-postgresql + testcontainers | 单元 + 集成 + 端到端 |

依赖矩阵见 [pyproject.toml](../../pyproject.toml)；Poetry extras 支持按需安装方言驱动、Flight、嵌入等。

## 与同类项目的差异

> *"When AI agents write raw SQL, they can get joins wrong, and produce metrics that drift between queries. Existing semantic layers (Cube, dbt semantic layer) were built for dashboards — heavy infrastructure, slow model refresh cycles, and not enough flexibility for ad-hoc agent queries."*
> — [docs/index.md](https://docs.motley.ai/slayer)

SLayer 的几个差异化设计点：

1. **运行时编辑模型**：不要求 build / refresh，没有 DAG 调度器。
2. **查询时聚合**：度量定义只是公式表达式，聚合在查询时决定 —— 同一个 `revenue` 列可以同时用 `:sum` / `:avg` / `:median`。
3. **不可知数据库**：一份模型可同时跨 Postgres / ClickHouse / SQLite 运行（前提是数据库都存在同结构表）。
4. **多 transport**：同一个后端能服务 MCP / REST / Flight / pg-wire 客户端。

## 不在范围

- **不内置物化 / 缓存层**（`query_cache` 是内存级可选，roadmap 中的"caching / pre-aggregations"未实现）。
- **不内置访问控制台**（`Access controls & governance` 在 roadmap 标注未实现）。
- **不支持 unpivot / asof join**（roadmap 第 8、9 项未实现）。
