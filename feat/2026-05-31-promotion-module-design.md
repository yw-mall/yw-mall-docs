# 优惠活动模块产品说明（PRD）· v0.1

> 对标京东 / 淘宝主流玩法，规划 yw-mall 优惠活动产品体系
> 目标：覆盖 80% 主流场景 + 留扩展位，分阶段实施

---

## 一、模块定位与目标

### 1.1 业务定位

优惠活动是电商平台的核心增长引擎，承担三件事：
1. **拉新**：新人券、注册礼包、首单立减
2. **促活/转化**：满减、秒杀、限时折扣、拼团
3. **留存/复购**：会员等级、积分、回购券、会员日

### 1.2 模块目标

- **商家侧**：能 5 分钟创建一个促销活动，配置灵活但不至于踩规则坑
- **C 端侧**：购物车自动算出最优优惠组合，所见即所得，零误解
- **平台侧**：能办大促（618 / 双 11 跨店满减），数据可观测
- **技术侧**：领域模型干净、规则可扩展、与订单/库存/支付解耦

### 1.3 与现有模块的依赖关系

```
              ┌──────────────┐
              │ 优惠活动模块  │
              └──────┬───────┘
        ┌────────────┼─────────────┐
        ▼            ▼             ▼
   ┌─────────┐  ┌─────────┐  ┌──────────┐
   │ 商品/SKU│  │ 用户/会员│  │ 订单/购物车│
   │ (已有)   │  │ (已有)   │  │ (已有)    │
   └─────────┘  └─────────┘  └──────────┘
        │            │             │
        └────────────┼─────────────┘
                     ▼
              ┌──────────────┐
              │ 支付/钱包/退款│
              │  (已有, 需对接)│
              └──────────────┘
```

---

## 二、活动类型分类（对标京东/淘宝）

### 2.1 券类 Coupon —— "用户领了再用"

| 类型 | 说明 | 京东/淘宝对照 | 优先级 |
|---|---|---|---|
| **满减券** | 满 X 减 Y | 满 199 减 30 | P0 |
| **折扣券** | X 折，可设最高优惠上限 | 7.5 折券（最高减 50） | P0 |
| **立减券** | 无门槛直接减 X | 无门槛 5 元券 | P0 |
| **包邮券** | 免运费 | 包邮券 | P1 |
| **品类券** | 限指定品类商品 | 服饰类满减 | P1 |
| **店铺券** | 仅在本店有效 | 店铺优惠券 | P0 |
| **平台券** | 全平台通用 | 京东券 / 淘宝券 | P1 |
| **新人券** | 仅注册 N 天内首单可用 | 新人专享 | P1 |
| **定向券** | 按用户标签/等级精准发放 | 定向券 | P2 |

**领取方式**：
- 主动领取（券中心 / 商品详情页 / 店铺主页）
- 自动发放（注册 / 生日 / 复购触发）
- 任务奖励（签到 / 评价 / 分享）

### 2.2 促销活动 Promotion —— "商家发起、自动作用于商品"

| 类型 | 说明 | 京东/淘宝对照 | 优先级 |
|---|---|---|---|
| **限时秒杀** | 指定 SKU + 时段 + 库存抢 | 京东秒杀 / 淘抢购 | P0 |
| **满减/满折** | 满 X 减/折 Y，阶梯式 | 满 199 减 30 / 满 2 件 8 折 | P0 |
| **多件折扣** | 第 2 件半价 / 买 N 送 M | 第二件 0 元 / 买二送一 | P1 |
| **直降** | 原价 → 活动价 | 一口价 | P0 |
| **预售** | 定金 + 尾款 + 享优惠价 | 双 11 预售 | P2 |
| **套装组合** | A+B 组合特价 | 套餐价 | P2 |
| **拼团** | N 人成团享优惠价 | 拼多多式 / 淘宝拼团 | P2 |
| **砍价** | 社交分享降到底价 | 拼多多砍价 | P2 |

### 2.3 会员/积分体系 —— "持续运营"

| 类型 | 说明 | 优先级 |
|---|---|---|
| **会员等级** | 普通/银/金/钻，按消费额升级 | P1 |
| **会员价** | 不同等级看到不同价格 | P1 |
| **积分获取** | 下单 / 评价 / 签到送积分 | P1 |
| **积分兑换** | 积分换券 / 兑商品 / 直接抵现 | P2 |
| **会员日** | 每月 X 日会员专享折扣 | P2 |
| **成长值** | 与等级体系联动 | P2 |

### 2.4 大促框架 —— "平台维度的统筹"

| 能力 | 说明 | 优先级 |
|---|---|---|
| **跨店满减** | 平台主导，多店商品凑单满减 | P2 |
| **大促主会场** | 双 11 / 618 主题页聚合 | P2 |
| **预热/正式/返场** | 三阶段活动节奏 | P2 |
| **限购规则** | 每人/每日/每订单限购 N 件 | P1 |

### 2.5 优先级规划总览

- **P0 (MVP)**：店铺券、满减券、立减券、折扣券；限时秒杀、满减/满折、直降
- **P1**：包邮券、品类券、平台券、新人券；多件折扣；会员等级 + 会员价 + 积分；限购规则
- **P2**：定向券、预售、拼团、砍价、套装；会员日 + 成长值；大促框架；跨店满减

---

## 三、核心领域模型

### 3.1 实体关系图

```
   activity (活动主体)
       │ 1
       │
       │ N
   activity_rule (活动规则)
       │ 1                    1
       │                      │
       │ N                    N
   activity_target           activity_action
   (适用范围)                  (优惠动作)
   ├─ 全店                    ├─ 满减
   ├─ 指定 SKU                 ├─ 折扣
   ├─ 指定品类                 ├─ 立减
   └─ 指定用户                 ├─ 包邮
                              └─ 赠品

   coupon_template (券模板)
       │ 1
       │
       │ N
   coupon (用户领到的券)
       │
       └── user_id + status (未用/已用/过期)
```

### 3.2 关键表设计

#### `activity` —— 活动主体

| 字段 | 类型 | 说明 |
|---|---|---|
| id | bigint PK | |
| type | varchar(32) | seckill / fullreduce / discount / coupon / group_buy / ... |
| name | varchar(200) | 活动名称 |
| shop_id | bigint | 0 = 平台活动 |
| status | tinyint | 0 草稿 / 1 待开始 / 2 进行中 / 3 已结束 / 4 已下线 |
| start_time | bigint | 开始时间戳 |
| end_time | bigint | 结束时间戳 |
| priority | int | 多活动叠加时的优先级（数字大优先） |
| stackable | tinyint | 0 互斥 / 1 可叠加 |
| description | text | |
| create_time, update_time | datetime | |

#### `activity_target` —— 适用范围

| 字段 | 说明 |
|---|---|
| activity_id | FK |
| target_type | sku / category / shop / user_tag / all |
| target_id | sku_id / category_id / shop_id / tag_id / 0 |

> 例：满减活动「满 199 减 30」适用于 shop_id=1 全店 → activity_target(shop, 1)
> 例：秒杀活动只对 sku=88 → activity_target(sku, 88)

#### `activity_action` —— 优惠动作

| 字段 | 说明 |
|---|---|
| activity_id | FK |
| action_type | reduce(满减) / discount(折扣) / cash(直接减) / gift(赠品) / freeship(包邮) / fix_price(一口价) |
| threshold_type | none / amount / quantity |
| threshold_value | 满减门槛（满 X 元 / X 件） |
| benefit_value | 减 Y 元 / 打 Y 折（百分比） |
| max_discount | 最高优惠上限（折扣类必需） |
| gift_sku_id | 赠品 SKU（gift 类型用） |

> 阶梯满减用多行表示：满 199-30、满 299-50、满 499-100

#### `coupon_template` —— 券模板

| 字段 | 类型 | 说明 |
|---|---|---|
| id | bigint PK | |
| activity_id | bigint | 关联活动主体 (券也是 activity 的一种) |
| name | varchar(200) | |
| type | varchar(32) | full_reduce / discount / cash / freeship |
| value | bigint | 减/折面值 |
| min_amount | bigint | 门槛 |
| total_count | int | 总发放量 |
| received_count | int | 已领数 |
| used_count | int | 已使用数 |
| per_user_limit | int | 每人限领 |
| valid_type | tinyint | 0 固定日期 / 1 领取后 N 天 |
| valid_days | int | 领取后 N 天有效 |
| valid_start, valid_end | bigint | 固定日期模式 |
| receive_start, receive_end | bigint | 领取窗口 |

#### `coupon` —— 用户已领券

| 字段 | 说明 |
|---|---|
| id | PK |
| template_id | FK → coupon_template |
| user_id | 领券用户 |
| status | 0 未用 / 1 已锁定（下单中）/ 2 已使用 / 3 已过期 |
| order_id | 已使用时回填 |
| receive_time | 领取时间 |
| expire_time | 失效时间 |
| use_time | 使用时间 |

#### `seckill_session` —— 秒杀场次（专表，因高并发优化）

| 字段 | 说明 |
|---|---|
| id | PK |
| activity_id | |
| sku_id | |
| seckill_price | 秒杀价 |
| seckill_stock | 秒杀库存（Redis 预热 + 原子扣减） |
| limit_per_user | 每人限购 |
| start_time, end_time | |

---

## 四、核心业务规则

### 4.1 互斥与叠加

参考京东「平台券 + 店铺券 + 单品促销可叠加；同品类券互斥」。

```
SKU 级别活动 (秒杀 / 直降 / 一口价) — 互斥，取最低价
       ↓
店铺级活动 (店铺满减) — 在 SKU 价基础上加
       ↓
店铺券 — 可叠加在店铺满减后
       ↓
平台券 — 最后一层，可叠加
       ↓
积分抵现 — 最后用积分
```

**规则**：
1. **同优先级互斥**：秒杀 vs 直降 vs 一口价 三选一，按 activity.priority 最高的生效
2. **跨优先级可叠加**：SKU 级 + 店铺级 + 券 + 积分依次叠加
3. **券与券**：同 activity_id 的券每订单限用 1 张；不同 activity 的券（店铺+平台）可叠加
4. **赠品**：满减满折达到门槛触发赠品，不抵消金额

### 4.2 价格计算引擎

购物车选中商品后，价格计算顺序：

```
def calc_price(cart_items, user, selected_coupons):
    # 1. 每个 SKU 取活动后的成交价
    for item in cart_items:
        item.deal_price = min(
            item.original_price,
            seckill_price_if_active(item.sku_id),
            fix_price_if_active(item.sku_id),
            member_price_for(user, item.sku_id),
        )

    # 2. 按店铺分组算店铺活动
    for shop_id, items in group_by_shop(cart_items):
        total = sum(i.deal_price * i.qty for i in items)
        promo = best_shop_promotion(shop_id, total, items.count)
        shop_discount = promo.benefit_value if promo else 0
        shop_total = total - shop_discount

    # 3. 应用店铺券（在 shop_total 基础上）
    if selected_coupons.shop_coupon:
        validate(shop_coupon, shop_total)
        apply_coupon(shop_coupon, shop_total)

    # 4. 应用平台券（在 sum(shop_totals) 基础上）
    if selected_coupons.platform_coupon:
        ...

    # 5. 积分抵现（最后）
    if user.points_to_use:
        ...

    # 6. 运费 / 包邮券
    ...

    return final_amount, discount_breakdown
```

每一步的 `discount_breakdown` 要存到订单上，便于 C 端"明细展开"和退款时按比例退优惠。

### 4.3 退款时优惠分摊

- **整单退款**：整单优惠全部退给商家不可恢复（用户已用的券**不退回**，因为已锁定）
- **部分退款**：按 SKU 单价占比分摊优惠，退实付金额
- **券退回规则**：
  - 整单退款 + 券未参与计算（无法触发门槛）→ 券退回未用状态
  - 部分退款触发券退还：店铺券若退款后店铺金额低于券门槛，退给商家 + 退用户实付 + **不退券**（防套利）

### 4.4 限购规则

- **每人限购**：seckill_session.limit_per_user / activity 的 per_user_quota
- **每订单限购**：activity_rule.per_order_quota
- **每日限购**：activity_rule.per_day_quota（Redis 记日维度计数）
- **总库存**：seckill_session.seckill_stock（Redis 预热 + Lua 原子扣减）

### 4.5 反作弊

- **领券防刷**：IP+设备指纹+用户 ID 三维风控；新人券要求实名认证
- **秒杀防黄牛**：账号风控（昨日新注册 / 异地登录 / 关联号检测）
- **拼团防机器人**：分享链接带 token 签名 + 防重放
- **薅羊毛监控**：高频领券 / 大额低单价异常订单告警

---

## 五、用户流程

### 5.1 C 端用户视角

**领券**：
1. 在「优惠券中心」/ 商品详情 / 店铺主页看到券 → 点击领取
2. 领取成功 → 「我的券包」可见，状态 = 未使用 + 剩余有效期倒计时
3. 下单时 → 购物车自动勾选最优券组合 + 显示已优惠 ¥XX

**秒杀**：
1. 「秒杀」频道看到将开场的活动 → 设置提醒
2. 开场前 30s 进入活动页 → 倒计时
3. 开场瞬间点抢 → 进入排队 → 抢到进入下单页 → 5 分钟内必须支付否则释放库存

**拼团**（P2）：
1. 看到拼团商品 → 选择「开团」或「参团」
2. 开团 → 生成分享链接 → 分享给 N-1 人
3. N 人 24h 内付款成功 → 成团 → 享优惠价 + 正常发货
4. 24h 内未成团 → 全额退款

### 5.2 商家视角

**创建一个店铺满减活动**：
1. 进「营销中心」→「店铺满减」→「新建活动」
2. 填活动名、时间、规则（阶梯满减：满 199-30、满 299-50）
3. 选适用范围：全店 / 指定 SKU / 指定品类
4. 设互斥/叠加规则
5. 保存草稿 → 预览 → 提交（自动到时间生效）

**发券**：
1. 「优惠券」→「创建券模板」
2. 配置：类型、面值、门槛、有效期、发放量、每人限领
3. 选发放方式：用户主动领 / 注册自动发 / 商品详情页挂券位
4. 上架 → 监控领取/使用数据

**秒杀报名**（如果平台有大促）：
1. 平台公告「618 秒杀报名」→ 商家提交 SKU + 秒杀价
2. 平台审核（毛利率/库存达标）→ 通过则进入秒杀池
3. 活动当天到时间自动开抢

### 5.3 平台运营视角

**大促统筹**：
1. 配置大促主题（双 11 / 618）+ 时间窗 + 主会场链接
2. 配置跨店满减规则（满 300 减 40）
3. 审批参与活动的商家/SKU
4. 实时监控：参与店铺数 / GMV / UV / 转化率 / 投诉率

**数据看板**：
- 活动维度：UV、点击、加购、转化、GMV、ROI
- 券维度：发放量、领取率、使用率、过期率
- 商家维度：参加活动数、活动 GMV 占比
- 风控维度：领券异常、秒杀异常、退款异常

---

## 六、技术架构要点

### 6.1 服务边界

```
mall-promotion-rpc （新增）
  ├── activity 管理 (CRUD + 状态机)
  ├── coupon_template / coupon 管理
  ├── seckill 高并发模块 (独立, Redis + Lua)
  ├── 价格计算引擎 (CalcPrice RPC，被 cart / order 调用)
  └── 数据统计 (ReportAggregator)

mall-cart-rpc / mall-order-rpc （改造）
  ├── 算价改调 PromotionRpc.CalcPrice
  └── 下单时调 PromotionRpc.LockCoupon + ConsumeStock
```

### 6.2 高并发要点（秒杀）

1. **库存预热**：活动开始前把 seckill_stock 写 Redis（key: `seckill:{sku_id}`）
2. **原子扣减**：Lua 脚本 `if stock > 0 then DECR + LPUSH order_queue`
3. **异步落库**：扣库存后丢消息队列，order-rpc 消费创建订单
4. **限购**：用户参与时 `SETNX seckill:user:{user_id}:{sku_id} = 1 EX 86400`
5. **风控前置**：进入秒杀页前已做风控检查，扣库存阶段不再校验避免延迟

### 6.3 价格计算性能

- 一次 CalcPrice 调用要查：所有 SKU 的活动、用户的可用券、用户的会员等级
- 优化：活动数据全量 Redis 缓存（key 按 sku_id + shop_id），更新时双删
- 计算引擎走纯内存匹配 + 规则评估，目标 P99 < 20ms

### 6.4 数据一致性

- 券领取 → MySQL 主库写 `coupon` 表，Redis 写 `coupon:user:{uid}` set 加速查询
- 下单锁券 → status: 未用 → 已锁定（带 order_id），订单取消 5 分钟内自动解锁
- 退款 → 按 4.3 规则处理券回退

### 6.5 与现有模块的接口

| 调用方 | 调用 PromotionRpc 接口 | 触发场景 |
|---|---|---|
| mall-product-rpc | GetActivePromotions(sku_id) | 商品详情页显示活动标签/秒杀价 |
| mall-cart-rpc | CalcPrice(cart_items, user_id, coupons) | 购物车实时算价 |
| mall-order-rpc | LockCouponAndStock(order_id, ...) | 创建订单时锁券锁库存 |
| mall-order-rpc | ReleaseCouponAndStock(order_id) | 订单取消 / 超时释放 |
| mall-payment-rpc | ConfirmConsume(order_id) | 支付成功最终扣库存+核销券 |

---

## 七、分阶段实施

### Phase 1 — MVP (4 周)

**目标**：商家能办最基础的促销，C 端能领券用券

- 表：activity / activity_target / activity_action / coupon_template / coupon
- 活动类型：店铺满减、立减券、折扣券、直降（一口价）
- 价格引擎：基础叠加（活动价 + 店铺优惠 + 店铺券）
- 商家后台：营销中心首页 + 创建满减/优惠券两个工作流
- C 端：券中心、我的券包、购物车自动算价
- 订单：下单锁券 + 取消释放 + 退款基础规则

### Phase 2 — 限时秒杀 + 平台券 (3 周)

- 秒杀模块（Redis + Lua + 异步落库）
- 平台券（跨店通用）
- 包邮券 + 品类券
- 新人券 + 注册自动发券
- 限购规则（每人/每订单/每日）

### Phase 3 — 会员体系 + 高级玩法 (4 周)

- 会员等级 + 会员价
- 积分获取（下单送 / 签到 / 评价）+ 积分抵现
- 多件折扣（买二送一 / 第二件半价）
- 套装组合
- 数据看板：商家维度 + 平台维度

### Phase 4 — 大促与社交 (开放)

- 大促主会场 + 跨店满减
- 拼团 / 砍价
- 预售（定金+尾款）
- 反作弊深度（设备指纹 / 关联号检测）

---

## 八、风险与开放问题

### 8.1 已知风险

1. **价格计算正确性**：叠加规则复杂，必须有大量单元/集成测试覆盖
2. **秒杀高并发稳定性**：Redis Lua + 异步落库的兜底机制要充分
3. **退款分摊规则**：用户/商家都会算账，规则必须能解释清楚
4. **券与库存的一致性**：分布式锁 / 最终一致 / 补偿机制选型

### 8.2 待决策的产品问题

1. **同活动不同 SKU 不同价**：秒杀活动里 A 商品 5 折、B 商品 7 折，是一个 activity 多个 action，还是多个 activity？
2. **券是否可转让**：京东允许分享给朋友，淘宝不允许，yw-mall 选哪个？
3. **会员等级降级**：消费下降是否降级？京东不降，淘宝有降级机制
4. **积分到期**：滚动到期 vs 固定到期（每年 12.31 清零）

### 8.3 不在本 PRD 范围

- 直播带货优惠（独立模块）
- 分销/佣金体系
- 私域营销（企微 / 公众号）
- 广告/付费推广

---

## 九、下一步

建议产出顺序：
1. **本文档 review + 用户确认** —— 优先级 / 排期 / 不确定项决策
2. **Phase 1 详细技术设计** —— 写到 `2026-06-xx-promotion-phase1-design.md`
3. **数据库 DDL + proto 定义** —— mall-promotion-rpc 骨架
4. **价格计算引擎 + 单测** —— 这是核心逻辑，先 TDD 立住
5. **Phase 1 MVP 实现**
