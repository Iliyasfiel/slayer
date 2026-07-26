# 08 - BI 接入层（Facades）

为了让任何 BI 工具 / JDBC 客户端 / Postgres 客户端都能消费 SLayer 的模型作为"表"，SLayer 提供两个 wire-protocol facade。两者**共享一个 translator 层**（`slayer/facade/`），把传入的 SQL 翻译成 `SlayerQuery`，再交给 `engine.execute()`。

## 端口速查

| 端口 | 服务 | 启动命令 |
| --- | --- | --- |
| 5143 | REST API + MCP-SSE | `slayer serve` |
| 5144 | Arrow Flight SQL | `slayer flight-serve` |
| 5145 | Postgres wire protocol | `slayer pg-serve` |

`loopback-no-token fallback`：非回环绑定且无 token / authenticator → 启动失败。

## 共享层：`slayer/facade/`

[`slayer/facade/`](../../slayer/facade/) 是 DEV-1486 抽出的、与具体协议无关的翻译/目录层。

```
slayer/facade/
├── translator.py         -- SQL → SlayerQuery 流水线（DEV-1390 §6, DEV-1486, DEV-1558）
├── catalog.py            -- FacadeCatalog/FacadeTable/FacadeMetric/FacadeDimension/FacadeSchema
├── catalog_sql.py        -- pg_catalog / information_schema 的 DuckDB SQL 视图
├── info_schema.py        -- INFORMATION_SCHEMA 探测查询
├── probe_queries.py      -- SELECT 1/SELECT NULL/version() 等白名单
├── rows.py               -- RowBatch（pyarrow-free typed columns + row dicts）
└── datatypes.py          -- SUPPORTED_DATATYPES, datatype_to_jdbc
```

### `translate(sql, catalog, *, dialect=None, probe_matcher=None, catalog_matchers=()) -> TranslatorResult`

流水线：

1. **Parse** sqlglot（可选 `dialect`）
2. **Probe-query whitelist** → 直接给 `RowBatch`
3. **AST 根分类**：
   - DML/DDL → 拒绝
   - `BEGIN/COMMIT/ROLLBACK/SET/SHOW/...` → `NoOpResult(command_tag=...)`（pg-facade 用它驱动事务状态）
   - `SELECT` → 继续
4. **`pg_catalog` 路径**（仅 pg-facade）：如果 `is_catalog_only(parsed) == True` 且有 `catalog_sql_executor`，把 SQL 投到内存 DuckDB 执行，返回 `PgCatalogResult`（Flight 走 INFORMATION_SCHEMA 路径）
5. **`SELECT *` 拒绝**（真实 model 上）
6. **SLayer 表翻译** → `SlayerQuery` + 列名映射

返回的 `TranslatorResult` 是 tagged union：
- `QueryResult` —— 普通查询（带 `facade_table`、投影列、列类型等）
- `NoOpResult` —— 命令无副作用
- `PgCatalogResult` —— pg_catalog 查询结果
- `QueryListResult` —— 多条语句（pg-facade 事务用）

### LEFT JOIN-with-subquery 识别（DEV-1565，共享）

Metabase 的 MBQL 会发 `FROM "public"."orders" LEFT JOIN (SELECT "public"."stores".* FROM "public"."stores") AS "Stores" ON ...`。共享层识别这种**单跳**形态并映射 `<JoinAlias>.<col>` → SLayer 跨模型 dotted form。

- 匹配到 existing join → 用 `SlayerQuery.source_model = parent.name`
- 不匹配 → 构建 inline `ModelExtension` 并 `logger.warning("pg-facade: dynamic join ...")`
- 配置 INNER 但匹配 → 仍用 existing join + 卡片警告
- Phase 1 范围：恰好 1 个 LEFT JOIN；右侧必须是 `(SELECT ... FROM <单表>) AS <alias>`；ON 必须是单等值；多跳 / 复合键不在范围

### 聚合 → 度量映射（DEV-1486 decision 21，共享/Facade-agnostic — Flight 同样适用）

投影中的 `SUM(col)` / `AVG` / `MIN` / `MAX` / `COUNT(col)` / `COUNT(*)` / `COUNT(DISTINCT col)` 自动映射为 `col:sum` / `col:avg` / `*:count` / `col:count_distinct`：

- 在 catalog 的 pre-expanded metrics 中查
- 同样的 `_eligible_aggregations` 规则
- `ORDER BY <agg>` 与 `HAVING <agg> <cmp> <literal>` 也覆盖
- 命中已保存 `ModelMeasure` 或非列表达式 → 抛错指向 **DEV-1493**（多阶段重写）
- 实际为 base-column only：跨表聚合（`COUNT(customers.region)`）在 SLayerQuery measure 名中不容许点号（DEV-1448 范围）

## `SELECT *` browse-mode 扩展

[`slayer/facade/translator.py`](../../slayer/facade/translator.py) 的 `_is_browse_mode_select` + `_expand_select_star`：

- `SELECT * FROM <model>` 无 GROUP BY / HAVING / 投影中的聚合 → 展开为每个非隐藏列
- `SELECT *, COUNT(*) FROM t` / `SELECT * FROM t GROUP BY x` → 仍拒绝并提示 "project specific names"
- 度量**不**被自动包含（强制聚合上下文会反人类）—— 想用则按名引用 saved measure

## Flight SQL 端口（`slayer/flight/`）

[`slayer/flight/`](../../slayer/flight/) 实现 Apache Arrow Flight SQL v18.3.0 协议（与 dbt Semantic Layer 同 JAR）。

### 启动

```bash
slayer flight-serve --host 0.0.0.0 --port 5144 \
    --token pick-a-secret \
    --tls-cert /path/cert.pem --tls-key /path/key.pem \
    [--demo]
```

### 模块分工

| 文件 | 作用 |
| --- | --- |
| `server.py` | `FlightSqlServer(fl.FlightServerBase)` 主体，绑定 `BearerTokenMiddlewareFactory` |
| `handlers.py` | `FlightHandlers`：dispatch `CreatePreparedStatement` / `GetFlightInfo` / `DoGet` / `GetCatalogs` / ... |
| `auth.py` | BearerToken 中间件工厂 + `validate_bind_address` / `validate_tls_pair` |
| `translator.py` | 旧名 shim：re-export `slayer.facade.translator` |
| `catalog.py` | shim：re-export `slayer.facade.catalog` 的 `Facade*` 类 |
| `info_schema.py` / `probe_queries.py` | shim：re-export |
| `types.py` | pyarrow 转换（`row_batch_to_arrow`） |
| `cli.py` | `slayer flight-serve` 入口 |
| `FlightSql.proto` | 协议源（编译产物 `_flight_sql_pb2.py`） |

### Path A vs Path B（"LIMIT 0 双回程"）

上游 Apache JDBC 驱动走 prepared-statement triplet。翻译器/handler 链路每条 BI 查询运行三次：

1. `CreatePreparedStatement` —— validate + 解析列类型
2. `get_flight_info(CommandPreparedStatementQuery)` —— 描述 schema
3. `do_get` —— 拉数据

DB 实际只两次回程：1 次 `LIMIT 0` 拿 schema、1 次全量取数。

### Any wrapping（server.py / handlers.py）

- 驱动发来的 `do_action` body 都是 `google.protobuf.Any` 包装（`type_url` = action 类的全名）
- pyarrow Python 客户端发 raw bytes
- `_parse_action_body` 同时支持；响应永远用 `Any` 包装

### 目录约定

点形式端到端：`customers.regions.name`。同形式在 `INFORMATION_SCHEMA.*`、BI 投影列表、`WHERE`、SLayer DSL 中。无 `__` → `.` 重写步骤。

### Phase 1 限制（CLAUDE.md 明示）

- `JDBC token=X` 不工作（驱动 pre-handshake bearer token，SLayer 中间件只在每 RPC 验 header）
- `flight-sql-jdbc-driver` 反射 `java.nio.Buffer.address` 在 Java 17+ 需要 `--add-opens`
- wire schema 当前从 `QueryResult.projection_types`（`Column.type` / `ModelMeasure.type`）声明，**不**驱动自 `LIMIT 0` 执行结果 —— `ModelMeasure.type` 缺失会让 `ArrowTypeError` 上 wire
- 测试用 `jaydebeapi` + JPype 驱动真 JAR（需要 Java ≥ 11 + Maven Central）
- `tests/integration/test_integration_flight_pyarrow_client.py` 不需 Java，全 Python 覆盖

### 客户端例子

```python
import pyarrow.flight as fl
client = fl.FlightClient("grpc+tls://localhost:5144")
client.authenticate_basic_token("ignored", "pick-a-secret")
desc = fl.FlightDescriptor.for_command(b"SELECT ...")
info = client.get_flight_info(desc)
reader = client.do_get(info.endpoints[0].ticket)
table = reader.read_all()
```

## Postgres 协议端口（`slayer/pg_facade/`）

[`slayer/pg_facade/`](../../slayer/pg_facade/)（DEV-1486）是**纯标准库**的 Postgres wire-protocol v3 服务端：

- 无新增运行时依赖（不走 asyncpg —— 反而是给 asyncpg 用的）
- 默认端口 **5145**
- `slayer pg-serve --host 0.0.0.0 --port 5145 --token pick-a-secret [--demo]`

### 启动

```bash
slayer pg-serve --demo --host 0.0.0.0 --token pick-a-secret
# 然后 BI 工具（Metabase / Superset / Tableau / Power BI / Looker）连：
#   host=host.docker.internal  port=5145  database=jaffle_shop
#   user=anything  password=pick-a-secret  SSL=off
```

### 多 DS 路由

- 客户端 startup 的 `database` 参数 → 锁定一个 SLayer datasource
- 那个 DS 的所有 model 在 PG schema `public` 下作为"表"出现
- `current_database()` → datasource name
- `current_schema()` → `public`
- 未知 / 缺失 `database` → `FATAL: database "<name>" does not exist`（SQLSTATE `3D000`）at startup
- 跨 DS 查询不支持

### 鉴权（`auth.py`）

- `AuthenticationCleartextPassword` + 可选 `--token`
- 绑定规则与 Flight 一致：loopback 可无 token
- TLS 升级当 `--tls-cert` / `--tls-key` 设置；否则对 `SSLRequest` 回复 `N`
- `verify_password` 常数时间；`None` token 接受任意非空 password，空 password 永远拒绝

### 线格式

- simple-query（`Q`）结果永远 text
- extended-query（`Parse/Bind/Describe/Execute/Sync`）尊重 Bind 的 per-column format code —— **text 与 binary encoder 都有** 6 个内置类型
- binary 遵循 `integer_datetimes=on`：int8 大端、float8 IEEE、date int32 自 2000-01-01 起、timestamp int64 微秒 自 2000-01-01 起
- `value_to_text` 输出 Postgres 格式 float（`NaN/Infinity/-Infinity`）和空格分隔 timestamp
- 类型 → OID：`TEXT→25`, `INT→20`, `DOUBLE→701`, `BOOLEAN→16`, `DATE→1082`, `TIMESTAMP→1114`；未知 → text
- **只**这些内置 OID 出现 → asyncpg 不触发 `pg_type` 自省路径

### 绑定参数

字面替换：每值按 Bind format code decode（text/binary），渲染为安全 quoted literal，`$N` 替换进 SQL **之后**才送翻译。`ParameterDescription` 从 `$N` 占位符数推断参数（asyncpg 留 Parse OID 空），未指定参数默认 text。

### 事务状态机（`connection.py`）

- 每连接 `I` / `T` / `E` 在每个 `ReadyForQuery` 报告
- `BEGIN / START TRANSACTION` → `T`
- `COMMIT / ROLLBACK / END` → `I`
- `T` 中出错 → `E`（直到 tx end 发 `25P02`）
- **单条** simple-query `Q` 消息可携带多条分号分隔语句 —— 每条自己的 `CommandComplete`，恰好一个 `ReadyForQuery` 收尾
- `max_rows` on `Execute` 忽略（无 `PortalSuspended`）
- Empty query → `EmptyQueryResponse`
- `FunctionCall` / `CopyData` / `CopyDone` / `CopyFail` → `0A000`

### `pg_catalog`（`pg_catalog.py`）

Phase 1 覆盖：`pg_namespace` / `pg_class` / `pg_attribute` / `pg_type` / `pg_proc` / `pg_settings`。`WHERE` 被忽略（返回所有行；客户端 filter）；`pg_catalog.X` 与裸 `X` 都解析。OID 用 `zlib.crc32`（不是 `hash()`，因为后者进程内 salted）确定性生成，冲突构建时检查。

**DS-aware probes**（`probes.py`）：

- `version()` → PostgreSQL 字符串
- `current_database()` → datasource
- `current_schema()` → `public`
- `SHOW <x>` → 单行
- `current_setting('jit')` → `off`
- `set_config('jit', ...)` → 值
- 回退到共享 `match_probe` 处理 `SELECT 1` / `SELECT NULL WHERE 1=0`

### psql backslash 命令覆盖

`\d` / `\du` / `\l` 都走 `pg_class + pg_namespace + pg_am` / `pg_roles` / `pg_database` 三个 stub。`_build_pg_am` 暴露 canonical `heap` access-method 行（`pg_class.relam = 2`），`_build_pg_roles` 暴露一个 synthetic `slayer` 角色（共享 token auth 无 per-user identities），`_build_pg_database` 暴露一行 for connected datasource。`pg_encoding_to_char(6) → 'UTF8'` 常量 stub 闭环 `\l` 的 encoding lookup。

**扩展性**：`build_catalog_relations(catalog, datasource, *, extra_relations=...)` 接受可迭代 `CatalogRelation`；每个要么按表名 match 替换默认 builder（override 场景，如把真实 per-tenant 行投影到 `pg_roles`），要么新增表（`pg_extension`、自定义视图等）。通过 `CatalogSqlExecutor` → `executor_for(..., extra_relations=...)`（绕过 fingerprint 缓存当 extras 传）→ `PgConnection(catalog_extra_relations=...)` → `serve(catalog_extra_relations=...)` 串通。Storyline 风格 embedder 用此把真实 principal/tenant 行投影到 `pg_roles` / `pg_database` 而不派生 facade。

## 启动脚本示例

```bash
# 三服务同时跑（分别端口）
slayer serve           --demo --host 127.0.0.1 --port 5143 &
slayer flight-serve    --demo --host 127.0.0.1 --port 5144 &
slayer pg-serve        --demo --host 127.0.0.1 --port 5145 &

# 让 Metabase 走 pg-facade
docker run -d -p 3000:3000 --name metabase \
    --add-host=host.docker.internal:host-gateway \
    metabase/metabase
# Metabase: Add database -> PostgreSQL
#   host=host.docker.internal  port=5145  database=jaffle_shop
#   user=anything  password=pick-a-secret  SSL=off
```

## 下一步

- 看 CLI / REST / MCP 三个"直接面向 Agent"的接口：[09 - 对外接口](09-Interfaces.md)
- 看怎么让存储模型驱动 dbt 或 OSI 项目：[11 - 导入器](11-Importers.md)
