# yw-mall-docs

yw-mall 电商平台**唯一**的文档汇总仓库 — 产品需求、技术规划、工作日志、架构决策、运维手册、复盘记录都汇集在这里。

> 业务代码仓库（yw-mall / yw-mall-admin / yw-mall-admin-fe / yw-mall-fe / yw-mall-deploy）原则上不再单独维护 `docs/` 目录，避免双写同步问题。

---

## 目录结构

```
yw-mall-docs/
├── prd/                  # 产品需求文档
├── planning/             # 路线图、Sprint 计划、差距分析
├── feat/                 # 跨 sprint 特性方案 / 架构改造专题（含时间预算 + 跟踪表）
├── daily/                # 每日工作日志（按日期归档，跨 repo 合并）
├── architecture/         # 架构决策记录 (ADR) / 系统图 / 数据流图
├── bugs/                 # 按根因分类的 bug 案例库（避免同类问题重复犯）
├── api/                  # 统一 API 规范（OpenAPI / proto 参考）
├── runbooks/             # 运维手册 / 故障预案 / 上下线流程
└── retrospectives/       # 项目复盘 / Sprint 回顾
```

---

## 本地走查入口与测试账号

启动前置：`cd yw-mall && ./start.sh start`（拉起 16 个 Go 服务）+
FE 项目按需 `pnpm dev` / `pnpm run build`。

> **FE dev 端口需要手动启**：5173/5174/5175 都是各 FE 项目的 vite dev server，
> 不会自动起。要走查哪个就 `cd <fe-dir> && pnpm dev`。
> 容器化端口 18080/18082/18083 由 `yw-mall-deploy/compose.yml` 起。

### Web 入口（浏览器）

| 端 | URL | 说明 | 默认账号 |
|---|---|---|---|
| **C 端 H5（用户购物）** | http://localhost:5173 | uni-app vite dev (proxy /api → :18888) | `alice` / `alice123`<br>`bob` / `bob123`<br>`demo` / `demo123` |
| C 端 H5（容器化） | http://localhost:18080 | yw-mall-deploy 起的 nginx | 同上 |
| **商家工作台 SPA** | http://localhost:5175 | Vue 3 vite dev (proxy → :18999) | `alice` / `alice123` (shop 1 owner)<br>`bob` / `bob123` (shop 3 owner) |
| 商家工作台（容器化） | http://localhost:18083 | yw-mall-deploy 起的 nginx | 同上 |
| **Admin 后台** | http://localhost:18082 | Vue 3 + Element Plus (compose 内) | `admin` / `admin123` |
| Admin FE dev | http://localhost:5174 | vite dev (proxy → :18999) | 同上 |
| **Swagger UI** | http://localhost:18081 | C 端 + Admin API 文档双切换 | — |
| MinIO Console | http://localhost:9001 | 文件桶管理 | `admin` / `admin123` |

### API 网关端口（curl 直调）

| 服务 | URL | 用途 |
|---|---|---|
| C 端 API | http://localhost:18888 | mall-api（/api/auth/login 等） |
| Admin + Merchant API | http://localhost:18999 | yw-mall-admin（/admin/v1 + /merchant/v1） |

### 走查账号细节

| 账号 | 密码 | 角色 | 可走查内容 |
|---|---|---|---|
| `alice` | `alice123` | C 端用户（user.id=1）+ shop 1 owner | 全场景：下单 / 支付 / 退款 / 商家工作台所有模块 |
| `bob` | `bob123` | C 端用户（user.id=3）+ shop 3 owner | 多店铺隔离、跨店购物车验证 |
| `demo` | `demo123` | C 端用户（user.id=5） | 纯用户视角，无商家权限 |
| `bob48971` | （邀请注册分配） | shop 1 finance（status=0 未激活） | M1 邀请链接 + 角色权限测试 |
| `admin` | `admin123` | 平台管理员 | 提现审批 / 仲裁 / 全局数据看板 |

### 高频走查链路

1. **下单 + 优惠** (Phase 1 联动):
   ```
   alice 商家：创建活动「满 199 减 30」+ 创建券模板 → C 端 alice 领券
   → 加购下单 ¥250 + 用券 → /api/cart/calc-price → mock-pay
   → 看 /api/order/detail/:id 优惠明细
   ```
2. **商家退款流转** (M4): C 端 submit refund → 商家 approve / 拒绝 / 仲裁
3. **退款 3 类**: 仅退款 / 退货退款 / 换货 完整状态机
4. **多店并发 Dashboard**: alice + bob 同时调 `/merchant/v1/dashboard`
5. **店铺装修 → C 端店铺主页**: 商家编辑 → C 端 `/api/shop/detail/:id`

### 常用查询

```bash
# 服务运行状态
cd yw-mall && ./start.sh status

# 重启某个服务
cd yw-mall && ./start.sh restart

# Swagger 入口可一站式看所有 API
open http://localhost:18081
```

---

## 关键文档索引

### 产品需求

- [`prd/admin-merchant-portal.md`](prd/admin-merchant-portal.md) — 后台管理 PRD（管理员 + 店家门户）

### 规划与差距分析

- [`planning/platform_completeness.md`](planning/platform_completeness.md) — 整个电商平台完整度评估（C 端 + B 端 + 平台 + 基础设施）
- [`planning/commercial_readiness_gap.md`](planning/commercial_readiness_gap.md) — 后台专项商用差距分析（P3/P4/P5 路线图）
- [`planning/mvp_sprint_plan.md`](planning/mvp_sprint_plan.md) — MVP 12-Sprint 详细计划（6 个月、8 人团队）
- [`planning/q1q2_tracking.md`](planning/q1q2_tracking.md) — Sprint Story 跟踪表 + 角色甘特图

### 特性方案

- [`feat/login-revamp.md`](feat/login-revamp.md) — 登录体系改造（Opaque Token + Redis Session，参考淘宝/京东）— P0 / P1 / P2 / P3 时间预算 + 跟踪表

### 架构 / ADR

- [`architecture/read-write-split-option-c-gtid.md`](architecture/read-write-split-option-c-gtid.md) — 读写分离方案 C：GTID 会话一致性（生产级正解）— 原理 / 7 人天预算 / 收益对比 A/B/C/D
- [`architecture/replication-lag-idle-investigation.md`](architecture/replication-lag-idle-investigation.md) — Idle 系统下 ~1s 复制延迟根因调查（实测数据 + 拓扑分析 + 修复路径）

### Bug 分类记录

- [`bugs/`](bugs/README.md) — 按根因类型归档（前后端契约 / 基础设施 / 数据库复制 / schema drift / placeholder）— 不按时间，看同类先例先翻这里

### 运维手册

- [`runbooks/logs.md`](runbooks/logs.md) — 日志查询 (Grafana Loki / podman logs / Loki HTTP API 三档速查 + 典型故障复盘场景)

### 工作日志

- [`daily/2026-05-10.md`](daily/2026-05-10.md) — etcd 配置中心修复 + rebuild.sh
- [`daily/2026-05-12.md`](daily/2026-05-12.md) — P2 后台管理 9 个 Story 全部完成 + admin-fe 14 模块

---

## 写作约定

### 通用

- 一律使用 **Markdown**（GitHub Flavored Markdown）
- 文件命名：英文小写 + 连字符（`commercial-readiness-gap.md` 这种），**不要**空格/驼峰/中文
- 日期格式：ISO 8601 `YYYY-MM-DD`（`2026-05-12`）
- 表格 / 代码块 / 链接 比纯文本更受欢迎
- 头部 metadata 写明：版本、日期、适用范围、关联文档

### 工作日志（daily/）

每个工作日**一个文件**（`YYYY-MM-DD.md`），跨 repo 的工作合并：

```markdown
# 工作日志 · YYYY-MM-DD

> 涉及 repo：repo-a · repo-b · ...

## 📦 repo-a
（该 repo 当日内容）

## 🚢 repo-b
（该 repo 当日内容）
```

主题区分用 emoji + repo 名作为二级标题。这样的好处是：日历跳转 → 单页看到当天所有 repo 的工作。

### 规划文档（planning/）

包含**估算工时**或**优先级矩阵**时，请保持单位一致：
- 工时单位：**人天**（≈ 1 人 1 个工作日）
- 月度估算：**人月** = 22 人天
- 优先级：P0（必须立即做）/ P1（本季度）/ P2（下季度）/ P3-P5（参见 `commercial_readiness_gap.md` §2-4）

### 架构决策（architecture/）

按 [ADR](https://adr.github.io/) 格式：

```markdown
# ADR-NNNN: 标题

- 日期：YYYY-MM-DD
- 状态：proposed / accepted / superseded
- 决策者：@xxx

## 背景

## 决策

## 后果（正面 + 负面）

## 备选方案

## 关联
```

---

## 关联仓库

| Repo | 内容 |
|---|---|
| [yw-mall](https://github.com/carter4cn/yw-mall) | 13 个 mall-*-rpc 后端服务 + mall-api C 端网关 + mall-activity-async-worker |
| [yw-mall-admin](https://github.com/carter4cn/yw-mall-admin) | 后台管理 HTTP API 网关（admin + merchant 双路由） |
| [yw-mall-admin-fe](https://github.com/carter4cn/yw-mall-admin-fe) | 后台前端（admin/ + merchant/ 两个 SPA） |
| yw-mall-fe | C 端商城前端（uni-app H5） |
| [yw-mall-deploy](https://github.com/carter4cn/yw-mall-deploy) | podman-compose 编排 + 启动脚本 + DDL bootstrap |

---

## 贡献

直接 `main` 分支提交（小团队 / 个人项目模式）。如未来引入 review 流程：

1. 新建 `docs/feat-xxx` 分支
2. 完成文档 → PR
3. 自动 trigger markdown lint（暂未配，未来加 GitHub Actions）

不可逆变更（删 / 大规模重命名）走 PR + review。
