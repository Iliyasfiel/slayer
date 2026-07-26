# 06 - 存储后端

## 抽象与工厂

[`slayer/storage/base.py`](../../slayer/storage/base.py) 定义了所有后端的协议：

- `class StorageBackend(ABC)` —— 异步协议（`async def`）
- `def resolve_storage(path: str) -> StorageBackend` —— 工厂：按后缀或路径判断用哪个后端
  - 路径以 `.db` / `.sqlite` / `.sqlite3` 结尾 → `SQLiteStorage`
  - 否则 → `YAMLStorage`

默认存储路径 [`default_storage_path`](../../slayer/storage/base.py)：

1. `$SLAYER_STORAGE` 环境变量
2. `$SLAYER_MODELS_DIR`（legacy）
3. 平台默认：
   - Linux: `$XDG_DATA_HOME/slayer`（默认 `~/.local/share/slayer`）
   - macOS: `~/Library/Application Support/slayer`（忽略 `XDG_DATA_HOME`）
   - Windows: `%LOCALAPPDATA%/slayer`

## 两种内置后端

| 特性 | `YAMLStorage` | `SQLiteStorage` |
| --- | --- | --- |
| 适用 | git 友好、单文件、CI 友好 | embedded、跨进程一致性、嵌入 app |
| 模型布局 | `<base>/models/<data_source>/<name>.yaml` | SQLite 单库 `models` 表 |
| 数据源布局 | `<base>/datasources/<name>.yaml` | 同 |
| Memory 布局 | `<base>/memories/<id>.md` | `memories` 表 |
| Embeddings | `<base>/embeddings.db` 侧车 | 主库内表 |
| 文件锁 | POSIX `fcntl`（Windows 退化无锁） | SQLite 内置 |
| 原子写 | 临时文件 + `os.replace` | 事务 |
| 迁移 | YAML 旧版 v1 flat layout 移入子目录 | `migrate_sqlite_schema` 升级 v3 单 PK → v4 复合 PK |

## 关键方法（节选自 `StorageBackend`）

```python
class StorageBackend(ABC):
    # model CRUD
    async def get_model(self, name: str, data_source: str | None = None) -> SlayerModel: ...
    async def save_model(self, model: SlayerModel) -> None: ...
    async def delete_model(self, name: str, data_source: str | None = None) -> None: ...
    async def list_models(self, data_source: str | None = None) -> list[SlayerModel]: ...
    # datasource
    async def list_datasources(self) -> list[DatasourceConfig]: ...
    async def get_datasource(self, name: str) -> DatasourceConfig: ...
    async def save_datasource(self, ds: DatasourceConfig) -> None: ...
    # memory
    async def save_memory(self, memory: Memory) -> Memory: ...
    async def get_memory(self, id: str) -> Memory: ...
    async def delete_memory(self, id: str) -> None: ...
    async def list_memories(self, datasource: str | None = None) -> list[Memory]: ...
    # settings
    async def get_datasource_priority(self) -> list[str]: ...
    async def set_datasource_priority(self, priority: list[str]) -> None: ...
    # sample
    async def update_column_sampled(self, *, data_source, model_name, column_name, sampled, sampled_values, distinct_count): ...
    # cascade
    async def strip_dangling_entities_from_memories(self, *, predicate): ...
    # fingerprint（搜索缓存失效）
    def graph_fingerprint(self) -> str: ...
```

> **保存链路**：`StorageBackend.save_model` 是 template method —— 调 `_validate_cycle` 校验派生列无环，然后委托 `_save_model_impl`。迁移写回在 `_migrate_and_refine_on_load` 传 `_validate=False` 保留旧循环数据。

## 持久化模型

存储层只看到 Pydantic dict 形态。三个核心类型都带 `version: int`：

- `SlayerModel.version: int = 7`（当前）
- `SlayerQuery.version: int = 3`
- `DatasourceConfig.version: int = 1`

写入永远 emit 当前版本；读取时通过 `@model_validator(mode="before")` 钩子调 `slayer/storage/migrations.py::migrate` 链式升级。

## 迁移链（`migrations.py`）

`migrate(obj, kind)` 调度到各版本的 `vN_migration.py`：

```
v1 ──► v2 ──► v3 ──► v4 ──► v5 ──► v6 ──► v7
```

| 版本 | 关键变更 | 文件 |
| --- | --- | --- |
| v1 → v2 | 合并 dimensions/measures 到 columns；measures 改存 ModelMeasure 公式 | `v2_migration.py` |
| v2 → v3 | 删除 SlayerQuery.dry_run/explain（改 engine kwargs） | `v3_migration.py` |
| v3 → v4 | 表底 model 强制非空 data_source；模型文件 layout 改 `<ds>/<name>.yaml`；SQLite 复合 PK | `v4_migration.py` |
| v4 → v5 | 旧 `string/number/time/...` → 新 `TEXT/DOUBLE/TIMESTAMP/...`；丢弃伪类型 | `v5_migration.py` |
| v5 → v6 | 新增 `Column.sampled: Optional[str]` | `v6_migration.py` |
| v6 → v7 | 新增 `Column.sampled_values: Optional[List[str]]` 和 `Column.distinct_count: Optional[int]` | `v7_migration.py` |

数据源层还有两个独立迁移：

- `v2_datasource_migration.py` —— datasource 形态独立升级
- `v2_memory_migration.py` —— memory PK 从 int 升为 text

## Type Refinement（`type_refinement.py`）

每次 load model 时按目标数据库 live schema 二次校验 `DataType`：

- `refine_dict_with_live_schema(...)` —— 把 `DOUBLE → INT` 缩窄到 live integer 列
- `has_refineable_columns(...)` / `has_sqlite_widenable_columns(...)` —— CLI 入口 `slayer storage migrate-types` 复用

> 加上 SQLite 的 `probe_sqlite_integer_column`（亲和性拓宽），DOUBLE/INT 之间的双向校准在所有 Tier-1 方言都可靠。

## Embeddings 侧车（`sidecar_embedding_store.py`）

- `SidecarEmbeddingStore` —— 统一接口，YAML / SQLite 都混入 `SidecarEmbeddingsMixin`
- SQLite 后端：嵌入表与主库同 db
- YAML 后端：`<base>/embeddings.db`（避免 `embeddings.yaml` 整体重写成为 ingest 瓶颈，DEV-1405）
- 升级：旧 `embeddings.yaml` 在首次 open 时改名为 `embeddings.yaml.legacy`
- `SlayerClient` 缓存键：`(canonical_id, embedding_model_name)`；切换 `SLAYER_EMBEDDING_MODEL` 后旧行保留但不命中

## 模型 + Memory + Embedding 布局示意

### YAML 存储

```
<base>/
├── models/
│   ├── jaffle_shop/
│   │   ├── customers.yaml
│   │   └── orders.yaml
│   └── my_postgres/
│       └── orders.yaml
├── datasources/
│   ├── jaffle_shop.yaml
│   └── my_postgres.yaml
├── memories/
│   ├── 1.md
│   ├── 2.md
│   └── kb_policy_42.md
├── priority.yaml
├── embeddings.db
└── (legacy on first open)
    embeddings.yaml.legacy
    counters.yaml.legacy
```

### SQLite 存储

```
<base>.db  (单文件)
├── models           -- (data_source, name) composite PK
├── datasources
├── memories         -- TEXT PK
├── embeddings
└── settings         -- 单例 (datasource_priority 等)
```

## Cascade-on-Delete（DEV-1428）

`StorageBackend.strip_dangling_entities_from_memories(*, predicate)`：

- 触发点：`delete_model` / `delete_datasource` / `forget_memory` / `edit_model_remove`
- 匹配策略：
  - `<ds>.<model>[.<leaf>]` —— 精确 OR 严格 dotted-path 后代（`mydb.orders` 同时 strip `mydb.orders.amount`，但不动 `mydb.orders_archive`）
  - `memory:<id>` —— 仅精确匹配（`memory:42` 不会 strip `memory:421`）
- 写入路径：`_save_memory_row`（绕过 `MemoryService.save_memory`，避免重新算嵌入 hash）
- 记忆全空后保留（learning 文本独立存在）
- 嵌入文本是 learning only（不含 tags），所以 cascade-strip 不会触发嵌入重算

## Graph Fingerprint

`StorageBackend.graph_fingerprint() -> str`：返回存储路径上的文件 mtime（或 SQLite schema hash）。Search 缓存用它在 graph 变化时重建。

## 下一步

- 看引擎如何使用存储后端做 ingest、validate、edit：[07 - 引擎内部](07-Engine-Internals.md)
- 看 `slayer/cli.py` 的 `slayer datasources/models/memories/...` 子命令族：[09 - 对外接口](09-Interfaces.md)
