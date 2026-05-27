# 商家工作台 M1 — 商家登录 + 员工 RBAC + 邀请链接 实施计划

> **关联**: `plan/2026-05-27-merchant-workstation-design.md` 第 8 节 M1
> **For agentic workers**: 每个 Task 独立 commit，commit message 用 `feat(m1):` 前缀

**Goal**：店主和员工子账号均可登录商家工作台；店主可邀请/管理员工；FE 脚手架 + 登录页 + 接受邀请页 + 员工列表页就位。

**Architecture**：复用 `SessionAuthMiddleware{role:"merchant"}` 已就绪的鉴权层。新增 `merchant_staff` + `merchant_staff_invitation` 表 + shop-rpc 加 staff/invitation RPC。改造 `MerchantLogin` 从 `GetShopByOwnerId` 改成 `GetStaffByUserId`，注入 `(shopId, staffRole, perms)` 到 session.Perms。FE 复制 admin/ 脚手架到 merchant/，端口 18083。

**Tech Stack**：Go 1.26 / go-zero 1.10.1 / proto3 / Redis / MySQL / Vue 3 + Element Plus + Vite + Pinia

---

## 已就绪资产（不重做）

- ✅ `yw-mall-admin/internal/middleware/rbac.go` `SessionAuthMiddleware` (`merchantMw := NewSessionAuthMiddleware(userRpc, "merchant")`)
- ✅ `merchantLoginHandler` / `merchantRefreshHandler` / `merchantLogoutHandler` (yw-mall-admin/internal/handler/handlers.go:278)
- ✅ `MerchantLogin` / `MerchantRefresh` / `MerchantLogout` logic (yw-mall-admin/internal/logic/auth_logic.go) — **但 MerchantLogin 不支持员工子账号**，T5 改造
- ✅ `/merchant/v1/login` 路由已注册 (yw-mall-admin/internal/handler/routes.go:127)
- ✅ user-rpc 的 `CreateSession` 已接 Perms 字段（loginrevamp P0.5 加的）
- ✅ L1.6 destroyUserSessionsByRole 已按 role 过滤（本方案 Role="merchant" 直接受益）
- ✅ user-rpc 的 sendcodehelpers / emailhelpers / SendVerifyCode 已就绪（邀请短信/邮件复用）

## File Structure

```
yw-mall (backend)
  mall-user-rpc/sql/
    merchant_workstation.sql                CREATE — 2 张表 DDL
  mall-user-rpc/cmd/backfill_merchant_owner/
    main.go                                 CREATE — 历史 shop 回填脚本

  mall-common/proto/shop/
    shop.proto                              MODIFY — 加 8 个 staff/invitation RPC

  mall-shop-rpc/internal/logic/
    staffhelpers.go                         CREATE — staff/invitation 通用 helper
    merchantrolepermslogic.go               CREATE — role → perms 映射常量
    getstaffbyuseridlogic.go                CREATE
    listshopstafflogic.go                   CREATE
    updatestaffrolelogic.go                 CREATE
    disablestafflogic.go                    CREATE
    createinvitationlogic.go                CREATE
    listpendinginvitationslogic.go          CREATE
    revokeinvitationlogic.go                CREATE
    acceptinvitationlogic.go                CREATE
  mall-shop-rpc/internal/server/
    shopserviceserver.go                    AUTO-REGEN
  mall-shop-rpc/shop/
    shop.pb.go / shop_grpc.pb.go            AUTO-REGEN
  mall-shop-rpc/shopservice/
    shop.go                                 MANUAL — 加 8 个 client wrapper

  yw-mall-admin/internal/types/types.go     MODIFY — 加 staff/invitation Req/Resp
  yw-mall-admin/internal/logic/
    auth_logic.go                           MODIFY — MerchantLogin 接 staff 表
    merchant_staff_logic.go                 CREATE — staff CRUD
    merchant_invitation_logic.go            CREATE — invitation CRUD + accept
  yw-mall-admin/internal/handler/
    merchant_staff_handlers.go              CREATE
    merchant_invitation_handlers.go         CREATE
    routes.go                               MODIFY — 加 7 个 /merchant/v1/staff/* 路由

  start.sh                                  MODIFY — SPRINT_MIGRATIONS 加 merchant_workstation.sql

yw-mall-admin-fe/merchant/                  整个目录从 admin/ 克隆 + 改造
  package.json                              MODIFY — name + port
  vite.config.ts                            MODIFY — proxy /merchant -> :18999
  Dockerfile / nginx.conf                   CREATE — 复刻 admin 同款
  src/
    api/request.ts                          复制 admin
    api/auth.ts                             merchantLogin/refresh/logout
    api/staff.ts                            list/role/disable + invitation CRUD/accept
    api/shop.ts                             getMyShop
    stores/session.ts                       token+csrf+shopId+staffRole+perms
    router/index.ts                         路由 + 权限守卫
    layouts/MerchantLayout.vue              left nav + top user dropdown
    views/login/index.vue                   京东风登录页
    views/invite/accept.vue                 接受邀请页
    views/staff/list.vue                    员工列表 + 邀请/改角色/冻结
    views/staff/invite.vue                  发邀请 modal
    views/dashboard/index.vue               占位卡片（M6 完整接）
    composables/usePermission.ts            v-perm 指令 + can(perm)

yw-mall-deploy/compose.yml                  MODIFY — 加 mall-merchant-fe 服务 18083:80
```

---

## Task 1: 数据表 DDL + start.sh 迁移

**Files**:
- Create: `mall-user-rpc/sql/merchant_workstation.sql`
- Modify: `yw-mall/start.sh` (SPRINT_MIGRATIONS 数组)

- [ ] **Step 1.1: 写 DDL**

`mall-user-rpc/sql/merchant_workstation.sql`:
```sql
-- M1 商家工作台：员工 RBAC + 邀请链接
-- 表归属 mall_user 库（与 user 表同库，因 FK user.id）

CREATE TABLE IF NOT EXISTS merchant_staff (
  id              BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  shop_id         BIGINT UNSIGNED NOT NULL,
  user_id         BIGINT UNSIGNED NOT NULL,
  role            VARCHAR(32) NOT NULL,            -- owner|service|warehouse|finance
  status          TINYINT NOT NULL DEFAULT 1,      -- 1=active, 0=disabled
  invited_by      BIGINT UNSIGNED NOT NULL DEFAULT 0,
  joined_at       BIGINT NOT NULL,
  create_time     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  update_time     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uk_shop_user (shop_id, user_id),
  KEY idx_user (user_id),
  KEY idx_shop_status (shop_id, status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS merchant_staff_invitation (
  id              BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  shop_id         BIGINT UNSIGNED NOT NULL,
  invited_by      BIGINT UNSIGNED NOT NULL,        -- staff.user_id（店主 uid）
  target_phone    VARCHAR(20) NOT NULL DEFAULT '',
  target_email    VARCHAR(255) NOT NULL DEFAULT '',
  role            VARCHAR(32) NOT NULL,
  invitation_code VARCHAR(64) NOT NULL UNIQUE,
  status          TINYINT NOT NULL DEFAULT 0,      -- 0=pending, 1=accepted, 2=expired, 3=revoked
  expires_at      BIGINT NOT NULL,                 -- unix +7d
  accepted_by     BIGINT UNSIGNED NOT NULL DEFAULT 0,
  accepted_at     BIGINT NOT NULL DEFAULT 0,
  create_time     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  KEY idx_shop (shop_id),
  KEY idx_status (status, expires_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

- [ ] **Step 1.2: start.sh 注入**

找到 `SPRINT_MIGRATIONS` 数组（约 line 200 附近），追加：
```bash
"mall_user:mall-user-rpc/sql/merchant_workstation.sql"
```

- [ ] **Step 1.3: 跑迁移验证**

```bash
cd ~/workspace/go/mall/yw-mall && ./start.sh bootstrap 2>&1 | grep -E "merchant_workstation|OK.*mall_user"
podman exec mysql-master1 mysql -uroot -proot123 mall_user -e "SHOW TABLES LIKE 'merchant_%';" 2>&1 | grep -v Warning
```

预期：两表存在。

- [ ] **Step 1.4: commit**

```
feat(m1): 数据表 merchant_staff + merchant_staff_invitation

mall_user 库新增 2 张表支持商家工作台员工 RBAC。staff 表唯一约束
(shop_id, user_id) 防止重复绑定；invitation 表用 invitation_code 当
不可猜 token，7 天过期。start.sh SPRINT_MIGRATIONS 注册迁移。
```

---

## Task 2: 历史 shop 数据回填

**Files**:
- Create: `mall-user-rpc/cmd/backfill_merchant_owner/main.go`

- [ ] **Step 2.1: 写脚本**

```go
// mall-user-rpc/cmd/backfill_merchant_owner/main.go
// 一次性脚本：每条 shop 自动 INSERT 一条 role=owner 的 staff 记录。
// 幂等：UNIQUE(shop_id, user_id) 保护，重跑无副作用。
package main

import (
    "database/sql"
    "flag"
    "fmt"
    "log"
    "time"

    _ "github.com/go-sql-driver/mysql"
)

func main() {
    userDSN := flag.String("user-dsn",
        "proxysql:proxysql123@tcp(127.0.0.1:6033)/mall_user?charset=utf8mb4&parseTime=true&loc=Local",
        "mall_user DSN")
    shopDSN := flag.String("shop-dsn",
        "proxysql:proxysql123@tcp(127.0.0.1:6033)/mall_shop?charset=utf8mb4&parseTime=true&loc=Local",
        "mall_shop DSN")
    flag.Parse()

    udb, err := sql.Open("mysql", *userDSN)
    if err != nil { log.Fatal(err) }
    defer udb.Close()
    sdb, err := sql.Open("mysql", *shopDSN)
    if err != nil { log.Fatal(err) }
    defer sdb.Close()

    rows, err := sdb.Query("SELECT id, owner_id FROM shop WHERE owner_id > 0")
    if err != nil { log.Fatal(err) }
    defer rows.Close()

    inserted, skipped := 0, 0
    now := time.Now().Unix()
    for rows.Next() {
        var sid, oid uint64
        if err := rows.Scan(&sid, &oid); err != nil { log.Fatal(err) }
        res, err := udb.Exec(`
            INSERT IGNORE INTO merchant_staff
              (shop_id, user_id, role, status, invited_by, joined_at)
            VALUES (?, ?, 'owner', 1, ?, ?)`, sid, oid, oid, now)
        if err != nil { log.Fatal(err) }
        n, _ := res.RowsAffected()
        if n > 0 { inserted++ } else { skipped++ }
    }
    fmt.Printf("backfill done: inserted=%d skipped=%d\n", inserted, skipped)
}
```

- [ ] **Step 2.2: 跑一遍**

```bash
cd ~/workspace/go/mall/yw-mall/mall-user-rpc && go run ./cmd/backfill_merchant_owner/
```

预期：`backfill done: inserted=N skipped=0`（N=现存 shop 数）

验证：
```bash
podman exec mysql-master1 mysql -uroot -proot123 mall_user -e "SELECT shop_id,user_id,role FROM merchant_staff;" 2>&1 | grep -v Warning
```

- [ ] **Step 2.3: commit**

```
feat(m1): 历史 shop 回填 owner staff 记录

cmd/backfill_merchant_owner 一次性脚本，对每条 shop 自动 INSERT IGNORE
一条 role=owner 的 merchant_staff。幂等可重跑。
```

---

## Task 3: shop-rpc proto 加 8 个 RPC

**Files**:
- Modify: `mall-common/proto/shop/shop.proto`
- Auto-regen: `mall-shop-rpc/shop/shop.pb.go`, `shop_grpc.pb.go`, `mall-shop-rpc/internal/server/shopserviceserver.go`
- Manual: `mall-shop-rpc/shopservice/shop.go`

- [ ] **Step 3.1: proto 文件加 messages + service rpc**

在 `mall-common/proto/shop/shop.proto` 末尾追加（具体位置看现有结构调整）：

```protobuf
// ===== M1 merchant staff & invitation =====
message StaffInfo {
  int64 id          = 1;
  int64 shop_id     = 2;
  int64 user_id     = 3;
  string username   = 4;
  string role       = 5;
  int32 status      = 6;
  int64 joined_at   = 7;
}

message GetStaffByUserIdReq { int64 user_id = 1; }
message GetStaffByUserIdResp {
  bool found       = 1;
  StaffInfo staff  = 2;
  repeated string perms = 3;
}

message ListShopStaffReq  { int64 shop_id = 1; }
message ListShopStaffResp { repeated StaffInfo items = 1; }

message UpdateStaffRoleReq {
  int64 shop_id   = 1;  // 用作权限校验（操作者所在 shop）
  int64 staff_id  = 2;
  string new_role = 3;
}
message DisableStaffReq {
  int64 shop_id  = 1;
  int64 staff_id = 2;
}
message OkResp { bool ok = 1; }

message InvitationInfo {
  int64 id              = 1;
  int64 shop_id         = 2;
  string target_phone   = 3;
  string target_email   = 4;
  string role           = 5;
  int32 status          = 6;
  int64 expires_at      = 7;
  int64 create_time     = 8;
}

message CreateInvitationReq {
  int64 shop_id        = 1;
  int64 invited_by_uid = 2;
  string target_phone  = 3;
  string target_email  = 4;
  string role          = 5;
}
message CreateInvitationResp {
  string invitation_code = 1;
  int64 expires_at       = 2;
}

message ListPendingInvitationsReq  { int64 shop_id = 1; }
message ListPendingInvitationsResp { repeated InvitationInfo items = 1; }

message RevokeInvitationReq { int64 shop_id = 1; int64 invitation_id = 2; }

message AcceptInvitationReq {
  string invitation_code = 1;
  int64 acceptor_uid     = 2;   // 当前登录用户
  string acceptor_phone  = 3;   // 校验与 target 一致
  string acceptor_email  = 4;
}
message AcceptInvitationResp {
  int64 shop_id     = 1;
  string role       = 2;
  string shop_name  = 3;
}

service ShopService {
  // ... existing rpcs ...

  rpc GetStaffByUserId      (GetStaffByUserIdReq)      returns (GetStaffByUserIdResp);
  rpc ListShopStaff         (ListShopStaffReq)         returns (ListShopStaffResp);
  rpc UpdateStaffRole       (UpdateStaffRoleReq)       returns (OkResp);
  rpc DisableStaff          (DisableStaffReq)          returns (OkResp);

  rpc CreateInvitation      (CreateInvitationReq)      returns (CreateInvitationResp);
  rpc ListPendingInvitations(ListPendingInvitationsReq) returns (ListPendingInvitationsResp);
  rpc RevokeInvitation      (RevokeInvitationReq)      returns (OkResp);
  rpc AcceptInvitation      (AcceptInvitationReq)      returns (AcceptInvitationResp);
}
```

- [ ] **Step 3.2: 重生成 stub**

```bash
export PATH=/home/carter/workspace/go/bin:$PATH
cd ~/workspace/go/mall/yw-mall/mall-shop-rpc
protoc --go_out=. --go-grpc_out=. \
  --proto_path=. --proto_path=../mall-common/proto \
  ../mall-common/proto/shop/shop.proto
```

- [ ] **Step 3.3: 手动同步 shopservice/shop.go**

打开 `mall-shop-rpc/shopservice/shop.go`，按现有 method 风格（NewXxxLogic + 直接 forward）加 8 个 client wrapper。参考相邻已有 method 的模板。

- [ ] **Step 3.4: build 确认 server 编译过**

```bash
cd ~/workspace/go/mall/yw-mall/mall-shop-rpc && go build ./... 2>&1 | head -10
```

预期：报"undefined: NewGetStaffByUserIdLogic" 等 8 个错（server stub 引用了 logic，Task 4-7 实现）。这是预期的。

- [ ] **Step 3.5: commit**

```
feat(m1 proto): shop 加 8 个 staff + invitation RPC

GetStaffByUserId 给 MerchantLogin 用；List/Update/Disable 给店主管理员工；
Create/List/Revoke/Accept Invitation 给邀请链接流程。
proto regen + shopservice 手动同步 wrapper.

WIP: build 待 Task 4-7 实现 logic 才通过.
```

---

## Task 4: shop-rpc 实现 staff CRUD logic + role-perms 映射

**Files**:
- Create: `mall-shop-rpc/internal/logic/merchantrolepermslogic.go` (role→perms 常量)
- Create: `mall-shop-rpc/internal/logic/staffhelpers.go`
- Create: `mall-shop-rpc/internal/logic/getstaffbyuseridlogic.go`
- Create: `mall-shop-rpc/internal/logic/listshopstafflogic.go`
- Create: `mall-shop-rpc/internal/logic/updatestaffrolelogic.go`
- Create: `mall-shop-rpc/internal/logic/disablestafflogic.go`

- [ ] **Step 4.1: 写 role→perms 常量**

```go
// mall-shop-rpc/internal/logic/merchantrolepermslogic.go
package logic

const (
    RoleOwner     = "owner"
    RoleService   = "service"
    RoleWarehouse = "warehouse"
    RoleFinance   = "finance"
)

// RolePerms 各角色固定权限码集合。owner 用 "*" 占位表示全开,
// 中间件可特判 "*" 跳过 perm 比对。
var RolePerms = map[string][]string{
    RoleOwner: {"*"},
    RoleService: {
        "shop.read", "product.read", "order.read", "order.write",
        "refund.read", "refund.handle", "staff.read",
    },
    RoleWarehouse: {
        "shop.read", "product.read", "product.write",
        "order.read", "order.ship",
        "refund.read", "refund.inspect",
        "staff.read", "freight.read",
    },
    RoleFinance: {
        "shop.read", "order.read",
        "finance.read", "finance.write",
        "staff.read",
    },
}

func IsValidRole(r string) bool {
    _, ok := RolePerms[r]
    return ok && r != "" 
}
```

- [ ] **Step 4.2: 写 staffhelpers.go**

```go
// mall-shop-rpc/internal/logic/staffhelpers.go
package logic

import (
    "context"
    "errors"

    "mall-shop-rpc/internal/svc"

    "github.com/zeromicro/go-zero/core/stores/sqlx"
)

type staffRow struct {
    Id       uint64 `db:"id"`
    ShopId   uint64 `db:"shop_id"`
    UserId   uint64 `db:"user_id"`
    Role     string `db:"role"`
    Status   int32  `db:"status"`
    JoinedAt int64  `db:"joined_at"`
}

func queryStaffByUserId(ctx context.Context, svcCtx *svc.ServiceContext, uid int64) (*staffRow, error) {
    var s staffRow
    err := svcCtx.UserDB.QueryRowCtx(ctx, &s, `
        SELECT id, shop_id, user_id, role, status, joined_at
        FROM merchant_staff WHERE user_id=? AND status=1 LIMIT 1`, uid)
    if err == sqlx.ErrNotFound {
        return nil, nil
    }
    if err != nil {
        return nil, err
    }
    return &s, nil
}

// requireShopOwner 校验调用方（uid）是 shopId 的 owner，否则拒。
func requireShopOwner(ctx context.Context, svcCtx *svc.ServiceContext, shopId, uid int64) error {
    if shopId <= 0 || uid <= 0 {
        return errors.New("invalid shop or uid")
    }
    var role string
    err := svcCtx.UserDB.QueryRowCtx(ctx, &role,
        `SELECT role FROM merchant_staff
         WHERE shop_id=? AND user_id=? AND status=1 LIMIT 1`, shopId, uid)
    if err != nil {
        return errors.New("not a staff of this shop")
    }
    if role != RoleOwner {
        return errors.New("only shop owner can perform this action")
    }
    return nil
}
```

注：`svcCtx.UserDB` 需要在 svc.ServiceContext 加一个指向 `mall_user` 库的 sqlx.SqlConn（因为 merchant_staff 表在 mall_user 库）。

加到 `mall-shop-rpc/internal/svc/servicecontext.go`：
```go
type ServiceContext struct {
    Config    config.Config
    DB        sqlx.SqlConn  // mall_shop 已有
    UserDB    sqlx.SqlConn  // 新增 mall_user
    // ... rest
}

// 构造里加：
UserDB: sqlx.NewMysql(c.UserDataSource),
```

config.go 加：
```go
type Config struct {
    // ...
    DataSource     string
    UserDataSource string  // 新增
}
```

etc/shop.yaml 加：
```yaml
DataSource: proxysql:proxysql123@tcp(127.0.0.1:6033)/mall_shop?...
UserDataSource: proxysql:proxysql123@tcp(127.0.0.1:6033)/mall_user?...
```

- [ ] **Step 4.3: 写 getstaffbyuseridlogic.go**

```go
// mall-shop-rpc/internal/logic/getstaffbyuseridlogic.go
package logic

import (
    "context"

    "mall-shop-rpc/internal/svc"
    "mall-shop-rpc/shop"

    "github.com/zeromicro/go-zero/core/logx"
)

type GetStaffByUserIdLogic struct {
    ctx    context.Context
    svcCtx *svc.ServiceContext
    logx.Logger
}

func NewGetStaffByUserIdLogic(ctx context.Context, svcCtx *svc.ServiceContext) *GetStaffByUserIdLogic {
    return &GetStaffByUserIdLogic{ctx: ctx, svcCtx: svcCtx, Logger: logx.WithContext(ctx)}
}

func (l *GetStaffByUserIdLogic) GetStaffByUserId(in *shop.GetStaffByUserIdReq) (*shop.GetStaffByUserIdResp, error) {
    s, err := queryStaffByUserId(l.ctx, l.svcCtx, in.UserId)
    if err != nil {
        return nil, err
    }
    if s == nil {
        return &shop.GetStaffByUserIdResp{Found: false}, nil
    }
    // 拿 username
    var username string
    _ = l.svcCtx.UserDB.QueryRowCtx(l.ctx, &username,
        "SELECT username FROM `user` WHERE id=?", s.UserId)
    return &shop.GetStaffByUserIdResp{
        Found: true,
        Staff: &shop.StaffInfo{
            Id: int64(s.Id), ShopId: int64(s.ShopId), UserId: int64(s.UserId),
            Username: username, Role: s.Role, Status: s.Status, JoinedAt: s.JoinedAt,
        },
        Perms: RolePerms[s.Role],
    }, nil
}
```

- [ ] **Step 4.4: 写 listshopstafflogic.go**

```go
// mall-shop-rpc/internal/logic/listshopstafflogic.go
package logic

import (
    "context"

    "mall-shop-rpc/internal/svc"
    "mall-shop-rpc/shop"

    "github.com/zeromicro/go-zero/core/logx"
)

type ListShopStaffLogic struct {
    ctx    context.Context
    svcCtx *svc.ServiceContext
    logx.Logger
}

func NewListShopStaffLogic(ctx context.Context, svcCtx *svc.ServiceContext) *ListShopStaffLogic {
    return &ListShopStaffLogic{ctx: ctx, svcCtx: svcCtx, Logger: logx.WithContext(ctx)}
}

func (l *ListShopStaffLogic) ListShopStaff(in *shop.ListShopStaffReq) (*shop.ListShopStaffResp, error) {
    var rows []struct {
        Id       uint64 `db:"id"`
        UserId   uint64 `db:"user_id"`
        Username string `db:"username"`
        Role     string `db:"role"`
        Status   int32  `db:"status"`
        JoinedAt int64  `db:"joined_at"`
    }
    err := l.svcCtx.UserDB.QueryRowsCtx(l.ctx, &rows, `
        SELECT s.id, s.user_id, u.username, s.role, s.status, s.joined_at
        FROM merchant_staff s
        LEFT JOIN `+"`user`"+` u ON u.id=s.user_id
        WHERE s.shop_id=?
        ORDER BY (s.role='owner') DESC, s.joined_at`, in.ShopId)
    if err != nil {
        return nil, err
    }
    out := make([]*shop.StaffInfo, 0, len(rows))
    for _, r := range rows {
        out = append(out, &shop.StaffInfo{
            Id: int64(r.Id), ShopId: in.ShopId, UserId: int64(r.UserId),
            Username: r.Username, Role: r.Role, Status: r.Status, JoinedAt: r.JoinedAt,
        })
    }
    return &shop.ListShopStaffResp{Items: out}, nil
}
```

- [ ] **Step 4.5: 写 updatestaffrolelogic.go**

```go
// mall-shop-rpc/internal/logic/updatestaffrolelogic.go
package logic

import (
    "context"
    "errors"

    "mall-shop-rpc/internal/svc"
    "mall-shop-rpc/shop"

    "github.com/zeromicro/go-zero/core/logx"
)

type UpdateStaffRoleLogic struct {
    ctx    context.Context
    svcCtx *svc.ServiceContext
    logx.Logger
}

func NewUpdateStaffRoleLogic(ctx context.Context, svcCtx *svc.ServiceContext) *UpdateStaffRoleLogic {
    return &UpdateStaffRoleLogic{ctx: ctx, svcCtx: svcCtx, Logger: logx.WithContext(ctx)}
}

func (l *UpdateStaffRoleLogic) UpdateStaffRole(in *shop.UpdateStaffRoleReq) (*shop.OkResp, error) {
    if !IsValidRole(in.NewRole) {
        return nil, errors.New("invalid role")
    }
    if in.NewRole == RoleOwner {
        return nil, errors.New("cannot transfer owner via this endpoint")
    }
    // 防止把唯一 owner 改成普通角色
    var currentRole string
    if err := l.svcCtx.UserDB.QueryRowCtx(l.ctx, &currentRole,
        "SELECT role FROM merchant_staff WHERE id=? AND shop_id=?",
        in.StaffId, in.ShopId); err != nil {
        return nil, errors.New("staff not found")
    }
    if currentRole == RoleOwner {
        return nil, errors.New("cannot change owner role here")
    }
    if _, err := l.svcCtx.UserDB.ExecCtx(l.ctx,
        "UPDATE merchant_staff SET role=? WHERE id=? AND shop_id=?",
        in.NewRole, in.StaffId, in.ShopId); err != nil {
        return nil, err
    }
    return &shop.OkResp{Ok: true}, nil
}
```

- [ ] **Step 4.6: 写 disablestafflogic.go**

```go
// mall-shop-rpc/internal/logic/disablestafflogic.go
package logic

import (
    "context"
    "errors"

    "mall-shop-rpc/internal/svc"
    "mall-shop-rpc/shop"

    "github.com/zeromicro/go-zero/core/logx"
)

type DisableStaffLogic struct {
    ctx    context.Context
    svcCtx *svc.ServiceContext
    logx.Logger
}

func NewDisableStaffLogic(ctx context.Context, svcCtx *svc.ServiceContext) *DisableStaffLogic {
    return &DisableStaffLogic{ctx: ctx, svcCtx: svcCtx, Logger: logx.WithContext(ctx)}
}

func (l *DisableStaffLogic) DisableStaff(in *shop.DisableStaffReq) (*shop.OkResp, error) {
    var role string
    if err := l.svcCtx.UserDB.QueryRowCtx(l.ctx, &role,
        "SELECT role FROM merchant_staff WHERE id=? AND shop_id=?",
        in.StaffId, in.ShopId); err != nil {
        return nil, errors.New("staff not found")
    }
    if role == RoleOwner {
        return nil, errors.New("cannot disable owner")
    }
    if _, err := l.svcCtx.UserDB.ExecCtx(l.ctx,
        "UPDATE merchant_staff SET status=0 WHERE id=? AND shop_id=?",
        in.StaffId, in.ShopId); err != nil {
        return nil, err
    }
    return &shop.OkResp{Ok: true}, nil
}
```

- [ ] **Step 4.7: build 验证**

```bash
cd ~/workspace/go/mall/yw-mall/mall-shop-rpc && go build ./... 2>&1 | head -10
```

预期：仍报缺 invitation logic 4 个，Task 5 补完。

- [ ] **Step 4.8: commit**

```
feat(m1 shop-rpc): staff CRUD + role-perms 映射

- merchantrolepermslogic.go: RolePerms[role] 映射 4 角色的权限码集合,
  owner 用 "*" 占位
- staffhelpers.go: queryStaffByUserId 给 MerchantLogin 用,
  requireShopOwner 给改员工角色/冻结校验权限
- 4 个 logic: GetStaffByUserId / ListShopStaff / UpdateStaffRole / DisableStaff
- svc 加 UserDB (mall_user 库) 因为 merchant_staff 表跨库

WIP: build 待 invitation 4 个 logic.
```

---

## Task 5: shop-rpc 实现 invitation logic

**Files**:
- Create: `mall-shop-rpc/internal/logic/createinvitationlogic.go`
- Create: `mall-shop-rpc/internal/logic/listpendinginvitationslogic.go`
- Create: `mall-shop-rpc/internal/logic/revokeinvitationlogic.go`
- Create: `mall-shop-rpc/internal/logic/acceptinvitationlogic.go`

- [ ] **Step 5.1: 加 invitation helper（在 staffhelpers.go 续写）**

```go
// 续 staffhelpers.go
import (
    "crypto/rand"
    "encoding/base64"
    "regexp"
)

const invitationTTL = 7 * 24 * 3600 // 7d

var (
    invPhoneRE = regexp.MustCompile(`^\d{11}$`)
    invEmailRE = regexp.MustCompile(`^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$`)
)

func newInvitationCode() (string, error) {
    b := make([]byte, 32)
    if _, err := rand.Read(b); err != nil {
        return "", err
    }
    return base64.RawURLEncoding.EncodeToString(b), nil
}
```

- [ ] **Step 5.2: createinvitationlogic.go**

```go
package logic

import (
    "context"
    "errors"
    "time"

    "mall-shop-rpc/internal/svc"
    "mall-shop-rpc/shop"

    "github.com/zeromicro/go-zero/core/logx"
)

type CreateInvitationLogic struct {
    ctx    context.Context
    svcCtx *svc.ServiceContext
    logx.Logger
}

func NewCreateInvitationLogic(ctx context.Context, svcCtx *svc.ServiceContext) *CreateInvitationLogic {
    return &CreateInvitationLogic{ctx: ctx, svcCtx: svcCtx, Logger: logx.WithContext(ctx)}
}

func (l *CreateInvitationLogic) CreateInvitation(in *shop.CreateInvitationReq) (*shop.CreateInvitationResp, error) {
    if !IsValidRole(in.Role) || in.Role == RoleOwner {
        return nil, errors.New("invalid role; owner cannot be invited")
    }
    if in.TargetPhone == "" && in.TargetEmail == "" {
        return nil, errors.New("target phone or email required")
    }
    if in.TargetPhone != "" && !invPhoneRE.MatchString(in.TargetPhone) {
        return nil, errors.New("invalid phone")
    }
    if in.TargetEmail != "" && !invEmailRE.MatchString(in.TargetEmail) {
        return nil, errors.New("invalid email")
    }
    if err := requireShopOwner(l.ctx, l.svcCtx, in.ShopId, in.InvitedByUid); err != nil {
        return nil, err
    }

    // 软上限：每店最多 20 staff（含 pending invitation）
    var cnt int64
    _ = l.svcCtx.UserDB.QueryRowCtx(l.ctx, &cnt, `
        SELECT (SELECT COUNT(*) FROM merchant_staff WHERE shop_id=? AND status=1)
             + (SELECT COUNT(*) FROM merchant_staff_invitation WHERE shop_id=? AND status=0 AND expires_at>?)`,
        in.ShopId, in.ShopId, time.Now().Unix())
    if cnt >= 20 {
        return nil, errors.New("staff cap reached (20)")
    }

    code, err := newInvitationCode()
    if err != nil {
        return nil, err
    }
    now := time.Now().Unix()
    expires := now + invitationTTL
    _, err = l.svcCtx.UserDB.ExecCtx(l.ctx, `
        INSERT INTO merchant_staff_invitation
          (shop_id, invited_by, target_phone, target_email, role, invitation_code, status, expires_at)
        VALUES (?, ?, ?, ?, ?, ?, 0, ?)`,
        in.ShopId, in.InvitedByUid, in.TargetPhone, in.TargetEmail, in.Role, code, expires)
    if err != nil {
        return nil, err
    }

    // mock SMS / Email 发送（与 reg-v2 同模式）
    target := in.TargetPhone
    if target == "" {
        target = in.TargetEmail
    }
    logx.WithContext(l.ctx).Infof("[mock-invite] target=%s code=%s role=%s shop_id=%d",
        target, code, in.Role, in.ShopId)

    return &shop.CreateInvitationResp{InvitationCode: code, ExpiresAt: expires}, nil
}
```

- [ ] **Step 5.3: listpendinginvitationslogic.go**

```go
package logic

import (
    "context"
    "time"

    "mall-shop-rpc/internal/svc"
    "mall-shop-rpc/shop"

    "github.com/zeromicro/go-zero/core/logx"
)

type ListPendingInvitationsLogic struct {
    ctx    context.Context
    svcCtx *svc.ServiceContext
    logx.Logger
}

func NewListPendingInvitationsLogic(ctx context.Context, svcCtx *svc.ServiceContext) *ListPendingInvitationsLogic {
    return &ListPendingInvitationsLogic{ctx: ctx, svcCtx: svcCtx, Logger: logx.WithContext(ctx)}
}

func (l *ListPendingInvitationsLogic) ListPendingInvitations(in *shop.ListPendingInvitationsReq) (*shop.ListPendingInvitationsResp, error) {
    var rows []struct {
        Id          uint64 `db:"id"`
        TargetPhone string `db:"target_phone"`
        TargetEmail string `db:"target_email"`
        Role        string `db:"role"`
        Status      int32  `db:"status"`
        ExpiresAt   int64  `db:"expires_at"`
        CreateTime  int64  `db:"create_time_unix"`
    }
    err := l.svcCtx.UserDB.QueryRowsCtx(l.ctx, &rows, `
        SELECT id, target_phone, target_email, role, status, expires_at,
               UNIX_TIMESTAMP(create_time) AS create_time_unix
        FROM merchant_staff_invitation
        WHERE shop_id=? AND status=0 AND expires_at>?
        ORDER BY id DESC`, in.ShopId, time.Now().Unix())
    if err != nil {
        return nil, err
    }
    out := make([]*shop.InvitationInfo, 0, len(rows))
    for _, r := range rows {
        out = append(out, &shop.InvitationInfo{
            Id: int64(r.Id), ShopId: in.ShopId,
            TargetPhone: r.TargetPhone, TargetEmail: r.TargetEmail,
            Role: r.Role, Status: r.Status,
            ExpiresAt: r.ExpiresAt, CreateTime: r.CreateTime,
        })
    }
    return &shop.ListPendingInvitationsResp{Items: out}, nil
}
```

- [ ] **Step 5.4: revokeinvitationlogic.go**

```go
package logic

import (
    "context"

    "mall-shop-rpc/internal/svc"
    "mall-shop-rpc/shop"

    "github.com/zeromicro/go-zero/core/logx"
)

type RevokeInvitationLogic struct {
    ctx    context.Context
    svcCtx *svc.ServiceContext
    logx.Logger
}

func NewRevokeInvitationLogic(ctx context.Context, svcCtx *svc.ServiceContext) *RevokeInvitationLogic {
    return &RevokeInvitationLogic{ctx: ctx, svcCtx: svcCtx, Logger: logx.WithContext(ctx)}
}

func (l *RevokeInvitationLogic) RevokeInvitation(in *shop.RevokeInvitationReq) (*shop.OkResp, error) {
    if _, err := l.svcCtx.UserDB.ExecCtx(l.ctx,
        "UPDATE merchant_staff_invitation SET status=3 WHERE id=? AND shop_id=? AND status=0",
        in.InvitationId, in.ShopId); err != nil {
        return nil, err
    }
    return &shop.OkResp{Ok: true}, nil
}
```

- [ ] **Step 5.5: acceptinvitationlogic.go**

```go
package logic

import (
    "context"
    "errors"
    "time"

    "mall-common/cryptox"
    "mall-shop-rpc/internal/svc"
    "mall-shop-rpc/shop"

    "github.com/zeromicro/go-zero/core/logx"
    "github.com/zeromicro/go-zero/core/stores/sqlx"
)

type AcceptInvitationLogic struct {
    ctx    context.Context
    svcCtx *svc.ServiceContext
    logx.Logger
}

func NewAcceptInvitationLogic(ctx context.Context, svcCtx *svc.ServiceContext) *AcceptInvitationLogic {
    return &AcceptInvitationLogic{ctx: ctx, svcCtx: svcCtx, Logger: logx.WithContext(ctx)}
}

func (l *AcceptInvitationLogic) AcceptInvitation(in *shop.AcceptInvitationReq) (*shop.AcceptInvitationResp, error) {
    if in.InvitationCode == "" || in.AcceptorUid <= 0 {
        return nil, errors.New("invalid request")
    }
    var inv struct {
        Id          uint64 `db:"id"`
        ShopId      uint64 `db:"shop_id"`
        TargetPhone string `db:"target_phone"`
        TargetEmail string `db:"target_email"`
        Role        string `db:"role"`
        ExpiresAt   int64  `db:"expires_at"`
        InvitedBy   uint64 `db:"invited_by"`
    }
    err := l.svcCtx.UserDB.QueryRowCtx(l.ctx, &inv, `
        SELECT id, shop_id, target_phone, target_email, role, expires_at, invited_by
        FROM merchant_staff_invitation
        WHERE invitation_code=? AND status=0 LIMIT 1`, in.InvitationCode)
    if err == sqlx.ErrNotFound {
        return nil, errors.New("invitation not found or already used")
    }
    if err != nil {
        return nil, err
    }
    if inv.ExpiresAt < time.Now().Unix() {
        _, _ = l.svcCtx.UserDB.ExecCtx(l.ctx,
            "UPDATE merchant_staff_invitation SET status=2 WHERE id=?", inv.Id)
        return nil, errors.New("invitation expired")
    }

    // 校验 acceptor 身份与 target 匹配（防代领）
    if inv.TargetPhone != "" {
        if cryptox.Hmac(in.AcceptorPhone) == "" || in.AcceptorPhone != inv.TargetPhone {
            return nil, errors.New("phone mismatch with invitation target")
        }
    } else if inv.TargetEmail != "" {
        if in.AcceptorEmail == "" || in.AcceptorEmail != inv.TargetEmail {
            return nil, errors.New("email mismatch with invitation target")
        }
    }

    // 防重复绑定
    var existed int64
    _ = l.svcCtx.UserDB.QueryRowCtx(l.ctx, &existed,
        "SELECT COUNT(*) FROM merchant_staff WHERE shop_id=? AND user_id=?",
        inv.ShopId, in.AcceptorUid)
    if existed > 0 {
        return nil, errors.New("already a staff of this shop")
    }

    now := time.Now().Unix()
    if _, err := l.svcCtx.UserDB.ExecCtx(l.ctx, `
        INSERT INTO merchant_staff (shop_id, user_id, role, status, invited_by, joined_at)
        VALUES (?, ?, ?, 1, ?, ?)`,
        inv.ShopId, in.AcceptorUid, inv.Role, inv.InvitedBy, now); err != nil {
        return nil, err
    }
    if _, err := l.svcCtx.UserDB.ExecCtx(l.ctx,
        "UPDATE merchant_staff_invitation SET status=1, accepted_by=?, accepted_at=? WHERE id=?",
        in.AcceptorUid, now, inv.Id); err != nil {
        return nil, err
    }

    var shopName string
    _ = l.svcCtx.DB.QueryRowCtx(l.ctx, &shopName,
        "SELECT shop_name FROM shop WHERE id=?", inv.ShopId)

    return &shop.AcceptInvitationResp{
        ShopId: int64(inv.ShopId), Role: inv.Role, ShopName: shopName,
    }, nil
}
```

- [ ] **Step 5.6: build + restart shop-rpc 验证**

```bash
cd ~/workspace/go/mall/yw-mall/mall-shop-rpc && go build ./...
cd ~/workspace/go/mall/yw-mall && ./start.sh restart 2>&1 | grep shop-rpc
sleep 2
tail -10 logs/shop-rpc.log
```

预期：build 0 错，启动正常。

- [ ] **Step 5.7: commit**

```
feat(m1 shop-rpc): invitation CRUD + accept

- CreateInvitation: 20 人软上限、校验 owner 才能发、生成 32 字节
  unguessable code、mock SMS/Email logx
- ListPendingInvitations: 未过期 + status=0
- RevokeInvitation: 软删 status=3
- AcceptInvitation: 校验 code 未过期+未消费、target 与 acceptor
  phone/email 匹配防代领、INSERT staff + UPDATE invitation status=1
```

---

## Task 6: 改造 MerchantLogin 接 staff 表

**Files**:
- Modify: `yw-mall-admin/internal/logic/auth_logic.go`

- [ ] **Step 6.1: 改 MerchantLogin**

```go
// 改造后 MerchantLogin (替换原实现)
func MerchantLogin(ctx context.Context, svcCtx *svc.ServiceContext, req *types.LoginReq) (*types.LoginResp, error) {
    loginResp, err := svcCtx.UserRpc.Login(ctx, &userclient.LoginReq{
        Username: req.Username, Password: req.Password,
    })
    if err != nil {
        return nil, err
    }

    // 新：查 merchant_staff，员工 / 店主都能登录
    sr, err := svcCtx.ShopRpc.GetStaffByUserId(ctx, &shopservice.GetStaffByUserIdReq{
        UserId: loginResp.Id,
    })
    if err != nil {
        return nil, fmt.Errorf("staff lookup failed: %w", err)
    }
    if !sr.Found {
        return nil, errors.New("user is not a staff of any shop; apply for one first")
    }

    sess, err := svcCtx.UserRpc.CreateSession(ctx, &userclient.CreateSessionReq{
        Uid:      loginResp.Id,
        Username: req.Username,
        Role:     "merchant",
        ShopId:   sr.Staff.ShopId,
        Perms:    sr.Perms,
    })
    if err != nil {
        return nil, err
    }
    return &types.LoginResp{
        Token: sess.AccessToken, RefreshToken: sess.RefreshToken,
        ExpiresIn: sess.ExpiresIn, CsrfToken: sess.CsrfToken,
        Id:       loginResp.Id, Username: req.Username,
        Role:     "merchant", ShopId: sr.Staff.ShopId,
        // LoginResp 可能没有 StaffRole 字段；types.go 加之
    }, nil
}
```

types.go 加字段（LoginResp 已有的话扩）：
```go
type LoginResp struct {
    // ...existing...
    StaffRole string   `json:"staffRole,omitempty"`
    Perms     []string `json:"perms,omitempty"`
}
```

回写 `Resp` 里：
```go
return &types.LoginResp{
    // ...
    StaffRole: sr.Staff.Role,
    Perms:     sr.Perms,
}, nil
```

- [ ] **Step 6.2: build + 重启 admin-api**

```bash
cd ~/workspace/go/mall/yw-mall/yw-mall-admin && go build ./...
cd ~/workspace/go/mall/yw-mall && ./start.sh restart 2>&1 | grep admin-api
sleep 2
tail -5 logs/admin-api.log
```

- [ ] **Step 6.3: e2e — 店主登录验证**

```bash
# alice 已是 shop_id=1 的 owner (T2 backfill 后)
curl -s -X POST http://localhost:18999/merchant/v1/login -H 'Content-Type: application/json' \
  -d '{"username":"alice","password":"Reset!2026X"}' | python3 -m json.tool
```

预期：返回 `token / role:"merchant" / shopId:1 / staffRole:"owner" / perms:["*"]`。

- [ ] **Step 6.4: commit**

```
feat(m1 admin-api): MerchantLogin 接 merchant_staff 支持员工子账号登录

原实现 GetShopByOwnerId 只支持店主一种角色。改为 GetStaffByUserId,
拿 (shopId, role, perms) 三元组，注入 session.Perms。
LoginResp 新增 staffRole + perms 字段给 FE 路由权限守卫用。
```

---

## Task 7: admin-api staff CRUD HTTP 路由

**Files**:
- Modify: `yw-mall-admin/internal/types/types.go`
- Create: `yw-mall-admin/internal/logic/merchant_staff_logic.go`
- Create: `yw-mall-admin/internal/handler/merchant_staff_handlers.go`
- Modify: `yw-mall-admin/internal/handler/routes.go`

- [ ] **Step 7.1: types.go 加 Req/Resp**

```go
// 加到 types.go 末尾
type StaffItemDTO struct {
    Id       int64  `json:"id"`
    UserId   int64  `json:"userId"`
    Username string `json:"username"`
    Role     string `json:"role"`
    Status   int32  `json:"status"`
    JoinedAt int64  `json:"joinedAt"`
}
type ListStaffResp struct{ Items []StaffItemDTO `json:"items"` }

type UpdateStaffRoleReq struct {
    NewRole string `json:"newRole"`
}
```

- [ ] **Step 7.2: merchant_staff_logic.go**

```go
package logic

import (
    "context"
    "errors"

    "yw-mall-admin/internal/middleware"
    "yw-mall-admin/internal/svc"
    "yw-mall-admin/internal/types"

    "mall-shop-rpc/shopservice"
)

func ListMerchantStaff(ctx context.Context, svcCtx *svc.ServiceContext) (*types.ListStaffResp, error) {
    c, _ := middleware.ClaimsFromContext(ctx)
    if c == nil || c.ShopId <= 0 {
        return nil, errors.New("not in a shop")
    }
    res, err := svcCtx.ShopRpc.ListShopStaff(ctx, &shopservice.ListShopStaffReq{ShopId: c.ShopId})
    if err != nil {
        return nil, err
    }
    out := make([]types.StaffItemDTO, 0, len(res.Items))
    for _, s := range res.Items {
        out = append(out, types.StaffItemDTO{
            Id: s.Id, UserId: s.UserId, Username: s.Username,
            Role: s.Role, Status: s.Status, JoinedAt: s.JoinedAt,
        })
    }
    return &types.ListStaffResp{Items: out}, nil
}

func UpdateMerchantStaffRole(ctx context.Context, svcCtx *svc.ServiceContext, staffId int64, req *types.UpdateStaffRoleReq) (*types.OkResp, error) {
    c, _ := middleware.ClaimsFromContext(ctx)
    if c == nil || c.ShopId <= 0 {
        return nil, errors.New("not in a shop")
    }
    if !hasPerm(c, "staff.write") {
        return nil, errors.New("permission denied: staff.write")
    }
    if _, err := svcCtx.ShopRpc.UpdateStaffRole(ctx, &shopservice.UpdateStaffRoleReq{
        ShopId: c.ShopId, StaffId: staffId, NewRole: req.NewRole,
    }); err != nil {
        return nil, err
    }
    return &types.OkResp{Ok: true}, nil
}

func DisableMerchantStaff(ctx context.Context, svcCtx *svc.ServiceContext, staffId int64) (*types.OkResp, error) {
    c, _ := middleware.ClaimsFromContext(ctx)
    if c == nil || c.ShopId <= 0 {
        return nil, errors.New("not in a shop")
    }
    if !hasPerm(c, "staff.write") {
        return nil, errors.New("permission denied: staff.write")
    }
    if _, err := svcCtx.ShopRpc.DisableStaff(ctx, &shopservice.DisableStaffReq{
        ShopId: c.ShopId, StaffId: staffId,
    }); err != nil {
        return nil, err
    }
    return &types.OkResp{Ok: true}, nil
}

// hasPerm: owner 的 "*" 直接放行
func hasPerm(c *middleware.Claims, perm string) bool {
    for _, p := range c.Perms {
        if p == "*" || p == perm {
            return true
        }
    }
    return false
}
```

- [ ] **Step 7.3: merchant_staff_handlers.go**

```go
package handler

import (
    "net/http"
    "strconv"

    "github.com/zeromicro/go-zero/rest/httpx"
    "github.com/zeromicro/go-zero/rest/pathvar"

    "yw-mall-admin/internal/logic"
    "yw-mall-admin/internal/svc"
    "yw-mall-admin/internal/types"
)

func listMerchantStaffHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        resp, err := logic.ListMerchantStaff(r.Context(), svcCtx)
        if err != nil { writeErr(r, w, err); return }
        writeOk(r, w, resp)
    }
}

func updateMerchantStaffRoleHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        idStr := pathvar.Vars(r)["id"]
        id, _ := strconv.ParseInt(idStr, 10, 64)
        var req types.UpdateStaffRoleReq
        if err := httpx.Parse(r, &req); err != nil { writeErr(r, w, err); return }
        resp, err := logic.UpdateMerchantStaffRole(r.Context(), svcCtx, id, &req)
        if err != nil { writeErr(r, w, err); return }
        writeOk(r, w, resp)
    }
}

func disableMerchantStaffHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        idStr := pathvar.Vars(r)["id"]
        id, _ := strconv.ParseInt(idStr, 10, 64)
        resp, err := logic.DisableMerchantStaff(r.Context(), svcCtx, id)
        if err != nil { writeErr(r, w, err); return }
        writeOk(r, w, resp)
    }
}
```

- [ ] **Step 7.4: routes.go 加 3 个路由**

找到 `/merchant/v1` protected group，在内追加：
```go
{Method: http.MethodGet, Path: "/staff", Handler: listMerchantStaffHandler(svcCtx)},
{Method: http.MethodPost, Path: "/staff/:id/role", Handler: updateMerchantStaffRoleHandler(svcCtx)},
{Method: http.MethodPost, Path: "/staff/:id/disable", Handler: disableMerchantStaffHandler(svcCtx)},
```

- [ ] **Step 7.5: build + 重启 admin-api**

```bash
cd ~/workspace/go/mall/yw-mall/yw-mall-admin && go build ./...
cd ~/workspace/go/mall/yw-mall && ./start.sh restart 2>&1 | grep admin-api
```

- [ ] **Step 7.6: commit**

```
feat(m1 admin-api): /merchant/v1/staff CRUD 路由

GET    /staff           列出本店员工
POST   /staff/:id/role  改员工角色 (需 staff.write)
POST   /staff/:id/disable 冻结员工 (需 staff.write)

权限检查在 logic 层用 hasPerm helper, owner 的 "*" 直接通过.
```

---

## Task 8: admin-api invitation HTTP 路由

**Files**:
- Modify: `yw-mall-admin/internal/types/types.go`
- Create: `yw-mall-admin/internal/logic/merchant_invitation_logic.go`
- Create: `yw-mall-admin/internal/handler/merchant_invitation_handlers.go`
- Modify: `yw-mall-admin/internal/handler/routes.go`

- [ ] **Step 8.1: types**

```go
type CreateInvitationReq struct {
    TargetPhone string `json:"targetPhone,optional"`
    TargetEmail string `json:"targetEmail,optional"`
    Role        string `json:"role"`
}
type CreateInvitationResp struct {
    InvitationCode string `json:"invitationCode"`
    ExpiresAt      int64  `json:"expiresAt"`
}

type InvitationItemDTO struct {
    Id          int64  `json:"id"`
    TargetPhone string `json:"targetPhone"`
    TargetEmail string `json:"targetEmail"`
    Role        string `json:"role"`
    Status      int32  `json:"status"`
    ExpiresAt   int64  `json:"expiresAt"`
    CreateTime  int64  `json:"createTime"`
}
type ListInvitationsResp struct{ Items []InvitationItemDTO `json:"items"` }

type AcceptInvitationReq struct {
    InvitationCode string `json:"invitationCode"`
}
type AcceptInvitationResp struct {
    ShopId   int64  `json:"shopId"`
    Role     string `json:"role"`
    ShopName string `json:"shopName"`
}
```

- [ ] **Step 8.2: logic + handler**

```go
// merchant_invitation_logic.go
package logic

import (
    "context"
    "errors"

    "yw-mall-admin/internal/middleware"
    "yw-mall-admin/internal/svc"
    "yw-mall-admin/internal/types"

    "mall-shop-rpc/shopservice"
)

func CreateInvitation(ctx context.Context, svcCtx *svc.ServiceContext, req *types.CreateInvitationReq) (*types.CreateInvitationResp, error) {
    c, _ := middleware.ClaimsFromContext(ctx)
    if c == nil || c.ShopId <= 0 {
        return nil, errors.New("not in a shop")
    }
    if !hasPerm(c, "staff.write") {
        return nil, errors.New("permission denied: staff.write")
    }
    res, err := svcCtx.ShopRpc.CreateInvitation(ctx, &shopservice.CreateInvitationReq{
        ShopId: c.ShopId, InvitedByUid: c.Uid,
        TargetPhone: req.TargetPhone, TargetEmail: req.TargetEmail,
        Role: req.Role,
    })
    if err != nil {
        return nil, err
    }
    return &types.CreateInvitationResp{InvitationCode: res.InvitationCode, ExpiresAt: res.ExpiresAt}, nil
}

func ListInvitations(ctx context.Context, svcCtx *svc.ServiceContext) (*types.ListInvitationsResp, error) {
    c, _ := middleware.ClaimsFromContext(ctx)
    if c == nil || c.ShopId <= 0 {
        return nil, errors.New("not in a shop")
    }
    res, err := svcCtx.ShopRpc.ListPendingInvitations(ctx, &shopservice.ListPendingInvitationsReq{ShopId: c.ShopId})
    if err != nil {
        return nil, err
    }
    out := make([]types.InvitationItemDTO, 0, len(res.Items))
    for _, it := range res.Items {
        out = append(out, types.InvitationItemDTO{
            Id: it.Id, TargetPhone: it.TargetPhone, TargetEmail: it.TargetEmail,
            Role: it.Role, Status: it.Status, ExpiresAt: it.ExpiresAt, CreateTime: it.CreateTime,
        })
    }
    return &types.ListInvitationsResp{Items: out}, nil
}

func RevokeInvitation(ctx context.Context, svcCtx *svc.ServiceContext, id int64) (*types.OkResp, error) {
    c, _ := middleware.ClaimsFromContext(ctx)
    if c == nil || c.ShopId <= 0 {
        return nil, errors.New("not in a shop")
    }
    if !hasPerm(c, "staff.write") {
        return nil, errors.New("permission denied: staff.write")
    }
    if _, err := svcCtx.ShopRpc.RevokeInvitation(ctx, &shopservice.RevokeInvitationReq{
        ShopId: c.ShopId, InvitationId: id,
    }); err != nil {
        return nil, err
    }
    return &types.OkResp{Ok: true}, nil
}

// AcceptInvitation 接受邀请——调用方是任意 merchant role session
// （新员工接受邀请前需要先登录成 merchant；但 chicken-and-egg：他还没绑店！
// 解决：这个接口走 c-side session 校验，需要 acceptor 是 c-user role）
// 简化版本：本接口放在 protected merchant 内 → 用户先用 c 端账号登录 ←
// 实际需要走另一条路径。下面给的版本是"已有店主 reuse"场景。
// 真实场景见 Step 8.4 注。
func AcceptInvitation(ctx context.Context, svcCtx *svc.ServiceContext, req *types.AcceptInvitationReq) (*types.AcceptInvitationResp, error) {
    c, _ := middleware.ClaimsFromContext(ctx)
    if c == nil {
        return nil, errors.New("login required")
    }
    // 查 user 的 phone/email 用于防代领校验（本期简化：直接从 user-rpc 查）
    u, err := svcCtx.UserRpc.GetUserById(ctx, &userclient.GetUserByIdReq{Id: c.Uid})
    if err != nil {
        return nil, err
    }
    res, err := svcCtx.ShopRpc.AcceptInvitation(ctx, &shopservice.AcceptInvitationReq{
        InvitationCode: req.InvitationCode,
        AcceptorUid:    c.Uid,
        AcceptorPhone:  u.Phone,
        AcceptorEmail:  u.Email,
    })
    if err != nil {
        return nil, err
    }
    return &types.AcceptInvitationResp{ShopId: res.ShopId, Role: res.Role, ShopName: res.ShopName}, nil
}
```

- [ ] **Step 8.3: handlers + 路由**

```go
// merchant_invitation_handlers.go
func createInvitationHandler(svcCtx *svc.ServiceContext) http.HandlerFunc { /* 标准 parse-call-write */ }
func listInvitationsHandler(svcCtx *svc.ServiceContext) http.HandlerFunc { /* ... */ }
func revokeInvitationHandler(svcCtx *svc.ServiceContext) http.HandlerFunc { /* ... */ }
func acceptInvitationHandler(svcCtx *svc.ServiceContext) http.HandlerFunc { /* ... */ }
```

routes.go protected 段内：
```go
{Method: http.MethodPost, Path: "/staff/invitations", Handler: createInvitationHandler(svcCtx)},
{Method: http.MethodGet,  Path: "/staff/invitations", Handler: listInvitationsHandler(svcCtx)},
{Method: http.MethodPost, Path: "/staff/invitations/:id/revoke", Handler: revokeInvitationHandler(svcCtx)},
{Method: http.MethodPost, Path: "/invitations/accept", Handler: acceptInvitationHandler(svcCtx)},
```

- [ ] **Step 8.4: 注意点**

`/invitations/accept` 当前放在 protected merchant group 下——意味着调用者必须已经是一个 merchant role 的用户。这对"已是某店店主，被邀请加入另一店"的场景可行。

**对"新员工，c-user，未绑店"**：他登录 c 端拿到 c-user token → c 端 SPA 跳到 merchant 工作台时调 `/invitations/accept`：
- 但他的 token role=user，admin-api 中间件 `requiredRole="merchant"` → 403

**解决（本 M1）**：把 `/invitations/accept` **挪到 public sub-routes**（无中间件），改成接收 `c-user` 的 access_token 而不是依赖 session middleware：
```go
// public 路由
{Method: http.MethodPost, Path: "/invitations/accept", Handler: acceptInvitationHandler(svcCtx)},
```
handler 自己调 user-rpc.ValidateSession 校验 token role=user，再 forward。

下一版 logic 改为：
```go
func AcceptInvitation(ctx context.Context, svcCtx *svc.ServiceContext, req *types.AcceptInvitationReq, accessToken string) (...) {
    // 直接调 user-rpc.ValidateSession 拿 uid (不走 middleware)
    sess, err := svcCtx.UserRpc.ValidateSession(ctx, &userclient.ValidateSessionReq{AccessToken: accessToken})
    if err != nil || sess == nil || sess.Uid <= 0 { return nil, errors.New("login required") }
    if sess.Role != "user" && sess.Role != "merchant" { return nil, errors.New("forbidden") }
    // ...rest
}
```

- [ ] **Step 8.5: build + restart**

```bash
cd ~/workspace/go/mall/yw-mall/yw-mall-admin && go build ./...
cd ~/workspace/go/mall/yw-mall && ./start.sh restart 2>&1 | grep admin-api
```

- [ ] **Step 8.6: commit**

```
feat(m1 admin-api): /merchant/v1/staff/invitations 路由 + /invitations/accept

POST /staff/invitations         发邀请 (需 staff.write)
GET  /staff/invitations         待处理邀请列表
POST /staff/invitations/:id/revoke 撤销
POST /invitations/accept        接受邀请（public，自校验 c-user token）
```

---

## Task 9: 后端 e2e smoke

- [ ] **Step 9.1: 完整 happy path**

```bash
# 1. 店主登录
T_ALICE=$(curl -s -X POST http://localhost:18999/merchant/v1/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"alice","password":"Reset!2026X"}' | jq -r .accessToken)
echo "T_ALICE=${T_ALICE:0:20}..."

# 2. 列员工（应只有 alice 一人 owner）
curl -s -H "Authorization: Bearer $T_ALICE" http://localhost:18999/merchant/v1/staff | jq

# 3. 发邀请给 bob (假设 bob 是已注册 c-user)
# 先 register bob
SUFFIX=$(date +%s | tail -c 6); BOB="bob$SUFFIX"; BOB_PHONE="138000$(date +%s | tail -c 5)"
# ... 注册 bob (复用前面 reg-v2 流程) ...
# 然后用 alice 发邀请：
curl -s -X POST -H "Authorization: Bearer $T_ALICE" -H 'Content-Type: application/json' \
  -d "{\"targetPhone\":\"$BOB_PHONE\",\"role\":\"warehouse\"}" \
  http://localhost:18999/merchant/v1/staff/invitations | jq

# 4. 从 logs/shop-rpc.log 拿到 mock invite code
grep "mock-invite" /home/carter/workspace/go/mall/yw-mall/logs/shop-rpc.log | tail -1

# 5. bob 登录拿 c-user token
T_BOB=$(curl -s -X POST http://localhost:18888/api/auth/login \
  -H 'Content-Type: application/json' \
  -d "{\"account\":\"$BOB\",\"password\":\"...\"}" | jq -r .accessToken)

# 6. bob 调 accept
curl -s -X POST -H "Authorization: Bearer $T_BOB" -H 'Content-Type: application/json' \
  -d '{"invitationCode":"<上面 code>"}' \
  http://localhost:18999/merchant/v1/invitations/accept | jq
# 预期：{"shopId":1,"role":"warehouse","shopName":"..."}

# 7. bob 用 merchant login 重新登
T_BOB_M=$(curl -s -X POST http://localhost:18999/merchant/v1/login \
  -H 'Content-Type: application/json' \
  -d "{\"username\":\"$BOB\",\"password\":\"...\"}" | jq -r .accessToken)

# 8. bob 调 staff 列表（应能看到自己 + alice）
curl -s -H "Authorization: Bearer $T_BOB_M" http://localhost:18999/merchant/v1/staff | jq

# 9. 跨权限测试：bob (warehouse) 尝试改员工角色 → 应 403
curl -s -X POST -H "Authorization: Bearer $T_BOB_M" -H 'Content-Type: application/json' \
  -d '{"newRole":"finance"}' http://localhost:18999/merchant/v1/staff/1/role | jq
# 预期：permission denied: staff.write
```

- [ ] **Step 9.2: 验证结果记到 daily**

写入 `daily/2026-MM-DD.md` 收尾时复用此清单。

- [ ] **Step 9.3: commit（如有 e2e 修复）**

```
fix(m1): e2e 排查中暴露的 xxx 问题
```

---

## Task 10: FE 脚手架（复制 admin/ → merchant/）

**Files**:
- Create: `yw-mall-admin-fe/merchant/*` (整个 SPA 目录)

- [ ] **Step 10.1: 复制 admin/ 整体到 merchant/**

```bash
cd ~/workspace/go/mall/yw-mall-admin-fe
rm -rf merchant/README.md
cp -r admin/ merchant/
cd merchant/
```

- [ ] **Step 10.2: 改 package.json**

```json
{
  "name": "yw-mall-merchant-fe",
  "version": "0.1.0",
  ...
}
```

- [ ] **Step 10.3: 改 vite.config.ts dev proxy**

```ts
server: {
  port: 5174,                 // 区别于 admin 5173
  proxy: {
    '/merchant': { target: 'http://localhost:18999', changeOrigin: true }
  }
}
```

- [ ] **Step 10.4: 清理 admin 专用页面，留壳**

```bash
# 保留 layouts/login 框架；删 admin 业务页
rm -rf src/views/{accounts,shop-applications,users,reviews,complaints,products,op-log}
# 留 src/views/login, src/views/dashboard 框架
```

- [ ] **Step 10.5: 改 axios baseURL**

`src/api/request.ts`：
```ts
const service = axios.create({ baseURL: '/merchant/v1', ... })
```

- [ ] **Step 10.6: build 试一遍**

```bash
pnpm install
pnpm run build
```

- [ ] **Step 10.7: 加 Dockerfile + nginx 同 admin 模板**

```bash
cp admin/Dockerfile merchant/Dockerfile
cp admin/nginx.conf merchant/nginx.conf
cp admin/.dockerignore merchant/.dockerignore
# nginx.conf 把 /admin/ proxy 改成 /merchant/
sed -i 's|/admin/|/merchant/|g' merchant/nginx.conf
```

- [ ] **Step 10.8: yw-mall-deploy compose 加 mall-merchant-fe (端口 18083)**

```yaml
mall-merchant-fe:
  build:
    context: ../yw-mall-admin-fe/merchant
    dockerfile: Dockerfile
  ports: ["18083:80"]
  networks: [env_infra]
  depends_on: [mall-admin-api]
```

- [ ] **Step 10.9: commit**

```
feat(m1 fe): merchant SPA 脚手架克隆 admin/

复制 yw-mall-admin-fe/admin -> merchant；调 baseURL /merchant/v1；
dev port 5174；Dockerfile + nginx (proxy /merchant/ → apisix:9080)
+ compose 加 mall-merchant-fe 服务 18083:80。
业务页清空，仅保留 login + dashboard + layout 框架。
```

---

## Task 11: FE 登录页 + session store + router

**Files**:
- Modify: `yw-mall-admin-fe/merchant/src/api/auth.ts`
- Modify: `yw-mall-admin-fe/merchant/src/stores/session.ts`
- Modify: `yw-mall-admin-fe/merchant/src/router/index.ts`
- Modify: `yw-mall-admin-fe/merchant/src/views/login/index.vue`

- [ ] **Step 11.1: api/auth.ts**

```ts
import request from './request'
export const merchantLogin = (username: string, password: string) =>
  request.post('/login', { username, password })
export const merchantLogout = () => request.post('/logout')
export const merchantRefresh = (refreshToken: string) =>
  request.post('/refresh', { refreshToken })
```

- [ ] **Step 11.2: stores/session.ts**

```ts
import { defineStore } from 'pinia'
export const useSession = defineStore('session', {
  state: () => ({
    accessToken: localStorage.getItem('m_at') || '',
    refreshToken: localStorage.getItem('m_rt') || '',
    csrfToken: localStorage.getItem('m_csrf') || '',
    shopId: Number(localStorage.getItem('m_shop_id') || 0),
    staffRole: localStorage.getItem('m_role') || '',
    perms: JSON.parse(localStorage.getItem('m_perms') || '[]') as string[],
    username: localStorage.getItem('m_user') || ''
  }),
  actions: {
    setSession(d: any) {
      this.accessToken = d.accessToken || d.token
      this.refreshToken = d.refreshToken
      this.csrfToken = d.csrfToken
      this.shopId = d.shopId
      this.staffRole = d.staffRole
      this.perms = d.perms || []
      this.username = d.username
      localStorage.setItem('m_at', this.accessToken)
      localStorage.setItem('m_rt', this.refreshToken)
      localStorage.setItem('m_csrf', this.csrfToken)
      localStorage.setItem('m_shop_id', String(this.shopId))
      localStorage.setItem('m_role', this.staffRole)
      localStorage.setItem('m_perms', JSON.stringify(this.perms))
      localStorage.setItem('m_user', this.username)
    },
    clear() {
      this.$reset()
      ['m_at','m_rt','m_csrf','m_shop_id','m_role','m_perms','m_user'].forEach(k => localStorage.removeItem(k))
    },
    can(perm: string) {
      return this.perms.includes('*') || this.perms.includes(perm)
    }
  }
})
```

- [ ] **Step 11.3: router/index.ts 权限守卫**

```ts
router.beforeEach((to, from, next) => {
  const sess = useSession()
  if (to.meta.public) return next()
  if (!sess.accessToken) return next({ path: '/login', query: { redirect: to.fullPath } })
  const required = to.meta.perms as string[] | undefined
  if (required && !required.some(p => sess.can(p))) {
    return next({ path: '/dashboard' })  // 没权限的路由直接打回 dashboard
  }
  next()
})
```

- [ ] **Step 11.4: views/login/index.vue 京东风（复刻 admin 同款）**

直接 cp `admin/src/views/login/index.vue` → `merchant/src/views/login/index.vue`，改：
- 品牌区文案：`yw-mall` → `yw-mall 商家工作台`
- 登录后调用：`adminLogin` → `merchantLogin`
- 跳转目标：`/dashboard` 保持
- 删 MFA 阶段（商家本期不强制 MFA）

- [ ] **Step 11.5: 本地 dev 验证**

```bash
cd ~/workspace/go/mall/yw-mall-admin-fe/merchant && pnpm dev
# 浏览器开 http://localhost:5174/login 用 alice/Reset!2026X 登录
# 预期：跳到 /dashboard 占位页，看到 username + shopId
```

- [ ] **Step 11.6: commit**

```
feat(m1 fe): 登录页 + session store + 权限路由守卫

- api/auth.ts: merchantLogin/refresh/logout
- stores/session.ts: token+csrf+shopId+staffRole+perms 全持久化
  localStorage，提供 can(perm) helper
- router: beforeEach 检查 meta.perms ∩ session.perms 非空才放行
- views/login/index.vue: 京东风复刻 admin，文案改"商家工作台"
```

---

## Task 12: FE 接受邀请页

**Files**:
- Create: `yw-mall-admin-fe/merchant/src/views/invite/accept.vue`
- Modify: `yw-mall-admin-fe/merchant/src/router/index.ts`
- Modify: `yw-mall-admin-fe/merchant/src/api/staff.ts`

- [ ] **Step 12.1: api/staff.ts 加 acceptInvitation**

```ts
export const acceptInvitation = (invitationCode: string) =>
  request.post('/invitations/accept', { invitationCode })
```

- [ ] **Step 12.2: views/invite/accept.vue**

```vue
<template>
  <div class="invite-page">
    <el-card class="invite-card">
      <h2>接受店铺邀请</h2>
      <p v-if="loading">处理中...</p>
      <div v-else-if="success">
        <el-result icon="success" :title="`已加入「${shopName}」`" :sub-title="`角色：${role}`">
          <template #extra>
            <el-button type="primary" @click="$router.replace('/login')">前往登录</el-button>
          </template>
        </el-result>
      </div>
      <div v-else>
        <el-alert :title="errMsg" type="error" />
        <el-button class="mt-4" @click="$router.replace('/login')">返回登录</el-button>
      </div>
    </el-card>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { useRoute } from 'vue-router'
import { acceptInvitation } from '@/api/staff'

const route = useRoute()
const loading = ref(true)
const success = ref(false)
const errMsg = ref('')
const shopName = ref('')
const role = ref('')

onMounted(async () => {
  const code = route.query.code as string
  if (!code) { errMsg.value = '邀请链接缺少 code 参数'; loading.value = false; return }
  try {
    const r = await acceptInvitation(code)
    shopName.value = r.shopName
    role.value = r.role
    success.value = true
  } catch (e: any) {
    errMsg.value = e?.message || '邀请已失效'
  } finally {
    loading.value = false
  }
})
</script>
```

- [ ] **Step 12.3: 路由注册**

```ts
{ path: '/accept-invite', component: () => import('@/views/invite/accept.vue'),
  meta: { public: true } }
```

- [ ] **Step 12.4: 注意点**

`/invitations/accept` 后端要求 c-user token。FE 在这个页面调用前要先有 c-user token——本期最小实现假设用户**已在 c 端登录** → 自动用 localStorage 里 c 端 token（如果有）发请求；否则提示先去 c 端登录然后回来。

简化路径：
- accept.vue 在 onMounted 检查 localStorage 是否有 c-user 的 token（约定 key 名 `c_at`）
- 如果有 → 用它调 accept
- 如果无 → 跳 `http://localhost:5173/login?redirect=/accept-invite?code=xxx`（c 端 login 完跳回）

**P1 改进**：用 SSO 子域统一 token，本期最小化先简化。

- [ ] **Step 12.5: commit**

```
feat(m1 fe): 接受邀请页 /accept-invite

读 URL query.code 调 POST /invitations/accept。成功展示 shopName + role
并提示去登录；失败展示错误信息。public 路由不需 token。

P1 改进：复用 c 端 token 自动调（本期假设 localStorage 里有 c_at）
```

---

## Task 13: FE 员工列表 + 邀请 modal + 改角色 + 冻结

**Files**:
- Modify: `yw-mall-admin-fe/merchant/src/api/staff.ts`
- Create: `yw-mall-admin-fe/merchant/src/views/staff/list.vue`
- Modify: `yw-mall-admin-fe/merchant/src/router/index.ts`

- [ ] **Step 13.1: api/staff.ts 完整**

```ts
import request from './request'
export const listStaff = () => request.get('/staff')
export const updateStaffRole = (id: number, newRole: string) =>
  request.post(`/staff/${id}/role`, { newRole })
export const disableStaff = (id: number) => request.post(`/staff/${id}/disable`)

export const listInvitations = () => request.get('/staff/invitations')
export const createInvitation = (payload: {targetPhone?: string; targetEmail?: string; role: string}) =>
  request.post('/staff/invitations', payload)
export const revokeInvitation = (id: number) =>
  request.post(`/staff/invitations/${id}/revoke`)

export const acceptInvitation = (invitationCode: string) =>
  request.post('/invitations/accept', { invitationCode })
```

- [ ] **Step 13.2: views/staff/list.vue**

参考 admin 同款 table+modal 风格，3 块：
1. 顶部：「邀请新成员」按钮（owner 可见）
2. 中部：员工 table（id/username/role/joinedAt/操作）
   - 操作：改角色 select（owner 之外可改）、冻结按钮
3. 底部：待处理邀请 table（targetPhone/role/expiresAt/操作）
   - 操作：撤销按钮

省略详细代码（标准 Element Plus el-table + el-dialog + el-form 用法）。

- [ ] **Step 13.3: 路由 + 导航**

```ts
{ path: '/staff', component: () => import('@/views/staff/list.vue'),
  meta: { perms: ['staff.read'] } }
```

`MerchantLayout.vue` 左侧 nav 加「员工管理」入口，按 `session.can('staff.read')` 条件渲染。

- [ ] **Step 13.4: 本地验证**

dev server 跑通：店主登录 → 看到员工管理 → 邀请 → 撤销 → 改角色 → 冻结 全流程。

- [ ] **Step 13.5: commit**

```
feat(m1 fe): 员工管理页 /staff

3 块布局: 邀请新成员按钮 (owner) + 员工 table (改角色/冻结) +
待处理邀请 table (撤销). 全部按 perms 条件渲染.
```

---

## Task 14: FE 占位 dashboard + layout

**Files**:
- Modify: `yw-mall-admin-fe/merchant/src/layouts/MerchantLayout.vue`
- Modify: `yw-mall-admin-fe/merchant/src/views/dashboard/index.vue`
- Modify: `yw-mall-admin-fe/merchant/src/router/index.ts`

- [ ] **Step 14.1: MerchantLayout.vue**

左侧 nav 菜单：
- 仪表盘（所有人）
- 商品管理（perms: product.read）— M2 接
- 订单管理（perms: order.read）— M3 接
- 退款处理（perms: refund.read）— M4 接
- 财务（perms: finance.read）— M5 接
- 装修（perms: decoration.write，owner）— M6 接
- 物流模板（perms: freight.read）— M3 接
- 员工管理（perms: staff.read）— M1 已接

```vue
<el-menu :collapse="collapsed">
  <el-menu-item index="/dashboard">
    <el-icon><Odometer/></el-icon>
    <span>仪表盘</span>
  </el-menu-item>
  <el-menu-item v-if="sess.can('staff.read')" index="/staff">
    <el-icon><User/></el-icon>
    <span>员工管理</span>
  </el-menu-item>
  <!-- 其他菜单项暂禁用，待 M2-M6 -->
  <el-menu-item index="/products" disabled><el-icon><Goods/></el-icon><span>商品 (M2)</span></el-menu-item>
  <el-menu-item index="/orders" disabled><el-icon><List/></el-icon><span>订单 (M3)</span></el-menu-item>
</el-menu>
```

顶部右侧：用户名 + dropdown「登出」。

- [ ] **Step 14.2: dashboard/index.vue 占位**

4 个 el-card 显示 N/A（M6 接 dashboard 聚合 RPC 后填真实数据）。

```vue
<template>
  <div class="dashboard">
    <h2>欢迎，{{ sess.username }}</h2>
    <p class="muted">店铺 ID: {{ sess.shopId }} · 角色: {{ sess.staffRole }}</p>
    <el-row :gutter="24">
      <el-col :span="6"><el-card><div class="metric-num">--</div><div>今日营业额 (M6)</div></el-card></el-col>
      <el-col :span="6"><el-card><div class="metric-num">--</div><div>今日订单 (M6)</div></el-card></el-col>
      <el-col :span="6"><el-card><div class="metric-num">--</div><div>待发货 (M6)</div></el-card></el-col>
      <el-col :span="6"><el-card><div class="metric-num">--</div><div>钱包余额 (M6)</div></el-card></el-col>
    </el-row>
  </div>
</template>
```

- [ ] **Step 14.3: commit**

```
feat(m1 fe): MerchantLayout + 占位 dashboard

左侧 nav 按 perms 条件渲染；其他模块 (M2-M6) 灰显占位；
dashboard 4 卡片显示 N/A 等 M6 接聚合 RPC.
```

---

## Task 15: E2E smoke + daily 日志收尾

- [ ] **Step 15.1: 启动栈 + 完整路径走通**

```bash
cd ~/workspace/go/mall/yw-mall && ./start.sh start
cd ~/workspace/go/mall/yw-mall-admin-fe/merchant && pnpm dev
```

浏览器跑：
1. alice 登录 http://localhost:5174 → 进 dashboard
2. 进员工管理 → 邀请 bob 手机号 → 注意 logs 拿 invite code
3. （后端 e2e）bob 登 c 端 → accept invite
4. bob 登录 merchant → 看自己店 + nav 没"装修"等 owner-only 项
5. alice 改 bob role 为 finance → bob 重新登录 → nav 变化
6. alice 冻结 bob → bob 重新登录 → "用户未绑店"

- [ ] **Step 15.2: 写 daily/2026-MM-DD.md**

按 daily 模板：
- 📌 当前状态
- 🔧 主要交付
- 🐛 排错记录
- ✅ 验证清单（含 e2e 实测证据）
- 📝 改动文件
- 🚧 已知遗留 / 下一步（M2）

- [ ] **Step 15.3: 回写 plan 跟踪**

更新本 plan 文件，把所有 Task 的 `- [ ]` 改 `- [x]`。

- [ ] **Step 15.4: commit + push**

```
docs(daily): M1 商家工作台登录+员工 RBAC 完成

店主 + 员工子账号双路径登录跑通；邀请链接全流程 e2e 验证。
[详细见 daily/2026-MM-DD.md]
```

---

## 验收清单

**全部 Task done 即视为 M1 交付**：
- [ ] alice (店主) 在 http://localhost:5174 用密码登录进 dashboard
- [ ] alice 邀请 bob 手机号 role=warehouse → bob 收 mock SMS (logs)
- [ ] bob 点链接调 /invitations/accept → 加入 alice 的店铺
- [ ] bob 在 merchant SPA 登录 → 看到 alice 的店铺；nav 没"装修"项
- [ ] alice 把 bob role 改 finance → bob 重新登录 → nav 变化
- [ ] alice 冻结 bob → bob 重新登录失败 ("user is not a staff")
- [ ] 跨权限：bob (warehouse) 调 POST /staff/1/role → 403 permission denied
- [ ] L1.6 联动：alice 改密 → 所有 merchant session 失效但同 uid c-user session 保留

---

## 关联文档

- `plan/2026-05-27-merchant-workstation-design.md` — 整体设计
- `feat/login-revamp.md` L1.6 — 改密强制下线（merchant role 复用）
- `feat/2026-05-21-multi-identifier-register-login-plan.md` — sendVerifyCode mock 模式
- `daily/2026-05-14.md` — admin RBAC SessionAuthMiddleware 实现参考
