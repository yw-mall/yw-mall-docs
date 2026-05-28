# 商家工作台 M4 — 退款 3 类工作流 实施计划

> **关联**: `plan/2026-05-27-merchant-workstation-design.md` 第 8 节 M4
> **For agentic workers**: 每个 Task 独立 commit，commit message 用 `feat(m4):` 前缀

**Goal**：商家可在 FE 看本店退款工单 + 处理三类工作流——仅退款（同意/拒绝直接退）/ 退货退款（同意+等用户寄回+验货+退）/ 换货（同意+等用户寄回+商家寄换货）。

**Architecture**：后端 RPC 完全就绪（`ListShopRefundRequests / GetRefundRequest / MerchantHandleRefund / MerchantInspectReturn / MerchantShipExchange`）+ admin-api 已有 list/handle 2 端点。M4 缺 3 个 admin-api 端点（detail / inspect / ship-exchange）+ FE 全套。

**Tech Stack**：Go 1.26 / go-zero / Vue 3 + Element Plus

---

## 已就绪资产

- ✅ order-rpc 5 个 refund method (List/Get/Handle/InspectReturn/ShipExchange)
- ✅ admin-api `/refunds` (list) + `/refunds/:id/handle` (同意/驳回)
- ✅ DB `refund_request` 表完整（status/refund_type/return_tracking_no 等）

## 缺口

| 缺口 | 范围 |
|---|---|
| BE: `GET /merchant/v1/refunds/:id` 详情 | 新 handler + logic |
| BE: `POST /merchant/v1/refunds/:id/inspect` 验货 | 新 handler + logic |
| BE: `POST /merchant/v1/refunds/:id/ship-exchange` 换货发货 | 新 handler + logic |
| FE: api/refunds.ts | list/get/handle/inspect/shipExchange |
| FE: views/refund/list.vue | 状态 tab + 表格 |
| FE: views/refund/detail.vue | stepper + 3 类工作流 action 按钮 |
| FE: 路由 + nav 解锁 | `/refunds` 从 (M4) 占位 |

---

## Task 1: admin-api 3 个新 endpoint

**Files**:
- Modify: `internal/types/types.go`（加 InspectReq / ShipExchangeReq）
- Modify: `internal/logic/refunds_logic.go`（加 3 个 logic）
- Modify: `internal/handler/handlers.go`（加 3 个 handler）
- Modify: `internal/handler/routes.go`（加 3 个路由）

- [ ] **Step 1.1: types**

```go
type InspectReturnReq struct {
    Passed bool   `json:"passed"`
    Remark string `json:"remark,optional"`
}

type ShipExchangeReq struct {
    TrackingNo string `json:"trackingNo"`
    Carrier    string `json:"carrier"`
}
```

- [ ] **Step 1.2: refunds_logic.go 加 3 函数**

```go
func GetRefundDetail(ctx context.Context, svcCtx *svc.ServiceContext, id int64) (*types.RefundInfo, error) {
    c, _ := middleware.ClaimsFromContext(ctx)
    if c == nil || c.ShopId <= 0 { return nil, errors.New("not in a shop") }
    r, err := svcCtx.OrderRpc.GetRefundRequest(ctx, &orderclient.GetRefundRequestReq{Id: id})
    if err != nil { return nil, err }
    if r.ShopId != c.ShopId { return nil, errors.New("refund not in this shop") }
    return refundProtoToInfo(r), nil
}

func MerchantInspectReturn(ctx context.Context, svcCtx *svc.ServiceContext, id int64, req *types.InspectReturnReq) (*types.OkResp, error) {
    c, _ := middleware.ClaimsFromContext(ctx)
    if c == nil || c.ShopId <= 0 { return nil, errors.New("not in a shop") }
    if !hasPerm(c, "refund.inspect") { return nil, errors.New("permission denied: refund.inspect") }
    if _, err := svcCtx.OrderRpc.MerchantInspectReturn(ctx, &orderclient.MerchantInspectReturnReq{
        RefundId: id, ShopId: c.ShopId, MerchantUserId: c.Uid,
        Passed: req.Passed, Remark: req.Remark,
    }); err != nil { return nil, err }
    return &types.OkResp{Ok: true}, nil
}

func MerchantShipExchange(ctx context.Context, svcCtx *svc.ServiceContext, id int64, req *types.ShipExchangeReq) (*types.OkResp, error) {
    c, _ := middleware.ClaimsFromContext(ctx)
    if c == nil || c.ShopId <= 0 { return nil, errors.New("not in a shop") }
    if !hasPerm(c, "order.ship") { return nil, errors.New("permission denied: order.ship") }
    if _, err := svcCtx.OrderRpc.MerchantShipExchange(ctx, &orderclient.MerchantShipExchangeReq{
        RefundId: id, ShopId: c.ShopId, MerchantUserId: c.Uid,
        TrackingNo: req.TrackingNo, Carrier: req.Carrier,
    }); err != nil { return nil, err }
    return &types.OkResp{Ok: true}, nil
}
```

- [ ] **Step 1.3: handler 3 个 + routes**

handlers.go 加：
```go
func merchantGetRefundDetailHandler(svcCtx *svc.ServiceContext) http.HandlerFunc { ... }
func merchantInspectReturnHandler(svcCtx *svc.ServiceContext) http.HandlerFunc { ... }
func merchantShipExchangeHandler(svcCtx *svc.ServiceContext) http.HandlerFunc { ... }
```

routes.go 在 /refunds 块下加：
```go
{Method: http.MethodGet,  Path: "/refunds/:id", Handler: merchantGetRefundDetailHandler(svcCtx)},
{Method: http.MethodPost, Path: "/refunds/:id/inspect", Handler: merchantInspectReturnHandler(svcCtx)},
{Method: http.MethodPost, Path: "/refunds/:id/ship-exchange", Handler: merchantShipExchangeHandler(svcCtx)},
```

- [ ] **Step 1.4: build + restart + e2e**

```
cd yw-mall-admin && go build ./...
./start.sh restart 2>&1 | grep admin-api
T=$(merchant login)
curl -H "Authorization: Bearer $T" /merchant/v1/refunds → list 0
```

- [ ] **Step 1.5: commit**

```
feat(m4 admin-api): /refunds/:id detail + /inspect + /ship-exchange
```

---

## Task 2: FE api/refunds.ts

**Files**: Create `src/api/refunds.ts`

- [ ] **Step 2.1**

```ts
import { get, post } from './request'
import type { ApiResponse } from '@/types/api'

export interface RefundItem {
  id: number; orderId: number; orderNo: string
  userId: number; shopId: number; amount: number
  reason: string; status: number; refundType: number
  evidence?: string[]
  merchantHandleTime?: number; merchantRemark?: string
  returnTrackingNo?: string; appealReason?: string
  createTime: number; updateTime: number
}
export interface RefundDetail extends RefundItem {
  items?: { productId: number; productName: string; quantity: number; price: number }[]
  adminRemark?: string
  refundCompleteTime?: number
}

function unwrap<T>(b: ApiResponse<T> & T): T {
  if (b && (b as ApiResponse<T>).data !== undefined) return (b as ApiResponse<T>).data as T
  return b as unknown as T
}

export async function listRefunds(params: { page?: number; pageSize?: number; status?: number }) {
  const qs = new URLSearchParams()
  qs.set('page', String(params.page ?? 1))
  qs.set('pageSize', String(params.pageSize ?? 20))
  if (params.status !== undefined && params.status >= 0) qs.set('status', String(params.status))
  type Body = { refunds: RefundItem[]; total: number }
  const body = await get<ApiResponse<Body> & Body>(`/refunds?${qs.toString()}`)
  return unwrap(body) as Body
}

export async function getRefund(id: number): Promise<RefundDetail> {
  const body = await get<ApiResponse<RefundDetail> & RefundDetail>(`/refunds/${id}`)
  return unwrap(body) as RefundDetail
}

export async function handleRefund(id: number, agreed: boolean, remark: string) {
  return post(`/refunds/${id}/handle`, { agreed, remark })
}

export async function inspectReturn(id: number, passed: boolean, remark: string) {
  return post(`/refunds/${id}/inspect`, { passed, remark })
}

export async function shipExchange(id: number, carrier: string, trackingNo: string) {
  return post(`/refunds/${id}/ship-exchange`, { carrier, trackingNo })
}
```

---

## Task 3: views/refund/list.vue + detail.vue

详见代码块（参考 M3 order/list + detail 同款模板）。
detail.vue 关键：4 类 status × 3 类 refund_type 的状态机决定显示哪个 action：

| status | refund_type=1 仅退款 | =2 退货退款 | =3 换货 |
|---|---|---|---|
| 0 待商家处理 | 同意（退款）/ 拒绝 | 同意（待用户寄回）/ 拒绝 | 同意（待用户寄回）/ 拒绝 |
| 1 同意（待寄回） | — | 验货按钮（passed/不 passed）| 验货按钮 |
| 2 用户已寄回 | — | 验货 + 同款 | 验货 + 同款 |
| 3 验货通过待发货换货 | — | 等待退款（自动）| 换货发货按钮 |
| 3 已完成 / 已驳回 / 已退款 | 只读 | 只读 | 只读 |

UI 用 el-steps 显示进度 + 底部 action 按钮组按状态条件渲染。

---

## Task 4: 路由 + nav 解锁 + build

```ts
// router
{ path: 'refunds', component: () => import('@/views/refund/list.vue'),
  meta: { title: '退款处理', perms: ['refund.read'] } },
{ path: 'refunds/:id', component: () => import('@/views/refund/detail.vue'),
  meta: { title: '退款详情', perms: ['refund.read'] } },

// AdminLayout 菜单
{ path: '/refunds', title: '退款处理', icon: 'RefreshLeft', perm: 'refund.read' },
```

---

## 验收

- [ ] alice 进 /refunds 看列表（0 单时空表）
- [ ] 进详情看到 el-steps 进度
- [ ] perm 守卫：bob (warehouse, 无 refund.handle) 看不到「同意」按钮，看得到「验货」（refund.inspect）

---

## 关联

- `plan/2026-05-28-merchant-workstation-M3-plan.md` — 同款 FE 模板
- `daily/2026-05-28.md` — M3 交付
