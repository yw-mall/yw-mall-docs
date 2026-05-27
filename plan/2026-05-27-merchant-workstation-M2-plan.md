# 商家工作台 M2 — 商品 + 多 SKU + 图片上传 实施计划

> **关联**: `plan/2026-05-27-merchant-workstation-design.md` 第 8 节 M2
> **For agentic workers**: 每个 Task 独立 commit，commit message 用 `feat(m2):` 前缀

**Goal**：商品支持多 SKU 多规格上传 + 图片上传到 MinIO；商家可在 FE 用矩阵生成器创建/编辑 SPU + 自动生成 SKU 矩阵；上下架 + 库存调整保持单 SKU 兼容路径。

**Architecture**：新建 `sku` 表挂在 `product` 表下，product 行保留 price/stock/images 作为「default SKU」聚合视图，避免 C 端读路径大改造。多 SKU 商品 product.price 取 SKU min price、stock 取 SKU sum、images 取 sku.images 合并。`mall-api` 复用现成 `/upload/review-media` 模式新增 `/upload/product-image` 给 merchant FE 用（出于 c-side cookie 隔离不放 admin-api）。FE 复刻 admin 同款表单 + 自研「属性矩阵 → SKU 笛卡尔积生成器」组件。

**Tech Stack**：Go 1.26 / go-zero 1.10.1 / proto3 / MySQL 9 / MinIO 7 / Vue 3 + Element Plus + Pinia / TypeScript

---

## 已就绪资产（不重做）

- ✅ `/merchant/v1/products` 6 个路由（list/create/get/update/status/stock）
- ✅ product-rpc `merchantListProducts/createProduct/updateProduct/setProductStatus/setProductStock/lockSku/unlockSku/getProductDetail` 等 15 个 method
- ✅ mall-common/minioutil 已有；mall-api `/upload/review-media` 是现成模板
- ✅ M1 FE 京东风脚手架 + MerchantLayout + perm 守卫已就位

## 真实缺口（M2 工作面）

| 缺口 | 范围 |
|---|---|
| sku 表不存在 | mall_product 库新增 sku 表 + 索引 |
| createProduct 不接 SKU | proto + logic 加 `skus []SkuInput` 字段 |
| SKU 批量 upsert | 新 RPC `BatchUpsertSkus`（编辑商品时改 SKU 矩阵） |
| 图片上传 endpoint | admin-api 加 `/merchant/v1/upload/image` + minioutil bridge |
| 单 SKU → 多 SKU 兼容读路径 | product 详情接口加 `skus` 字段；list 接口聚合 min price + sum stock |
| FE 商品列表页 | 表格 + 上下架 + 库存调整 inline + 编辑跳转 |
| FE 商品编辑页 | name/描述/分类/品牌 + 多规格属性矩阵 + SKU 生成器 + 图片上传 |
| FE 通用上传组件 | 复用 Element `el-upload` 包装 |

## File Structure

```
yw-mall (backend)
  mall-product-rpc/sql/
    sku.sql                                  CREATE — 新表 + 索引

  mall-product-rpc/cmd/backfill_default_sku/
    main.go                                  CREATE — 历史 product 行回填 1 个默认 SKU

  mall-common/proto/product/
    product.proto                            MODIFY — 加 SkuInput/SkuItem messages + BatchUpsertSkus rpc
                                                       + CreateProductReq.skus + ProductDetailResp.skus

  mall-product-rpc/internal/logic/
    skuhelpers.go                            CREATE — sku CRUD 通用 helper
    batchupsertskuslogic.go                  CREATE
    createproductlogic.go                    MODIFY — 落 default SKU; 多 SKU 时聚合 price/stock
    updateproductlogic.go                    MODIFY — 改 SPU 字段不动 SKU
    setproductstocklogic.go                  MODIFY — 单 SKU 路径保留兼容
    getproductdetaillogic.go                 MODIFY — 返回 skus[] 字段
    listshopproductslogic.go                 MODIFY — 聚合 sku.stock SUM 给 list 视图
  mall-product-rpc/internal/server/
    productserviceserver.go                  AUTO-REGEN
  mall-product-rpc/product/
    product.pb.go / product_grpc.pb.go       AUTO-REGEN
  mall-product-rpc/productservice/
    productservice.go                        MANUAL — 加 BatchUpsertSkus wrapper

  yw-mall-admin/internal/types/types.go      MODIFY — Sku/SkuInput DTO + MerchantCreateProductReq.skus
                                                      + MerchantUpdateProductReq.skus
                                                      + UploadImageResp
  yw-mall-admin/internal/logic/
    merchant_products_logic.go               MODIFY — Create/Update 透传 skus
    merchant_upload_logic.go                 CREATE — 校文件大小+类型 + minioutil.Put + 返 URL
  yw-mall-admin/internal/handler/
    merchant_upload_handler.go               CREATE
    routes.go                                MODIFY — 加 POST /upload/image + POST /products/:id/skus

  start.sh                                   MODIFY — SPRINT_MIGRATIONS 加 sku.sql

yw-mall-admin-fe/merchant/src/
  api/products.ts                            CREATE — list/get/create/update/setStatus/setStock + upsertSkus
  api/upload.ts                              CREATE — uploadImage 包装 axios POST multipart
  components/SkuMatrix.vue                   CREATE — 属性 → SKU 笛卡尔积生成器
  components/ImageUploader.vue               CREATE — el-upload 包装，调 api/upload.ts
  views/product/
    list.vue                                 CREATE — 列表 + 操作
    edit.vue                                 CREATE — SPU 表单 + SkuMatrix + ImageUploader
  router/index.ts                            MODIFY — 加 /products + /products/:id/edit + /products/new
  layouts/AdminLayout.vue                    MODIFY — 商品 nav 项从 disabled 改 active
```

---

## Task 1: SKU 数据表

**Files**:
- Create: `mall-product-rpc/sql/sku.sql`
- Modify: `yw-mall/start.sh` (SPRINT_MIGRATIONS)

- [ ] **Step 1.1: 写 DDL**

`mall-product-rpc/sql/sku.sql`:
```sql
-- M2 商品多 SKU：一个 product 对应 0..N 个 sku 行。
-- product 表保持单 SKU 兼容字段（price/stock/images = 默认 SKU 聚合视图）。

CREATE TABLE IF NOT EXISTS sku (
  id              BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  product_id      BIGINT UNSIGNED NOT NULL,
  shop_id         BIGINT UNSIGNED NOT NULL,         -- 冗余，按店铺查 SKU 时用
  sku_code        VARCHAR(64) NOT NULL,             -- 商家自定义编码 / 或 P{pid}-S{idx} 自动生成
  spec_text       VARCHAR(255) NOT NULL DEFAULT '', -- 规格展示文本「黑色/256GB」
  spec_json       VARCHAR(500) NOT NULL DEFAULT '', -- 规格 JSON {"颜色":"黑色","存储":"256GB"}
  price           BIGINT NOT NULL DEFAULT 0,        -- 分
  stock           BIGINT NOT NULL DEFAULT 0,
  image           VARCHAR(500) NOT NULL DEFAULT '', -- SKU 主图（可与 product.images 不同）
  status          TINYINT NOT NULL DEFAULT 1,       -- 1=active, 0=disabled (软删)
  create_time     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  update_time     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  KEY idx_product_status (product_id, status),
  KEY idx_shop (shop_id, status),
  UNIQUE KEY uk_product_code (product_id, sku_code)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

- [ ] **Step 1.2: start.sh 注入**

```bash
"mall_product|mall-product-rpc/sql/sku.sql"
```

加到 `SPRINT_MIGRATIONS` 数组末尾。

- [ ] **Step 1.3: 跑迁移**

```bash
cd ~/workspace/go/mall/yw-mall && ./start.sh bootstrap 2>&1 | grep sku.sql
podman exec mysql-master1 mysql -uroot -proot123 mall_product -e "SHOW TABLES LIKE 'sku';" 2>&1 | grep -v Warning
```

预期：表存在。

- [ ] **Step 1.4: commit**

```
feat(m2): 数据表 sku — 多 SKU 多规格

mall_product 库新增 sku 表挂在 product 表下:
- product_id FK + UNIQUE(product_id, sku_code) 防同 SPU 重复 code
- spec_text 给前端展示用「黑色/256GB」, spec_json 结构化存
- status 0/1 软删
- shop_id 冗余便于按店铺扫
product 表保留 price/stock/images 字段作单 SKU 兼容路径
(M2 兼容性: 单 SKU 商品 sku 表只存 1 行 default).

start.sh SPRINT_MIGRATIONS 注册迁移.
```

---

## Task 2: 历史 product 回填 default SKU

**Files**:
- Create: `mall-product-rpc/cmd/backfill_default_sku/main.go`

- [ ] **Step 2.1: 写脚本**

```go
// mall-product-rpc/cmd/backfill_default_sku/main.go
// 一次性脚本：每条 product 自动 INSERT 一行 default SKU
// 复制 product 的 price/stock/images。幂等：UNIQUE(product_id, sku_code) 保护。
package main

import (
	"database/sql"
	"flag"
	"fmt"
	"log"

	_ "github.com/go-sql-driver/mysql"
)

func main() {
	dsn := flag.String("dsn",
		"proxysql:proxysql123@tcp(127.0.0.1:6033)/mall_product?charset=utf8mb4&parseTime=true&loc=Local",
		"mall_product DSN")
	flag.Parse()

	db, err := sql.Open("mysql", *dsn)
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	rows, err := db.Query("SELECT id, shop_id, price, stock, images FROM product")
	if err != nil {
		log.Fatal(err)
	}
	defer rows.Close()

	inserted, skipped := 0, 0
	for rows.Next() {
		var pid, shopId, price, stock uint64
		var imgs string
		if err := rows.Scan(&pid, &shopId, &price, &stock, &imgs); err != nil {
			log.Fatal(err)
		}
		mainImg := ""
		// product.images 是逗号分隔；取第一张做 SKU 主图
		for i := 0; i < len(imgs); i++ {
			if imgs[i] == ',' {
				mainImg = imgs[:i]
				break
			}
		}
		if mainImg == "" {
			mainImg = imgs
		}
		code := fmt.Sprintf("P%d-S1", pid)
		res, err := db.Exec(`
            INSERT IGNORE INTO sku
              (product_id, shop_id, sku_code, spec_text, spec_json, price, stock, image, status)
            VALUES (?, ?, ?, '', '', ?, ?, ?, 1)`,
			pid, shopId, code, price, stock, mainImg)
		if err != nil {
			log.Fatalf("insert pid=%d: %v", pid, err)
		}
		n, _ := res.RowsAffected()
		if n > 0 {
			inserted++
		} else {
			skipped++
		}
	}
	fmt.Printf("backfill default sku done: inserted=%d skipped=%d\n", inserted, skipped)
}
```

- [ ] **Step 2.2: 跑**

```bash
cd ~/workspace/go/mall/yw-mall/mall-product-rpc && go run ./cmd/backfill_default_sku/
podman exec mysql-master1 mysql -uroot -proot123 mall_product -e "SELECT COUNT(*) AS sku_count FROM sku; SELECT COUNT(*) AS product_count FROM product;" 2>&1 | grep -v Warning
```

预期：sku_count == product_count（每条 product 一条 default SKU）。

- [ ] **Step 2.3: commit**

```
feat(m2): 历史 product 回填 default SKU

cmd/backfill_default_sku 一次性脚本，每条 product 自动 INSERT IGNORE
一行 sku (sku_code=P{pid}-S1, spec_text=空, 复制 product 的 price/stock,
image 取 product.images 第一张). 幂等可重跑.
```

---

## Task 3: product-rpc proto 加 SKU + BatchUpsertSkus RPC

**Files**:
- Modify: `mall-common/proto/product/product.proto`
- Auto-regen: `mall-product-rpc/product/product.pb.go`, `product_grpc.pb.go`, `internal/server/productserviceserver.go`
- Manual: `mall-product-rpc/productservice/productservice.go`

- [ ] **Step 3.1: proto 加 messages + rpc**

在 `mall-common/proto/product/product.proto` 末尾追加：

```protobuf
// ===== M2 multi-SKU =====
message SkuInput {
  // id=0 表示新建; >0 表示 update 已有 SKU; status=0 表示软删
  int64  id        = 1;
  string sku_code  = 2;  // 空则后端按 P{pid}-S{idx} 自动生成
  string spec_text = 3;
  string spec_json = 4;
  int64  price     = 5;
  int64  stock     = 6;
  string image     = 7;
  int32  status    = 8;  // 1=active 0=disabled
}

message SkuItem {
  int64  id          = 1;
  int64  product_id  = 2;
  string sku_code    = 3;
  string spec_text   = 4;
  string spec_json   = 5;
  int64  price       = 6;
  int64  stock       = 7;
  string image       = 8;
  int32  status      = 9;
}

message BatchUpsertSkusReq {
  int64 product_id = 1;
  int64 shop_id    = 2; // 校验所有权
  repeated SkuInput skus = 3;
}
message BatchUpsertSkusResp {
  repeated SkuItem items = 1;
}
```

修改 `CreateProductReq` 加 skus 字段（追加到末尾保持向后兼容）：
```protobuf
message CreateProductReq {
  string name = 1;
  string description = 2;
  int64 price = 3;
  int64 stock = 4;
  int64 category_id = 5;
  string images = 6;
  int64 shop_id = 7;
  // M2: 多 SKU 数组，空时后端建 1 个 default SKU（id=0, spec_text=空）
  repeated SkuInput skus = 8;
}
```

修改 `ProductDetailResp` 加 skus 字段：
```protobuf
// 查现有定义，在末尾加：
repeated SkuItem skus = N+1;  // M2: 多 SKU 列表
```

service block 末尾加：
```protobuf
rpc BatchUpsertSkus(BatchUpsertSkusReq) returns (BatchUpsertSkusResp);
```

- [ ] **Step 3.2: regen**

```bash
export PATH=/home/carter/workspace/go/bin:$PATH
cd ~/workspace/go/mall/yw-mall/mall-product-rpc
protoc --go_out=. --go-grpc_out=. \
  --proto_path=. --proto_path=../mall-common/proto \
  ../mall-common/proto/product/product.proto
```

- [ ] **Step 3.3: 手工补 server stub**

在 `internal/server/productserviceserver.go` 末尾加：
```go
func (s *ProductServiceServer) BatchUpsertSkus(ctx context.Context, in *product.BatchUpsertSkusReq) (*product.BatchUpsertSkusResp, error) {
	l := logic.NewBatchUpsertSkusLogic(ctx, s.svcCtx)
	return l.BatchUpsertSkus(in)
}
```

- [ ] **Step 3.4: 手工补 productservice/productservice.go**

加 type alias + interface 方法 + impl，参考 M1 T3 同款手法：
```go
// type aliases 区块加
SkuInput               = product.SkuInput
SkuItem                = product.SkuItem
BatchUpsertSkusReq     = product.BatchUpsertSkusReq
BatchUpsertSkusResp    = product.BatchUpsertSkusResp

// interface 加
BatchUpsertSkus(ctx context.Context, in *BatchUpsertSkusReq, opts ...grpc.CallOption) (*BatchUpsertSkusResp, error)

// impl 加
func (m *defaultProductService) BatchUpsertSkus(ctx context.Context, in *BatchUpsertSkusReq, opts ...grpc.CallOption) (*BatchUpsertSkusResp, error) {
	client := product.NewProductServiceClient(m.cli.Conn())
	return client.BatchUpsertSkus(ctx, in, opts...)
}
```

- [ ] **Step 3.5: build 确认（应报 NewBatchUpsertSkusLogic undefined）**

```bash
cd ~/workspace/go/mall/yw-mall/mall-product-rpc && go build ./... 2>&1 | head -5
```

预期：报 `undefined: logic.NewBatchUpsertSkusLogic`。

- [ ] **Step 3.6: commit**

```
feat(m2 proto): product 加 SkuInput/SkuItem + BatchUpsertSkus RPC

CreateProductReq 加 skus 字段 (空表示走 default SKU 兼容路径).
ProductDetailResp 加 skus 列表给详情页用.
BatchUpsertSkus 给编辑商品时批量改 SKU 矩阵用.

proto regen + server stub + productservice wrapper 手工同步.

WIP: build 待 BatchUpsertSkusLogic 实现 (T4).
```

---

## Task 4: product-rpc SKU helper + BatchUpsertSkus logic + 改造 createproductlogic

**Files**:
- Create: `mall-product-rpc/internal/logic/skuhelpers.go`
- Create: `mall-product-rpc/internal/logic/batchupsertskuslogic.go`
- Modify: `mall-product-rpc/internal/logic/createproductlogic.go`
- Modify: `mall-product-rpc/internal/logic/getproductdetaillogic.go`

- [ ] **Step 4.1: skuhelpers.go — 通用 helper**

```go
// mall-product-rpc/internal/logic/skuhelpers.go
package logic

import (
	"context"
	"fmt"

	"mall-product-rpc/internal/svc"
	"mall-product-rpc/product"

	"github.com/zeromicro/go-zero/core/stores/sqlx"
)

// upsertSkusAtomic 在外层 transaction 内调，对 skus 数组做：
// - id=0 INSERT (sku_code 空则 P{pid}-S{idx} 自动生成)
// - id>0 UPDATE 字段
// - status=0 软删
// 返回插入后行 (含 auto-increment id)。
func upsertSkusAtomic(ctx context.Context, tx sqlx.Session, shopId, productId int64, skus []*product.SkuInput) ([]*product.SkuItem, error) {
	out := make([]*product.SkuItem, 0, len(skus))
	for idx, s := range skus {
		code := s.SkuCode
		if code == "" {
			code = fmt.Sprintf("P%d-S%d", productId, idx+1)
		}
		if s.Id == 0 {
			res, err := tx.ExecCtx(ctx, `
                INSERT INTO sku
                  (product_id, shop_id, sku_code, spec_text, spec_json, price, stock, image, status)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)`,
				productId, shopId, code, s.SpecText, s.SpecJson,
				s.Price, s.Stock, s.Image, statusOrDefault(s.Status))
			if err != nil {
				return nil, err
			}
			newId, _ := res.LastInsertId()
			out = append(out, &product.SkuItem{
				Id: newId, ProductId: productId, SkuCode: code,
				SpecText: s.SpecText, SpecJson: s.SpecJson,
				Price: s.Price, Stock: s.Stock, Image: s.Image,
				Status: statusOrDefault(s.Status),
			})
		} else {
			if _, err := tx.ExecCtx(ctx, `
                UPDATE sku SET sku_code=?, spec_text=?, spec_json=?, price=?, stock=?, image=?, status=?
                WHERE id=? AND product_id=? AND shop_id=?`,
				code, s.SpecText, s.SpecJson,
				s.Price, s.Stock, s.Image, statusOrDefault(s.Status),
				s.Id, productId, shopId); err != nil {
				return nil, err
			}
			out = append(out, &product.SkuItem{
				Id: s.Id, ProductId: productId, SkuCode: code,
				SpecText: s.SpecText, SpecJson: s.SpecJson,
				Price: s.Price, Stock: s.Stock, Image: s.Image,
				Status: statusOrDefault(s.Status),
			})
		}
	}
	return out, nil
}

func statusOrDefault(s int32) int32 {
	if s == 0 {
		return 1
	}
	return s
}

// listSkusByProduct 给 detail 接口用，返 status=1 的 SKU。
func listSkusByProduct(ctx context.Context, db sqlx.SqlConn, productId int64) ([]*product.SkuItem, error) {
	var rows []struct {
		Id        uint64 `db:"id"`
		ProductId uint64 `db:"product_id"`
		SkuCode   string `db:"sku_code"`
		SpecText  string `db:"spec_text"`
		SpecJson  string `db:"spec_json"`
		Price     int64  `db:"price"`
		Stock     int64  `db:"stock"`
		Image     string `db:"image"`
		Status    int32  `db:"status"`
	}
	if err := db.QueryRowsCtx(ctx, &rows, `
        SELECT id, product_id, sku_code, spec_text, spec_json, price, stock, image, status
        FROM sku WHERE product_id=? AND status=1 ORDER BY id`, productId); err != nil {
		return nil, err
	}
	out := make([]*product.SkuItem, 0, len(rows))
	for _, r := range rows {
		out = append(out, &product.SkuItem{
			Id: int64(r.Id), ProductId: int64(r.ProductId), SkuCode: r.SkuCode,
			SpecText: r.SpecText, SpecJson: r.SpecJson,
			Price: r.Price, Stock: r.Stock, Image: r.Image, Status: r.Status,
		})
	}
	return out, nil
}

// 复用 svc.ServiceContext.DB
var _ = svc.ServiceContext{}
```

- [ ] **Step 4.2: batchupsertskuslogic.go**

```go
// mall-product-rpc/internal/logic/batchupsertskuslogic.go
package logic

import (
	"context"
	"errors"

	"mall-product-rpc/internal/svc"
	"mall-product-rpc/product"

	"github.com/zeromicro/go-zero/core/logx"
	"github.com/zeromicro/go-zero/core/stores/sqlx"
)

type BatchUpsertSkusLogic struct {
	ctx    context.Context
	svcCtx *svc.ServiceContext
	logx.Logger
}

func NewBatchUpsertSkusLogic(ctx context.Context, svcCtx *svc.ServiceContext) *BatchUpsertSkusLogic {
	return &BatchUpsertSkusLogic{ctx: ctx, svcCtx: svcCtx, Logger: logx.WithContext(ctx)}
}

// BatchUpsertSkus 给商家编辑商品时改 SKU 矩阵用。
// 校验 product 归属 shopId，否则拒。SKU 列表全量替换语义：未在 skus 中的现存 SKU
// 不动（要软删需在 skus 数组里传 status=0）。
func (l *BatchUpsertSkusLogic) BatchUpsertSkus(in *product.BatchUpsertSkusReq) (*product.BatchUpsertSkusResp, error) {
	if in.ProductId <= 0 || in.ShopId <= 0 {
		return nil, errors.New("product_id and shop_id required")
	}
	// 校验 product 归属
	var ownerShopId int64
	if err := l.svcCtx.DB.QueryRowCtx(l.ctx, &ownerShopId,
		"SELECT shop_id FROM product WHERE id=? LIMIT 1", in.ProductId); err != nil {
		return nil, errors.New("product not found")
	}
	if ownerShopId != in.ShopId {
		return nil, errors.New("product does not belong to this shop")
	}

	var out []*product.SkuItem
	err := l.svcCtx.DB.TransactCtx(l.ctx, func(ctx context.Context, tx sqlx.Session) error {
		items, e := upsertSkusAtomic(ctx, tx, in.ShopId, in.ProductId, in.Skus)
		if e != nil {
			return e
		}
		out = items

		// 同步 product 表的聚合字段（min price / sum stock / 第一张 SKU image）
		if e := syncProductAggregateInTx(ctx, tx, in.ProductId); e != nil {
			return e
		}
		return nil
	})
	if err != nil {
		return nil, err
	}
	return &product.BatchUpsertSkusResp{Items: out}, nil
}

// syncProductAggregateInTx 把多 SKU 聚合写回 product 行：
// price = MIN(active sku price), stock = SUM(active sku stock).
// 这是给 C 端列表/搜索路径用的兼容字段。
func syncProductAggregateInTx(ctx context.Context, tx sqlx.Session, productId int64) error {
	var agg struct {
		MinPrice int64 `db:"min_price"`
		SumStock int64 `db:"sum_stock"`
	}
	if err := tx.QueryRowCtx(ctx, &agg, `
        SELECT COALESCE(MIN(price),0) AS min_price, COALESCE(SUM(stock),0) AS sum_stock
        FROM sku WHERE product_id=? AND status=1`, productId); err != nil {
		return err
	}
	_, err := tx.ExecCtx(ctx,
		"UPDATE product SET price=?, stock=? WHERE id=?",
		agg.MinPrice, agg.SumStock, productId)
	return err
}
```

- [ ] **Step 4.3: 改造 createproductlogic.go — 落 default SKU 或解析 skus 数组**

打开 `mall-product-rpc/internal/logic/createproductlogic.go`，在 INSERT product 拿到 pid 后追加：

```go
// 落 SKU：若 in.Skus 为空 → 建 1 个 default SKU 兼容老路径
// 否则按数组建多 SKU + 聚合回写 product
if len(in.Skus) == 0 {
    // default SKU
    mainImg := ""
    if in.Images != "" {
        if comma := strings.Index(in.Images, ","); comma >= 0 {
            mainImg = in.Images[:comma]
        } else {
            mainImg = in.Images
        }
    }
    _, err = l.svcCtx.DB.ExecCtx(l.ctx, `
        INSERT INTO sku (product_id, shop_id, sku_code, spec_text, spec_json,
                         price, stock, image, status)
        VALUES (?, ?, ?, '', '', ?, ?, ?, 1)`,
        pid, in.ShopId, fmt.Sprintf("P%d-S1", pid),
        in.Price, in.Stock, mainImg)
    if err != nil { return nil, err }
} else {
    if err := l.svcCtx.DB.TransactCtx(l.ctx, func(ctx context.Context, tx sqlx.Session) error {
        if _, e := upsertSkusAtomic(ctx, tx, in.ShopId, pid, in.Skus); e != nil {
            return e
        }
        return syncProductAggregateInTx(ctx, tx, pid)
    }); err != nil {
        return nil, err
    }
}
```

注：`strings` import 新增。其他依赖（fmt/sqlx）应已有。

- [ ] **Step 4.4: getproductdetaillogic.go 加 skus 返回**

在原 logic 拼 ProductDetailResp 的位置追加：
```go
skus, _ := listSkusByProduct(l.ctx, l.svcCtx.DB, in.Id)
resp.Skus = skus
```

- [ ] **Step 4.5: build + restart**

```bash
cd ~/workspace/go/mall/yw-mall/mall-product-rpc && go build ./... 2>&1 | head -5
cd ~/workspace/go/mall/yw-mall && ./start.sh restart 2>&1 | grep product-rpc
sleep 2
tail -5 logs/product-rpc.log
```

预期：build 0 错，启动成功。

- [ ] **Step 4.6: commit**

```
feat(m2 product-rpc): SKU upsert + 详情返 skus 数组 + 自动聚合

- skuhelpers.go: upsertSkusAtomic 单事务支持新增/更新/软删;
  listSkusByProduct 给详情接口用; syncProductAggregateInTx
  按 MIN(price)/SUM(stock) 聚合写回 product 行 (C 端兼容字段)
- BatchUpsertSkus logic: 校验 product 归属 shopId, 全量上送语义
  (未传的 sku 不动, 软删需显式 status=0)
- createproductlogic 改造: in.Skus 空走 default SKU 兼容路径,
  非空走 SKU 数组 + 聚合
- getproductdetaillogic: 返 skus 列表

build + restart 验证通过.
```

---

## Task 5: admin-api 图片上传 endpoint

**Files**:
- Modify: `yw-mall-admin/internal/types/types.go`
- Create: `yw-mall-admin/internal/logic/merchant_upload_logic.go`
- Create: `yw-mall-admin/internal/handler/merchant_upload_handler.go`
- Modify: `yw-mall-admin/internal/handler/routes.go`
- Modify: `yw-mall-admin/etc/admin.yaml`（确保 MinIO 配置块在）
- Modify: `yw-mall-admin/internal/svc/servicecontext.go`（注入 MinIO client）

- [ ] **Step 5.1: types 加 UploadImageResp**

```go
type UploadImageResp struct {
    Url string `json:"url"`
    Key string `json:"key"`
}
```

- [ ] **Step 5.2: merchant_upload_logic.go**

```go
package logic

import (
	"context"
	"errors"
	"fmt"
	"io"
	"path/filepath"
	"strings"
	"time"

	"mall-admin-api/internal/middleware"
	"mall-admin-api/internal/svc"
	"mall-admin-api/internal/types"
)

const (
	maxImageBytes = 5 * 1024 * 1024 // 5 MB
)

var allowedExts = map[string]bool{".jpg": true, ".jpeg": true, ".png": true, ".webp": true}

// UploadProductImage 把图片放到 MinIO 的 product-images bucket，按 shop_id 分目录。
// 返回可访问 URL（生产 nginx 代理 /minio/，dev 直 :9000）+ key 给 FE 删除用。
func UploadProductImage(ctx context.Context, svcCtx *svc.ServiceContext, filename string, body io.Reader, size int64) (*types.UploadImageResp, error) {
	c, _ := middleware.ClaimsFromContext(ctx)
	if c == nil || c.ShopId <= 0 {
		return nil, errors.New("not in a shop")
	}
	if size <= 0 || size > maxImageBytes {
		return nil, fmt.Errorf("file size out of range (0, %d]", maxImageBytes)
	}
	ext := strings.ToLower(filepath.Ext(filename))
	if !allowedExts[ext] {
		return nil, errors.New("unsupported file type (allow jpg/jpeg/png/webp)")
	}
	key := fmt.Sprintf("product-images/shop-%d/%d%s", c.ShopId, time.Now().UnixNano(), ext)
	url, err := svcCtx.MinIO.Put(ctx, key, body, size, "image/"+strings.TrimPrefix(ext, "."))
	if err != nil {
		return nil, fmt.Errorf("upload failed: %w", err)
	}
	return &types.UploadImageResp{Url: url, Key: key}, nil
}
```

注：`svcCtx.MinIO` 接口签名约定 `Put(ctx, key, reader, size, contentType) (url string, err error)`。如 admin-api 现没装 MinIO，参考 mall-api/internal/svc/servicecontext.go 引同款。

- [ ] **Step 5.3: merchant_upload_handler.go**

```go
package handler

import (
	"net/http"

	"mall-admin-api/internal/logic"
	"mall-admin-api/internal/svc"
)

func uploadProductImageHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		// 限 6 MB（5 MB 文件 + 表单开销）
		if err := r.ParseMultipartForm(6 << 20); err != nil {
			writeErr(r, w, err)
			return
		}
		file, hdr, err := r.FormFile("file")
		if err != nil {
			writeErr(r, w, err)
			return
		}
		defer file.Close()
		resp, err := logic.UploadProductImage(r.Context(), svcCtx, hdr.Filename, file, hdr.Size)
		if err != nil {
			writeErr(r, w, err)
			return
		}
		writeOk(r, w, resp)
	}
}
```

- [ ] **Step 5.4: routes.go 加路由 + svc 注入 MinIO**

在 `/merchant/v1` protected group 加：
```go
{Method: http.MethodPost, Path: "/upload/image", Handler: uploadProductImageHandler(svcCtx)},
```

- [ ] **Step 5.5: svc 注入 MinIO client**

打开 `yw-mall-admin/internal/svc/servicecontext.go`，仿照 mall-api 同款引 minioutil。详细看 `mall-api/internal/svc/servicecontext.go` 的 `hotMinioClient` 区块，admin-api 不要 hot reload，直接 new client 即可。

- [ ] **Step 5.6: build + e2e**

```bash
cd ~/workspace/go/mall/yw-mall-admin && go build ./... 2>&1 | head -5

# 用 alice token 试上传一张 1KB 测试图
echo "fake-jpg-bytes" > /tmp/test.jpg
T=$(curl -s -X POST http://localhost:18999/merchant/v1/login -H 'Content-Type: application/json' \
  -d '{"username":"alice","password":"alice123"}' | python3 -c "import json,sys;print(json.load(sys.stdin)['token'])")
curl -s -X POST -H "Authorization: Bearer $T" \
  -F "file=@/tmp/test.jpg" \
  http://localhost:18999/merchant/v1/upload/image
```

预期：返 `{"url":"http://...","key":"product-images/shop-1/..."}`。

- [ ] **Step 5.7: commit**

```
feat(m2 admin-api): /merchant/v1/upload/image 图片上传

POST multipart, 限 5MB + jpg/jpeg/png/webp 白名单.
按 shop_id 分目录 product-images/shop-{id}/{ts}.{ext},
返 (url, key) 给 FE 用. 5MB 上限挡批量恶意.

需 staff.shopId 命中, 暂未做 PermGate (商品上传图算 product.write
但 owner/warehouse 双角色都可上传, 后续 RBAC 收紧到 product.write).
```

---

## Task 6: admin-api merchant CreateProduct/UpdateProduct/Skus 透传

**Files**:
- Modify: `yw-mall-admin/internal/types/types.go`（加 SkuInput/SkuItem + Req 字段）
- Modify: `yw-mall-admin/internal/logic/merchant_products_logic.go`（透传 skus）
- Create: 新增 BatchUpsertSkus handler/logic
- Modify: `yw-mall-admin/internal/handler/routes.go`（加路由）

- [ ] **Step 6.1: types 加 SKU DTOs + 扩 product req**

```go
type SkuInputDTO struct {
    Id        int64  `json:"id,optional"`
    SkuCode   string `json:"skuCode,optional"`
    SpecText  string `json:"specText,optional"`
    SpecJson  string `json:"specJson,optional"`
    Price     int64  `json:"price"`
    Stock     int64  `json:"stock"`
    Image     string `json:"image,optional"`
    Status    int32  `json:"status,optional"`
}
type SkuItemDTO struct {
    Id         int64  `json:"id"`
    ProductId  int64  `json:"productId"`
    SkuCode    string `json:"skuCode"`
    SpecText   string `json:"specText"`
    SpecJson   string `json:"specJson"`
    Price      int64  `json:"price"`
    Stock      int64  `json:"stock"`
    Image      string `json:"image"`
    Status     int32  `json:"status"`
}

// 扩 MerchantCreateProductReq：
type MerchantCreateProductReq struct {
    // ...existing fields...
    Skus []SkuInputDTO `json:"skus,optional"`
}
// 扩 ProductDetailDTO：加 Skus []SkuItemDTO

type BatchUpsertSkusReq struct {
    Skus []SkuInputDTO `json:"skus"`
}
type BatchUpsertSkusResp struct {
    Items []SkuItemDTO `json:"items"`
}
```

- [ ] **Step 6.2: logic 透传 skus**

在 `MerchantCreateProduct` 内构造 RPC req 时把 skus 数组 map 过去；GetProduct/Detail 时把 skus 字段读出来。

加 `MerchantBatchUpsertSkus` 函数：
```go
func MerchantBatchUpsertSkus(ctx context.Context, svcCtx *svc.ServiceContext, productId int64, req *types.BatchUpsertSkusReq) (*types.BatchUpsertSkusResp, error) {
    c, _ := middleware.ClaimsFromContext(ctx)
    if c == nil || c.ShopId <= 0 { return nil, errors.New("not in a shop") }
    if !hasPerm(c, "product.write") { return nil, errors.New("permission denied: product.write") }
    skus := make([]*productservice.SkuInput, 0, len(req.Skus))
    for _, s := range req.Skus {
        skus = append(skus, &productservice.SkuInput{
            Id: s.Id, SkuCode: s.SkuCode, SpecText: s.SpecText, SpecJson: s.SpecJson,
            Price: s.Price, Stock: s.Stock, Image: s.Image, Status: s.Status,
        })
    }
    r, err := svcCtx.ProductRpc.BatchUpsertSkus(ctx, &productservice.BatchUpsertSkusReq{
        ProductId: productId, ShopId: c.ShopId, Skus: skus,
    })
    if err != nil { return nil, err }
    out := make([]types.SkuItemDTO, 0, len(r.Items))
    for _, it := range r.Items {
        out = append(out, types.SkuItemDTO{
            Id: it.Id, ProductId: it.ProductId, SkuCode: it.SkuCode,
            SpecText: it.SpecText, SpecJson: it.SpecJson,
            Price: it.Price, Stock: it.Stock, Image: it.Image, Status: it.Status,
        })
    }
    return &types.BatchUpsertSkusResp{Items: out}, nil
}
```

- [ ] **Step 6.3: handler + 路由**

```go
// merchant_products_handlers 同文件加：
func merchantBatchUpsertSkusHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        id, _ := parseId(r)
        var req types.BatchUpsertSkusReq
        if err := httpx.Parse(r, &req); err != nil { writeErr(r, w, err); return }
        resp, err := logic.MerchantBatchUpsertSkus(r.Context(), svcCtx, id, &req)
        if err != nil { writeErr(r, w, err); return }
        writeOk(r, w, resp)
    }
}
```

routes.go protected 内加：
```go
{Method: http.MethodPost, Path: "/products/:id/skus", Handler: merchantBatchUpsertSkusHandler(svcCtx)},
```

- [ ] **Step 6.4: build + restart + e2e**

```bash
cd ~/workspace/go/mall/yw-mall-admin && go build ./...
cd ~/workspace/go/mall/yw-mall && ./start.sh restart 2>&1 | grep admin-api
sleep 2

# alice 创建带 2 SKU 商品
T=$(curl -s -X POST http://localhost:18999/merchant/v1/login -H 'Content-Type: application/json' \
  -d '{"username":"alice","password":"alice123"}' | python3 -c "import json,sys;print(json.load(sys.stdin)['token'])")

curl -s -X POST -H "Authorization: Bearer $T" -H 'Content-Type: application/json' \
  -d '{
    "name":"测试多SKU商品",
    "description":"M2 e2e",
    "categoryId":1,
    "skus":[
      {"specText":"黑色/256GB","price":99900,"stock":10,"image":"img1.jpg"},
      {"specText":"白色/256GB","price":99900,"stock":5,"image":"img2.jpg"}
    ]
  }' \
  http://localhost:18999/merchant/v1/products | python3 -m json.tool

# 然后 GET 详情看 skus 数组
NEW_PID=$(...)  # 从上面 resp 拿
curl -s -H "Authorization: Bearer $T" http://localhost:18999/merchant/v1/products/$NEW_PID | python3 -m json.tool
```

预期：detail 返回包含 2 个 SKU + product.price=99900 + product.stock=15。

- [ ] **Step 6.5: commit**

```
feat(m2 admin-api): /merchant/v1/products/:id/skus + CreateProduct 接 skus

CreateProduct 透传 skus 数组给 product-rpc, 空数组走 default SKU 路径.
新增 POST /products/:id/skus → BatchUpsertSkus (需 product.write).
GetProduct detail 返 skus 列表.

e2e: alice 创建 2-SKU 商品 → detail 见 2 SKU + product 行 price=MIN/stock=SUM.
```

---

## Task 7: FE products list 页

**Files**:
- Create: `yw-mall-admin-fe/merchant/src/api/products.ts`
- Create: `yw-mall-admin-fe/merchant/src/views/product/list.vue`
- Modify: `yw-mall-admin-fe/merchant/src/router/index.ts` (加 /products 路由)
- Modify: `yw-mall-admin-fe/merchant/src/layouts/AdminLayout.vue` (商品菜单 disabled → enabled)

- [ ] **Step 7.1: api/products.ts**

```ts
import { get, post, put as putReq } from './request'
import type { ApiResponse } from '@/types/api'

export interface SkuInput {
  id?: number; skuCode?: string; specText?: string; specJson?: string
  price: number; stock: number; image?: string; status?: number
}
export interface SkuItem {
  id: number; productId: number; skuCode: string; specText: string
  specJson: string; price: number; stock: number; image: string; status: number
}
export interface ProductSummary {
  id: number; name: string; price: number; stock: number; images: string
  status: number; reviewStatus: number; categoryId: number
}
export interface ProductDetail extends ProductSummary {
  description?: string; brand?: string; detail?: string; weight?: number
  skus: SkuItem[]
}

function unwrap<T>(b: ApiResponse<T> & T): T {
  return (b as ApiResponse<T>).data ?? (b as unknown as T)
}

export async function listProducts(p: { page?: number; pageSize?: number; name?: string }) {
  const body = await get<ApiResponse<{ items: ProductSummary[]; total: number }> & { items: ProductSummary[]; total: number }>(
    `/products?page=${p.page ?? 1}&pageSize=${p.pageSize ?? 20}&name=${encodeURIComponent(p.name ?? '')}`,
  )
  return unwrap(body)
}
export async function getProduct(id: number) {
  const body = await get<ApiResponse<ProductDetail> & ProductDetail>(`/products/${id}`)
  return unwrap(body)
}
export async function createProduct(req: {
  name: string; description?: string; categoryId: number; brand?: string;
  detail?: string; images: string; skus: SkuInput[]
}) {
  const body = await post<ApiResponse<{ id: number }> & { id: number }>('/products', req)
  return unwrap(body)
}
export async function updateProduct(id: number, req: { name?: string; description?: string; brand?: string; detail?: string; images?: string; categoryId?: number }) {
  const body = await putReq<ApiResponse<{ ok: boolean }> & { ok: boolean }>(`/products/${id}`, req)
  return unwrap(body)
}
export async function setProductStatus(id: number, status: number) {
  return post(`/products/${id}/status`, { status })
}
export async function setProductStock(id: number, stock: number) {
  return post(`/products/${id}/stock`, { stock })
}
export async function batchUpsertSkus(productId: number, skus: SkuInput[]) {
  const body = await post<ApiResponse<{ items: SkuItem[] }> & { items: SkuItem[] }>(
    `/products/${productId}/skus`, { skus },
  )
  return unwrap(body)
}
```

- [ ] **Step 7.2: views/product/list.vue**

参考 admin SPA 的 product list 风格做，元素：
- 顶部：搜索 input + 「新建商品」按钮（perm: product.write）
- 表格：缩略图 / 名称 / 价格(分→元) / 库存 / 状态 tag / 审核状态 / 操作
- 操作：编辑（跳 `/products/${id}/edit`）+ 上下架 switch + 删除（软删 status=2）+ 改库存（inline 编辑）
- 分页

详细代码省略——参考 `views/staff/list.vue` 模板，把 listStaff 换成 listProducts，加金额换算 `(p / 100).toFixed(2)`。

- [ ] **Step 7.3: 路由 + 菜单**

`router/index.ts` `/` 下加：
```ts
{ path: 'products', name: 'Products', component: () => import('@/views/product/list.vue'),
  meta: { title: '商品管理', icon: 'Goods', perms: ['product.read'] } },
{ path: 'products/new', name: 'ProductNew', component: () => import('@/views/product/edit.vue'),
  meta: { title: '新建商品', perms: ['product.write'] } },
{ path: 'products/:id/edit', name: 'ProductEdit', component: () => import('@/views/product/edit.vue'),
  meta: { title: '编辑商品', perms: ['product.write'] } },
```

`layouts/AdminLayout.vue` 菜单数组里去掉 `disabled: true` 的 `/products` 项的 disabled，并把 title 从「商品 (M2)」改回「商品管理」。

- [ ] **Step 7.4: build + dev smoke**

```bash
cd ~/workspace/go/mall/yw-mall-admin-fe/merchant && pnpm run build
```

预期：0 错。

- [ ] **Step 7.5: commit**

```
feat(m2 fe): 商品列表页 /products + nav 解锁

api/products.ts: list/get/create/update/setStatus/setStock + batchUpsertSkus
views/product/list.vue: 表格 + 搜索 + 上下架 + 改库存 inline
菜单「商品管理」从 disabled 解锁
```

---

## Task 8: FE SkuMatrix + ImageUploader 组件

**Files**:
- Create: `yw-mall-admin-fe/merchant/src/components/SkuMatrix.vue`
- Create: `yw-mall-admin-fe/merchant/src/components/ImageUploader.vue`
- Create: `yw-mall-admin-fe/merchant/src/api/upload.ts`

- [ ] **Step 8.1: api/upload.ts**

```ts
import axios from 'axios'
import { useUserStore } from '@/stores/user'
export async function uploadImage(file: File): Promise<{ url: string; key: string }> {
  const fd = new FormData()
  fd.append('file', file)
  const sess = useUserStore()
  const resp = await axios.post('/merchant/v1/upload/image', fd, {
    headers: { Authorization: `Bearer ${sess.accessToken}`, 'Content-Type': 'multipart/form-data' },
    timeout: 30000,
  })
  const body = resp.data?.data ?? resp.data
  return body
}
```

- [ ] **Step 8.2: components/ImageUploader.vue**

包装 el-upload，handler 调 uploadImage：

```vue
<script setup lang="ts">
import { computed } from 'vue'
import { ElMessage } from 'element-plus'
import { Plus } from '@element-plus/icons-vue'
import { uploadImage } from '@/api/upload'

const props = defineProps<{ modelValue: string[]; max?: number }>()
const emit = defineEmits<{ 'update:modelValue': [string[]] }>()

const max = computed(() => props.max ?? 5)
const list = computed({
  get: () => props.modelValue.map((url, idx) => ({ uid: idx, name: `img-${idx}`, url, status: 'success' as const })),
  set: () => {},
})

async function beforeUpload(file: File) {
  if (file.size > 5 * 1024 * 1024) { ElMessage.error('图片超过 5MB'); return false }
  if (!/^image\/(jpeg|png|webp)$/.test(file.type)) { ElMessage.error('只支持 jpg/png/webp'); return false }
  return true
}

async function handle(item: { file: File }) {
  try {
    const r = await uploadImage(item.file)
    emit('update:modelValue', [...props.modelValue, r.url])
    ElMessage.success('上传成功')
  } catch (e) {
    ElMessage.error('上传失败')
  }
}

function remove(idx: number) {
  const next = [...props.modelValue]
  next.splice(idx, 1)
  emit('update:modelValue', next)
}
</script>

<template>
  <div class="img-uploader">
    <div v-for="(url, idx) in modelValue" :key="idx" class="img-tile">
      <el-image :src="url" fit="cover" style="width:100px;height:100px" />
      <span class="del" @click="remove(idx)">×</span>
    </div>
    <el-upload
      v-if="modelValue.length < max"
      :show-file-list="false"
      :before-upload="beforeUpload"
      :http-request="handle"
      accept="image/jpeg,image/png,image/webp"
    >
      <div class="add-tile"><el-icon><Plus/></el-icon></div>
    </el-upload>
  </div>
</template>

<style scoped>
.img-uploader { display:flex; flex-wrap:wrap; gap:8px }
.img-tile { position:relative; width:100px; height:100px }
.img-tile .del { position:absolute; top:-6px; right:-6px; background:#000; color:#fff;
                 width:20px; height:20px; border-radius:50%; text-align:center;
                 line-height:18px; cursor:pointer; font-size:14px }
.add-tile { width:100px; height:100px; border:1px dashed #ccc; border-radius:6px;
            display:flex; align-items:center; justify-content:center;
            cursor:pointer; color:#999 }
</style>
```

- [ ] **Step 8.3: components/SkuMatrix.vue**

属性矩阵 → SKU 笛卡尔积生成器。结构：

```vue
<script setup lang="ts">
import { ref, watch, computed } from 'vue'
import { ElButton, ElInput, ElTag } from 'element-plus'
import type { SkuInput } from '@/api/products'

const props = defineProps<{ modelValue: SkuInput[] }>()
const emit = defineEmits<{ 'update:modelValue': [SkuInput[]] }>()

interface AttrGroup { name: string; values: string[] }
const groups = ref<AttrGroup[]>([])
const newValueDraft = ref<Record<number, string>>({})

function addGroup() { groups.value.push({ name: '', values: [] }) }
function removeGroup(idx: number) { groups.value.splice(idx, 1) }
function addValue(idx: number) {
  const v = (newValueDraft.value[idx] ?? '').trim()
  if (v) { groups.value[idx].values.push(v); newValueDraft.value[idx] = '' }
}
function removeValue(gIdx: number, vIdx: number) { groups.value[gIdx].values.splice(vIdx, 1) }

// Cartesian product
function cartesian(): SkuInput[] {
  if (!groups.value.length || groups.value.some((g) => !g.name || !g.values.length)) return []
  let combos: string[][] = [[]]
  for (const g of groups.value) {
    combos = combos.flatMap((c) => g.values.map((v) => [...c, v]))
  }
  return combos.map((c) => {
    const specText = c.join('/')
    const specJson = JSON.stringify(Object.fromEntries(groups.value.map((g, i) => [g.name, c[i]])))
    // 保留已有 SKU 的 price/stock/id（按 spec_text 匹配）
    const existing = props.modelValue.find((s) => s.specText === specText)
    return existing ?? { specText, specJson, price: 0, stock: 0, image: '', status: 1 }
  })
}

function regenerate() {
  emit('update:modelValue', cartesian())
}

function updateField(idx: number, field: 'price' | 'stock' | 'image', value: number | string) {
  const next = [...props.modelValue]
  ;(next[idx] as Record<string, unknown>)[field] = value
  emit('update:modelValue', next)
}
</script>

<template>
  <div class="sku-matrix">
    <div class="groups">
      <h4>规格属性</h4>
      <div v-for="(g, gIdx) in groups" :key="gIdx" class="group-row">
        <el-input v-model="g.name" placeholder="属性名（颜色/容量）" style="width:140px" />
        <span class="colon">:</span>
        <el-tag v-for="(v, vIdx) in g.values" :key="vIdx" closable @close="removeValue(gIdx, vIdx)" style="margin-right:6px">{{ v }}</el-tag>
        <el-input
          v-model="newValueDraft[gIdx]" placeholder="加值后回车"
          style="width:120px" @keyup.enter="addValue(gIdx)"
        />
        <el-button link type="danger" @click="removeGroup(gIdx)">删除属性</el-button>
      </div>
      <el-button @click="addGroup">+ 加属性</el-button>
      <el-button type="primary" plain @click="regenerate">生成 SKU 矩阵</el-button>
    </div>

    <h4 style="margin-top:24px">SKU 列表（{{ modelValue.length }} 个）</h4>
    <el-table :data="modelValue" border>
      <el-table-column prop="specText" label="规格" width="200" />
      <el-table-column label="价格 (元)" width="140">
        <template #default="{ row, $index }">
          <el-input :model-value="(row.price/100).toFixed(2)" @change="(v: string) => updateField($index, 'price', Math.round(Number(v)*100))" />
        </template>
      </el-table-column>
      <el-table-column label="库存" width="120">
        <template #default="{ row, $index }">
          <el-input-number :model-value="row.stock" :min="0" @change="(v) => updateField($index, 'stock', v as number)" />
        </template>
      </el-table-column>
      <el-table-column label="SKU 主图" width="200">
        <template #default="{ row, $index }">
          <el-input :model-value="row.image" placeholder="图片 URL" @change="(v: string) => updateField($index, 'image', v)" />
        </template>
      </el-table-column>
    </el-table>
  </div>
</template>

<style scoped>
.groups { padding:16px; background:#f9f9f9; border-radius:8px }
.group-row { display:flex; align-items:center; gap:8px; margin-bottom:12px }
.colon { color:#999 }
</style>
```

- [ ] **Step 8.4: build**

```bash
cd ~/workspace/go/mall/yw-mall-admin-fe/merchant && pnpm run build 2>&1 | tail -5
```

预期：0 错。

- [ ] **Step 8.5: commit**

```
feat(m2 fe): SkuMatrix + ImageUploader 组件

SkuMatrix.vue: 属性组多值 → 笛卡尔积自动生成 SKU 列表;
保留已有 SKU 的价格/库存按 specText 匹配 (避免重新生成丢数据)
ImageUploader.vue: el-upload 包装, 校 5MB+jpg/png/webp,
调 api/upload.ts → /merchant/v1/upload/image
api/upload.ts: multipart axios POST
```

---

## Task 9: FE product/edit.vue 商品编辑页

**Files**:
- Create: `yw-mall-admin-fe/merchant/src/views/product/edit.vue`

- [ ] **Step 9.1: 写完整编辑页**

逻辑：
- mounted: 路由 :id 存在 → 调 getProduct 填表，路由 /new → 空表
- 表单：name, description, brand, categoryId, detail (textarea), images (ImageUploader, 多张)
- SkuMatrix 子组件绑定 sku 数组
- 提交：
  - 新建模式：createProduct({...form, skus}) → 拿 newId → batchUpsertSkus(newId, skus) 不再需要因为 create 已收 skus → 跳列表
  - 编辑模式：先 updateProduct({...form, 不含 skus}) 再 batchUpsertSkus(id, skus)

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { ElMessage, type FormInstance, type FormRules } from 'element-plus'
import PageContainer from '@/components/PageContainer.vue'
import SkuMatrix from '@/components/SkuMatrix.vue'
import ImageUploader from '@/components/ImageUploader.vue'
import {
  getProduct, createProduct, updateProduct, batchUpsertSkus,
  type SkuInput,
} from '@/api/products'

const route = useRoute()
const router = useRouter()
const productId = ref<number>(0)
const isEdit = ref(false)
const loading = ref(false)
const submitting = ref(false)
const formRef = ref<FormInstance>()

const form = ref({
  name: '', description: '', brand: '', categoryId: 1, detail: '',
  images: [] as string[],
})
const skus = ref<SkuInput[]>([])

const rules: FormRules = {
  name: [{ required: true, message: '商品名必填', trigger: 'blur' }],
  categoryId: [{ required: true, message: '分类必选', trigger: 'change' }],
}

onMounted(async () => {
  const idStr = route.params.id as string | undefined
  if (idStr && idStr !== 'new') {
    isEdit.value = true
    productId.value = Number(idStr)
    loading.value = true
    try {
      const p = await getProduct(productId.value)
      form.value = {
        name: p.name, description: p.description ?? '', brand: p.brand ?? '',
        categoryId: p.categoryId, detail: p.detail ?? '',
        images: (p.images ?? '').split(',').filter(Boolean),
      }
      skus.value = (p.skus ?? []).map((s) => ({
        id: s.id, skuCode: s.skuCode, specText: s.specText, specJson: s.specJson,
        price: s.price, stock: s.stock, image: s.image, status: s.status,
      }))
    } finally { loading.value = false }
  }
})

async function onSubmit() {
  if (!formRef.value) return
  try { await formRef.value.validate() } catch { return }
  if (skus.value.length === 0) {
    ElMessage.warning('至少 1 个 SKU；如无规格请生成 1 行 default')
    return
  }
  submitting.value = true
  try {
    const imagesStr = form.value.images.join(',')
    if (isEdit.value) {
      await updateProduct(productId.value, {
        name: form.value.name, description: form.value.description,
        brand: form.value.brand, detail: form.value.detail,
        categoryId: form.value.categoryId, images: imagesStr,
      })
      await batchUpsertSkus(productId.value, skus.value)
      ElMessage.success('已保存')
    } else {
      const r = await createProduct({
        name: form.value.name, description: form.value.description,
        brand: form.value.brand, detail: form.value.detail,
        categoryId: form.value.categoryId, images: imagesStr,
        skus: skus.value,
      })
      ElMessage.success(`已创建商品 #${r.id}`)
    }
    router.replace('/products')
  } finally { submitting.value = false }
}
</script>

<template>
  <PageContainer :title="isEdit ? '编辑商品' : '新建商品'" v-loading="loading">
    <el-form ref="formRef" :model="form" :rules="rules" label-width="100px" style="max-width:900px">
      <el-form-item label="商品名" prop="name">
        <el-input v-model="form.name" />
      </el-form-item>
      <el-form-item label="简介">
        <el-input v-model="form.description" type="textarea" :rows="2" />
      </el-form-item>
      <el-form-item label="品牌">
        <el-input v-model="form.brand" style="width:240px" />
      </el-form-item>
      <el-form-item label="分类" prop="categoryId">
        <el-input-number v-model="form.categoryId" :min="1" />
        <span class="hint">（分类树 M3 接，先填 id）</span>
      </el-form-item>
      <el-form-item label="主图（多张）">
        <ImageUploader v-model="form.images" :max="5" />
      </el-form-item>
      <el-form-item label="详情">
        <el-input v-model="form.detail" type="textarea" :rows="6" />
      </el-form-item>
      <el-form-item label="SKU 规格">
        <SkuMatrix v-model="skus" style="width:100%" />
      </el-form-item>
      <el-form-item>
        <el-button type="primary" :loading="submitting" @click="onSubmit">
          {{ isEdit ? '保存修改' : '创建商品' }}
        </el-button>
        <el-button @click="$router.back()">取消</el-button>
      </el-form-item>
    </el-form>
  </PageContainer>
</template>

<style scoped>
.hint { color: #999; font-size: 12px; margin-left: 12px }
</style>
```

- [ ] **Step 9.2: build**

```bash
cd ~/workspace/go/mall/yw-mall-admin-fe/merchant && pnpm run build 2>&1 | tail -5
```

预期：0 错。

- [ ] **Step 9.3: commit**

```
feat(m2 fe): /products/:id/edit + /products/new 商品编辑页

整合 SkuMatrix + ImageUploader. 新建/编辑双模式.
保存路径: createProduct(含 skus) 或 updateProduct (SPU 字段) +
batchUpsertSkus (SKU 矩阵).
```

---

## Task 10: E2E smoke + daily 收尾

- [ ] **Step 10.1: 浏览器 e2e**

```bash
cd ~/workspace/go/mall/yw-mall-admin-fe/merchant && pnpm dev
# 浏览器开 http://localhost:5175/login alice / alice123 登录
# 进 /products → 「新建商品」
# 填表 + 多张图 + 加 2 个属性 (颜色:红/黑, 容量:128/256) → 生成 4 SKU
# 每 SKU 填价格库存 → 保存
# 回 /products 看到新商品 + price=MIN/stock=SUM
# 点编辑 → 改个 SKU 价格 → 保存 → 列表立刻反映
```

- [ ] **Step 10.2: 写 daily 2026-MM-DD.md**

按既定模板写 M2 daily，含 e2e 实测证据。

- [ ] **Step 10.3: 回写 plan**

把本 plan 所有 task 的 `- [ ]` 改 `- [x]`。

- [ ] **Step 10.4: 推送**

5 个 repo（yw-mall + yw-mall-admin + yw-mall-admin-fe + env + yw-mall-docs）逐个 push。

---

## 验收清单

- [ ] sku 表存在 + 历史 product 全部有 default SKU
- [ ] alice 创建带 4 SKU 多规格商品 → product 行 price=MIN/stock=SUM
- [ ] 编辑现有商品改 SKU 矩阵 → 列表立刻反映
- [ ] alice 上传 3 张图片 → MinIO bucket 存在 URL 可访问
- [ ] bob (warehouse, perm=product.write) 能改库存 + 上下架，但不能改店铺信息
- [ ] bob (finance, 无 product.write) 调 POST /products → 403 permission denied
- [ ] 老 c 端列表读 product 行（不读 sku）→ 见 price=MIN/stock=SUM 不报错

---

## 关联文档

- `plan/2026-05-27-merchant-workstation-design.md` — 整体设计
- `plan/2026-05-27-merchant-workstation-M1-plan.md` — M1 (登录 + RBAC)
- `daily/2026-05-27-evening.md` — M1 交付记录
