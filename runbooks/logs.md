# 日志查询 Runbook

> 适用栈：yw-mall-deploy 容器化运行 + env Loki/Promtail（`logs` profile）
> 创建：2026-05-21 — Sprint 4 后

---

## TL;DR — 3 种方式速选

| 场景 | 用哪个 |
|------|--------|
| 复盘 / 查历史 / 多服务关联 | **Grafana Explore (Loki)** |
| 实时盯一个服务、最快上手 | `podman logs -f` |
| 脚本 / CI / 告警系统取数 | Loki HTTP API |

---

## 1. Grafana Explore（推荐）

入口：**http://localhost:3000** → 用 `admin / admin123` 登录 → 左侧菜单 **Explore** → 顶部数据源下拉选 **Loki** → 在查询框粘 LogQL。

右上角可：选时间范围、开 **Live tail**（实时滚屏）、加 split view（左右分屏对比两个服务）。

### LogQL 速查

```logql
# === 按服务过滤 ===
{service="mall-api"}                          # C 端 API 网关
{service="mall-admin-api"}                    # Admin API 网关
{service=~"mall-(api|admin-api)"}             # C + Admin 同时看
{service=~"mall-.*-rpc"}                      # 所有 RPC
{stack="yw-mall"}                             # yw-mall 全部 17 个服务

# === 按级别过滤（level 在 Promtail pipeline 提为 label）===
{service="mall-api", level="error"}
{stack="yw-mall", level=~"error|severe"}

# === 全文搜索（go-zero 的 content 字段已被 Promtail 解析为正文）===
{service="mall-api"} |~ "alice"               # 包含 "alice"
{service="mall-order-rpc"} |~ "/api/order/create"
{service="mall-payment-rpc"} |~ "(?i)refund"  # 忽略大小写

# === 排除噪声（go-zero 默认 stat 日志很吵）===
{service="mall-api"} != "stat"
{service=~"mall-.*-rpc", level!="stat"}

# === metrics（按服务计算 error 速率）===
sum by (service) (rate({stack="yw-mall", level="error"}[5m]))

# === APISIX 网关访问日志（service 标签是 mall-* 系列；APISIX 容器单独查）===
{container="apisix"}                          # 用 container 标签，不是 service
{container="apisix"} |~ "502"
```

### LogQL 提示

- `=` 精确，`=~` 正则，`!=` 不等，`!~` 不匹配
- `|~ "pattern"` 是 line filter，跑在 chunk 内、便宜
- `| json` 显式解析（Promtail 已经 pipeline 解过一遍，正常不用再 `| json`）
- 时间范围越窄越快；先 5m 看趋势再放宽

---

## 2. podman logs（实时 tail，无需 UI）

```bash
# 单服务 follow
podman logs -f yw-mall-deploy_mall-api_1
podman logs -f yw-mall-deploy_mall-admin-api_1
podman logs -f yw-mall-deploy_mall-user-rpc_1

# 限制行数（默认全部，长服务会刷屏）
podman logs --tail 50 -f yw-mall-deploy_mall-order-rpc_1

# 同时盯多个服务（后台 + 标签前缀，按 Ctrl+C 全停）
for svc in mall-api mall-admin-api mall-payment-rpc; do
  podman logs -f yw-mall-deploy_${svc}_1 2>&1 | sed "s/^/[${svc}] /" &
done; wait

# APISIX 网关：每个请求落到哪个 upstream + 耗时 + 响应码
podman logs -f apisix

# 看 APISIX 错误日志（502/504 排查）
podman logs apisix 2>&1 | grep -E 'error|502|504' | tail -20
```

容器名规则：`yw-mall-deploy_<服务名>_1`（podman-compose 后缀 `_1`）。
gateway/audit 类（APISIX / proxysql / kafka1）容器名就是裸名。

---

## 3. Loki HTTP API（脚本 / 告警）

```bash
# 最近 5 分钟 mall-api 的 error，提取 content
curl -sG http://localhost:3100/loki/api/v1/query_range \
  --data-urlencode 'query={service="mall-api",level="error"}' \
  --data-urlencode 'start='$(($(date +%s)-300))'000000000' \
  --data-urlencode 'limit=50' | jq -r '.data.result[].values[][1]'

# 列出所有 service 标签值（确认收到了哪些服务的日志）
curl -s http://localhost:3100/loki/api/v1/label/service/values | jq

# 统计某时段 error 数（用于告警阈值）
curl -sG http://localhost:3100/loki/api/v1/query \
  --data-urlencode 'query=sum(count_over_time({stack="yw-mall",level="error"}[10m]))' \
  | jq '.data.result[0].value[1]'
```

参数：`start` / `end` 是 nanosecond unix；`limit` 默认 100；`direction=backward` 倒序（默认）。

---

## 各服务 service 标签速查

| 想查 | service= | 容器 |
|------|----------|------|
| C 端 HTTP 入口 | `mall-api` | `yw-mall-deploy_mall-api_1` |
| Admin HTTP 入口 | `mall-admin-api` | `yw-mall-deploy_mall-admin-api_1` |
| 用户 / 登录 / KYC RPC | `mall-user-rpc` | `yw-mall-deploy_mall-user-rpc_1` |
| 订单 / 退款 RPC | `mall-order-rpc` | `yw-mall-deploy_mall-order-rpc_1` |
| 购物车 RPC | `mall-cart-rpc` | `yw-mall-deploy_mall-cart-rpc_1` |
| 商品 RPC | `mall-product-rpc` | `yw-mall-deploy_mall-product-rpc_1` |
| 支付 RPC | `mall-payment-rpc` | `yw-mall-deploy_mall-payment-rpc_1` |
| 店铺 RPC | `mall-shop-rpc` | `yw-mall-deploy_mall-shop-rpc_1` |
| 风控 / 规则 / 工作流 / 奖励 / 物流 / 评价 / 活动 | `mall-<x>-rpc` | `yw-mall-deploy_mall-<x>-rpc_1` |
| 活动异步 worker | `mall-activity-async-worker` | `yw-mall-deploy_mall-activity-async-worker_1` |
| C 端 H5 静态 | `mall-fe` | `yw-mall-deploy_mall-fe_1` |
| **APISIX 网关** | — *(用 container 标签)* | `apisix` |
| **infra**（mysql / redis / kafka / etcd / pg / mongo） | — *(用 container 标签 + stack="infra")* | 各自裸名 |

---

## 典型场景

### 场景 0：拿用户提供的 ID 关联整条请求链路

每条经 nginx 的请求都被打了两个 ID，对应 Loki 里的不同字段：

| 来源 | 头 | 哪里能拿 | Loki 字段 |
|------|----|---------|----------|
| mall-fe nginx | `X-Request-Id` | 浏览器 F12 → Network → Response Headers | APISIX access log 文本里 |
| go-zero (mall-api / RPC) | W3C `traceparent` | go-zero 自动；mall-api → user-rpc 自动透传同一 trace_id | `trace` label / JSON 字段 |

**用户拿 X-Request-Id 给你时**：先在 APISIX 容器日志里找对应行的 trace_id，再去 Loki：

```bash
# 1) APISIX 日志里搜 request id（access log 文本格式）
podman logs apisix 2>&1 | grep d16abad2a68f2595e5a589438e2992c5

# 2) 看 mall-api loghandler 行（同时间窗内即可），拿 trace=
podman logs --since 10m yw-mall-deploy_mall-api_1 | grep -E '"trace":"[a-f0-9]+"'
```

**已知 trace_id 反查全链路**（Grafana）：

```logql
{stack="yw-mall"} | json | trace="dcfa233e4320819853237c3558e4f0f6"
```

会一行行排出 mall-api / user-rpc / product-rpc / ... 在同一请求里的所有日志（含 SQL），按时间排序就是完整调用栈。

### 场景 1：用户报错"下单 500"

```logql
# 1) 看 mall-api 网关在那个时间窗有没有 5xx
{service="mall-api"} |~ "/api/order/create"

# 2) 跟到 order-rpc 看真正的 stack trace
{service="mall-order-rpc", level="error"}

# 3) 关联同一 trace（如有 trace_id 透传）
{stack="yw-mall"} |~ "trace=abc123"
```

### 场景 2：APISIX 502

```bash
# 1) 看是哪个 upstream connection refused
podman logs --tail 50 apisix 2>&1 | grep -E "upstream|connect"

# 2) 确认下游服务还活着
podman ps --format 'table {{.Names}}\t{{.Status}}' | grep yw-mall-deploy

# 3) DNS 漂移类问题：重启 APISIX 让 resolver 重新解析（30s TTL 已配，应该自愈）
podman restart apisix
```

### 场景 3：S4 安全事件复盘（失败登录 / IP 拦截）

```logql
# admin 失败登录
{service="mall-admin-api"} |~ "login_fail|账号已锁定"

# IP 白名单拦截
{service="mall-admin-api"} |~ "IP 不在白名单"

# MFA 验证失败
{service="mall-admin-api"} |~ "MFA"
```

### 场景 4：性能下降 / 慢请求定位

```logql
# go-zero stat 行已被 Promtail 解析 content。粗看 qps：
{service=~"mall-.*-rpc"} |~ "qps:"

# APISIX 自己的 latency 在 prometheus，不在 Loki
# 看 http://localhost:9091/apisix/prometheus/metrics
```

---

## 启停 / 维护

```bash
# 启日志栈
cd ~/workspace/go/mall/env
podman-compose --profile logs up -d loki promtail

# 停（数据保留在 ./data/loki/）
podman-compose --profile logs stop loki promtail

# 看 Promtail 自己抓的状态（哪些容器命中规则）
podman logs --tail 20 promtail | grep -i target

# 重置（删存档）
podman-compose --profile logs down
podman unshare rm -rf ./data/loki/* ./data/promtail/*
```

---

## 配置位置（要改时去哪里）

| 想改 | 文件 | 改后 |
|------|------|------|
| 保留时长（默认 30 天） | `env/loki/loki-config.yaml` → `limits_config.retention_period` | `podman restart loki` |
| 监控哪些容器 | `env/loki/promtail-config.yaml` → `scrape_configs.docker_sd_configs.filters` | `podman restart promtail` |
| Pipeline 解析字段 | `env/loki/promtail-config.yaml` → `pipeline_stages.json.expressions` | 同上 |
| Grafana datasource | `env/grafana/provisioning/datasources/datasources.yml` | `podman restart grafana` |

---

## 已知局限

- 单实例 Loki，无 HA；底层 filesystem 存储 ≤ 50GB 量级够用，超出请上 boltdb-shipper + S3/MinIO 后端
- `level` label 用 go-zero 输出的字符串，**go-zero 没有 warn 级别**（只有 info/error/stat/severe/slow）
- APISIX 访问日志当前**没**进 Loki（容器 stdout 写的是 nginx access log 格式而非 JSON）。如需结构化，应在 APISIX 配 `http-logger` 插件推 Loki，或加专用 Promtail pipeline
- mall-fe 容器是 nginx，access log 在 Loki 里裸文本，搜不动 path/status code；要正经分析得换 JSON access log 格式
