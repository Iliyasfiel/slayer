# 12 - 运行与部署

## 安装

### 推荐方式：`uv`（`pip` / `pipx` 同样可用）

```bash
# 完整安装（含所有 extras）
uv tool install 'motley-slayer[all]'

# 最小安装
uv tool install motley-slayer

# 装额外方言驱动
uv tool install 'motley-slayer[postgres,mysql,clickhouse,sqlserver,bigquery,snowflake]'
uv tool install 'motley-slayer[flight]'          # Flight SQL 需要的 pyarrow
uv tool install 'motley-slayer[advanced_search]' # litellm + numpy + ladybug (graph + embedding)
uv tool install 'motley-slayer[dbt]'             # dbt import

# 找不到 slayer 时
uv tool update-shell
```

### Poetry（开发）

```bash
git clone https://github.com/MotleyAI/slayer
cd slayer
poetry install -E all

# 跑命令
poetry run slayer --help
poetry run slayer serve --demo

# 跑测试（永远用 poetry run）
poetry run pytest                                # unit
poetry run pytest -m integration                 # 集成
poetry run pytest tests/integration/test_integration_postgres.py -m integration
poetry run pytest tests/integration/test_integration_duckdb.py -m integration
```

## 快速起步（Jaffle Shop demo）

零配置方式让 SLayer 跑起来，database 是内置的 DuckDB + `jafgen` 生成的 Jaffle Shop：

```bash
# 一步启动 MCP（stdio 给 Claude Code）
claude mcp add slayer_demo -- slayer mcp --demo

# 或启 REST API（端口 5143）
slayer serve --demo

# 或把 BI 工具接上
slayer pg-serve --demo --host 127.0.0.1   # 端口 5145
slayer flight-serve --demo --host 127.0.0.1   # 端口 5144
```

## 接入自己的数据

### 一步创建数据源

```bash
slayer datasources create 'postgresql://user:${DB_PASSWORD}@hostname/db_name'
# 密码运行时从环境读，不落盘

# 一步 + 立即 ingest
slayer datasources create 'postgresql://user:${DB_PASSWORD}@localhost/analytics' --ingest
```

### 手动 YAML 配置

```yaml
# datasources/my_postgres.yaml
name: my_postgres
type: postgres
host: ${DB_HOST}
port: 5432
database: ${DB_NAME}
username: ${DB_USER}
password: ${DB_PASSWORD}
```

> `${ENV_VAR}` 在 read time 解析（CLAUDE.md / `DatasourceConfig.resolve_env_vars()`）。

### 自动建模

```bash
slayer ingest --datasource my_postgres --schema public
```

幂等：可重复跑，新表/新列追加，已有 metadata 不覆盖。

### 启动时自动 ingest（DEV-1392）

```bash
slayer serve --ingest-on-startup
slayer mcp --ingest-on-startup

# 或
SLAYER_INGEST_ON_STARTUP=1 slayer serve
```

## Docker Compose 示例

仓库自带 6 个 dialect 示例（`examples/`）：

| 目录 | 数据库 | 启动 |
| --- | --- | --- |
| `examples/embedded/` | SQLite (in-process) | `python run.py` |
| `examples/postgres/` | Postgres | `docker compose up -d && bash start.sh` |
| `examples/mysql/` | MySQL | `docker compose up -d && bash start.sh` |
| `examples/clickhouse/` | ClickHouse | `docker compose up -d && bash start.sh` |
| `examples/sqlserver/` | SQL Server 2022 | `docker compose up -d && bash start.sh` |
| `examples/snowflake/` | Snowflake（云） | 设置 `~/.snowflake/connections.toml` 后 `python verify.py` |
| `examples/bigquery/` | BigQuery（云） | 设置 `GOOGLE_APPLICATION_CREDENTIALS` + `GCP_PROJECT_ID` 后 `python verify.py` |

每个示例都自带 `verify.py` 跑一遍 sanity check。

## CLI 速查

```bash
# 模型
slayer models list
slayer models get my_postgres orders
slayer models inspect my_postgres orders
slayer models delete my_postgres orders

# 数据源
slayer datasources list
slayer datasources create 'postgresql://...'
slayer datasources get my_postgres
slayer datasources test my_postgres
slayer datasources priority                          # 显示
slayer datasources priority db_a db_b db_c           # 设置

# 查询
slayer query '{"source_model": "orders", "measures": ["*:count", "revenue:sum"], "dimensions": ["status"]}'
slayer query @query.json --format json
slayer query my_postgres.orders --variables region=EU  # run-by-name

# 漂移
slayer validate-models --datasource my_postgres
slayer validate-models --datasource my_postgres --force-clean --yes

# 搜索
slayer search --entity mydb.orders.revenue --max-results 10
slayer search --question "what was last quarter revenue"
slayer search --cypher-filter "MATCH (n:Model) WHERE n.datasource = 'mydb' RETURN n.id AS id"
slayer search refresh-samples --data-source my_postgres

# 记忆
slayer memory save "revenue excludes refunds" --entity mydb.orders.revenue
slayer memory list
slayer memory get 1
slayer memory forget 1

# 导入
slayer import-osi /path/to/osi.yaml --datasource my_postgres
slayer import-dbt /path/to/dbt_project --datasource my_postgres

# 其他
slayer recommend-root-model mydb.orders.revenue mydb.customers.email
slayer storage migrate-types --dry-run
slayer inspect mydb.orders --type model --no-compact --format json
```

## REST API 速查

```bash
# 启动
slayer serve --host 0.0.0.0 --port 5143 --ingest-on-startup

# 查询
curl -X POST http://localhost:5143/query \
  -H "Content-Type: application/json" \
  -d '{"source_model": "orders", "measures": ["*:count"], "dimensions": ["status"]}'

# 多阶段 DAG
curl -X POST http://localhost:5143/query \
  -H "Content-Type: application/json" \
  -d '{"queries": [{"source_model": "orders", ...}, {"name": "agg", "source_model": "orders", ...}]}'

# 模型 / 数据源
curl http://localhost:5143/models
curl http://localhost:5143/models/my_postgres/orders
curl http://localhost:5143/datasources/my_postgres          # credentials masked
curl -X PUT http://localhost:5143/datasources/priority \
  -H "Content-Type: application/json" \
  -d '["db_a", "db_b"]'

# 搜索 / 记忆 / 导入 / 漂移 / 推荐 / 检视
curl -X POST http://localhost:5143/search -d '{"question":"recent revenue","max_results":5}'
curl -X POST http://localhost:5143/memories -d '{"learning":"...","linked_entities":["mydb.orders.revenue"]}'
curl -X POST http://localhost:5143/inspect -d '{"reference":"mydb.orders","entity_type":"model","compact":true}'
curl -X POST http://localhost:5143/validate-models -d '{"datasource":"my_postgres"}'
curl -X POST http://localhost:5143/recommend-root-model \
  -d '{"items":["my_postgres.orders.revenue","my_postgres.customers.email"]}'
```

## MCP

```bash
# stdio（默认）
claude mcp add slayer -- slayer mcp

# stdio + demo
claude mcp add slayer -- slayer mcp --demo

# stdio + ingest-on-startup
claude mcp add slayer -- slayer mcp --ingest-on-startup

# SSE / streamable HTTP（REST 服务在跑时）
claude mcp add slayer-remote --transport sse --url http://localhost:5143/mcp/sse
```

> CLAUDE Code 启动的 shell 必须 `export DB_PASSWORD` —— MCP 子进程继承环境。

## Python 客户端

```bash
# 服务端 + 远程客户端
slayer serve --demo &
python - <<'PY'
from slayer.client.slayer_client import SlayerClient
from slayer.core.query import SlayerQuery
import pandas as pd

client = SlayerClient(url="http://localhost:5143")
df: pd.DataFrame = client.query_df(
    SlayerQuery(source_model="orders", measures=["*:count","revenue:sum"], dimensions=["status"])
)
print(df)
PY
```

```python
# 本地模式（无 server）
from slayer.client.slayer_client import SlayerClient
from slayer.storage.yaml_storage import YAMLStorage
client = SlayerClient(storage=YAMLStorage(base_dir="./my_models"))
```

## Flight SQL / Postgres Facade 客户端

### Metabase 走 pg-facade

```bash
# SLayer 端
slayer pg-serve --demo --host 0.0.0.0 --token pick-a-secret

# Metabase 端
docker run -d -p 3000:3000 --name metabase \
  --add-host=host.docker.internal:host-gateway \
  -e MB_DB_FILE=/metabase.data/metabase.db \
  -v metabase-data:/metabase.data \
  metabase/metabase
# Metabase UI: Add database → PostgreSQL
#   host=host.docker.internal  port=5145  database=jaffle_shop
#   user=anything  password=pick-a-secret  SSL=off
```

### DBeaver / JDBC 走 Flight

```bash
slayer flight-serve --demo --host 0.0.0.0 --port 5144
# DBeaver → New Connection → Apache Arrow Flight SQL
#   host=localhost  port=5144  URL=jdbc:arrow-flight-sql://localhost:5144
#   Username=ignored  Password=任何非空串（除 loopback 都需正确 token）
```

> Java 17+ 用户需要在 DBeaver 的 JVM flags 加：
> ```
> --add-opens=java.base/java.nio=ALL-UNNAMED
> --add-opens=java.base/java.lang=ALL-UNNAMED
> --add-opens=java.base/java.util=ALL-UNNAMED
> ```

## 环境变量清单

| 变量 | 用途 | 默认 |
| --- | --- | --- |
| `SLAYER_STORAGE` | 存储路径覆盖 | 平台默认 |
| `SLAYER_MODELS_DIR` | 旧版路径 | — |
| `SLAYER_INGEST_ON_STARTUP` | `1` / `true` / `yes` 启动时 ingest | off |
| `SLAYER_FLIGHT_TOKEN` | Flight 服务 token | — |
| `SLAYER_PG_TOKEN` | pg-facade token | — |
| `SLAYER_EMBEDDING_MODEL` | 嵌入模型名 | `openai/text-embedding-3-small` |
| `GOOGLE_APPLICATION_CREDENTIALS` | BigQuery 认证 JSON 路径 | — |
| `GCP_PROJECT_ID` | BigQuery billing project | — |
| `DB_PASSWORD` 等 | datasource `${ENV}` 引用 | — |

## 测试矩阵

| 测试 | 命令 |
| --- | --- |
| 单元 | `poetry run pytest` |
| SQLite 集成 | `poetry run pytest tests/integration/test_integration.py -m integration` |
| Postgres 集成 | `poetry run pytest tests/integration/test_integration_postgres.py -m integration` |
| DuckDB 集成 | `poetry run pytest tests/integration/test_integration_duckdb.py -m integration` |
| MySQL 集成 | `poetry run pytest tests/integration/test_integration_mysql.py -m integration` |
| ClickHouse 集成 | `poetry run pytest tests/integration/test_integration_clickhouse.py -m integration` |
| SQL Server 集成 | `poetry run pytest tests/integration/test_integration_sqlserver.py -m integration` |
| Snowflake 集成 | `poetry run pytest tests/integration/test_integration_snowflake.py -m integration`（需要凭证） |
| Flight 集成 | `poetry run pytest tests/integration/test_integration_flight.py -m integration` |
| pg-facade 集成 | `poetry run pytest tests/integration/test_integration_pg_facade.py -m integration` |
| Metabase e2e | `poetry run pytest -m metabase_e2e tests/integration/test_metabase_e2e.py -v`（需 Docker） |
| Lint | `poetry run ruff check slayer/ tests/` |

CI 编排见 `.github/workflows/`：
- `ci.yml` —— 主流程（Postgres / DuckDB / SQLite / Flight / pg-facade）
- `integration-{mysql,clickhouse,sqlserver}.yml` —— 三方言
- `pg-facade-e2e.yml` —— 真 Metabase e2e
- `publish.yml` / `publish-test.yml` —— PyPI 发布
- `notify-docs.yml` —— 文档更新

## Docker

`Dockerfile` 在仓库根：

```bash
docker build -t motley-slayer:latest .
docker run -p 5143:5143 -p 5144:5144 -p 5145:5145 \
  -v $PWD/slayer_data:/data \
  -e SLAYER_STORAGE=/data \
  motley-slayer:latest \
  slayer serve --host 0.0.0.0 --ingest-on-startup
```

## License

MIT —— 见 [LICENSE](../../LICENSE)。
