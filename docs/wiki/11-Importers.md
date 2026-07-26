# 11 - 导入器（dbt / OSI）

SLayer 提供两个 CLI 导入器把外部语义层规范翻译成 `SlayerModel`：

- `slayer import-dbt` —— dbt Semantic Layer YAML
- `slayer import-osi` —— Open Semantic Interchange（OSI）YAML/JSON

两者都遵循"**构造不丢语义，失败不静默**"原则：能转的字段转成 `SlayerModel` 对应字段；转不出的进 `ConversionResult` 报告，**绝不**静默丢。报告类型 (`ConversionResult` / `ConversionWarning`) 在 [`slayer/ingest_report.py`](../../slayer/ingest_report.py) 是中性的，`slayer/dbt/converter.py` 再 re-export 出去。

## `slayer import-dbt`

[`slayer/dbt/`](../../slayer/dbt/) 完整端口 dbt Semantic Layer 的 schema：

| 文件 | 作用 |
| --- | --- |
| `models.py` | Pydantic 端口 dbt `DbtProject` / `DbtSemanticModel` / `DbtMetric` 等 |
| `parser.py` | YAML 解析 |
| `manifest.py` | 读取 dbt `manifest.json` 跟踪 `ref()` 解析 |
| `entities.py` | `EntityRegistry` —— 跨数据集共享 entity 名解析 |
| `filters.py` | dbt filter 字符串 → SLayer filter 字符串 |
| `sql_resolver.py` | resolve `ref(...)` 引用 |
| `converter.py` | `DbtToSlayerConverter` 主编排 |

### 转换规则（节选）

- 每个 `DbtSemanticModel` → 1 个 `SlayerModel`
- dbt `dimensions`（`categorical` / `time`）→ `Column`（`type` 按语义设）
- dbt `measures` → `Column`（base）+ `ModelMeasure`（命名公式）
- dbt `metrics` 按类型折叠到源语义 model 的 `ModelMeasure`：
  - `simple` with filter → `ModelMeasure.formula`
  - `ratio` → 算术公式
  - `derived` → 嵌套公式
  - `cumulative` → `cumsum` / 时间窗口
- 失败 clean 的（conversion metrics / windowed grain-to-date cumulatives / semi-additive）→ 进报告，原始 construct 存进拥有实体的 `meta`

### 启动

```bash
slayer import-dbt /path/to/dbt_project --datasource my_postgres
```

需要 `[dbt]` extra 装 `dbt-core`。

## `slayer import-osi`

[`slayer/osi/`](../../slayer/osi/)（DEV-1643）端口 Open Semantic Interchange 规范：

| 文件 | 作用 |
| --- | --- |
| `models.py` | Pydantic 端口 OSI schema（v1.0 / 0.1.0 / 0.1.1 / 0.2.0.dev0 结构同义） |
| `source.py` | `parse_source` 切 `[cat.]db.schema.table`；`resolve_datasource(database, default)` 是 per-database routing 钩子（当前 drop db） |
| `expression.py` | `convert_expression` —— sqlglot-AST → SLayer 冒号公式：agg leaves、算术 + 常量、`SCALAR_PASSTHROUGH` funcs、非裸操作数物化为 hidden derived column、percentile → `col:percentile(p=…)` / `median` |
| `converter.py` | `OsiToSlayerConverter` 主入口 |
| `parser.py` | YAML/JSON 解析 |

### 转换流程

1. **Live introspection** —— 对 OSI dataset 的物理表做实时 schema 反射（真列类型 + PK），OSI 语义 metadata（label / description / is_time / `ai_context` / `custom_extensions` / `unique_keys`）overlay 上去
2. **Relationships → ModelJoin** —— composite-safe 复合键，length mismatch clean-fail
3. **Metrics → ModelMeasure** —— 通过共享 `slayer/engine/join_graph.py::min_hops_root` 选锚点 model
4. **Cross-dataset 引用** → 锚点相对的 dotted path
5. **列-less `COUNT(*)`** → 锚在 semantic model 的**唯一 fact table**（永远不用作 join target 的那个 dataset）—— 0 或多个候选 → 干净失败为 orphan（grain ambiguous，永不猜测）

### 字段映射

| OSI 字段 | SLayer 字段 |
| --- | --- |
| `description` | `description` |
| `label` | `label` |
| `is_time: true` | `Column.type: TIMESTAMP` |
| `ai_context` | `description` + `meta['osi_ai_context']` |
| `custom_extensions` | `meta` |
| `unique_keys` | `meta` |

### 启动

```bash
slayer import-osi /path/to/osi.yaml --datasource my_postgres [--dialect ANSI_SQL]
```

CLI-only —— 无 MCP / REST 暴露。OSI 直接对 SLayer（**不**走 dbt 中转，避免 dbt converter 的两个结构不匹配）。

## 共同的设计原则

1. **Live introspection 优先**：从实表读真列类型 / PK，而不是信任用户提供的 schema 描述
2. **跨 dataset 引用走 join graph**：OSI 锚点选择复用 `min_hops_root`（`recommend_root_model` 也用），dbt 走 `EntityRegistry` 实体名解析
3. **构造不丢语义**：转不出就报告，绝不静默丢；原始 construct 进 meta 保留
4. **复合键 / 多跳 join 优先 composite-safe**：单边 PK 缺时 clean-fail
5. **行级失败聚合**：单 metric 失败不影响其他 metric 转换
6. **报告驱动**：返回 `ConversionResult(additions, warnings, errors)`，CLI / MCP 都基于同一 report 渲染

## 转换报告（`slayer/ingest_report.py`）

```python
class ConversionWarning(BaseModel):
    entity: str                  # 哪个 model / metric / measure 触发
    kind: str                    # "unsupported_construct" / "missing_entity" / "type_mismatch" / ...
    message: str
    location: str | None = None  # YAML 行号 / 字段路径

class ConversionResult(BaseModel):
    models_added: list[SlayerModel]
    models_updated: list[SlayerModel]
    warnings: list[ConversionWarning]
    errors: list[ConversionWarning]
```

CLI 输出按"已添加 / 已更新 / 警告 / 错误"分组打印。

## 下一步

- 看怎么把这些 model 跑起来：[12 - 运行与部署](12-Running-and-Deployment.md)
- 看 storage 如何持久化：[06 - 存储后端](06-Storage-Backends.md)
