# SLayer Code Wiki

> **SLayer (Semantic Layer)** 是由 [MotleyAI](https://github.com/MotleyAI) 开发的轻量级、面向 AI Agent 的开源语义层（MIT 协议）。
> 它位于数据库与 Agent / BI 工具 / 脚本之间，接收结构化查询描述，生成并执行目标方言的 SQL。

本 Wiki 是面向工程师的项目代码阅读指南。它与 [docs/](https://docs.motley.ai/slayer) 的用户文档互补 —— 官方文档讲"怎么用 SLayer"，本 Wiki 讲"SLayer 在源码层是怎么组织的、关键模块如何协作"。

---

## 快速跳转

| 章节 | 内容 |
| --- | --- |
| [01 - 项目概览](01-Overview.md) | 一句话定位、能力矩阵、技术栈、运行环境 |
| [02 - 架构总览](02-Architecture.md) | 分层架构图、查询生命周期、模块依赖图、异步模型 |
| [03 - 核心数据模型](03-Core-Data-Models.md) | `SlayerModel` / `Column` / `ModelMeasure` / `SlayerQuery` / `DatasourceConfig` 等 Pydantic 模型 |
| [04 - 查询生命周期](04-Query-Lifecycle.md) | `SlayerQuery` → `EnrichedQuery` → SQL → 执行 的端到端流水线 |
| [05 - SQL 生成与方言](05-SQL-Generation-and-Dialects.md) | `SQLGenerator`、方言策略模式、保留字、`SessionPolicy` |
| [06 - 存储后端](06-Storage-Backends.md) | `YAMLStorage` / `SQLiteStorage`、迁移链、embeddings 侧车 |
| [07 - 引擎内部](07-Engine-Internals.md) | `SlayerQueryEngine`、缓存、auto-ingestion、schema drift、profiling |
| [08 - BI 接入层（Facades）](08-BI-Facades.md) | Flight SQL 端口 5144、Postgres 端口 5145、共享 translator |
| [09 - 对外接口](09-Interfaces.md) | CLI、REST、Python 客户端、MCP 端口 |
| [10 - AI 记忆与搜索](10-AI-Memory-and-Search.md) | `Memory` / `search()` / 三通道 RRF 融合 / 嵌入 |
| [11 - 导入器（dbt / OSI）](11-Importers.md) | `slayer import-dbt`、`slayer import-osi` |
| [12 - 运行与部署](12-Running-and-Deployment.md) | 安装、CLI 速查、Docker 示例、demo 数据集 |
| [13 - 术语表](13-Glossary.md) | 核心名词解释 |

---

## 仓库基本信息

- **项目名**：`motley-slayer`（PyPI 包名 / 仓库名）
- **当前版本**：`0.9.9`（见 [pyproject.toml](../../pyproject.toml)）
- **Python 要求**：`>= 3.11`
- **协议**：MIT
- **默认端口**：REST `5143`、Flight SQL `5144`、Postgres 协议 `5145`
- **主要依赖**：`sqlglot`（AST 化 SQL 生成）、`sqlalchemy>=2.0`、`pydantic>=2.0`、`fastapi`、`mcp`（MCP 协议）、`tantivy` + `rank-bm25`（本地搜索）、`litellm`（可选嵌入）
- **关键可选项**：`postgres` / `mysql` / `clickhouse` / `sqlserver` / `snowflake` / `bigquery`（各 DB 驱动）、`flight`（`pyarrow`）、`advanced_search`（`litellm` + `numpy` + `ladybug`）、`dbt`

---

## 文档约定

- 文中"**Mode A (SQL)**"和"**Mode B (DSL)**"指两种引用模式，详见 [03 - 核心数据模型 / 引用模式](03-Core-Data-Models.md#引用模式-mode-a--mode-b)。
- 路径一律以仓库根目录为基准，例如 `slayer/engine/query_engine.py`。
- DEV-XXXX 是 MotleyAI 内部 Linear 工单编号，源码注释中频繁出现，用于追溯某条行为约束的来源 —— 引用时保留。
