# 09 - 对外接口

SLayer 暴露 5 个面向用户的接口。它们都最终构造一个 `SlayerQueryEngine`（或 `SlayerClient`）然后委派给它。

## 1. CLI —— `slayer`

[`slayer/cli.py`](../../slayer/cli.py) 是 Python 入口（`pyproject.toml` 中 `[project.scripts]` 注册）。子命令族：

| 子命令 | 说明 |
| --- | --- |
| `slayer serve [--host H] [--port 5143] [--storage S] [--demo] [--ingest-on-startup] [--mcp-transport stdio\|sse\|streamable-http]` | 启动 REST API + MCP server（默认 MCP stdio） |
| `slayer flight-serve [--host H] [--port 5144] [--token T] [--demo]` | Flight SQL 服务（端口 5144） |
| `slayer pg-serve [--host H] [--port 5145] [--token T] [--demo]` | Postgres wire-protocol 服务（端口 5145） |
| `slayer mcp [--demo] [--ingest-on-startup] [--transport ...]` | 仅 MCP（无 REST） |
| `slayer query <JSON\|@file> [--format json\|markdown\|csv] [--dry-run] [--explain]` | 单条 query |
| `slayer query @file.json` (list) | 多阶段 DAG |
| `slayer query <model_name> --variables k=v` | run-by-name（查询链模型） |
| `slayer ingest --datasource X [--schema S]` | 反射 + 自动建模（幂等） |
| `slayer validate-models [--datasource X] [--force-clean] [--yes]` | 漂移检测 + 可选清理 |
| `slayer datasources {create, list, get, delete, test, priority, create-inline, create demo}` | 数据源 CRUD |
| `slayer models {list, get, create, delete, inspect}` | 模型 CRUD |
| `slayer memory {save, list, get, forget}` | 记忆 CRUD |
| `slayer search [--entity ...] [--query ...] [--question ...] [--datasource DS] [--cypher-filter ...] [--max-results N] [--format json\|text] [--verbose]` | 三通道语义搜索 |
| `slayer search refresh-samples [--data-source X] [--model M ...]` | 重刷列 sample 缓存 |
| `slayer recommend-root-model ITEM... [--data-source X] [--root-hint M] [--format json\|text]` | 跨模型查询的根模型推荐 |
| `slayer import-osi <path> --datasource X [--dialect ANSI_SQL]` | 导入 OSI（YAML/JSON，文件或目录） |
| `slayer import-dbt <path> --datasource X` | 导入 dbt Semantic Layer |
| `slayer storage migrate-types [--data-source X] [--dry-run]` | 显式触发 DOUBLE → INT 缩窄 |
| `slayer inspect <reference> --type <entity_type> [--no-compact] [--format json]` | 单实体 point-lookup |
| `slayer models-summary` (via `inspect(None, 'model', ...)`) | 集合视图 |

所有命令接受 `--storage`（YAML 目录或 SQLite 文件）。默认路径由 `default_storage_path()` 解析（见 [06 - 存储后端](06-Storage-Backends.md)）。

### `--ingest-on-startup` 语义（DEV-1392）

- 触发 `engine/ingestion.py::ingest_all_datasources_idempotent`
- 对每个 DS 同步执行 idempotent ingest
- 单 DS 失败写到 stderr，不阻塞启动
- `to_delete` 漂移条目**只**打印，**不**自动应用 —— 破坏性清理仍需 `slayer validate-models --force-clean`
- 与 `--demo` 自由组合：先建 demo，再扫所有 DS
- 也通过 `SLAYER_INGEST_ON_STARTUP=1` 环境变量触发（flag 优先）
- 也通过 `create_app(ingest_on_startup=True)` / `create_mcp_server(ingest_on_startup=True)` kwarg

### CLI 中的样本缓存

`slayer search refresh-samples` 重新 profile 并 persist `Column.sampled`。Best-effort：单列失败报告但不中断。

## 2. REST API —— `slayer/api/server.py`

[`slayer/api/server.py`](../../slayer/api/server.py) 是 FastAPI 应用。`create_app(...)` 工厂：

```python
def create_app(
    storage: StorageBackend | None = None,
    *,
    policy: SessionPolicy | None = None,
    cache_config: CacheConfig | None = None,
    ingest_on_startup: bool = False,
    ...
) -> FastAPI
```

### 关键端点

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| `POST` | `/query` | 单查询或多阶段（`{"queries": [...]}`） |
| `POST` | `/sql` | 给出 DSL，等价 `/query` 但只返回 SQL |
| `POST` | `/explain` | 解释执行计划 |
| `GET`  | `/models` | 列出模型摘要 |
| `GET`  | `/models/{data_source}/{name}` | 取单模型 |
| `POST` | `/models` | 创建/更新模型 |
| `DELETE` | `/models/{data_source}/{name}` | 删除 |
| `POST` | `/models/{data_source}/{name}/inspect` | inspect（compact/verbose） |
| `GET`  | `/datasources` | 列出 |
| `GET`  | `/datasources/{name}` | 取（credentials masked） |
| `POST` | `/datasources` | 创建 |
| `DELETE` | `/datasources` | 删除 |
| `PUT`  | `/datasources/priority` | 设置 priority |
| `POST` | `/ingest` | 触发 ingest |
| `POST` | `/validate-models` | 漂移检测 |
| `POST` | `/inspect` | 单实体 point-lookup |
| `POST` | `/memories` | 保存 |
| `GET`  | `/memories` | 列出 |
| `GET`  | `/memories/{id}` | 取单 |
| `DELETE` | `/memories/{id}` | 删除 |
| `POST` | `/search` | 三通道搜索 |
| `POST` | `/recommend-root-model` | 根模型推荐 |
| `GET/POST/DELETE` | `/mcp/sse` | MCP-SSE（与 MCP HTTP 共享） |

### 错误约定

- `ValueError` → HTTP 400
- `SlayerError` → HTTP 422（部分场景）
- `SchemaDriftError` → HTTP 422 + `{"error": "schema_drift", "models": [...], "to_delete": [...], "original": "..."}`
- `EntityResolutionError` → 400/404
- `ForcedFilterError`（RLS）→ 400/403

### `QueryRequest` / `QueryListRequest`

- `QueryRequest` 允许 `extra="allow"`（向前兼容）
- `source_model` 接受 `str | dict`（dict 形态是 inline `ModelExtension` / `SlayerModel`）
- `measures` / `dimensions` 接受 `str | dict` 简写
- `QueryListRequest` 强制 `extra="forbid"`，含 `queries: list[dict]` + 顶层 `variables / dry_run / explain`

## 3. MCP —— `slayer/mcp/server.py`

[`slayer/mcp/server.py`](../../slayer/mcp/server.py) 暴露 MCP 工具。两个 transport：

- **stdio**（默认）：通过 `claude mcp add slayer -- slayer mcp` 拉起
- **SSE/HTTP**（与 REST 共享端口 5143）：`claude mcp add slayer-remote --transport sse --url http://localhost:5143/mcp/sse`

### 主要工具

| 工具 | 用途 |
| --- | --- |
| `query(measure/formula, dimensions, filters, order, limit, format, ...)` | 单阶段查询 |
| `query_nested(queries, ...)` | 多阶段 DAG |
| `models_summary(data_source=None, compact=True)` | 集合视图 |
| `inspect_model(name, data_source=None, compact=True, ...)` | **DEPRECATED**（DEV-1588）—— 转发到 `inspect` |
| `inspect(reference, entity_type, compact=True, ...)` | 单实体 point-lookup（DEV-1588） |
| `list_datasources()` | 数据源列表 |
| `get_datasource(name)` | 取（凭证脱敏） |
| `create_datasource(name, type, host, ...)` | 创建 |
| `update_datasource(name, ...)` | 更新 |
| `delete_datasource(name)` | 删除 |
| `set_datasource_priority(priority)` | 设置 priority |
| `list_models(data_source=None)` | 模型列表 |
| `get_model(name, data_source=None)` | 取单 |
| `create_model(model)` | 创建 |
| `update_model(name, model)` | 更新 |
| `delete_model(name, data_source=None)` | 删除 |
| `edit_model(name, data_source=None, op=...)` | 部分编辑 |
| `ingest_datasource_models(datasource, schema_name=None)` | 自动建模 |
| `validate_models(data_source=None)` | 漂移检测（只读） |
| `save_memory(learning, linked_entities, id=None, description=None, query=None)` | 保存记忆 |
| `forget_memory(id)` | 删记忆 |
| `search(entities, query, question, datasource, cypher_filter, max_results, compact=True)` | 搜索 |
| `recommend_root_model(items, data_source, root_hint)` | 根模型推荐 |

### 错误风格

MCP 把内部异常包装为 tool-error：包含原始 message + remediation hint（`AmbiguousModelError` 提示用 `data_source=` 或 `set_datasource_priority`）。

## 4. Python 客户端 —— `slayer/client/slayer_client.py`

[`SlayerClient`](../../slayer/client/slayer_client.py) 异步优先，支持两种模式：

### 远程模式

```python
from slayer.client.slayer_client import SlayerClient
client = SlayerClient(url="http://localhost:5143")
df = client.query_df(query)        # 同步
df = await client.query(query)     # 异步
```

### 本地模式

```python
from slayer.client.slayer_client import SlayerClient
from slayer.storage.yaml_storage import YAMLStorage

client = SlayerClient(storage=YAMLStorage(base_dir="./my_models"))
```

### 镜像引擎的输入联合（DEV-1437）

每个查询入口（`query / query_sync / sql / sql_sync / explain / explain_sync / query_df`）接受 `SlayerQuery | dict | list | str`。

### 其他公开方法

- `client.list_models(...)`, `client.get_model(...)`, `client.create_model(...)`, `client.update_model(...)`, `client.delete_model(...)`
- `client.save_memory(...)`, `client.forget_memory(...)`, `client.list_memories(...)`
- `client.search(...)` → `SearchResponse`
- `client.inspect(...)` / `inspect_sync(...)`
- `client.recommend_root_model(...)` / `_sync(...)`

## 5. 启动矩阵

| 模式 | 启动命令 | 入口 |
| --- | --- | --- |
| 仅 CLI | `slayer ...` | `slayer.cli.main` |
| REST API（同时带 MCP stdio） | `slayer serve` | `slayer.api.server.create_app` |
| REST API + MCP-SSE | `slayer serve --mcp-transport sse` | 同上 |
| 仅 MCP stdio | `slayer mcp` | `slayer.mcp.server.create_mcp_server` |
| 仅 MCP streamable-http | `slayer mcp --transport streamable-http` | 同上 |
| Flight SQL | `slayer flight-serve` | `slayer.flight.cli.run_flight_serve` |
| Postgres 协议 | `slayer pg-serve` | `slayer.pg_facade.cli` |
| 嵌入 Python | `from slayer.client.slayer_client import SlayerClient` | — |

## 下一步

- 真实运行/部署步骤：[12 - 运行与部署](12-Running-and-Deployment.md)
- 让 dbt / OSI 项目接入：[11 - 导入器](11-Importers.md)
