# 10 - AI 记忆与搜索

SLayer 给 Agent 提供两类互补能力：

- **Memory** —— 自由文本 + 规范化实体标签 的可检索条目（"过去学到的"）
- **Search** —— 单次调用三通道（BM25 + Tantivy + 嵌入）RRF 融合的统一搜索

两者在 `slayer/memories/` / `slayer/search/` / `slayer/embeddings/` 实现。

## Memory 数据模型

[`slayer/memories/models.py`](../../slayer/memories/models.py)：

```python
class Memory(BaseModel):
    version: int = 1
    id: str = ""                                # DEV-1428: 非空 string
    learning: str                               # 主要文本
    description: str | None = None              # DEV-1549: ≤500 字符紧凑预览
    entities: list[str] = []                    # 规范实体标签
    query: SlayerQuery | None = None            # 可选 example query
    created_at: datetime
    updated_at: datetime | None = None
    meta: dict[str, Any] = {}
```

### ID 规则（DEV-1428）

- 非空字符串
- 自动分配：纯数字（无前导零）从 `"1"`, `"2"`, ... 顺序
- 用户可指定（如 `"kb.policy.42"`），与自动分配共享命名空间 → 重复 id 视为 upsert
- 禁止字符集：`:`, `/`, `?`, `#`, `\`, 空白, ASCII 控制
- bare 名绝不解析为 memory（必须 `memory:<id>` 前缀）

### 实体规范形式

`<ds>` / `<ds>.<model>` / `<ds>.<model>.<leaf>` —— 或 DEV-1428 的 `memory:<id>`（跨 memory 引用）。

聚合后缀剥离：`revenue:sum` → `<ds>.<model>.revenue`；`*:count` → 收为源 model；多跳路径只保留 leaf。

解析器：[`slayer/memories/resolver.py::resolve_entity`](../../slayer/memories/resolver.py)。`memory:` 分支在 `_strip_agg_suffix` **之前** 跑。

## MemoryService

[`slayer/memories/service.py`](../../slayer/memories/service.py)（DEV-1357 v2）：

- 写侧入口：`save_memory(learning, linked_entities, id=None, description=None, query=None)` / `forget_memory(id)`
- `linked_entities` 多态：
  - `list[str]` —— 每项 strict 解析
  - `SlayerQuery | dict` —— 走 `extract_entities_from_query`，query 持久化在 memory 上
- 读侧从 `SearchService`（不在 MemoryService 里 —— 关注点分离）

`MemoryService.save_memory` 是**纯写入 + 嵌入刷新触发**，不重算 embedding（hash-skip 在 `SlayerClient`/`SidecarEmbeddingStore` 内部）。

## Search —— 三通道 RRF

[`slayer/search/service.py::SearchService`](../../slayer/search/service.py)：

```python
SearchService.search(
    entities: list[str] | None = None,
    query: str | list[str] | None = None,
    question: str | None = None,
    datasource: str | None = None,
    cypher_filter: str | None = None,
    max_results: int = 10,
    compact: bool = True,
) -> SearchResponse
```

返回 `SearchResponse` 含一个**扁平**的 `results: List[SearchHit]`（DEV-1532），每个 hit 带 `kind`（`memory` / `datasource` / `model` / `column` / `measure` / `aggregation`）。

### 三个 Retriever

| Retriever | 文件 | 通道 | 依赖 |
| --- | --- | --- | --- |
| `BM25Retriever` | `search/retrievers/bm25.py` | 词频倒排 over memory tags（`rank_bm25.BM25Plus`） | `rank-bm25`（核心依赖） |
| `TantivyRetriever` | `search/retrievers/tantivy.py` | 全文本 over memories ∪ 非隐藏 entities（`en_stem` analyzer） | `tantivy`（核心依赖） |
| `EmbeddingRetriever` | `search/retrievers/embeddings.py` | 密集 cosine over 持久化 embedding 侧车 | `litellm` + `numpy`（可选 extra） |

调用方可在构造时通过 `retrievers=` 注入自定义 list（默认 `[BM25, Tantivy, Embedding]`）。

### 融合（Reciprocal Rank Fusion）

[`slayer/search/rrf.py`](../../slayer/search/rrf.py)：

```python
score(item) = Σ_i (1 / (k + rank_i(item)))   # k = 60
```

三个通道都返回完整 per-kind ranking（不被 `max_results` 截断），融合后取 `max_results`。

### Ranking 稳定性（DEV-1414）

- 每个 retriever 产生完整 per-kind ranking，不被任何 candidate-pool budget 截断
- 子集相对顺序稳定
- 只改 `max_results` 不会重排已有项，也不会让项出现 / 消失，除非跨过 cap 边界

### 空输入回退

- 三个 channel 都没命中 → 返回最新的 memory 限到 `max_results` + 一条 warning
- Embedding 通道在 extra 缺失 / 无 key / 空语料时贡献为空 + 1 条 warning 进 `SearchResponse.warnings`

### 写侧（`upsert_memory` / `refresh_model_subtree` / `refresh_datasource`）

fan-out 到所有 retriever；单 retriever 异常被隔离为 prefixed warning，所以 fan-out 总能跑到最后一个 retriever。Warning aggregation 按声明顺序，不按 gather 完成顺序。

### Post-fusion column hit refresh（DEV-1516 / DEV-1615）

`SearchService` 构造时可选 `engine: Optional[SlayerQueryEngine] = None`：

- MCP `create_mcp_server` + REST `create_app` 都 wire 它
- post-fusion hook：走 column hits，按 `(data_source, model_name)` 分组，组内串行，组间 `asyncio.gather` 并行
- 每列通过 `slayer.engine.profiling.ensure_column_sample_fresh` 补 sample
- `engine=None` 时静默 no-op（存储-only 测试上下文仍能工作）

## Cypher filter（DEV-1464 / DEV-1532）

可选 `cypher_filter` 字符串：

- **Advanced search 装好时**（含 `ladybug`）：运行 openCypher `MATCH … RETURN … AS id` against ephemeral in-memory LadybugDB property graph
- **未装时**：仅支持 `MATCH (n:Label1:Label2) RETURN n.id AS id` kind-filter（naive fallback）
- 命中 ID 成为 hard allowlist 跨三个 channel

Graph 节点：

- `Memory (id=memory:<id>, learning)`
- `Datasource (id, name)`
- `Model (id=<ds>.<model>, name, description)`
- `ModelColumn (id=<ds>.<model>.<col>, name, data_type, description)`
- `Measure (id, name, description)`
- `Aggregation (id, name)`

关系：`MENTIONS` / `CONTAINS` / `JOINS`。Hidden models/columns 排除。

Cache：per storage path，asyncio double-checked locking。Graph 在 `storage.graph_fingerprint()` 变化时重建。

## Indexed text 渲染

[`slayer/search/render.py`](../../slayer/search/render.py)：

- Hidden models/columns 排除
- `meta` 不被索引
- 命名 children（columns/measures/aggregations/join targets）按 `name + kind` 索引（每个 child 自己的 indexed doc）
- `compact=True` 时输出 `description`；`compact=False` 输出 `text`（DEV-1549）

## Embedding 子系统

[`slayer/embeddings/`](../../slayer/embeddings/)：

### `client.py` —— `EmbeddingClient`

- 通过 `litellm.completion` 调嵌入 provider
- 模型名 from `SLAYER_EMBEDDING_MODEL`（默认 `openai/text-embedding-3-small`）
- 批处理 + 错误捕获

### `models.py` —— `Embedding` Pydantic 模型

```python
class Embedding(BaseModel):
    version: int = 1
    canonical_id: str          # "memory:42" / "jaffle_shop.orders.revenue"
    embedding_model_name: str
    vector: list[float]
    content_hash: str          # sha256 of rendered text
    entity_kind: Literal["memory", "datasource", "model", "column", "measure", "aggregation"]
    updated_at: datetime
```

主键：`(canonical_id, embedding_model_name)`。换 `SLAYER_EMBEDDING_MODEL` 后旧行保留但不命中。

### `ranker.py` —— 实际 cosine similarity 排序

### `storage/sidecar_embedding_store.py` —— `SidecarEmbeddingStore`

- 单一接口；YAML 后端走 `<base>/embeddings.db`，SQLite 后端走主 db
- `SidecarEmbeddingsMixin` 混入两个后端
- 升级：旧 `embeddings.yaml` 在首次 open 时改名为 `.legacy`

## Embedding 刷新

触发点：

- `slayer ingest` 每次
- `edit_model`
- `save_memory`
- `--ingest-on-startup`（DEV-1416 同步触发，per-DS 重嵌该 DS 下所有 memory）

每条流水线：

1. 计算 content hash
2. 与持久化 hash 比较，相同 → skip
3. 不同 → 批量 litellm 调用 + 批量写回
4. per-entity 失败不致命
5. per-memory 失败 → `IngestionError(model_name="memory:<id>", ...)` 进 `IdempotentIngestResult.errors`

## Cascade-on-delete（DEV-1428）

写在 `StorageBackend.strip_dangling_entities_from_memories`：

- 触发点：`delete_model` / `delete_datasource` / `forget_memory` / `edit_model_remove`
- 匹配策略：
  - `<ds>.<model>[.<leaf>]` —— 精确 OR dotted-path 后代（`mydb.orders` 同时 strip `mydb.orders.amount`，但不动 `mydb.orders_archive`）
  - `memory:<id>` —— 仅精确匹配
- 写入绕过 `MemoryService.save_memory`（直接 `_save_memory_row`），避免重算 hash
- 记忆全空后保留（learning 文本独立存在）
- ingest-time cleanup：`slayer ingest` / `--ingest-on-startup` 重走每条 memory 引用，strip "definitively not found" 的；transient 失败保留引用
- stale `Memory.query` 上的 example-queries 记忆 → `IngestionError(model_name="memory:<id>")` 抛出，**不**重写

## Sample-value 缓存

详见 [07 - 引擎内部 / `profiling.py`](07-Engine-Internals.md#profilinpy--sample-value-cache)。

要点：

- `Column.sampled` (text) / `sampled_values` (top-50 list) / `distinct_count` (≤50 真实)
- 类别列按 count desc + 字典序 tie-break 排
- 嵌入文本 = learning only（不含 tags）—— cascade-strip 不触发重嵌
- categorical 缓存有效要求 `sampled_values is not None`

## Inspect（DEV-1588 / DEV-1549）

[`slayer/inspect/service.py::InspectService`](../../slayer/inspect/service.py) 是单实体 point-lookup（**不**带 ranking / fusion / 记忆 bundling）：

```python
inspect(reference, entity_type, *, compact=True, format="markdown", ...)
```

`entity_type ∈ {datasource, model, column, measure, aggregation, memory}` 必填（消歧 3-part canonical collision）。

`compact=True`（默认）：

- leaf（column/measure/aggregation）/ datasource / memory：description-only
- **model**：cheapest schema skeleton —— 列/度量/聚合的**名字** + join targets，**零 DB 调用**（`render_model_skeleton`）

`compact=False`：

- leaf / memory：description + 详情
- **datasource**：per-model skeleton
- **model**：full table + samples

### Batch（DEV-1612）

`reference` 接受 `str | list[str] | None`：

- `str` —— 保持单 id 输出字节相同
- `list[str]` —— 同 kind batch，markdown 用 `## <canonical>` 头分隔，error block 头用 input ref
- `None` / 缺省 —— `entity_type ∈ {model, datasource}` 时为 collection view（DEV-1667）

### Compact 三层（按 verbosity 升序）

1. `models_summary(compact=True)` —— 列**计数**
2. `inspect(model, compact=True)` —— 列**名字**
3. `inspect(model, compact=False)` —— full table + samples

### JSON 形 compact

`compact=True` JSON 删除 `text` key。`compact=False` 保留。

## 下一步

- 看 CLI / REST 怎么暴露这套：[09 - 对外接口](09-Interfaces.md)
- 看怎么把现成 dbt / OSI 项目接进来：[11 - 导入器](11-Importers.md)
