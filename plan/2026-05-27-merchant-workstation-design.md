# 商家工作台 (Merchant Workstation) — 设计方案

> **文档类型**: Design Spec (S7 商家工作台)
> **状态**: Draft v1 — 待审查方案可行性
> **作者**: Carter + Claude Opus 4.7
> **日期**: 2026-05-27
> **关联**: `planning/q1q2_tracking.md` S7、`planning/platform_completeness.md`、`feat/login-revamp.md`

---

## 1. 背景

### 1.1 当前痛点

平台已经跑通 C 端注册/登录/购物/下单/支付/退款全链路，**店家侧完全没有 Web 工作台**：

| 能力 | 现状 |
|---|---|
| 商家登录 | ❌ 无 Web 入口（仅 admin-api 有公共 `/merchant/v1/login` 路由占位） |
| 上传商品 | ❌ 仅 RPC 层有 `createProduct`，没 HTTP 暴露，更没 FE |
| 库存管理 | ❌ 同上（`setProductStock` RPC 已就绪） |
| 订单发货 | ❌ 同上（`markShipped` RPC 已就绪） |
| 退款处理 | ❌ 同上（`merchantHandleRefund` 等 4 个 RPC 已就绪） |
| 财务对账 | ❌ 同上（`getMerchantWallet`、`listLedger`、`createWithdrawal` 已就绪） |
| 物流模板 | ❌ 同上 |
| 评价管理 | ❌ 同上 |

> "yw-mall-admin-fe/merchant/ 目录是占位（一行 README），店家无法登录 web 后台 —— 所有店家侧只能 RPC/curl 测试" — `planning/platform_completeness.md`

### 1.2 已有的资产

**后端 RPC 大部分齐全**（约 30+ method 已实现）：

| RPC 服务 | 商家相关已实现 method |
|---|---|
| mall-shop-rpc | `applyShop` / `createShop` / `getShopByOwnerId` / `updateShop` / `submitLevelApplication` / `submitShopLifecycleRequest` |
| mall-product-rpc | `merchantListProducts` / `createProduct` / `updateProduct` / `setProductStatus` / `setProductStock` / `lockSku` / `unlockSku` |
| mall-order-rpc | `listShopOrders` / `getShopOrder` / `markShipped` / `shipOrder` / `listShopRefundRequests` / `merchantHandleRefund` / `merchantInspectReturn` / `merchantRejectRefund` / `merchantShipExchange` |
| mall-payment-rpc | `getMerchantWallet` / `getShopLedgerSummary` / `listLedger` / `createWithdrawal` / `listWithdrawals` / `listBillRecords` |
| mall-logistics-rpc | `createShipment` / `getShipment` / `listShipmentsByOrder` / freight templates CRUD / `injectTrack` |
| mall-review-rpc | `merchantListReviews` / `requestDeleteReview` |

**HTTP 网关已有 ~15 个 `/merchant/v1/*` 路由**（见 `yw-mall-admin/internal/handler/routes.go:123-180`）：商品 CRUD、订单列表+发货+拒退、评价、投诉、活动浏览。

### 1.3 缺口

| 层 | 缺口 |
|---|---|
| **数据模型** | merchant_staff RBAC、staff_invitation、shop_decoration（装修配置）3 张新表 |
| **后端 RPC** | 仅缺 `aggregateMerchantDashboard`（聚合 4 服务）+ 邀请 SMS/邮件发送 + staff CRUD |
| **后端 HTTP** | `/merchant/v1/*` 缺约 15 个端点：refund/return 流程、wallet/withdrawal/ledger、freight template、staff CRUD、invitation、dashboard、shop decoration |
| **前端 SPA** | 完全空白，需要从零搭 10 模块 |

---

## 2. 目标与非目标

### 2.1 In Scope (本方案覆盖)

1. **商家登录体系**：店主用 owner_id 登录；员工用子账号登录；JWT/Session 复用 L0+L0.5 opaque token 体系，Role 字段统一为 `"merchant"`
2. **员工子账号 RBAC**：4 个预定义角色（店主/客服/仓管/财务），通过 SMS/邮箱邀请链接添加成员；店主可踢除/改角色
3. **商品管理**：多 SKU 多规格上传，矩阵生成器；库存批量调整；上下架；图片上传到 MinIO
4. **订单管理**：列表+筛选、详情、单条发货（选物流公司+面单号）、批量发货 CSV 导入（P1 可选）
5. **退款处理**：仅退款/退货退款/换货 3 类工作流；同意/拒绝/验货/换货发货
6. **财务**：钱包余额展示、收支流水分页、提现申请+列表
7. **店铺信息**：商家自助编辑店铺名/Logo/banner/公告/联系方式
8. **店铺装修**：banner 轮播图 + 公告 + 推荐商品 3 个简单模块（不做拖拽编辑器）
9. **物流模板**：列表 + CRUD（按地区计费）
10. **数据看板**：4 核心指标卡片（今日营业额/今日订单/待发货/待退款）+ 钱包余额 + 跳转入口
11. **评价管理**：列表 + 申请删除
12. **活动报名**：列表浏览 + 报名按钮

### 2.2 Out of Scope (后续单独排期)

- ❌ IM 客服系统（platform_completeness 已标为 M4-M5 单独项）
- ❌ 数据分析（7 日趋势图、商品销售排行、客户画像）
- ❌ 拖拽式店铺装修编辑器
- ❌ 批量发货 CSV 导入工具（如需保留，纳入 M3 末尾 P1 选项）
- ❌ 营销工具：优惠券、限时折扣、满减自助（后台 admin 已有，商家自配是 P2）
- ❌ 数据导出（订单/账单 Excel 导出）
- ❌ 多店铺管理（一个 user 多个 shop）
- ❌ 移动端 H5 商家工作台（先 Web）

---

## 3. 已收敛决策（brainstorm 输出）

| # | 决策 | 选择 | 理由 |
|---|---|---|---|
| 1 | Scope | S7 完整工作台（12-15 天） | 用户明确选择，能闭环 |
| 2 | FE 脚手架 | 复制 `admin/`（Vue 3 + Element Plus + Vite + Pinia） | 一致性、复用京东风设计令牌 |
| 3 | 账号模型 | 店主 + 4 角色员工 RBAC | 真实使用场景需要 |
| 4 | 员工加入 | SMS/邮箱邀请链接 | 复用 sendVerifyCode 基建 |
| 5 | 财务颗粒度 | 钱包余额 + 流水 + 提现（基本盘） | 后端 RPC 已完整对应 |
| 6 | 商品规格 | 多 SKU 多规格 + 属性矩阵 | sku 表已存在 |
| 7 | Dashboard | 4 核心指标 + 待办提醒 | 新增 1 个聚合 RPC |
| 8 | 实施路径 | 方案 B 垂直切片 M1-M6 | 跟 S1-S5 节奏一致 |

### 3.1 默认值（未单独问，可在 spec review 阶段挑战）

| 项 | 默认 |
|---|---|
| 退款处理 UX | 列表 + 详情页 + action 按钮（同意/拒绝/验货） |
| 店铺装修模块 | banner 轮播 + 公告 + 推荐商品 3 块，简单表单 |
| 物流模板 | 列表 + CRUD（按省份/地区/重量计费） |
| 评价管理 | 列表 + 申请删除（后端已有） |
| 活动报名 | 列表浏览 + 报名按钮 |
| Token TTL | 复用 c-side / admin：30 分钟 access + 7 天 refresh |
| Role 字段值 | `"merchant"`（区别于 `"user"`/`"admin"`，确保 L1.6 force-logout 过滤正确） |
| Dashboard 待办 | 「N 单待发货」+「N 笔待审退款」可点击跳转 |
| 端口 / 容器 | 宿主端口 18083，沿用 admin 同款 Dockerfile + nginx + apisix 透传 |
| 邀请链接有效期 | 7 天 |
| 单店员工上限 | 20 人（含店主） |

---

## 4. 系统架构

### 4.1 总览

```
                          ┌─────────────────────────────┐
                          │  yw-mall-admin-fe/merchant/  │
                          │  Vue3 + Element Plus + Vite  │
                          │  http://localhost:18083      │
                          └──────────────┬──────────────┘
                                         │ /merchant/v1/*
                                         ▼
                          ┌─────────────────────────────┐
                          │   yw-mall-admin (admin-api)  │
                          │   :18999  /merchant/v1/*     │
                          │   MerchantAuth middleware    │
                          │   OpLog middleware           │
                          └──────────────┬──────────────┘
                                         │ gRPC
            ┌────────────┬────────────┬──┴────────┬──────────┬──────────┐
            ▼            ▼            ▼           ▼          ▼          ▼
       user-rpc     shop-rpc    product-rpc  order-rpc  payment-rpc logistics-rpc
       (session)    (shop+staff) (prod+sku)  (order+refund)(wallet)  (shipment)
```

### 4.2 调用链举例（"商家给一单发货"）

```
店主点击"发货"→ FE POST /merchant/v1/orders/123/ship {company, trackingNo}
              → admin-api MerchantAuth 校验 session.role=merchant + shop_id
              → admin-api logic 检查 staff.permissions 包含 order.ship
              → order-rpc.MarkShipped(orderId=123, shopId=session.shopId)
              → order-rpc 校验订单归属、状态机：PAID → SHIPPED
              → logistics-rpc.CreateShipment(orderId=123, company, trackingNo)
              → 返回 200 {ok:true}
              → admin-api OpLog middleware 记录 (uid=staff.id, action=ship, target=order:123)
```

### 4.3 安全模型

#### 4.3.1 Token 与 Role
- 复用 L0/L0.5 opaque token + Redis session
- Role = `"merchant"`（与 `"user"`/`"admin"` 互斥）
- sessionPayload 新增 `staffId int64`（=自己 user.id）+ `shopId int64`
- 共享 user_sessions:{uid} set 不冲突：destroyUserSessionsByRole 按 Role 过滤已就绪（L1.6 已交付）

#### 4.3.2 中间件链
- `MerchantAuth`（新）：
  1. Bearer token → user-rpc.ValidateSession
  2. 校验 sess.Role == "merchant"
  3. 查询 merchant_staff WHERE user_id=sess.Uid → 拿 shopId + staffRole + permissions
  4. 校验 staff.status == 1（未冻结）
  5. ctx 注入 (shopId, staffId, staffRole, permissions)
- `OpLog`（复用 admin 同款）：每个写操作记录 op_log
- 写操作路由可加 `PermGate("order.ship")` 装饰器做权限二次校验

#### 4.3.3 RBAC 角色与权限矩阵

| 角色 | shop 信息 | 商品 | 订单 | 退款 | 财务 | 装修 | 员工 | 物流模板 |
|---|---|---|---|---|---|---|---|---|
| **店主 owner** | RW | RW | RW + 发货 | RW + 仲裁 | RW + 提现 | RW | RW（加减员工） | RW |
| **客服 service** | R | R | R | RW（同意/拒绝） | — | — | R | — |
| **仓管 warehouse** | R | R（改库存） | RW + 发货 | R（验货）| — | — | R | R |
| **财务 finance** | R | — | R | — | RW + 提现 | — | R | — |

权限码格式：`shop.read` / `shop.write` / `product.write` / `order.ship` / `refund.handle` / `finance.write` / `staff.write` / `decoration.write` / `freight.write`

owner 默认拥有所有权限（`"*"`），其他角色拥有固定子集（hardcode 在 `merchant_role_perms.go` 里）。

---

## 5. 数据模型变更

### 5.1 新表

#### `merchant_staff`（员工绑定关系）
```sql
CREATE TABLE merchant_staff (
  id              BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  shop_id         BIGINT UNSIGNED NOT NULL,
  user_id         BIGINT UNSIGNED NOT NULL,          -- FK user.id
  role            VARCHAR(32) NOT NULL,              -- owner|service|warehouse|finance
  status          TINYINT NOT NULL DEFAULT 1,        -- 1=active, 0=disabled
  invited_by      BIGINT UNSIGNED NOT NULL,          -- 谁邀请的（owner staff.id）
  joined_at       BIGINT NOT NULL,                   -- unix
  create_time     DATETIME NOT NULL,
  update_time     DATETIME NOT NULL,
  UNIQUE KEY uk_shop_user (shop_id, user_id),
  KEY idx_user (user_id),
  KEY idx_shop (shop_id, status)
);
```

每个 shop 一定有一条 role=owner 的 staff 记录，在 `createShop` 时自动插入（一次性迁移脚本补齐历史 shop）。

#### `merchant_staff_invitation`（邀请单）
```sql
CREATE TABLE merchant_staff_invitation (
  id              BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  shop_id         BIGINT UNSIGNED NOT NULL,
  invited_by      BIGINT UNSIGNED NOT NULL,          -- staff.id 谁发起
  target_phone    VARCHAR(20),                       -- phone/email 二选一
  target_email    VARCHAR(255),
  role            VARCHAR(32) NOT NULL,
  invitation_code VARCHAR(64) NOT NULL UNIQUE,       -- 邀请链接 token（URL-safe base64）
  status          TINYINT NOT NULL DEFAULT 0,        -- 0=pending, 1=accepted, 2=expired, 3=revoked
  expires_at      BIGINT NOT NULL,                   -- 默认 +7 天
  accepted_by     BIGINT UNSIGNED,                   -- user.id of acceptor
  accepted_at     BIGINT,
  create_time     DATETIME NOT NULL,
  KEY idx_shop (shop_id),
  KEY idx_status (status, expires_at)
);
```

邀请链接：`https://merchant.yw-mall.com/accept-invite?code=<invitation_code>`
接受流程：
1. 被邀请人点击链接 → 跳到 FE
2. FE 调 `/merchant/v1/invitations/accept {invitationCode}`
3. 后端校验 code + 未过期 + 未使用
4. 被邀请人**必须已是 c-user**（未注册则跳注册页带 redirect）
5. 校验 phone/email 与邀请的 target 一致（防代领）
6. INSERT merchant_staff (shop_id, user_id=acceptor, role, status=1)
7. UPDATE invitation status=1, accepted_by, accepted_at
8. 调 user-rpc.DestroyAllUserSessionsByRole(user_id, "user") 不需要，但要让接受者下次登录时进入商家工作台路由（FE 自己做：登录后查 staff → 有则进 /merchant SPA，否则进 c 端）

#### `shop_decoration`（店铺装修配置）
```sql
CREATE TABLE shop_decoration (
  id              BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  shop_id         BIGINT UNSIGNED NOT NULL UNIQUE,
  banners         TEXT,                              -- JSON: [{image,link,sort}]
  announcement    VARCHAR(500),                      -- 公告纯文本
  featured_pids   VARCHAR(500),                      -- JSON: [pid1, pid2, ...] 最多 10 个
  update_time     DATETIME NOT NULL
);
```

C 端店铺主页（`/shop/detail.vue`）需要新增逻辑读这张表渲染（C 端改动单独排期，本 sprint 只做商家端写）。

### 5.2 表迁移

文件：`mall-user-rpc/sql/merchant_workstation.sql`（与 user-rpc 同库，因为 staff 表跟 user.id 强绑）
注入到 `start.sh SPRINT_MIGRATIONS`。

历史 shop 数据回填：写一次性脚本 `cmd/backfill_merchant_owner/main.go`，对每条 shop 自动 INSERT merchant_staff (shop_id, user_id=shop.owner_id, role=owner, invited_by=自身)。

---

## 6. 后端 HTTP 网关变更

### 6.1 新增 `/merchant/v1/*` 端点

**Staff & Invitation 模块（M1）**
```
POST   /merchant/v1/staff/invitations         发送邀请（仅 owner）
GET    /merchant/v1/staff/invitations         待处理邀请列表（仅 owner）
POST   /merchant/v1/staff/invitations/:id/revoke  撤销邀请
POST   /merchant/v1/invitations/accept        接受邀请（公开,登录后可调）
GET    /merchant/v1/staff                     员工列表（all）
POST   /merchant/v1/staff/:id/role            改员工角色（仅 owner）
POST   /merchant/v1/staff/:id/disable         冻结员工（仅 owner）
```

**Product & Inventory（M2）— 已有的不动，仅补**
```
POST   /merchant/v1/products/:id/skus         批量 upsert SKU
POST   /merchant/v1/products/:id/skus/:skuId/stock  调单 SKU 库存
POST   /merchant/v1/upload                    图片上传 → MinIO，返 URL
```

**Order & Refund（M3-M4）— 补**
```
GET    /merchant/v1/refunds                   退款工单列表
GET    /merchant/v1/refunds/:id               退款详情
POST   /merchant/v1/refunds/:id/agree         同意退款（含验货后退款）
POST   /merchant/v1/refunds/:id/inspect       验货（退货退款）
POST   /merchant/v1/refunds/:id/ship-exchange 换货发货
```

**Wallet & Finance（M5）— 补**
```
GET    /merchant/v1/wallet                    钱包余额 + summary
GET    /merchant/v1/wallet/ledger             收支流水（分页）
GET    /merchant/v1/withdrawals               提现记录
POST   /merchant/v1/withdrawals               发起提现
GET    /merchant/v1/bills                     账单记录（按月）
```

**Freight Template（M3）— 补**
```
GET    /merchant/v1/freight-templates         模板列表
POST   /merchant/v1/freight-templates         创建
PUT    /merchant/v1/freight-templates/:id     更新
DELETE /merchant/v1/freight-templates/:id     删除
```

**Shop Decoration（M6）— 补**
```
GET    /merchant/v1/decoration                获取装修配置
PUT    /merchant/v1/decoration                更新装修配置
```

**Dashboard（M6）— 补**
```
GET    /merchant/v1/dashboard                 聚合接口
```

### 6.2 新增聚合 RPC

`mall-shop-rpc` 加 `GetMerchantDashboard(shopId)` 方法，串行调：
- order-rpc：今日订单数、今日营业额、待发货数
- order-rpc：待审退款数
- payment-rpc：钱包余额 + T+3 未到账金额

返回：
```go
type MerchantDashboardResp struct {
  TodayRevenue       int64  // 分
  TodayOrders        int32
  PendingShipments   int32
  PendingRefunds     int32
  WalletAvailable    int64
  WalletPending      int64
}
```

**为什么放 shop-rpc 不放 admin-api logic 层？**
- 聚合逻辑需要并发调 4 个服务，统一一处用 errgroup 比 admin-api logic 一处一处串调省 60% 延迟
- shop-rpc 已经聚合了店铺相关业务，是天然的"商家域" anchor

---

## 7. 前端 SPA 结构

### 7.1 目录结构（复制 admin/ 改造）

```
yw-mall-admin-fe/merchant/
  src/
    api/
      request.ts          基础 axios + 401 单飞 refresh（直接复制 admin）
      auth.ts             merchantLogin / refresh / logout
      shop.ts             getMyShop / updateMyShop / decoration
      product.ts          CRUD + SKU batch
      order.ts            list / get / ship
      refund.ts           list / detail / agree / inspect / ship-exchange
      wallet.ts           balance / ledger / withdrawals
      staff.ts            list / invitations / accept / role / disable
      freight.ts          template CRUD
      dashboard.ts        get
      upload.ts           image upload
    stores/
      session.ts          accessToken / refreshToken / csrf / shopId / staffRole / perms
      shop.ts             my shop cache
    router/
      index.ts            vue-router 权限路由（meta.perms 与 staff.permissions 交集）
    layouts/
      MerchantLayout.vue  左侧 nav + 顶部 user dropdown + 内容区
    views/
      login/index.vue           京东风登录页（复刻 admin 京东风）
      invite/accept.vue         接受邀请页
      dashboard/index.vue       仪表盘
      shop/info.vue             店铺信息编辑
      shop/decoration.vue       装修
      product/list.vue          商品列表
      product/edit.vue          商品上传/编辑（含 SKU 矩阵生成器）
      order/list.vue            订单列表（多 tab：全部/待发货/已发货/已完成）
      order/detail.vue          订单详情 + 发货 modal
      refund/list.vue           退款列表
      refund/detail.vue         退款详情（带 stepper + action 按钮）
      wallet/index.vue          钱包余额 + ledger 流水
      wallet/withdraw.vue       提现
      staff/list.vue            员工列表
      staff/invite.vue          邀请员工 modal
      freight/list.vue          物流模板
      review/list.vue           评价管理
      activity/list.vue         活动列表
    composables/
      usePermission.ts          v-perm 指令 + can(perm) helper
      useFormatMoney.ts         分 → 元，¥1,234.56
    types/api.ts                所有 API 响应类型
```

### 7.2 路由权限

复用 admin 同款做法：`meta.perms: ['order.ship']`，路由进入前 `staffPermissions ∩ meta.perms` 非空才放行。owner 拥有 `"*"`，相当于全开。

### 7.3 设计令牌（继承 admin 京东风）

主色：红色渐变 `#e1251b → #ff4b4b`
背景：`#f5f5f7`
卡片：`border-radius: 16px; box-shadow: 0 12px 32px rgba(0,0,0,0.06)`
输入框：灰底 `#f5f5f7` + 12px 圆角 + Element prefix-icon

---

## 8. 实施里程碑

| 里程碑 | 工时 | 交付内容 | 可演示 |
|---|---|---|---|
| **M1 商家登录+RBAC** | 3d | merchant_staff/invitation 表 + RBAC 中间件 + 邀请 SMS/邮件 + Staff CRUD HTTP + FE 脚手架 + 登录页 + 邀请接受页 + 员工列表+邀请页 | 店主登录、邀请员工、员工接受邀请、员工登录看到自己店铺 |
| **M2 商品** | 3d | SKU upsert HTTP + 图片上传 HTTP + FE 商品列表+编辑（含 SKU 矩阵生成器） | 上传带规格商品、调库存、上下架 |
| **M3 订单+物流模板** | 3d | 退款列表/详情/操作 HTTP + freight template CRUD HTTP + FE 订单列表+详情+发货 modal + freight 模板页 | 看订单、单条发货带物流模板 |
| **M4 退款** | 3d | 上面 HTTP 完整接通 + FE 退款列表+详情 stepper + 3 类工单 action | 退款 / 退货退款 / 换货 全流程操作 |
| **M5 财务+评价+活动+投诉** | 2d | wallet/ledger/withdrawal HTTP + FE 钱包页+提现页+评价/活动/投诉列表 | 看余额、看流水、申请提现、申请删评、报名活动 |
| **M6 Dashboard+装修+收尾** | 2d | `aggregateMerchantDashboard` RPC + 装修 HTTP + FE 仪表盘+装修页 + Dockerfile + nginx + compose 18083 + e2e | 商家从登录到首屏全链路 + 容器化部署 |

**总工时**：16 天（含 1 天 buffer for buffer），跟原估算 12-15 天稍超，因为加了 RBAC（之前 brainstorm 选了"店主+员工"额外 +2-3d）。

每个 M 收尾跟 S1-S5 一样独立 commit 系列，最终上 docs/daily 日志总结。

---

## 9. 风险与缓解

| 风险 | 概率 | 影响 | 缓解 |
|---|---|---|---|
| RBAC 权限矩阵漏判 / 越权 | 中 | 高 | 中间件 + 路由 meta.perms 双层；写 e2e 覆盖跨角色（仓管不能提现 / 客服不能改库存）|
| SKU 矩阵生成器 UI 复杂 | 中 | 中 | 参考淘宝商家工作台拍照仿，先 MVP 出，迭代 |
| 邀请链接被钓鱼 | 中 | 中 | 链接带 invitation_code（不可猜）+ 7 天过期 + accept 时校验 target_phone/email 与登录账号一致 |
| 商家提现风控不够 | 高 | 高 | 本期只做提现申请，**真实打款留 placeholder**（payment-rpc.adminHandleWithdrawal 需要 admin 人工审批）；记 op_log |
| 多 SKU 商品兼容老数据 | 中 | 中 | createProduct 路径已经支持 SKU，老 SPU 单 SKU 数据 "默认 SKU" 自动生成兜底 |
| dashboard 聚合 RPC 延迟 | 低 | 中 | 用 errgroup 并发调 4 服务；任一失败用 fallback 0 + 日志 warn 不阻塞 |
| 店主无法登录新工作台时遗失店铺访问 | 低 | 高 | M1 优先级最高；上线前必须 e2e 跑通"老 shop_app 商家用 owner_id 登录"路径 |
| FE 路由进 c 端 vs merchant 错判 | 中 | 中 | 登录后调 `/api/user/me` 查 staff 表，有则进 merchant；无则进 c 端；登录响应里加 `hasMerchantRole bool` 字段帮 FE 决策 |

---

## 10. 与 L1.6 改密下线的交互

L1.6 已交付的 `destroyUserSessionsByRole` 按 sessionPayload.Role 过滤。本次新增 Role=`"merchant"`：
- 商家改密 → 调 helper 时传 role="merchant" → 只清商家自己的 session，不影响同一 user_id 的 c 端 session
- 这正是设计这个 helper 时考虑到的扩展性 — **无须改动**，传新 role string 即可

需要在哪里加 ChangePassword 调用？
- 商家在 `/merchant/v1/security/password` 改密 → admin-api 调 user-rpc.ChangePassword(SubjectType=3?  or SubjectType=1, with role="merchant")

**待决**：ChangePassword 的 SubjectType 是否新增 3=merchant，还是保持 1=user（因为 staff 其实就是 user 表里的人）？建议保留 SubjectType=1，但 destroyUserSessionsByRole 调用时传 `role="merchant"` —— 这两套维度独立。

---

## 11. 验收标准

**M1 通过**：
- 店主 alice 用密码登录 merchant SPA，进入 dashboard 占位页（数字暂时 mock）
- 店主邀请 bob（c-user, 已注册）→ bob 收 mock SMS → bob 点链接 → 加入店铺 role=warehouse
- bob 登录 → 看到同一店铺的工作台，但导航没有"提现"项（perm 缺失）
- 店主把 bob role 改 finance → bob 退出重新登录 → 看到"提现"，看不到"商品库存"
- e2e: 跨角色 403 强校验（bob warehouse 调 /merchant/v1/withdrawals → 403）

**M2-M6 各自的验收**在每个里程碑结尾 daily 里补，跟 L1.6 同款"实测证据"模式。

**整体上线门槛**：
- alice 完整跑完"登录 → 上传商品（多 SKU） → 顾客下单（C 端） → 仓管发货 → 顾客确认收货 → 店主看到入账"
- alice 完整跑完"顾客退款 → 客服同意 → 仓管验货 → 退款入账"
- 全程不需要任何 curl/SQL/admin 介入

---

## 12. 开放问题（spec review 阶段欢迎挑战）

| ID | 问题 | 建议默认 |
|---|---|---|
| OQ1 | 提现是否本期真打款？ | 否，本期仅"申请"。admin 端审批+模拟打款 |
| OQ2 | 多 shop 一个店主？ | 否，1 user = 0 or 1 shop。多店是 P2 |
| OQ3 | C 端店铺主页是否本期接装修配置？ | 否，本期只商家端写，C 端读单独 sprint |
| OQ4 | 邀请非注册用户怎么办？ | accept 页跳 c 端注册 → 注册完自动回 invite token 接受 |
| OQ5 | SubjectType 是否新增 3=merchant？ | 否，保留 1=user，仅 role 字符串区分 |
| OQ6 | 商家是否有独立的注册入口？ | 否，先 c 端注册 → 后申请 shop → 自动成为该 shop 的 owner |
| OQ7 | 文件上传走 admin-api 还是 mall-api？ | admin-api 加 `/merchant/v1/upload` 单独端点（避免引 c-side cookie） |
| OQ8 | 装修页改完 C 端何时生效？ | 立即（无审批），刷新即生效。若担心审核，加 admin 审批是 P1 |
| OQ9 | 物流模板是 shop 级还是平台级？ | shop 级（每个店铺独立模板，店主自管） |
| OQ10 | 4 个固定角色是否够？ | 够首期。自定义角色 + 自定义权限是 P2 |

---

## 13. 下一步

1. **本 spec 用户审查** — 期望反馈：scope、风险、OQ 1-10、M1-M6 顺序是否 OK
2. **审查通过后** → 调用 `superpowers:writing-plans` 把 design 翻译成 task-by-task 实施计划（保存在 `feat/2026-05-27-merchant-workstation-plan.md`）
3. 实施按 M1 → M6 顺序，每个 M 独立 commit 系列 + daily 日志收尾
4. 整体收尾后回写 `planning/q1q2_tracking.md` 把 S7.1-S7.4 全部标 ✅

---

## 14. 关联文档

- `planning/q1q2_tracking.md` — Sprint 7 商家工作台跟踪
- `planning/platform_completeness.md` — 平台完整度评估
- `feat/login-revamp.md` — Opaque token + Redis session + L1.6 force-logout（本方案直接复用）
- `feat/2026-05-21-multi-identifier-register-login-plan.md` — sendVerifyCode / cryptox 基建（本方案邀请链接复用）
- `daily/2026-05-14.md` — admin-api Sprint 4 安全 + RBAC 落地（参考实现）
- `daily/2026-05-25.md` — admin-fe 京东风 + Dockerfile + nginx + compose（本方案前端复刻）
- `daily/2026-05-27.md` — L1.6 改密强制下线（本方案 Role 过滤直接获益）
