# 商家工作台 M3 — 订单 + 物流模板 + 发货 实施计划

> **关联**: `plan/2026-05-27-merchant-workstation-design.md` 第 8 节 M3
> **For agentic workers**: 每个 Task 独立 commit，commit message 用 `feat(m3):` 前缀

**Goal**：商家可在 FE 看本店订单 + 筛状态 + 单条发货（选物流公司+面单号）+ 拒退；可 CRUD 物流模板。

**Architecture**：后端 RPC 完全就绪（order-rpc 25+ method + logistics 11 method）+ admin-api 11 个路由全注册。M3 几乎纯 FE：复刻 M2 同款 list/detail 模板，加 ship modal + freight template CRUD 页。零新增 proto / handler。

**Tech Stack**：Vue 3 + Element Plus + Pinia / TypeScript

---

## 已就绪资产（不重做）

- ✅ order-rpc: `ListShopOrders / GetShopOrder / ShipOrder / MarkShipped / MerchantRejectRefund` + 退款相关 8 个 method
- ✅ logistics-rpc: `CreateFreightTemplate / ListFreightTemplates / GetFreightTemplate / UpdateFreightTemplate / DeleteFreightTemplate` + shipment CRUD
- ✅ admin-api `/merchant/v1/orders` 4 路由（list/get/ship/reject-refund）+ batch-ship + freight-templates 5 路由
- ✅ DB 表全部就绪（order/shipment/freight_template/shipment_track）

## 缺口（M3 工作面）

| 缺口 | 范围 |
|---|---|
| FE orders 列表 | views/order/list.vue + 状态 tab + 分页 + 跳详情 |
| FE orders 详情 | views/order/detail.vue + 收货信息 + 商品列表 + 发货按钮 + 拒退按钮 |
| FE 发货 modal | 物流公司选择 + 面单号输入；调 ship endpoint |
| FE freight templates | views/freight/list.vue + 新建/编辑/删除 modal |
| FE api 层 | api/orders.ts + api/freight.ts |
| layout 解锁 | 订单管理 + 物流模板从 M3 占位解锁 |

---

## File Structure

```
yw-mall-admin-fe/merchant/src/
  api/
    orders.ts                CREATE — list/get/ship/rejectRefund
    freight.ts               CREATE — list/get/create/update/del
  views/
    order/
      list.vue               CREATE — 表格 + 状态 tab + 分页
      detail.vue             CREATE — 收货+商品+发货 modal+拒退
    freight/
      list.vue               CREATE — CRUD + modal
  layouts/AdminLayout.vue    MODIFY — 「订单」「物流模板」解锁
  router/index.ts            MODIFY — 加 4 个新路由
```

---

## Task 1: api/orders.ts + api/freight.ts

**Files**:
- Create: `src/api/orders.ts`
- Create: `src/api/freight.ts`

- [ ] **Step 1.1: 写 api/orders.ts**

```ts
import { get, post } from './request'
import type { ApiResponse } from '@/types/api'

export interface OrderItem {
  id: number; orderNo: string; userId: number
  totalAmount: number; status: number
  receiverName: string; receiverPhone: string
  receiverProvince: string; receiverCity: string
  receiverDistrict: string; receiverDetail: string
  trackingNo: string; carrier: string
  payTime: number; shipTime: number; createTime: number
  refundStatus: number
}
export interface OrderDetail extends OrderItem {
  refundReason?: string
  cancelTime?: number
  cancelReason?: string
  completeTime?: number
}

function unwrap<T>(b: ApiResponse<T> & T): T {
  return (b as ApiResponse<T>).data ?? (b as unknown as T)
}

export async function listOrders(params: { page?: number; pageSize?: number; status?: number }) {
  const qs = new URLSearchParams()
  qs.set('page', String(params.page ?? 1))
  qs.set('pageSize', String(params.pageSize ?? 20))
  if (params.status !== undefined && params.status >= 0) qs.set('status', String(params.status))
  const body = await get<ApiResponse<{ orders: OrderItem[]; total: number }> & { orders: OrderItem[]; total: number }>(
    `/orders?${qs.toString()}`,
  )
  return unwrap(body) as { orders: OrderItem[]; total: number }
}

export async function getOrder(id: number): Promise<OrderDetail> {
  const body = await get<ApiResponse<OrderDetail> & OrderDetail>(`/orders/${id}`)
  return unwrap(body) as OrderDetail
}

export async function shipOrder(id: number, carrier: string, trackingNo: string) {
  return post(`/orders/${id}/ship`, { carrier, trackingNo })
}

export async function rejectRefund(id: number, reason: string) {
  return post(`/orders/${id}/reject-refund`, { reason })
}
```

- [ ] **Step 1.2: 写 api/freight.ts**

```ts
import { get, post, put as putReq, del } from './request'
import type { ApiResponse } from '@/types/api'

export interface FreightTemplate {
  id: number; shopId: number; name: string
  calcType: number     // 1=按件, 2=按重(g)
  firstValue: number
  firstFee: number     // 分
  extraValue: number
  extraFee: number     // 分
  regions: string      // 省份逗号
  isDefault: boolean
  status: number
  createTime: number
}

function unwrap<T>(b: ApiResponse<T> & T): T {
  return (b as ApiResponse<T>).data ?? (b as unknown as T)
}

export async function listTemplates(): Promise<{ templates: FreightTemplate[]; total: number }> {
  const body = await get<ApiResponse<{ templates: FreightTemplate[]; total: number }> & { templates: FreightTemplate[]; total: number }>(
    '/freight-templates?page=1&pageSize=50',
  )
  return unwrap(body) as { templates: FreightTemplate[]; total: number }
}

export async function createTemplate(req: Omit<FreightTemplate, 'id' | 'shopId' | 'status' | 'createTime'>): Promise<{ id: number }> {
  const body = await post<ApiResponse<{ id: number }> & { id: number }>('/freight-templates', req)
  return unwrap(body) as { id: number }
}

export async function updateTemplate(id: number, req: Partial<FreightTemplate>) {
  return putReq(`/freight-templates/${id}`, req)
}

export async function deleteTemplate(id: number) {
  return del(`/freight-templates/${id}`)
}
```

- [ ] **Step 1.3: commit**

```
feat(m3 fe): api/orders.ts + api/freight.ts

OrderItem/OrderDetail/FreightTemplate 类型 + CRUD wrappers.
```

---

## Task 2: views/order/list.vue

**Files**: Create `src/views/order/list.vue`

- [ ] **Step 2.1: 写订单列表页**

```vue
<script setup lang="ts">
import { onMounted, ref, watch } from 'vue'
import { useRouter } from 'vue-router'
import PageContainer from '@/components/PageContainer.vue'
import { listOrders, type OrderItem } from '@/api/orders'

const router = useRouter()
const items = ref<OrderItem[]>([])
const total = ref(0)
const page = ref(1)
const pageSize = ref(20)
const status = ref<number>(-1)  // -1 = all
const loading = ref(false)

const STATUS_TABS = [
  { label: '全部', value: -1 },
  { label: '待付款', value: 0 },
  { label: '待发货', value: 1 },
  { label: '已发货', value: 2 },
  { label: '已完成', value: 3 },
  { label: '已取消', value: 4 },
]

const STATUS_LABEL: Record<number, string> = {
  0: '待付款', 1: '待发货', 2: '已发货', 3: '已完成', 4: '已取消',
}

async function reload() {
  loading.value = true
  try {
    const r = await listOrders({
      page: page.value, pageSize: pageSize.value,
      status: status.value >= 0 ? status.value : undefined,
    })
    items.value = r.orders
    total.value = r.total
  } finally {
    loading.value = false
  }
}

onMounted(reload)
watch([status, page], reload)

function fmtPrice(p: number) { return '¥' + (p / 100).toFixed(2) }
function fmtTs(ts: number) { return ts ? new Date(ts * 1000).toLocaleString() : '-' }
function goDetail(o: OrderItem) { router.push(`/orders/${o.id}`) }
</script>

<template>
  <PageContainer title="订单管理">
    <el-tabs v-model="status" @tab-change="(v: any) => (status = Number(v))" style="margin-bottom: 16px">
      <el-tab-pane v-for="t in STATUS_TABS" :key="t.value" :label="t.label" :name="t.value as any" />
    </el-tabs>

    <el-table v-loading="loading" :data="items" border @row-click="goDetail">
      <el-table-column prop="orderNo" label="订单号" width="200" />
      <el-table-column label="收件人" width="200">
        <template #default="{ row }">
          <div>{{ row.receiverName }} · {{ row.receiverPhone }}</div>
          <div class="muted">{{ row.receiverProvince }}{{ row.receiverCity }}{{ row.receiverDistrict }}</div>
        </template>
      </el-table-column>
      <el-table-column label="金额" width="120">
        <template #default="{ row }"><span class="price">{{ fmtPrice(row.totalAmount) }}</span></template>
      </el-table-column>
      <el-table-column label="状态" width="100">
        <template #default="{ row }"><el-tag>{{ STATUS_LABEL[row.status] || '-' }}</el-tag></template>
      </el-table-column>
      <el-table-column label="创建时间" width="180">
        <template #default="{ row }">{{ fmtTs(row.createTime) }}</template>
      </el-table-column>
      <el-table-column label="物流" min-width="200">
        <template #default="{ row }">
          <div v-if="row.carrier">{{ row.carrier }} · {{ row.trackingNo }}</div>
          <span v-else class="muted">—</span>
        </template>
      </el-table-column>
    </el-table>

    <div class="pager">
      <el-pagination v-model:current-page="page" :page-size="pageSize" :total="total"
        layout="prev, pager, next, jumper, total" background />
    </div>
  </PageContainer>
</template>

<style scoped>
.muted { color: #999; font-size: 12px }
.price { color: #e1251b; font-weight: 600 }
.pager { margin-top: 16px; display: flex; justify-content: flex-end }
.el-table :deep(.el-table__row) { cursor: pointer }
</style>
```

- [ ] **Step 2.2: build verify + commit**

```
feat(m3 fe): 订单列表页 /orders — 状态 tab + 表格 + 点击进详情
```

---

## Task 3: views/order/detail.vue + 发货 modal

**Files**: Create `src/views/order/detail.vue`

- [ ] **Step 3.1: 写详情 + ship modal**

```vue
<script setup lang="ts">
import { onMounted, ref, computed } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { ElMessage, ElMessageBox, type FormInstance, type FormRules } from 'element-plus'
import PageContainer from '@/components/PageContainer.vue'
import { useUserStore } from '@/stores/user'
import { getOrder, shipOrder, rejectRefund, type OrderDetail } from '@/api/orders'
import { listTemplates, type FreightTemplate } from '@/api/freight'

const route = useRoute()
const router = useRouter()
const user = useUserStore()

const order = ref<OrderDetail | null>(null)
const loading = ref(false)
const templates = ref<FreightTemplate[]>([])

const STATUS_LABEL: Record<number, string> = {
  0: '待付款', 1: '待发货', 2: '已发货', 3: '已完成', 4: '已取消',
}
const REFUND_LABEL: Record<number, string> = {
  0: '无退款', 1: '申请中', 2: '协商中', 3: '已退款', 4: '已驳回',
}
const CARRIERS = ['顺丰', '京东', '中通', '圆通', '韵达', '申通', 'EMS', '其他']

const shipDialog = ref(false)
const shipFormRef = ref<FormInstance>()
const shipForm = ref({ carrier: '顺丰', trackingNo: '' })
const shipLoading = ref(false)
const shipRules: FormRules = {
  carrier: [{ required: true, message: '请选择物流公司', trigger: 'change' }],
  trackingNo: [{ required: true, message: '请填面单号', trigger: 'blur' }],
}

const rejectDialog = ref(false)
const rejectReason = ref('')
const rejectLoading = ref(false)

const canShip = computed(() => order.value?.status === 1)
const canRejectRefund = computed(() => order.value && order.value.refundStatus === 1)

async function reload() {
  loading.value = true
  try {
    const id = Number(route.params.id)
    order.value = await getOrder(id)
  } finally {
    loading.value = false
  }
}

onMounted(async () => {
  reload()
  // 加载物流模板用作下拉提示（M3 不强制绑模板）
  try { templates.value = (await listTemplates()).templates } catch { /* OK */ }
})

function fmtPrice(p: number) { return '¥' + (p / 100).toFixed(2) }
function fmtTs(ts: number) { return ts ? new Date(ts * 1000).toLocaleString() : '-' }

function openShip() {
  shipForm.value = { carrier: '顺丰', trackingNo: '' }
  shipDialog.value = true
}

async function submitShip() {
  if (!shipFormRef.value || !order.value) return
  try { await shipFormRef.value.validate() } catch { return }
  shipLoading.value = true
  try {
    await shipOrder(order.value.id, shipForm.value.carrier, shipForm.value.trackingNo)
    ElMessage.success('已发货')
    shipDialog.value = false
    await reload()
  } finally { shipLoading.value = false }
}

function openReject() {
  rejectReason.value = ''
  rejectDialog.value = true
}

async function submitReject() {
  if (!order.value) return
  if (!rejectReason.value.trim()) {
    ElMessage.error('请填拒绝理由')
    return
  }
  rejectLoading.value = true
  try {
    await rejectRefund(order.value.id, rejectReason.value)
    ElMessage.success('已拒绝退款')
    rejectDialog.value = false
    await reload()
  } finally { rejectLoading.value = false }
}
</script>

<template>
  <PageContainer title="订单详情" v-loading="loading">
    <div v-if="order">
      <div class="header-bar">
        <span class="order-no">订单号: {{ order.orderNo }}</span>
        <span>
          <el-button v-if="canShip && user.hasPerm('order.ship')" type="primary" @click="openShip">发货</el-button>
          <el-button v-if="canRejectRefund && user.hasPerm('refund.handle')" type="danger" plain @click="openReject">
            拒绝退款
          </el-button>
          <el-button @click="$router.back()">返回</el-button>
        </span>
      </div>

      <el-descriptions :column="2" border style="margin-bottom: 16px">
        <el-descriptions-item label="订单状态"><el-tag>{{ STATUS_LABEL[order.status] }}</el-tag></el-descriptions-item>
        <el-descriptions-item label="退款状态"><el-tag type="warning">{{ REFUND_LABEL[order.refundStatus] }}</el-tag></el-descriptions-item>
        <el-descriptions-item label="金额"><span class="price">{{ fmtPrice(order.totalAmount) }}</span></el-descriptions-item>
        <el-descriptions-item label="创建时间">{{ fmtTs(order.createTime) }}</el-descriptions-item>
        <el-descriptions-item label="付款时间">{{ fmtTs(order.payTime) }}</el-descriptions-item>
        <el-descriptions-item label="发货时间">{{ fmtTs(order.shipTime) }}</el-descriptions-item>
        <el-descriptions-item label="物流" :span="2">
          <span v-if="order.carrier">{{ order.carrier }} · {{ order.trackingNo }}</span>
          <span v-else class="muted">未发货</span>
        </el-descriptions-item>
        <el-descriptions-item label="收件人">{{ order.receiverName }} · {{ order.receiverPhone }}</el-descriptions-item>
        <el-descriptions-item label="收件地址">
          {{ order.receiverProvince }}{{ order.receiverCity }}{{ order.receiverDistrict }}{{ order.receiverDetail }}
        </el-descriptions-item>
        <el-descriptions-item v-if="order.refundReason" label="退款理由" :span="2">{{ order.refundReason }}</el-descriptions-item>
      </el-descriptions>
    </div>

    <!-- Ship modal -->
    <el-dialog v-model="shipDialog" title="发货" width="480px">
      <el-form ref="shipFormRef" :model="shipForm" :rules="shipRules" label-width="100px">
        <el-form-item label="物流公司" prop="carrier">
          <el-select v-model="shipForm.carrier" style="width: 100%">
            <el-option v-for="c in CARRIERS" :key="c" :label="c" :value="c" />
          </el-select>
        </el-form-item>
        <el-form-item label="面单号" prop="trackingNo">
          <el-input v-model="shipForm.trackingNo" placeholder="请填快递单号" />
        </el-form-item>
        <el-form-item v-if="templates.length" label="物流模板">
          <span class="muted">已配置 {{ templates.length }} 个模板（M3 仅展示，不强制绑定）</span>
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="shipDialog = false">取消</el-button>
        <el-button type="primary" :loading="shipLoading" @click="submitShip">确认发货</el-button>
      </template>
    </el-dialog>

    <!-- Reject refund modal -->
    <el-dialog v-model="rejectDialog" title="拒绝退款" width="480px">
      <el-input v-model="rejectReason" type="textarea" :rows="4" placeholder="请简述拒绝理由（会通知给买家）" />
      <template #footer>
        <el-button @click="rejectDialog = false">取消</el-button>
        <el-button type="danger" :loading="rejectLoading" @click="submitReject">确认拒绝</el-button>
      </template>
    </el-dialog>
  </PageContainer>
</template>

<style scoped>
.header-bar { display: flex; justify-content: space-between; align-items: center; margin-bottom: 16px }
.order-no { font-size: 15px; color: #333; font-weight: 600 }
.muted { color: #999 }
.price { color: #e1251b; font-weight: 600; font-size: 16px }
</style>
```

- [ ] **Step 3.2: commit**

```
feat(m3 fe): 订单详情 + 发货 modal + 拒退 modal
```

---

## Task 4: views/freight/list.vue 物流模板 CRUD

**Files**: Create `src/views/freight/list.vue`

- [ ] **Step 4.1: 写模板页**

```vue
<script setup lang="ts">
import { onMounted, ref } from 'vue'
import { ElMessage, ElMessageBox, type FormInstance, type FormRules } from 'element-plus'
import PageContainer from '@/components/PageContainer.vue'
import { useUserStore } from '@/stores/user'
import {
  listTemplates, createTemplate, updateTemplate, deleteTemplate,
  type FreightTemplate,
} from '@/api/freight'

const user = useUserStore()
const items = ref<FreightTemplate[]>([])
const loading = ref(false)
const dialogVisible = ref(false)
const editing = ref<FreightTemplate | null>(null)
const formRef = ref<FormInstance>()
const submitting = ref(false)

const form = ref({
  name: '', calcType: 1,
  firstValue: 1, firstFee: 0,
  extraValue: 1, extraFee: 0,
  regions: '全国', isDefault: false,
})

const rules: FormRules = {
  name: [{ required: true, message: '模板名必填', trigger: 'blur' }],
  regions: [{ required: true, message: '至少填一个地区', trigger: 'blur' }],
}

const CALC_LABEL: Record<number, string> = { 1: '按件数', 2: '按重量(g)' }

async function reload() {
  loading.value = true
  try { items.value = (await listTemplates()).templates } finally { loading.value = false }
}
onMounted(reload)

function openNew() {
  editing.value = null
  form.value = { name: '', calcType: 1, firstValue: 1, firstFee: 0,
    extraValue: 1, extraFee: 0, regions: '全国', isDefault: false }
  dialogVisible.value = true
}

function openEdit(row: FreightTemplate) {
  editing.value = row
  form.value = {
    name: row.name, calcType: row.calcType,
    firstValue: row.firstValue, firstFee: row.firstFee,
    extraValue: row.extraValue, extraFee: row.extraFee,
    regions: row.regions, isDefault: row.isDefault,
  }
  dialogVisible.value = true
}

async function submit() {
  if (!formRef.value) return
  try { await formRef.value.validate() } catch { return }
  submitting.value = true
  try {
    if (editing.value) {
      await updateTemplate(editing.value.id, form.value)
      ElMessage.success('已保存')
    } else {
      await createTemplate(form.value)
      ElMessage.success('已创建')
    }
    dialogVisible.value = false
    await reload()
  } finally { submitting.value = false }
}

async function onDelete(row: FreightTemplate) {
  try {
    await ElMessageBox.confirm(`确定删除模板「${row.name}」？`, '删除', {
      type: 'warning', confirmButtonText: '删除', cancelButtonText: '取消',
    })
  } catch { return }
  await deleteTemplate(row.id)
  ElMessage.success('已删除')
  await reload()
}

function fmtFee(f: number) { return '¥' + (f / 100).toFixed(2) }
</script>

<template>
  <PageContainer title="物流模板">
    <div style="display: flex; justify-content: space-between; margin-bottom: 16px">
      <span class="muted">共 {{ items.length }} 个模板（默认模板显示在最前）</span>
      <el-button v-if="user.hasPerm('freight.write') || user.hasPerm('*')" type="primary" @click="openNew">
        + 新建模板
      </el-button>
    </div>

    <el-table v-loading="loading" :data="items" border>
      <el-table-column prop="name" label="名称" min-width="160">
        <template #default="{ row }">
          {{ row.name }}
          <el-tag v-if="row.isDefault" size="small" type="success" style="margin-left: 4px">默认</el-tag>
        </template>
      </el-table-column>
      <el-table-column label="计费方式" width="120">
        <template #default="{ row }">{{ CALC_LABEL[row.calcType] }}</template>
      </el-table-column>
      <el-table-column label="首段">
        <template #default="{ row }">{{ row.firstValue }} → {{ fmtFee(row.firstFee) }}</template>
      </el-table-column>
      <el-table-column label="续段">
        <template #default="{ row }">+{{ row.extraValue }} → +{{ fmtFee(row.extraFee) }}</template>
      </el-table-column>
      <el-table-column prop="regions" label="覆盖地区" min-width="180" />
      <el-table-column label="操作" width="180">
        <template #default="{ row }">
          <el-button size="small" type="primary" link @click="openEdit(row)">编辑</el-button>
          <el-button size="small" type="danger" link @click="onDelete(row)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>

    <el-dialog v-model="dialogVisible" :title="editing ? '编辑模板' : '新建模板'" width="560px">
      <el-form ref="formRef" :model="form" :rules="rules" label-width="100px">
        <el-form-item label="模板名" prop="name"><el-input v-model="form.name" /></el-form-item>
        <el-form-item label="计费方式">
          <el-radio-group v-model="form.calcType">
            <el-radio :value="1">按件数</el-radio>
            <el-radio :value="2">按重量(g)</el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="首段">
          <el-input-number v-model="form.firstValue" :min="1" style="width: 100px" />
          <span style="margin: 0 8px">单价 (元):</span>
          <el-input
            :model-value="(form.firstFee / 100).toFixed(2)"
            @update:model-value="(v: string) => (form.firstFee = Math.round(Number(v || 0) * 100))"
            style="width: 120px"
          />
        </el-form-item>
        <el-form-item label="续段每增">
          <el-input-number v-model="form.extraValue" :min="1" style="width: 100px" />
          <span style="margin: 0 8px">加价 (元):</span>
          <el-input
            :model-value="(form.extraFee / 100).toFixed(2)"
            @update:model-value="(v: string) => (form.extraFee = Math.round(Number(v || 0) * 100))"
            style="width: 120px"
          />
        </el-form-item>
        <el-form-item label="覆盖地区" prop="regions">
          <el-input v-model="form.regions" placeholder="如：广东,浙江 / 全国" />
        </el-form-item>
        <el-form-item label="设为默认"><el-switch v-model="form.isDefault" /></el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="dialogVisible = false">取消</el-button>
        <el-button type="primary" :loading="submitting" @click="submit">保存</el-button>
      </template>
    </el-dialog>
  </PageContainer>
</template>

<style scoped>
.muted { color: #999; font-size: 13px }
</style>
```

- [ ] **Step 4.2: commit**

```
feat(m3 fe): 物流模板 CRUD /freight
```

---

## Task 5: 路由 + nav 解锁 + 整体 build verify

**Files**:
- Modify: `src/router/index.ts`（加 3 个路由）
- Modify: `src/layouts/AdminLayout.vue`（订单 + 物流模板 disabled → enabled）

- [ ] **Step 5.1: router**

```ts
{ path: 'orders', component: () => import('@/views/order/list.vue'),
  meta: { title: '订单管理', perms: ['order.read'] } },
{ path: 'orders/:id', component: () => import('@/views/order/detail.vue'),
  meta: { title: '订单详情', perms: ['order.read'] } },
{ path: 'freight', component: () => import('@/views/freight/list.vue'),
  meta: { title: '物流模板', perms: ['freight.read'] } },
```

- [ ] **Step 5.2: AdminLayout nav 解锁**

```ts
// 把 disabled: true 删掉，title 「(M3)」标记去掉
{ path: '/orders', title: '订单管理', icon: 'List', perm: 'order.read' },
{ path: '/freight', title: '物流模板', icon: 'Van', perm: 'freight.read' },
```

- [ ] **Step 5.3: pnpm run build**

```
✓ 0 错
```

- [ ] **Step 5.4: commit + push**

```
feat(m3 fe): 解锁订单/物流 nav + 路由 + e2e build
```

---

## 验收清单

- [ ] alice (owner) 进 /orders 看订单列表（当前 0 单，UI 跑通）
- [ ] 状态 tab 切换 → 接口参数 status 跟着变
- [ ] 点击行进 /orders/:id 详情
- [ ] 进 /freight 看模板列表（M3 之前 e2e 创建过 1 个「全国包邮」）
- [ ] 编辑/删除模板生效
- [ ] bob (warehouse, perm=order.ship) 可发货，但无 freight.write 不能新建模板
- [ ] FE build vue-tsc 0 错

---

## 关联文档

- `plan/2026-05-27-merchant-workstation-design.md` — 整体设计 M3
- `plan/2026-05-27-merchant-workstation-M2-plan.md` — M2 商品（FE 同模板）
- `daily/2026-05-27-night.md` — M2 交付参考
