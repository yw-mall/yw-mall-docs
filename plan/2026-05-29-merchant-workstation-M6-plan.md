# 商家工作台 M6 — Dashboard + 装修 + 容器化 实施计划

> 关联：`plan/2026-05-27-merchant-workstation-design.md` 第 8 节 M6 + 6.2 聚合 RPC

**Goal**：商家进 dashboard 看 4 核心指标（今日营业额/订单/待发货/待退款/钱包余额）+ 简单店铺装修（banner/公告/推荐商品）+ merchant-fe 容器化部署 18083。

---

## Task 1: `shop_decoration` 表 + 迁移

- Create: `mall-shop-rpc/sql/shop_decoration.sql`
- Modify: `yw-mall/start.sh` SPRINT_MIGRATIONS

```sql
CREATE TABLE IF NOT EXISTS shop_decoration (
  id           BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  shop_id      BIGINT UNSIGNED NOT NULL UNIQUE,
  banners      TEXT NOT NULL,
  announcement VARCHAR(500) NOT NULL DEFAULT '',
  featured_pids VARCHAR(500) NOT NULL DEFAULT '',
  update_time  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Task 2: shop-rpc 加 3 RPC

proto:
- `GetMerchantDashboard(shopId) → MerchantDashboardResp` 聚合 4 服务（errgroup 并发）
- `GetShopDecoration(shopId) → ShopDecoration`
- `UpdateShopDecoration(shopId, banners, announcement, featuredPids) → OkResp`

logic:
- dashboardlogic 用 errgroup 并发调 order-rpc.ListShopOrders（today filter + status=1 count）+ payment-rpc.GetMerchantWallet + order-rpc.ListShopRefundRequests
- decoration get/put 走 UPSERT 模式

## Task 3: admin-api 3 endpoint

- `GET /merchant/v1/dashboard`（任意 merchant role 可调）
- `GET /merchant/v1/decoration`（perm shop.read）
- `PUT /merchant/v1/decoration`（perm decoration.write，仅 owner）

## Task 4: FE api + dashboard 真实数据

- api/dashboard.ts + api/decoration.ts
- 改造 views/dashboard/index.vue：4 卡片用真实数据 + 待办区块

## Task 5: FE 装修页

- views/decoration/edit.vue：banner 图片数组（ImageUploader 多张）+ 公告 textarea + 推荐商品 ID 数组
- 路由 + nav 解锁

## Task 6: 容器化

- merchant/.dockerignore
- yw-mall-deploy/compose.yml 加 mall-merchant-fe 服务 18083:80

## Task 7: 收尾

- daily + 整体里程碑回写

---

## 验收

- [ ] alice 进 /dashboard 看 4 卡片真实数据（非占位 --）
- [ ] alice 进 /decoration 填 banner + 公告 + 保存
- [ ] `podman-compose config` 看到 mall-merchant-fe service @ 18083
- [ ] design spec 6 里程碑全部 ✅
