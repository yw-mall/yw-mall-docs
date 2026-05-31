# 优惠活动模块 Phase 1 技术设计 · v0.1

> 配套 PRD: [2026-05-31-promotion-module-design.md](./2026-05-31-promotion-module-design.md)
> Story: [2026-05-31-promotion-module-stories.md](./2026-05-31-promotion-module-stories.md)
> 目标：DDL 完整 / proto 完整 / 价格引擎可直接 TDD

---

## 一、整体架构

### 1.1 新增服务

```
mall-promotion-rpc            新建，端口 9018
├── DB: mall_promotion        新建库
├── Cache: cache:promotion:*  共用 Redis
└── etcd: yw-mall/promotion-rpc
```

### 1.2 上游调用关系

```
       ┌─────────────────────┐
       │ mall-promotion-rpc  │
       │                     │
       │  • Activity CRUD    │
       │  • Coupon Template  │
       │  • CalcPrice (★)    │
       │  • Lock/Release/    │
       │    Consume Coupon   │
       │  • CalcRefund       │
       └──┬───────┬───────┬──┘
          │       │       │
   ┌──────┘  ┌────┴──┐ ┌──┴──────┐
   ▼         ▼       ▼ ▼         ▼
┌─────┐  ┌─────┐  ┌─────┐  ┌─────────┐
│ api │  │cart │  │order│  │ admin   │
│ (C) │  │ rpc │  │ rpc │  │ api     │
└─────┘  └─────┘  └─────┘  └─────────┘
   ↑        ↑        ↑          ↑
   │        │        │          │
   FE       FE       内部          商家/平台
   领券     算价     用券+核销      管理活动
```

### 1.3 服务内部模块

```
mall-promotion-rpc/
├── promotion.go                     main
├── etc/promotion.yaml               配置
├── promotion/                       proto 生成
├── promotionclient/                 客户端 wrapper
├── sql/
│   ├── promotion.sql               初始 DDL
│   └── migrations/                 后续迁移
├── cmd/seed/                        seed 数据（示例活动+券）
└── internal/
    ├── config/
    ├── svc/
    ├── server/                      grpc 接线
    ├── model/                       sqlx 模型 (activity, coupon, ...)
    ├── logic/                       业务逻辑
    │   ├── activity_*.go
    │   ├── coupon_template_*.go
    │   ├── coupon_*.go
    │   ├── calcprice_logic.go      ★ 核心
    │   ├── lockcoupon_logic.go
    │   └── calcrefund_logic.go
    └── engine/                      ★ 价格引擎核心包
        ├── engine.go               入口
        ├── sku_level.go            SKU 级评估
        ├── shop_level.go           店铺级满减
        ├── coupon.go               券应用
        ├── breakdown.go            优惠明细
        └── engine_test.go          全量 TDD
```

---

## 二、数据库 DDL

### 2.1 数据库初始化

```sql
CREATE DATABASE IF NOT EXISTS mall_promotion
  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

USE mall_promotion;
```

### 2.2 activity 活动主体

```sql
CREATE TABLE IF NOT EXISTS activity (
  id              BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  type            VARCHAR(32)     NOT NULL                COMMENT '活动类型: fullreduce/discount/fixprice/coupon',
  name            VARCHAR(200)    NOT NULL                COMMENT '活动名称',
  shop_id         BIGINT UNSIGNED NOT NULL DEFAULT 0      COMMENT '所属店铺, 0=平台活动',
  status          TINYINT         NOT NULL DEFAULT 0      COMMENT '0草稿/1待开始/2进行中/3已结束/4已下线',
  start_time      BIGINT          NOT NULL DEFAULT 0      COMMENT '开始时间 unix',
  end_time        BIGINT          NOT NULL DEFAULT 0      COMMENT '结束时间 unix',
  priority        INT             NOT NULL DEFAULT 0      COMMENT '互斥时数字大优先生效',
  stackable       TINYINT         NOT NULL DEFAULT 1      COMMENT '0互斥/1可叠加',
  description     VARCHAR(500)    NOT NULL DEFAULT ''     COMMENT '活动描述',
  create_user_id  BIGINT UNSIGNED NOT NULL DEFAULT 0      COMMENT '创建人 user_id',
  create_time     DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
  update_time     DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_shop_status (shop_id, status, end_time),
  INDEX idx_type_status (type, status),
  INDEX idx_time_range  (start_time, end_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='促销活动主表';
```

### 2.3 activity_target 适用范围

```sql
CREATE TABLE IF NOT EXISTS activity_target (
  id              BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  activity_id     BIGINT UNSIGNED NOT NULL                COMMENT 'FK activity.id',
  target_type     VARCHAR(16)     NOT NULL                COMMENT 'sku/category/shop/user_tag/all',
  target_id       BIGINT UNSIGNED NOT NULL                COMMENT 'sku_id / category_id / shop_id / tag_id / 0=all',
  create_time     DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_activity (activity_id),
  INDEX idx_lookup   (target_type, target_id)            COMMENT '价格引擎反查 sku 命中的 activity'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='活动适用范围';
```

### 2.4 activity_action 优惠动作

```sql
CREATE TABLE IF NOT EXISTS activity_action (
  id                BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  activity_id       BIGINT UNSIGNED NOT NULL                COMMENT 'FK activity.id',
  action_type       VARCHAR(16)     NOT NULL                COMMENT 'reduce/discount/cash/fixprice/freeship/gift',
  threshold_type    VARCHAR(8)      NOT NULL DEFAULT 'none' COMMENT 'none/amount/quantity',
  threshold_value   BIGINT          NOT NULL DEFAULT 0      COMMENT '满 X 元(分)或 X 件',
  benefit_value     BIGINT          NOT NULL DEFAULT 0      COMMENT '减 Y 元(分) / 打 Y 折(75 表示7.5折) / 一口价(分)',
  max_discount      BIGINT          NOT NULL DEFAULT 0      COMMENT '折扣类的最高优惠上限(分), 0=不限',
  gift_sku_id       BIGINT UNSIGNED NOT NULL DEFAULT 0      COMMENT '赠品 sku',
  step_order        INT             NOT NULL DEFAULT 0      COMMENT '阶梯满减时排序, 引擎按 threshold_value DESC 取首个达标项',
  create_time       DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_activity (activity_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='活动优惠动作 (一对多支持阶梯)';
```

**阶梯满减示例**：满 199 减 30、满 299 减 50、满 499 减 100 → 同一 activity_id 写 3 行。

### 2.5 coupon_template 券模板

```sql
CREATE TABLE IF NOT EXISTS coupon_template (
  id                BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  activity_id       BIGINT UNSIGNED NOT NULL                COMMENT 'FK activity.id (券也是 activity 的一种)',
  shop_id           BIGINT UNSIGNED NOT NULL DEFAULT 0      COMMENT '0=平台券',
  name              VARCHAR(200)    NOT NULL,
  type              VARCHAR(16)     NOT NULL                COMMENT 'full_reduce/discount/cash/freeship',
  value             BIGINT          NOT NULL                COMMENT '满减面值(分) / 折扣(75=7.5折) / 立减值(分)',
  min_amount        BIGINT          NOT NULL DEFAULT 0      COMMENT '满 X 元可用(分)',
  max_discount      BIGINT          NOT NULL DEFAULT 0      COMMENT '折扣类最高优惠(分), 0=不限',
  category_id       BIGINT UNSIGNED NOT NULL DEFAULT 0      COMMENT '品类券限定, 0=全品类',
  total_count       INT             NOT NULL                COMMENT '总发放量',
  received_count    INT             NOT NULL DEFAULT 0      COMMENT '已领数',
  used_count        INT             NOT NULL DEFAULT 0      COMMENT '已使用数',
  per_user_limit    INT             NOT NULL DEFAULT 1      COMMENT '每人限领',
  valid_type        TINYINT         NOT NULL DEFAULT 0      COMMENT '0固定日期 / 1领取后N天',
  valid_days        INT             NOT NULL DEFAULT 0      COMMENT '领取后N天有效(valid_type=1时用)',
  valid_start       BIGINT          NOT NULL DEFAULT 0      COMMENT '固定日期模式起 unix',
  valid_end         BIGINT          NOT NULL DEFAULT 0      COMMENT '固定日期模式止 unix',
  receive_start     BIGINT          NOT NULL DEFAULT 0      COMMENT '领取窗口起',
  receive_end       BIGINT          NOT NULL DEFAULT 0      COMMENT '领取窗口止',
  status            TINYINT         NOT NULL DEFAULT 1      COMMENT '0下架/1上架',
  create_time       DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
  update_time       DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_shop_status (shop_id, status, receive_end),
  INDEX idx_activity    (activity_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='券模板';
```

### 2.6 coupon 用户领到的券

```sql
CREATE TABLE IF NOT EXISTS coupon (
  id                BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  template_id       BIGINT UNSIGNED NOT NULL                COMMENT 'FK coupon_template.id',
  user_id           BIGINT UNSIGNED NOT NULL,
  shop_id           BIGINT UNSIGNED NOT NULL DEFAULT 0      COMMENT '冗余 shop_id 加速查询',
  status            TINYINT         NOT NULL DEFAULT 0      COMMENT '0未用/1已锁定/2已使用/3已过期',
  order_id          BIGINT UNSIGNED NOT NULL DEFAULT 0      COMMENT '已使用时回填',
  receive_time      BIGINT          NOT NULL                COMMENT '领取 unix',
  expire_time       BIGINT          NOT NULL                COMMENT '失效 unix',
  lock_time         BIGINT          NOT NULL DEFAULT 0      COMMENT '锁定时间, 5min 未支付则释放',
  use_time          BIGINT          NOT NULL DEFAULT 0,
  create_time       DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uk_user_template_one (user_id, template_id, status) COMMENT '同模板同 user 同状态防重(配合 per_user_limit=1)',
  INDEX idx_user_status (user_id, status, expire_time),
  INDEX idx_order       (order_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户已领券';
```

> **说明**: `uk_user_template_one` 仅对每模板限 1 张的场景能 enforce；per_user_limit > 1 时需要 logic 层校验 + 行锁。

### 2.7 order 表新增列 (改造 mall_order)

```sql
ALTER TABLE mall_order.`order`
  ADD COLUMN promotion_discount BIGINT NOT NULL DEFAULT 0
    COMMENT '活动总优惠(分), 含 SKU 级 + 店铺级 + 券' AFTER total_amount,
  ADD COLUMN coupon_discount    BIGINT NOT NULL DEFAULT 0
    COMMENT '券优惠总额(分)' AFTER promotion_discount,
  ADD COLUMN paid_amount        BIGINT NOT NULL DEFAULT 0
    COMMENT '用户实付(分) = total_amount - promotion_discount' AFTER coupon_discount,
  ADD COLUMN discount_detail    JSON   DEFAULT NULL
    COMMENT '优惠明细 JSON, 退款分摊用';
```

`discount_detail` JSON 示例：
```json
{
  "sku_level": [
    {"sku_id": 81, "type": "fixprice", "activity_id": 5, "original": 12900, "deal": 9900, "saved": 3000}
  ],
  "shop_level": {
    "shop_id": 1, "type": "fullreduce", "activity_id": 7, "threshold": 19900, "saved": 3000
  },
  "coupons": [
    {"coupon_id": 17, "template_id": 4, "type": "cash", "scope": "shop", "saved": 500}
  ],
  "shipping_saved": 0
}
```

### 2.8 order_coupon_use 订单-券映射 (可选, 简单场景可不要)

如果一单可能用多张券（店铺券 + 平台券）：

```sql
CREATE TABLE IF NOT EXISTS mall_order.order_coupon_use (
  id          BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  order_id    BIGINT UNSIGNED NOT NULL,
  coupon_id   BIGINT UNSIGNED NOT NULL,
  template_id BIGINT UNSIGNED NOT NULL,
  saved       BIGINT          NOT NULL    COMMENT '本张券实际抵扣(分)',
  create_time DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uk_order_coupon (order_id, coupon_id),
  INDEX idx_order (order_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='订单与券的映射 (一单多券)';
```

### 2.9 seed 数据示例

```sql
-- 示例 1: shop 1 满减活动「满 199 减 30」
INSERT INTO activity (type, name, shop_id, status, start_time, end_time, priority, stackable)
  VALUES ('fullreduce', '满 199 减 30', 1, 2, UNIX_TIMESTAMP()-3600, UNIX_TIMESTAMP()+86400*7, 100, 1);
SET @aid = LAST_INSERT_ID();
INSERT INTO activity_target (activity_id, target_type, target_id) VALUES (@aid, 'shop', 1);
INSERT INTO activity_action (activity_id, action_type, threshold_type, threshold_value, benefit_value, step_order)
  VALUES (@aid, 'reduce', 'amount', 19900, 3000, 1);

-- 示例 2: shop 1 立减券 ¥5
INSERT INTO activity (type, name, shop_id, status, start_time, end_time, stackable)
  VALUES ('coupon', '店铺立减 5 元', 1, 2, UNIX_TIMESTAMP()-3600, UNIX_TIMESTAMP()+86400*30, 1);
SET @cid = LAST_INSERT_ID();
INSERT INTO coupon_template (activity_id, shop_id, name, type, value, min_amount, total_count, per_user_limit,
                             valid_type, valid_days, receive_start, receive_end, status)
  VALUES (@cid, 1, '店铺立减 5 元', 'cash', 500, 5000, 1000, 1,
          1, 14, UNIX_TIMESTAMP()-3600, UNIX_TIMESTAMP()+86400*30, 1);
```

---

## 三、Proto 定义

### 3.1 完整 promotion.proto

```protobuf
syntax = "proto3";
package promotion;
option go_package = "./promotion";

// ========== 通用 ==========
message Empty {}
message OkResp {
  bool ok = 1;
}

// ========== 活动管理 ==========
message Activity {
  int64  id          = 1;
  string type        = 2;          // fullreduce/discount/fixprice/coupon
  string name        = 3;
  int64  shop_id     = 4;
  int32  status      = 5;
  int64  start_time  = 6;
  int64  end_time    = 7;
  int32  priority    = 8;
  bool   stackable   = 9;
  string description = 10;
  repeated ActivityTarget targets = 11;
  repeated ActivityAction actions = 12;
  int64  create_time = 13;
  int64  update_time = 14;
}

message ActivityTarget {
  int64  id          = 1;
  int64  activity_id = 2;
  string target_type = 3;          // sku/category/shop/all
  int64  target_id   = 4;
}

message ActivityAction {
  int64  id              = 1;
  int64  activity_id     = 2;
  string action_type     = 3;      // reduce/discount/cash/fixprice/freeship/gift
  string threshold_type  = 4;      // none/amount/quantity
  int64  threshold_value = 5;
  int64  benefit_value   = 6;
  int64  max_discount    = 7;
  int64  gift_sku_id     = 8;
  int32  step_order      = 9;
}

message CreateActivityReq {
  string type        = 1;
  string name        = 2;
  int64  shop_id     = 3;
  int64  start_time  = 4;
  int64  end_time    = 5;
  int32  priority    = 6;
  bool   stackable   = 7;
  string description = 8;
  int64  create_user_id = 9;
  repeated ActivityTarget targets = 10;
  repeated ActivityAction actions = 11;
}
message CreateActivityResp { int64 id = 1; }

message GetActivityReq  { int64 id = 1; }
message GetActivityResp { Activity activity = 1; }

message UpdateActivityReq {
  int64  id          = 1;
  string name        = 2;
  int64  start_time  = 3;
  int64  end_time    = 4;
  int32  priority    = 5;
  bool   stackable   = 6;
  string description = 7;
  repeated ActivityTarget targets = 8;
  repeated ActivityAction actions = 9;
}

message ChangeActivityStatusReq {
  int64 id     = 1;
  int32 status = 2;                // 1上线/4下线 (草稿→上线/进行中→下线)
}

message ListActivitiesReq {
  int64  shop_id   = 1;
  string type      = 2;            // 空=全部
  int32  status    = 3;            // -1=全部
  int32  page      = 4;
  int32  page_size = 5;
}
message ListActivitiesResp {
  repeated Activity activities = 1;
  int64 total = 2;
}

// ========== 券模板 ==========
message CouponTemplate {
  int64  id              = 1;
  int64  activity_id     = 2;
  int64  shop_id         = 3;
  string name            = 4;
  string type            = 5;      // full_reduce/discount/cash/freeship
  int64  value           = 6;
  int64  min_amount      = 7;
  int64  max_discount    = 8;
  int64  category_id     = 9;
  int32  total_count     = 10;
  int32  received_count  = 11;
  int32  used_count      = 12;
  int32  per_user_limit  = 13;
  int32  valid_type      = 14;
  int32  valid_days      = 15;
  int64  valid_start     = 16;
  int64  valid_end       = 17;
  int64  receive_start   = 18;
  int64  receive_end     = 19;
  int32  status          = 20;
  int64  create_time     = 21;
  int64  update_time     = 22;
}

message CreateCouponTemplateReq {
  int64  shop_id        = 1;
  string name           = 2;
  string type           = 3;
  int64  value          = 4;
  int64  min_amount     = 5;
  int64  max_discount   = 6;
  int64  category_id    = 7;
  int32  total_count    = 8;
  int32  per_user_limit = 9;
  int32  valid_type     = 10;
  int32  valid_days     = 11;
  int64  valid_start    = 12;
  int64  valid_end      = 13;
  int64  receive_start  = 14;
  int64  receive_end    = 15;
}
message CreateCouponTemplateResp { int64 id = 1; }

message ListCouponTemplatesReq {
  int64 shop_id   = 1;             // 0 表示平台券
  int32 status    = 2;             // -1=全部
  int32 page      = 3;
  int32 page_size = 4;
}
message ListCouponTemplatesResp {
  repeated CouponTemplate templates = 1;
  int64 total = 2;
}

message ChangeCouponTemplateStatusReq {
  int64 id     = 1;
  int32 status = 2;                // 0下架/1上架
}

// ========== 用户领券 ==========
message Coupon {
  int64  id           = 1;
  int64  template_id  = 2;
  int64  user_id      = 3;
  int64  shop_id      = 4;
  int32  status       = 5;
  int64  order_id     = 6;
  int64  receive_time = 7;
  int64  expire_time  = 8;
  int64  use_time     = 9;
  // 冗余字段方便 FE 展示
  string template_name = 10;
  string type          = 11;
  int64  value         = 12;
  int64  min_amount    = 13;
}

message ReceiveCouponReq {
  int64 user_id     = 1;
  int64 template_id = 2;
}
message ReceiveCouponResp { int64 coupon_id = 1; }

message ListMyCouponsReq {
  int64 user_id   = 1;
  int32 status    = 2;             // -1=全部
  int32 page      = 3;
  int32 page_size = 4;
}
message ListMyCouponsResp {
  repeated Coupon coupons = 1;
  int64 total = 2;
}

// ========== 价格计算引擎 (★ 核心) ==========
message CartItem {
  int64 sku_id        = 1;
  int64 product_id    = 2;
  int64 shop_id       = 3;
  int64 category_id   = 4;
  int64 original_price = 5;        // sku 原价(分)
  int32 quantity      = 6;
}

message CalcPriceReq {
  int64 user_id              = 1;
  repeated CartItem items    = 2;
  repeated int64 coupon_ids  = 3;  // 用户主动选定的 coupon.id 列表
  int64  shipping_fee        = 4;  // 由调用方算好运费传入
}

message CalcPriceResp {
  int64  total_amount       = 1;   // 原价合计
  int64  promotion_discount = 2;   // 活动总优惠
  int64  coupon_discount    = 3;   // 券总优惠
  int64  shipping_fee       = 4;   // 实际运费 (可能被包邮券减为 0)
  int64  paid_amount        = 5;   // 应付 = total - promotion - coupon + shipping
  string discount_detail    = 6;   // JSON, 见 § 2.7
  repeated PriceConflict conflicts = 7;  // 若用户传的 coupon 不可用, 此处说明
}

message PriceConflict {
  int64  coupon_id = 1;
  string reason    = 2;
}

// ========== 订单锁/释放/核销 ==========
message LockCouponReq {
  int64 user_id   = 1;
  int64 order_id  = 2;
  repeated int64 coupon_ids = 3;
}
message ReleaseCouponReq { int64 order_id = 1; }
message ConsumeCouponReq { int64 order_id = 1; }   // 支付成功调用

// ========== 退款分摊 ==========
message CalcRefundReq {
  int64 order_id           = 1;
  repeated RefundItem items = 2;     // 部分退款时, 指定哪些 sku 退多少件
  bool   full_refund       = 3;      // true=整单退
}
message RefundItem {
  int64 sku_id   = 1;
  int32 quantity = 2;
}
message CalcRefundResp {
  int64  refund_amount   = 1;        // 应退给用户的实付金额(分)
  string refund_detail   = 2;        // JSON: 各项分摊明细
}

// ========== 服务 ==========
service Promotion {
  // 活动 CRUD
  rpc CreateActivity        (CreateActivityReq)        returns (CreateActivityResp);
  rpc GetActivity           (GetActivityReq)           returns (GetActivityResp);
  rpc UpdateActivity        (UpdateActivityReq)        returns (OkResp);
  rpc ChangeActivityStatus  (ChangeActivityStatusReq)  returns (OkResp);
  rpc ListActivities        (ListActivitiesReq)        returns (ListActivitiesResp);

  // 券模板
  rpc CreateCouponTemplate        (CreateCouponTemplateReq)        returns (CreateCouponTemplateResp);
  rpc ListCouponTemplates         (ListCouponTemplatesReq)         returns (ListCouponTemplatesResp);
  rpc ChangeCouponTemplateStatus  (ChangeCouponTemplateStatusReq)  returns (OkResp);

  // 用户领券
  rpc ReceiveCoupon  (ReceiveCouponReq)  returns (ReceiveCouponResp);
  rpc ListMyCoupons  (ListMyCouponsReq)  returns (ListMyCouponsResp);

  // ★ 价格引擎
  rpc CalcPrice  (CalcPriceReq)  returns (CalcPriceResp);

  // 订单生命周期
  rpc LockCoupon     (LockCouponReq)     returns (OkResp);
  rpc ReleaseCoupon  (ReleaseCouponReq)  returns (OkResp);
  rpc ConsumeCoupon  (ConsumeCouponReq)  returns (OkResp);

  // 退款
  rpc CalcRefund  (CalcRefundReq)  returns (CalcRefundResp);
}
```

### 3.2 RPC 行为契约

| RPC | 幂等 | 事务 | 缓存 | 失败处理 |
|---|---|---|---|---|
| CreateActivity | 否 | 跨 3 表事务 | 无 | 任一失败回滚 |
| GetActivity | 是 | 否 | Redis cache:promotion:activity:{id} TTL 300s | DB 失败返 error |
| ChangeActivityStatus | 是 | 否 | 失效 cache | 状态机校验失败返 ParamError |
| ReceiveCoupon | 否 (但通过 unique key 防重) | 单表事务 | Redis 计数 received_count | duplicate key 返 "已领取" |
| **CalcPrice** | **是** | 否 (只读) | 强烈依赖 cache | 任一字段失败兜底 0, 不阻塞下单 |
| LockCoupon | 否 (但通过 status CAS 防重) | 跨表事务 | 失效 cache:coupon:user:* | RowsAffected=0 返 "券状态已变" |
| ReleaseCoupon | 是 | 跨表事务 | | 不存在的 order_id 返 ok |
| ConsumeCoupon | 是 | 跨表事务 | | 已 consumed 的再调返 ok |
| CalcRefund | 是 | 否 (只读) | 无 | order 不存在返 NotFound |

---

## 四、价格计算引擎 (★ Phase 1 核心)

### 4.1 设计原则

1. **纯函数**：输入 (cart_items + user + coupons + activities) 全显式传入，无隐式 DB 查询
2. **可测试**：所有 DB 读在 logic 层完成，engine 层只接收数据 → 100% 可单测
3. **顺序确定性**：固定 5 步流程，每步产出可解释的 breakdown
4. **失败兜底**：单步失败不阻塞整体，记录 conflict 返回给 FE

### 4.2 引擎调用流程

```
logic/CalcPriceLogic:
  1. 拉 cart_items 涉及的所有 SKU 命中的 active activities    ← DB/cache
     SELECT a.* FROM activity a
       JOIN activity_target t ON t.activity_id=a.id
      WHERE a.status=2 AND a.end_time>NOW()
        AND ((t.target_type='sku' AND t.target_id IN (...))
          OR (t.target_type='shop' AND t.target_id IN (...))
          OR (t.target_type='all'))
  2. 拉 user 拥有的、未过期、未使用 + 用户主动选的 coupon
  3. engine.Calculate(activities, coupons, items, user) → CalcPriceResp
  4. 返回
```

### 4.3 核心算法伪代码

```go
package engine

type Item struct {
    SkuID         int64
    ProductID     int64
    ShopID        int64
    CategoryID    int64
    OriginalPrice int64        // 单价
    Quantity      int32
    // 评估后填充
    DealPrice     int64        // 单价 (经 SKU 级活动)
    SkuActivityID int64        // 命中的 SKU 级活动
}

type Result struct {
    TotalAmount       int64
    PromotionDiscount int64
    CouponDiscount    int64
    ShippingFee       int64
    PaidAmount        int64
    Breakdown         Breakdown  // 结构化, 后续序列化为 JSON
    Conflicts         []Conflict
}

func Calculate(
    items []Item,
    activities []Activity,    // 已过滤 status=2 + 时间窗内
    coupons []Coupon,         // 用户主动选定的, 已校验 status=0
    shippingFee int64,
) Result {
    bd := Breakdown{}

    // ============ Step 1: SKU 级评估 ============
    // 每个 SKU 取所有命中的 SKU 级活动 (fixprice/discount), 选最优价
    skuActivities := indexSkuActivities(activities)  // map[sku_id][]Activity
    for i := range items {
        item := &items[i]
        bestPrice := item.OriginalPrice
        bestActID := int64(0)
        for _, act := range skuActivities[item.SkuID] {
            for _, action := range act.Actions {
                price := applySkuAction(item.OriginalPrice, action)
                if price < bestPrice {
                    bestPrice, bestActID = price, act.ID
                }
            }
        }
        item.DealPrice = bestPrice
        item.SkuActivityID = bestActID
        if bestActID > 0 {
            bd.SkuLevel = append(bd.SkuLevel, SkuLine{
                SkuID: item.SkuID, ActivityID: bestActID,
                Original: item.OriginalPrice, Deal: bestPrice,
                Saved: (item.OriginalPrice - bestPrice) * int64(item.Quantity),
            })
        }
    }

    // ============ Step 2: 店铺级满减 ============
    // 按店铺分组, 计算每个店铺的 DealPrice 小计
    shopGroups := groupByShop(items)  // map[shop_id][]*Item
    shopSubtotal := map[int64]int64{}
    for shopID, group := range shopGroups {
        sub := int64(0)
        for _, it := range group {
            sub += it.DealPrice * int64(it.Quantity)
        }
        shopSubtotal[shopID] = sub

        // 取该店铺命中的店铺级活动 (fullreduce/discount, target=shop)
        shopActs := shopLevelActivitiesFor(activities, shopID)
        bestSaved := int64(0)
        var bestAct *Activity
        for _, act := range shopActs {
            saved := evaluateShopActivity(act, sub, group)
            if saved > bestSaved {
                bestSaved, bestAct = saved, act
            }
        }
        if bestAct != nil {
            bd.ShopLevel = append(bd.ShopLevel, ShopLine{
                ShopID: shopID, ActivityID: bestAct.ID,
                Type: bestAct.Type, Subtotal: sub, Saved: bestSaved,
            })
            shopSubtotal[shopID] -= bestSaved  // 用于后续券计算
        }
    }

    // ============ Step 3: 券应用 ============
    for _, c := range coupons {
        scope := couponScope(c)              // shop / platform
        validatePool := []int64{}            // 哪些金额可用
        if scope == "shop" {
            // 检查该店铺小计是否满 min_amount
            if sub, ok := shopSubtotal[c.ShopID]; ok && sub >= c.MinAmount {
                saved := applyCoupon(c, sub)
                shopSubtotal[c.ShopID] -= saved
                bd.Coupons = append(bd.Coupons, CouponLine{
                    CouponID: c.ID, TemplateID: c.TemplateID, Type: c.Type,
                    Scope: "shop", ShopID: c.ShopID, Saved: saved,
                })
            } else {
                bd.Conflicts = append(bd.Conflicts, Conflict{
                    CouponID: c.ID,
                    Reason:   fmt.Sprintf("店铺金额未满 %d 元门槛", c.MinAmount/100),
                })
            }
        } else if scope == "platform" {
            allSub := int64(0)
            for _, v := range shopSubtotal {
                allSub += v
            }
            if allSub >= c.MinAmount {
                saved := applyCoupon(c, allSub)
                // 按各店铺占比分摊扣减
                for sid := range shopSubtotal {
                    share := saved * shopSubtotal[sid] / allSub
                    shopSubtotal[sid] -= share
                }
                bd.Coupons = append(bd.Coupons, CouponLine{
                    CouponID: c.ID, Type: c.Type, Scope: "platform", Saved: saved,
                })
            } else {
                bd.Conflicts = append(bd.Conflicts, Conflict{
                    CouponID: c.ID, Reason: "总金额未满平台券门槛",
                })
            }
        }
    }

    // ============ Step 4: 运费 / 包邮券 ============
    finalShipping := shippingFee
    for _, c := range coupons {
        if c.Type == "freeship" && /* 命中 freeship 条件 */ true {
            bd.ShippingSaved = shippingFee
            finalShipping = 0
            break
        }
    }

    // ============ Step 5: 汇总 ============
    totalOrig := int64(0)
    for _, it := range items {
        totalOrig += it.OriginalPrice * int64(it.Quantity)
    }
    promoDiscount := int64(0)
    for _, l := range bd.SkuLevel {
        promoDiscount += l.Saved
    }
    for _, l := range bd.ShopLevel {
        promoDiscount += l.Saved
    }
    couponDiscount := int64(0)
    for _, l := range bd.Coupons {
        couponDiscount += l.Saved
    }
    paid := totalOrig - promoDiscount - couponDiscount + finalShipping
    if paid < 0 {
        paid = 0  // 防呆: 优惠总额 > 商品额时不能负
    }

    return Result{
        TotalAmount:       totalOrig,
        PromotionDiscount: promoDiscount,
        CouponDiscount:    couponDiscount,
        ShippingFee:       finalShipping,
        PaidAmount:        paid,
        Breakdown:         bd,
        Conflicts:         bd.Conflicts,
    }
}
```

### 4.4 各 action 单步算法

```go
// applySkuAction: SKU 级活动评估 (返回新单价)
func applySkuAction(orig int64, a Action) int64 {
    switch a.ActionType {
    case "fixprice":
        return a.BenefitValue            // 一口价直接覆盖
    case "discount":
        // a.BenefitValue 是 75 表示 7.5 折
        newPrice := orig * a.BenefitValue / 100
        if a.MaxDiscount > 0 && orig - newPrice > a.MaxDiscount {
            newPrice = orig - a.MaxDiscount
        }
        return newPrice
    case "reduce":
        // SKU 级直接减 (少见, 通常用于单品立减)
        if a.ThresholdType == "amount" && orig < a.ThresholdValue {
            return orig
        }
        return max(orig - a.BenefitValue, 0)
    }
    return orig
}

// evaluateShopActivity: 店铺级满减, 返回省了多少钱
func evaluateShopActivity(act Activity, subtotal int64, items []*Item) int64 {
    // 阶梯满减: 取 step_order DESC + threshold_value <= subtotal 的第一个
    var best *Action
    for i := range act.Actions {
        a := &act.Actions[i]
        if a.ThresholdType == "amount" && subtotal >= a.ThresholdValue {
            if best == nil || a.ThresholdValue > best.ThresholdValue {
                best = a
            }
        }
        if a.ThresholdType == "quantity" {
            totalQty := int64(0)
            for _, it := range items {
                totalQty += int64(it.Quantity)
            }
            if totalQty >= a.ThresholdValue {
                if best == nil || a.ThresholdValue > best.ThresholdValue {
                    best = a
                }
            }
        }
    }
    if best == nil {
        return 0
    }
    switch best.ActionType {
    case "reduce":
        return best.BenefitValue
    case "discount":
        saved := subtotal - subtotal * best.BenefitValue / 100
        if best.MaxDiscount > 0 && saved > best.MaxDiscount {
            saved = best.MaxDiscount
        }
        return saved
    }
    return 0
}

// applyCoupon: 算单张券能省多少 (调用方已校验 min_amount)
func applyCoupon(c Coupon, basis int64) int64 {
    switch c.Type {
    case "cash":
        return min(c.Value, basis)        // 立减不超过基数
    case "full_reduce":
        return c.Value                     // 满减面值
    case "discount":
        saved := basis - basis * c.Value / 100
        if c.MaxDiscount > 0 && saved > c.MaxDiscount {
            saved = c.MaxDiscount
        }
        return saved
    }
    return 0
}
```

### 4.5 单元测试矩阵 (必须 TDD)

| Case ID | 场景 | 预期 |
|---|---|---|
| TC-01 | 单 SKU 无活动 | paid = orig × qty |
| TC-02 | SKU 命中一口价 | paid 用一口价 |
| TC-03 | SKU 同时命中一口价 + 7 折，取最优 | 取价低的 |
| TC-04 | 店铺满 199 减 30，购物车 250 元 | 减 30 |
| TC-05 | 店铺阶梯满减，购物车 350 元 | 取 299 减 50 档 |
| TC-06 | 阶梯门槛差 1 分钱不达标 | 不减 |
| TC-07 | SKU 级 + 店铺级 + 店铺券叠加 | 三层都减 |
| TC-08 | 店铺券 + 平台券同时用 | 平台券基于店铺券后的总额 |
| TC-09 | 平台券按店铺占比分摊 | 各店铺扣减比例正确 |
| TC-10 | 用户传的券不满门槛 | 返 conflict 不应用 |
| TC-11 | 折扣券有 max_discount 触发上限 | 取上限 |
| TC-12 | 多店铺购物车 + 店铺活动只对 shop1 生效 | shop2 不享 |
| TC-13 | 包邮券触发，运费归 0 | shipping=0 |
| TC-14 | 优惠总额 > 商品总额 | paid=0 不能负 |
| TC-15 | 0 quantity item | 跳过不算入 |
| TC-16 | 同一活动 priority 不同，只生效高优先级 | 互斥取最高 priority |

### 4.6 性能优化

- **活动缓存**：activities 按 shop_id 全量 Redis 缓存，TTL 60s，更新时双删
- **券缓存**：用户 coupon 列表按 user_id 缓存，TTL 30s，领券/用券时失效
- **计算复杂度**：O(items × activities + coupons)；典型场景 (10 件 / 5 活动 / 3 券) 在 1ms 内
- **目标 P99**：< 20ms（不含 DB IO）

---

## 五、订单生命周期

### 5.1 下单时序

```
C-end POST /api/order/create
   │
   ▼
mall-api / CreateOrder
   │
   ├─ 1. PromotionRpc.CalcPrice(...)              → 拿到 paid_amount + discount_detail
   ├─ 2. mall-order-rpc / CreateOrder             → 写 order 表 (含 promotion_discount/coupon_discount/paid_amount/discount_detail)
   ├─ 3. PromotionRpc.LockCoupon(order_id, [...]) → 券 status 0→1 / write order_coupon_use
   └─ 4. 返回 order_id + paid_amount
```

### 5.2 状态同步表

| 事件 | order | coupon | merchant_wallet |
|---|---|---|---|
| 下单 | 创建 status=0 | 锁定 0→1 | — |
| 取消 / 5min 未付 | status→4 | 释放 1→0 | — |
| 支付成功 | status→1 + paid_amount | 核销 1→2 + use_time | frozen += paid_amount |
| 退款执行 | status→4 + cancel_reason | **不变** (券不退) | frozen -= refund_amount |

### 5.3 LockCoupon 防重 (SQL 模板)

```sql
-- 一条 SQL 完成乐观锁 status=0 → 1
UPDATE coupon
   SET status = 1, lock_time = ?, order_id = ?
 WHERE id IN (?, ?, ...)
   AND user_id = ?
   AND status = 0
   AND expire_time > ?;
-- 检查 RowsAffected == len(coupon_ids), 否则 rollback
```

---

## 六、退款分摊 (CalcRefund)

### 6.1 整单退款

```
refund_amount = order.paid_amount
```

### 6.2 部分退款 (按 SKU 单价占比)

```
对每个退款 SKU:
  原 SKU 实付 = order.discount_detail 中该 sku 经 sku 级活动后的单价 × qty
  店铺级 + 券优惠按"该 SKU 原价占店铺/订单总原价比例"分摊
  退款额 = 原 SKU 实付 - 该 SKU 分摊的优惠
```

### 6.3 算法伪代码

```go
func CalcPartialRefund(order Order, refundItems []RefundItem) int64 {
    detail := parseDiscountDetail(order.DiscountDetail)
    refundAmount := int64(0)

    // 索引: sku_id -> sku 级 deal price
    skuDealPrice := map[int64]int64{}
    for _, line := range detail.SkuLevel {
        skuDealPrice[line.SkuID] = line.Deal
    }

    // 计算原订单各 sku 的"sku 级价"小计 (用于占比分摊)
    skuSubtotal := map[int64]int64{}  // sku_id -> sku_deal_price * qty
    totalSubtotal := int64(0)
    for _, item := range order.Items {
        price := skuDealPrice[item.SkuID]
        if price == 0 {
            price = item.OriginalPrice  // 该 sku 无 sku 级活动
        }
        skuSubtotal[item.SkuID] = price * int64(item.Quantity)
        totalSubtotal += skuSubtotal[item.SkuID]
    }

    // 店铺级 + 券总优惠
    extraSaved := int64(0)
    for _, l := range detail.ShopLevel { extraSaved += l.Saved }
    for _, l := range detail.Coupons   { extraSaved += l.Saved }

    // 对每个退款 sku 算退款额
    for _, ri := range refundItems {
        skuDeal := skuDealPrice[ri.SkuID]
        skuPaid := skuDeal * int64(ri.Quantity)
        // 该 sku 分摊到的优惠 = 该 sku 在订单中占比 * 总额外优惠 * 退款占该 sku 的比例
        skuShare := extraSaved * skuSubtotal[ri.SkuID] / totalSubtotal
        skuShareRefund := skuShare * int64(ri.Quantity) /
                         (skuSubtotal[ri.SkuID] / skuDeal)  // 该 sku 退款数 / 总数
        refundAmount += skuPaid - skuShareRefund
    }
    return refundAmount
}
```

### 6.4 边界 case

- **整单退**：直接 `order.paid_amount`，不走分摊
- **退款后剩余金额低于券门槛**：本 Phase 简化为「券不退、退实付」；后续 Phase 处理「触发券回退」
- **赠品**：本 Phase 不在范围，Phase 3 起处理

---

## 七、关键 API 契约 (HTTP 层)

### 7.1 商家后台 (admin-api / 加 /merchant/v1 前缀)

```
POST   /merchant/v1/promotions                     创建活动
GET    /merchant/v1/promotions                     活动列表 (支持 type/status 过滤)
GET    /merchant/v1/promotions/:id                 活动详情
PUT    /merchant/v1/promotions/:id                 编辑 (草稿态)
POST   /merchant/v1/promotions/:id/online          上线
POST   /merchant/v1/promotions/:id/offline         下线

POST   /merchant/v1/coupon-templates               创建券模板
GET    /merchant/v1/coupon-templates               券模板列表
PUT    /merchant/v1/coupon-templates/:id/status    上下架
```

### 7.2 C 端 (mall-api / /api 前缀)

```
GET    /api/coupon/list                            可领券列表 (按 shop_id/category 过滤)
POST   /api/coupon/receive                         领券 {templateId}
GET    /api/coupon/my                              我的券包 (status 过滤)

POST   /api/cart/calc-price                        实时算价 {items, couponIds}
```

### 7.3 内部 (RPC, 不暴露 HTTP)

`mall-api / mall-order-rpc / mall-payment-rpc` 通过 promotionclient 调用：
- `CalcPrice`
- `LockCoupon` / `ReleaseCoupon` / `ConsumeCoupon`
- `CalcRefund`

---

## 八、迁移 & 部署

### 8.1 上线步骤

1. **DB**: db-init.sh 加 `apply mall_promotion mall-promotion-rpc/sql/promotion.sql` + 改 mall_order 加列
2. **服务**: start.sh 加 promotion-rpc 启动
3. **etcd 配置**: 写 `/config/dev/yw-mall/promotion-rpc.yaml`
4. **mall-api / order-rpc / admin-api**: 加 PromotionRpc client 配置
5. **seed**: mall-promotion-rpc/cmd/seed 写 1 个示例满减 + 1 个示例券模板
6. **smoke test**: 跑 e2e 1 ~ 3

### 8.2 兼容性

- order 表加新列默认 0，对现有订单无影响
- 老订单 `paid_amount=0` → admin 数据补丁脚本回填 `paid_amount = total_amount`
- 老订单 `discount_detail=NULL` → 退款时按"无优惠"路径处理，等于 total_amount

### 8.3 灰度

- 阶段 A：促销 RPC 上线但 mall-api 不调用 (黑屏)
- 阶段 B：mall-api 在购物车开关位下调用，默认关
- 阶段 C：放量 10% 用户 → 监控 conflict 率 / 实付 != 预期数
- 阶段 D：全量放开

---

## 九、待解决问题

### 9.1 PRD § 8.2 的 4 个产品决策必须先定

1. **同活动不同 SKU 不同价** → 当前 Phase 1 用「每个 SKU 单独一个 fixprice activity」实现，未来如需「同活动多档」要扩 activity_action.target_sku 字段
2. **券是否可转让** → Phase 1 默认不可，coupon 表无 transfer_count 字段
3. **会员等级降级** → Phase 1 不涉及会员，留 Phase 3
4. **积分到期** → Phase 1 不涉及

### 9.2 技术开放问题

1. **平台券按占比分摊时的尾差处理** —— 各店铺占比扣减后总和可能差 1 分。约定：差额计入第一家店铺
2. **CalcPrice 是否要在 ProxySQL 走从库** —— 强一致但允许 50ms 内的延迟，可接受走从
3. **discount_detail JSON 列搜索性能** —— 暂不索引，未来如果有「按活动统计 GMV」需求再加 JSON virtual column

---

## 十、下一步

1. **本设计 review** —— 把不确定的算法 case 一起过
2. **DDL + proto 提交评审** —— 用代码 review 把字段类型/索引敲死
3. **engine 包 TDD 启动** —— 先把 § 4.5 的 16 个 test case 写成 _test.go，红了再写实现
4. **Phase 1 Sprint 排期** —— 按 Story 文档分人分周
