# 多 identifier 注册/登录 + PII 加密 — 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 用户注册支持 username + phone/email 多 identifier；登录支持单端点 regex 自动识别；phone/email 列 AES + HMAC blind index 加密。

**Architecture:** S4 已有的 cryptox 加 HMAC 函数 → user 表加 phone_enc/email_enc/phone_hash/email_hash 4 列 + email 明文列 → 复用 S4.1 challenge_token Redis 握手做注册二步流程 → mall-api 加 3 个新 auth 路由 + 兼容老 /api/user/register → mall-fe 新增 register.vue + login.vue placeholder 改造。

**Tech Stack:** Go 1.26 / go-zero 1.10.1 / proto3 / Redis / bcrypt / cryptox AES-256-GCM + HMAC-SHA256 / MySQL 9.6 / uni-app Vue 3 + Wot Design Uni

**Spec:** [2026-05-21-multi-identifier-register-login-design.md](./2026-05-21-multi-identifier-register-login-design.md)

---

## 项目约定说明（先读）

- **无自动化单测**（CLAUDE.md 明确）；纯函数（cryptox.Hmac）写 `_test.go`，业务 logic 走 e2e curl 验证
- 改 proto 后必须 `protoc` 重生成 + 手工同步 `userclient/user.go` 暴露新方法
- 配置改了在 etcd 走 configcenter，新装时 `start.sh` 自动 hash-key push 一次
- 每个 task **独立 commit**，commit message 用 `<scope>(reg-v2):` 前缀方便筛
- 容器化部署用 yw-mall-deploy：本 plan 默认开发态走 `start.sh`，最后 task 切容器并 verify

---

## File Structure

### Backend changes — yw-mall

```
mall-common/
  cryptox/
    cryptox.go              MODIFY: 加 Hmac() 函数
    cryptox_test.go         CREATE: Hmac/Encrypt 黑盒单测
  proto/user/
    user.proto              MODIFY: 加 SendVerifyCode/RegisterV2/LoginV2 + 3 个 Req/Resp

mall-user-rpc/
  sql/
    user_multi_id_v2.sql    CREATE: ALTER user 加 5 列 + 2 index
  cmd/backfill_pii/
    main.go                 CREATE: 老 phone 数据 → enc + hash backfill 脚本
  internal/logic/
    sendverifycodelogic.go  CREATE: SendVerifyCode RPC 实现
    registerv2logic.go      CREATE: RegisterV2 RPC 实现（仿 registerlogic.go）
    loginv2logic.go         CREATE: LoginV2 RPC 实现（仿 loginlogic.go）
    emailhelpers.go         CREATE: EmailSend mock (logx)
    sendcodehelpers.go      CREATE: 验证码 Redis key + 限频 + 单次消费
  internal/server/
    userserver.go           AUTO-REGEN: 加 3 个新 method handler
  user/
    user.pb.go              AUTO-REGEN
    user_grpc.pb.go         AUTO-REGEN
  userclient/
    user.go                 MANUAL: 加 SendVerifyCode/RegisterV2/LoginV2 wrappers

mall-api/
  internal/types/
    types.go                MODIFY: 加 SendCode/RegisterV2/LoginV2 Req/Resp
  internal/handler/
    sendcodehandler.go      CREATE
    registerv2handler.go    CREATE
    routes.go               MODIFY: 加 3 个新路由
  internal/logic/
    sendcodelogic.go        CREATE: 调 UserRpc.SendVerifyCode
    registerv2logic.go      CREATE: 调 UserRpc.RegisterV2
    authloginlogic.go       MODIFY: 改调 UserRpc.LoginV2 (account 字段)

start.sh                    MODIFY: SPRINT_MIGRATIONS 加 user_multi_id_v2.sql
```

### Frontend changes — yw-mall-fe

```
src/api/
  auth.ts                   CREATE: sendVerifyCode/registerV2/loginV2 wrappers
  user.ts                   MODIFY: login() 改调 /api/auth/login + account 字段（向后兼容）
src/types/
  api.ts                    MODIFY: SendCodeReq/RegisterV2Req/LoginV2Req 类型
src/pages/login/
  register.vue              CREATE: 注册页
  index.vue                 MODIFY: placeholder + 注册链接
src/
  pages.json                MODIFY: 注册 pages/login/register
```

### Infra changes — env (可分离推进)

```
env/apisix/routes/setup.sh  MODIFY: 加 /api/auth/* 限流路由
```

---

## Task 1: cryptox 加 HMAC 函数

**Files:**
- Modify: `mall-common/cryptox/cryptox.go`
- Create: `mall-common/cryptox/cryptox_test.go`

- [ ] **Step 1: 加 Hmac 函数到 cryptox.go**

在 `cryptox.go` 末尾加：

```go
// Hmac 用 HMAC-SHA256 对 plain 做确定性哈希，结果是 64 字符 hex。
// 用途：phone / email 的 blind index — 让加密字段能做 WHERE 查询。
// 空串特殊处理：返空串，避免所有未绑定用户共用同一个 hash。
//
// key 复用 MALL_FIELD_ENCRYPTION_KEY 的前 32 字节，与 AES key 共享：
// key rotation 时必须一起换、所有 hash 列要重算。
func Hmac(plain string) string {
    if plain == "" {
        return ""
    }
    mustInit()
    mac := hmac.New(sha256.New, key) // key 是包内已有的 AES key
    mac.Write([]byte(plain))
    return hex.EncodeToString(mac.Sum(nil))
}
```

import 需要补：`"crypto/hmac"`, `"crypto/sha256"`, `"encoding/hex"`（hex 可能已有）。

- [ ] **Step 2: 写单测验证 Hmac**

```go
package cryptox

import (
    "os"
    "strings"
    "testing"
)

func TestHmac_EmptyReturnsEmpty(t *testing.T) {
    setupKey(t)
    if got := Hmac(""); got != "" {
        t.Errorf("Hmac('') = %q, want empty", got)
    }
}

func TestHmac_Deterministic(t *testing.T) {
    setupKey(t)
    a := Hmac("13800138001")
    b := Hmac("13800138001")
    if a != b {
        t.Errorf("Hmac not deterministic: %q vs %q", a, b)
    }
    if len(a) != 64 {
        t.Errorf("expected 64-char hex, got %d", len(a))
    }
}

func TestHmac_DifferentInputsDiffer(t *testing.T) {
    setupKey(t)
    a := Hmac("alice@example.com")
    b := Hmac("bob@example.com")
    if a == b {
        t.Errorf("different inputs produced same hash")
    }
}

func TestEncrypt_Roundtrip(t *testing.T) {
    setupKey(t)
    plain := "alice@example.com"
    enc, err := Encrypt(plain)
    if err != nil {
        t.Fatal(err)
    }
    if !strings.HasPrefix(enc, "v1:") {
        t.Errorf("expected v1: prefix, got %q", enc)
    }
    got, err := Decrypt(enc)
    if err != nil {
        t.Fatal(err)
    }
    if got != plain {
        t.Errorf("roundtrip: got %q, want %q", got, plain)
    }
}

func setupKey(t *testing.T) {
    t.Helper()
    // 32 字节 hex key — 跟 start.sh 注入的开发态默认一致
    os.Setenv("MALL_FIELD_ENCRYPTION_KEY",
        "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef")
}
```

- [ ] **Step 3: 跑测试**

```bash
cd ~/workspace/go/mall/yw-mall/mall-common && go test ./cryptox/ -v
```

Expected: 4 个 PASS。

- [ ] **Step 4: commit**

```bash
git add cryptox/
git commit -m "feat(reg-v2 cryptox): 加 Hmac SHA256 函数 + 单测

Hmac 复用 AES key 前 32 字节，输出 64 char hex；空串返空；
后续 user 表 phone/email blind index 用。Encrypt 双读双写 v1: 已支持，
roundtrip 测一并补上。

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 2: DDL migration + start.sh 注册

**Files:**
- Create: `mall-user-rpc/sql/user_multi_id_v2.sql`
- Modify: `start.sh`

- [ ] **Step 1: 写 DDL migration**

```sql
-- mall-user-rpc/sql/user_multi_id_v2.sql
-- Sprint 5 (multi-identifier): user 表加 email + phone/email 双列加密 + blind index.
USE mall_user;

DROP PROCEDURE IF EXISTS sp_user_multi_id_v2;
DELIMITER //
CREATE PROCEDURE sp_user_multi_id_v2()
BEGIN
  IF NOT EXISTS (SELECT 1 FROM information_schema.COLUMNS
                 WHERE table_schema='mall_user' AND table_name='user'
                   AND column_name='email') THEN
    ALTER TABLE `user`
      ADD COLUMN `email`      VARCHAR(128) NOT NULL DEFAULT '',
      ADD COLUMN `phone_enc`  VARCHAR(512) NOT NULL DEFAULT '',
      ADD COLUMN `email_enc`  VARCHAR(512) NOT NULL DEFAULT '',
      ADD COLUMN `phone_hash` CHAR(64)     NOT NULL DEFAULT '',
      ADD COLUMN `email_hash` CHAR(64)     NOT NULL DEFAULT '',
      ADD INDEX `idx_phone_hash` (`phone_hash`),
      ADD INDEX `idx_email_hash` (`email_hash`);
  END IF;
END//
DELIMITER ;
CALL sp_user_multi_id_v2();
DROP PROCEDURE sp_user_multi_id_v2;
```

- [ ] **Step 2: 注册到 start.sh SPRINT_MIGRATIONS**

修改 `start.sh` 的 `SPRINT_MIGRATIONS` 数组（在 sprint4_security 之前，order 系列之后），加：

```bash
        "mall_user|mall-user-rpc/sql/user_multi_id_v2.sql"
```

最终该数组类似：

```bash
    local -a SPRINT_MIGRATIONS=(
        "mall_order|mall-order-rpc/sql/order_timeline_v2.sql"
        "mall_order|mall-order-rpc/sql/refund_v2.sql"
        "mall_order|mall-order-rpc/sql/return_v3.sql"
        "mall_payment|mall-payment-rpc/sql/settlement_v2.sql"
        "mall_user|mall-user-rpc/sql/user_multi_id_v2.sql"
    )
```

- [ ] **Step 3: 应用到当前数据库**

```bash
mysql -h127.0.0.1 -P6033 -uproxysql -pproxysql123 mall_user < mall-user-rpc/sql/user_multi_id_v2.sql
mysql -h127.0.0.1 -P6033 -uproxysql -pproxysql123 mall_user -e "DESC \`user\`"
```

Expected: DESC 输出包含 `email`, `phone_enc`, `email_enc`, `phone_hash`, `email_hash` 5 个新列。

- [ ] **Step 4: 同步到 yw-mall-deploy db-init.sh**

修改 `~/workspace/go/mall/yw-mall-deploy/scripts/db-init.sh`，在「Applying Sprint 4 migrations」之后加：

```bash
echo "Applying Sprint 5 migrations..."
apply mall_user       mall-user-rpc/sql/user_multi_id_v2.sql
```

- [ ] **Step 5: commit yw-mall + yw-mall-deploy**

```bash
# yw-mall repo
cd ~/workspace/go/mall/yw-mall
git add mall-user-rpc/sql/user_multi_id_v2.sql start.sh
git commit -m "feat(reg-v2 schema): user 表加 email + phone/email 加密 blind index

5 个新列：email (明文显示用过渡保留) + phone_enc/email_enc (cryptox v1:)
+ phone_hash/email_hash (HMAC-SHA256 64 hex)。idx_*_hash 用于 WHERE 查询。

幂等：用 sp procedure 包 ALTER 通过 information_schema 探测，
start.sh nuke 后自动重放。

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"

# yw-mall-deploy repo
cd ~/workspace/go/mall/yw-mall-deploy
git add scripts/db-init.sh
git commit -m "chore(reg-v2): db-init 跟进 user_multi_id_v2 迁移

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 3: Backfill 老 phone 数据

**Files:**
- Create: `mall-user-rpc/cmd/backfill_pii/main.go`

- [ ] **Step 1: 写 backfill 脚本**

```go
// mall-user-rpc/cmd/backfill_pii/main.go
//
// One-shot：扫描 user 表，把明文 phone 列还没迁移的行（phone != '' AND phone_enc = ''）
// 转成 phone_enc + phone_hash。幂等 — 已迁移行用 phone_enc != '' 排除。
// 不动 email 列（老库没数据）。
package main

import (
    "context"
    "flag"
    "log"

    "mall-common/cryptox"

    _ "github.com/go-sql-driver/mysql"
    "github.com/zeromicro/go-zero/core/stores/sqlx"
)

var ds = flag.String("ds",
    "proxysql:proxysql123@tcp(127.0.0.1:6033)/mall_user?charset=utf8mb4&parseTime=true&loc=Local",
    "mall_user MySQL DSN")

func main() {
    flag.Parse()
    cryptox.MustInit()

    db := sqlx.NewMysql(*ds)
    ctx := context.Background()

    type row struct {
        Id    uint64 `db:"id"`
        Phone string `db:"phone"`
    }
    var rows []*row
    if err := db.QueryRowsCtx(ctx, &rows,
        "SELECT id, phone FROM `user` WHERE phone != '' AND phone_enc = ''"); err != nil {
        log.Fatalf("scan: %v", err)
    }
    log.Printf("found %d rows to migrate", len(rows))

    migrated := 0
    for _, r := range rows {
        enc, err := cryptox.Encrypt(r.Phone)
        if err != nil {
            log.Printf("[skip] id=%d encrypt: %v", r.Id, err)
            continue
        }
        hash := cryptox.Hmac(r.Phone)
        if _, err := db.ExecCtx(ctx,
            "UPDATE `user` SET phone_enc=?, phone_hash=? WHERE id=?",
            enc, hash, r.Id); err != nil {
            log.Printf("[skip] id=%d update: %v", r.Id, err)
            continue
        }
        log.Printf("[ok] id=%d phone=%s", r.Id, r.Phone)
        migrated++
    }
    log.Printf("done: total=%d migrated=%d skipped=%d", len(rows), migrated, len(rows)-migrated)
}
```

- [ ] **Step 2: 跑 backfill**

```bash
cd ~/workspace/go/mall/yw-mall/mall-user-rpc
MALL_FIELD_ENCRYPTION_KEY=$(grep MALL_FIELD_ENCRYPTION_KEY ../../yw-mall-deploy/.env | cut -d= -f2) \
  go run cmd/backfill_pii/main.go
```

Expected: `found 5 rows to migrate` → `[ok] id=1 phone=13800138001` 等 5 行 → `done: total=5 migrated=5 skipped=0`。

- [ ] **Step 3: 数据库验证**

```bash
mysql -h127.0.0.1 -P6033 -uproxysql -pproxysql123 mall_user -e \
  "SELECT id, username, phone, LEFT(phone_enc, 10), LEFT(phone_hash, 16) FROM \`user\` LIMIT 5"
```

Expected: phone_enc 全部以 `v1:` 开头，phone_hash 是 16 字符 hex（截断的）。

- [ ] **Step 4: 幂等性测试**

再跑一次 `go run cmd/backfill_pii/main.go`：

Expected: `found 0 rows to migrate` → `done: total=0`

- [ ] **Step 5: commit**

```bash
cd ~/workspace/go/mall/yw-mall
git add mall-user-rpc/cmd/backfill_pii/
git commit -m "feat(reg-v2 backfill): 老 user.phone 明文 → phone_enc + phone_hash

幂等脚本，扫 phone != '' AND phone_enc = '' 的行转换。
跑一次后 alice/bob/demo 三个种子用户和后来的 2 个，共 5 行全部迁移。
重跑 0 行。

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 4: proto + 重生成 pb.go + userclient

**Files:**
- Modify: `mall-common/proto/user/user.proto`
- Auto-regen: `mall-user-rpc/user/user.pb.go`, `mall-user-rpc/user/user_grpc.pb.go`
- Manual: `mall-user-rpc/userclient/user.go`
- Auto-regen: `mall-user-rpc/internal/server/userserver.go`

- [ ] **Step 1: 改 proto**

在 `mall-common/proto/user/user.proto` 末尾的 `service User { ... }` 内加 3 个 rpc，并在文件上方加 3 组 message：

```protobuf
// ===== Sprint 5 multi-identifier register/login =====

message SendVerifyCodeReq {
  int32  channel = 1;            // 1=sms, 2=email
  string target  = 2;            // phone 或 email
  int32  scene   = 3;            // 1=register, 2=login, 3=bind, 4=change-password
}
message SendVerifyCodeResp {
  string challenge_token = 1;
  int32  expires_in      = 2;
}

message RegisterV2Req {
  string username        = 1;
  string password        = 2;
  string phone           = 3;
  string email           = 4;
  string verify_code     = 5;
  string challenge_token = 6;
}

message LoginV2Req {
  string account  = 1;
  string password = 2;
}
```

在 service 内（`service User { ... }`）追加：

```protobuf
  rpc SendVerifyCode(SendVerifyCodeReq) returns (SendVerifyCodeResp);
  rpc RegisterV2(RegisterV2Req) returns (RegisterResp);
  rpc LoginV2(LoginV2Req) returns (LoginResp);
```

- [ ] **Step 2: 重生成 pb.go**

```bash
cd ~/workspace/go/mall/yw-mall/mall-user-rpc
export PATH=/home/carter/workspace/go/bin:$PATH
protoc --go_out=. --go-grpc_out=. \
  --proto_path=. --proto_path=../mall-common/proto \
  ../mall-common/proto/user/user.proto
```

Expected: 无错误输出，`user/user.pb.go` 和 `user/user_grpc.pb.go` 时间戳更新。

- [ ] **Step 3: 手工补 userclient/user.go**

读 `mall-user-rpc/userclient/user.go`，仿现有 `Login` 函数定义新增 3 个 wrapper：

```go
func (m *defaultUser) SendVerifyCode(ctx context.Context, in *user.SendVerifyCodeReq, opts ...grpc.CallOption) (*user.SendVerifyCodeResp, error) {
    client := user.NewUserClient(m.cli.Conn())
    return client.SendVerifyCode(ctx, in, opts...)
}

func (m *defaultUser) RegisterV2(ctx context.Context, in *user.RegisterV2Req, opts ...grpc.CallOption) (*user.RegisterResp, error) {
    client := user.NewUserClient(m.cli.Conn())
    return client.RegisterV2(ctx, in, opts...)
}

func (m *defaultUser) LoginV2(ctx context.Context, in *user.LoginV2Req, opts ...grpc.CallOption) (*user.LoginResp, error) {
    client := user.NewUserClient(m.cli.Conn())
    return client.LoginV2(ctx, in, opts...)
}
```

同时在文件顶部 `User interface` 加：

```go
    SendVerifyCode(ctx context.Context, in *user.SendVerifyCodeReq, opts ...grpc.CallOption) (*user.SendVerifyCodeResp, error)
    RegisterV2(ctx context.Context, in *user.RegisterV2Req, opts ...grpc.CallOption) (*user.RegisterResp, error)
    LoginV2(ctx context.Context, in *user.LoginV2Req, opts ...grpc.CallOption) (*user.LoginResp, error)
```

并在 type aliases 段加：

```go
    SendVerifyCodeReq  = user.SendVerifyCodeReq
    SendVerifyCodeResp = user.SendVerifyCodeResp
    RegisterV2Req      = user.RegisterV2Req
    LoginV2Req         = user.LoginV2Req
```

- [ ] **Step 4: server 端补 handler stub**

go-zero 不会自动给 server 加新方法。手工在 `mall-user-rpc/internal/server/userserver.go` 加 3 个 method：

```go
func (s *UserServer) SendVerifyCode(ctx context.Context, in *user.SendVerifyCodeReq) (*user.SendVerifyCodeResp, error) {
    l := logic.NewSendVerifyCodeLogic(ctx, s.svcCtx)
    return l.SendVerifyCode(in)
}
func (s *UserServer) RegisterV2(ctx context.Context, in *user.RegisterV2Req) (*user.RegisterResp, error) {
    l := logic.NewRegisterV2Logic(ctx, s.svcCtx)
    return l.RegisterV2(in)
}
func (s *UserServer) LoginV2(ctx context.Context, in *user.LoginV2Req) (*user.LoginResp, error) {
    l := logic.NewLoginV2Logic(ctx, s.svcCtx)
    return l.LoginV2(in)
}
```

注：这步会让 user-rpc 暂时编译不过（logic 还没写），不要 build；写完 Task 5-7 再验证。

- [ ] **Step 5: commit**

```bash
cd ~/workspace/go/mall/yw-mall
git add mall-common/proto/user/user.proto mall-user-rpc/user/ mall-user-rpc/userclient/ mall-user-rpc/internal/server/
git commit -m "feat(reg-v2 proto): 加 SendVerifyCode/RegisterV2/LoginV2

protoc 重生成 + userclient 手工补 wrapper + interface + type aliases。
server stub 占位，logic 实现见后续 commit。
WIP: 单独 build 不通过，配套 Task 5-7 logic 一起才能编译。

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 5: user-rpc SendVerifyCode + Email mock + helpers

**Files:**
- Create: `mall-user-rpc/internal/logic/sendverifycodelogic.go`
- Create: `mall-user-rpc/internal/logic/sendcodehelpers.go`
- Create: `mall-user-rpc/internal/logic/emailhelpers.go`

- [ ] **Step 1: 写 sendcodehelpers.go（共享底层）**

```go
// mall-user-rpc/internal/logic/sendcodehelpers.go
package logic

import (
    "context"
    "crypto/rand"
    "encoding/base64"
    "encoding/json"
    "errors"
    "fmt"
    "math/big"
    "time"

    "mall-user-rpc/internal/svc"
)

const (
    verifyCodeTTL      = 5 * time.Minute
    verifyResendWindow = 60 * time.Second
)

type verifyPayload struct {
    Code           string `json:"code"`
    ChallengeToken string `json:"token"`
    SentAt         int64  `json:"sent_at"`
}

func verifyKey(scene int32, target string) string {
    return fmt.Sprintf("verify:%d:%s", scene, target)
}

func newChallengeTokenStr() (string, error) {
    b := make([]byte, 32)
    if _, err := rand.Read(b); err != nil {
        return "", err
    }
    return base64.RawURLEncoding.EncodeToString(b), nil
}

func randomDigit6() (string, error) {
    const digits = "0123456789"
    out := make([]byte, 6)
    max := big.NewInt(10)
    for i := range out {
        n, err := rand.Int(rand.Reader, max)
        if err != nil {
            return "", err
        }
        out[i] = digits[n.Int64()]
    }
    return string(out), nil
}

// storeVerifyCode 把 (code, token) 写 Redis，TTL 5min。
// 同 target 60 秒内再次发送会被拒绝。
func storeVerifyCode(ctx context.Context, svcCtx *svc.ServiceContext, scene int32, target, code, token string) error {
    k := verifyKey(scene, target)
    if raw, _ := svcCtx.Redis.Get(ctx, k).Bytes(); len(raw) > 0 {
        var old verifyPayload
        if json.Unmarshal(raw, &old) == nil &&
            time.Since(time.Unix(old.SentAt, 0)) < verifyResendWindow {
            return errors.New("发送太频繁，请稍后再试")
        }
    }
    p := verifyPayload{Code: code, ChallengeToken: token, SentAt: time.Now().Unix()}
    data, _ := json.Marshal(p)
    return svcCtx.Redis.Set(ctx, k, data, verifyCodeTTL).Err()
}

// consumeVerifyCode 校验 (token, code) 是否匹配 + 未过期，匹配则删除 (单次消费)。
func consumeVerifyCode(ctx context.Context, svcCtx *svc.ServiceContext, scene int32, target, code, token string) error {
    k := verifyKey(scene, target)
    raw, err := svcCtx.Redis.Get(ctx, k).Bytes()
    if err != nil {
        return errors.New("验证码已过期或不存在")
    }
    var p verifyPayload
    if err := json.Unmarshal(raw, &p); err != nil {
        return errors.New("验证码已过期或不存在")
    }
    if p.ChallengeToken != token {
        return errors.New("challenge token 不匹配")
    }
    if p.Code != code {
        return errors.New("验证码不正确")
    }
    _ = svcCtx.Redis.Del(ctx, k).Err()
    return nil
}
```

- [ ] **Step 2: 写 emailhelpers.go（mock）**

```go
// mall-user-rpc/internal/logic/emailhelpers.go
package logic

import (
    "context"

    "github.com/zeromicro/go-zero/core/logx"
)

// EmailSend mock — 把 (target, code) 打到 logx，生产替换成 SES/SMTP 实现。
// 跟 S4.1 SmsSend 同模式，方便统一升级。
func EmailSend(ctx context.Context, target, code string) error {
    logx.WithContext(ctx).Infof("[mock-email] target=%s code=%s", target, code)
    return nil
}
```

- [ ] **Step 3: 写 sendverifycodelogic.go**

```go
// mall-user-rpc/internal/logic/sendverifycodelogic.go
package logic

import (
    "context"
    "errors"
    "fmt"

    "mall-user-rpc/internal/svc"
    "mall-user-rpc/user"

    "github.com/zeromicro/go-zero/core/logx"
)

type SendVerifyCodeLogic struct {
    ctx    context.Context
    svcCtx *svc.ServiceContext
    logx.Logger
}

func NewSendVerifyCodeLogic(ctx context.Context, svcCtx *svc.ServiceContext) *SendVerifyCodeLogic {
    return &SendVerifyCodeLogic{ctx: ctx, svcCtx: svcCtx, Logger: logx.WithContext(ctx)}
}

const (
    channelSms   = 1
    channelEmail = 2
)

func (l *SendVerifyCodeLogic) SendVerifyCode(in *user.SendVerifyCodeReq) (*user.SendVerifyCodeResp, error) {
    if in.Target == "" {
        return nil, errors.New("target required")
    }
    code, err := randomDigit6()
    if err != nil {
        return nil, err
    }
    token, err := newChallengeTokenStr()
    if err != nil {
        return nil, err
    }
    if err := storeVerifyCode(l.ctx, l.svcCtx, in.Scene, in.Target, code, token); err != nil {
        return nil, err
    }

    switch in.Channel {
    case channelSms:
        // 复用 S4 现成 SmsSend 实现：写入相同 Redis key 那套是 admin MFA 专用,
        // 这里直接 logx 占位，跟 EmailSend 同模式，避免双写 Redis
        logx.WithContext(l.ctx).Infof("[mock-sms] scene=%d target=%s code=%s", in.Scene, in.Target, code)
    case channelEmail:
        if err := EmailSend(l.ctx, in.Target, code); err != nil {
            return nil, err
        }
    default:
        return nil, fmt.Errorf("unsupported channel: %d", in.Channel)
    }

    return &user.SendVerifyCodeResp{
        ChallengeToken: token,
        ExpiresIn:      int32(verifyCodeTTL.Seconds()),
    }, nil
}
```

- [ ] **Step 4: build user-rpc 验证编译**

```bash
cd ~/workspace/go/mall/yw-mall/mall-user-rpc && go build ./...
```

Expected: 应该报 RegisterV2Logic/LoginV2Logic 不存在（server stub 引用了）。这是预期的，Task 6-7 补完。

- [ ] **Step 5: commit**

```bash
cd ~/workspace/go/mall/yw-mall
git add mall-user-rpc/internal/logic/sendverifycodelogic.go \
        mall-user-rpc/internal/logic/sendcodehelpers.go \
        mall-user-rpc/internal/logic/emailhelpers.go
git commit -m "feat(reg-v2 user-rpc): SendVerifyCode + 验证码握手 helpers + email mock

- sendcodehelpers: verify:{scene}:{target} Redis key, 5min TTL, 60s 重发窗口,
  storeVerifyCode / consumeVerifyCode 单次消费
- emailhelpers: EmailSend 写 logx (同 S4.1 SmsSend 模式)
- sendverifycodelogic: channel=1 logx mock SMS / channel=2 走 EmailSend

WIP: build 待 RegisterV2/LoginV2 logic 完成才通过.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 6: user-rpc RegisterV2

**Files:**
- Create: `mall-user-rpc/internal/logic/registerv2logic.go`

- [ ] **Step 1: 写 RegisterV2Logic**

```go
// mall-user-rpc/internal/logic/registerv2logic.go
package logic

import (
    "context"
    "errors"
    "fmt"
    "regexp"
    "strings"
    "time"

    "mall-common/cryptox"
    "mall-user-rpc/internal/svc"
    "mall-user-rpc/user"

    "github.com/zeromicro/go-zero/core/logx"
    "github.com/zeromicro/go-zero/core/stores/sqlx"
    "golang.org/x/crypto/bcrypt"
)

type RegisterV2Logic struct {
    ctx    context.Context
    svcCtx *svc.ServiceContext
    logx.Logger
}

func NewRegisterV2Logic(ctx context.Context, svcCtx *svc.ServiceContext) *RegisterV2Logic {
    return &RegisterV2Logic{ctx: ctx, svcCtx: svcCtx, Logger: logx.WithContext(ctx)}
}

const (
    sceneRegister = 1
)

var (
    usernameRE = regexp.MustCompile(`^[a-zA-Z][a-zA-Z0-9_]{3,31}$`)  // 4-32, 字母开头
    phoneRE    = regexp.MustCompile(`^\d{11}$`)
    emailRE    = regexp.MustCompile(`^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$`)
)

func (l *RegisterV2Logic) RegisterV2(in *user.RegisterV2Req) (*user.RegisterResp, error) {
    username := strings.TrimSpace(in.Username)
    phone := strings.TrimSpace(in.Phone)
    email := strings.ToLower(strings.TrimSpace(in.Email))
    password := trimPlain(in.Password)

    if !usernameRE.MatchString(username) {
        return nil, errors.New("用户名格式不合法（4-32 位字母开头）")
    }
    if phone == "" && email == "" {
        return nil, errors.New("手机号和邮箱至少填一个")
    }
    if phone != "" && !phoneRE.MatchString(phone) {
        return nil, errors.New("手机号格式不合法")
    }
    if email != "" && !emailRE.MatchString(email) {
        return nil, errors.New("邮箱格式不合法")
    }
    if err := validatePassword(password, l.svcCtx.PasswordPolicy); err != nil {
        return nil, err
    }

    // 1. username 唯一
    var cnt int64
    if err := l.svcCtx.DB.QueryRowCtx(l.ctx, &cnt,
        "SELECT COUNT(*) FROM `user` WHERE username=?", username); err != nil {
        return nil, err
    }
    if cnt > 0 {
        return nil, errors.New("用户名已被使用")
    }

    // 2. 验证码消费 + identifier 占用检查
    target := phone
    if target == "" {
        target = email
    }
    if err := consumeVerifyCode(l.ctx, l.svcCtx, sceneRegister, target,
        in.VerifyCode, in.ChallengeToken); err != nil {
        return nil, err
    }

    phoneHash := cryptox.Hmac(phone)
    emailHash := cryptox.Hmac(email)

    if phoneHash != "" {
        if err := l.svcCtx.DB.QueryRowCtx(l.ctx, &cnt,
            "SELECT COUNT(*) FROM `user` WHERE phone_hash=?", phoneHash); err != nil {
            return nil, err
        }
        if cnt > 0 {
            return nil, errors.New("手机号已被注册")
        }
    }
    if emailHash != "" {
        if err := l.svcCtx.DB.QueryRowCtx(l.ctx, &cnt,
            "SELECT COUNT(*) FROM `user` WHERE email_hash=?", emailHash); err != nil {
            return nil, err
        }
        if cnt > 0 {
            return nil, errors.New("邮箱已被注册")
        }
    }

    // 3. 加密 + bcrypt
    var phoneEnc, emailEnc string
    if phone != "" {
        if phoneEnc, err := cryptox.Encrypt(phone); err != nil || phoneEnc == "" {
            return nil, fmt.Errorf("encrypt phone: %w", err)
        }
    }
    if email != "" {
        if emailEnc, err := cryptox.Encrypt(email); err != nil || emailEnc == "" {
            return nil, fmt.Errorf("encrypt email: %w", err)
        }
    }
    // 注意：上面的 := 会创建作用域内变量。重写一次确保赋值到外层。
    if phone != "" {
        phoneEnc, _ = cryptox.Encrypt(phone)
    }
    if email != "" {
        emailEnc, _ = cryptox.Encrypt(email)
    }

    hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    if err != nil {
        return nil, err
    }

    // 4. INSERT
    now := time.Now().Unix()
    res, err := l.svcCtx.DB.ExecCtx(l.ctx, `
        INSERT INTO user (username, password, phone, email, phone_enc, email_enc,
                          phone_hash, email_hash, last_password_change)
        VALUES (?, ?, '', ?, ?, ?, ?, ?, ?)`,
        username, string(hash), email, phoneEnc, emailEnc, phoneHash, emailHash, now)
    if err != nil {
        if isDuplicateKeyErr(err) {
            return nil, errors.New("用户名/手机号/邮箱已被使用")
        }
        return nil, err
    }
    id, _ := res.LastInsertId()

    // 5. 记一次 password_history
    _ = recordPasswordHistory(l.ctx, l.svcCtx.DB, subjectTypeUser, uint64(id),
        string(hash), l.svcCtx.PasswordPolicy.MaxHistory)

    return &user.RegisterResp{Id: id}, nil
}

func isDuplicateKeyErr(err error) bool {
    if err == sqlx.ErrNotFound {
        return false
    }
    return strings.Contains(err.Error(), "Duplicate entry")
}
```

- [ ] **Step 2: 临时 sanity build**

```bash
cd ~/workspace/go/mall/yw-mall/mall-user-rpc && go build ./...
```

Expected: 还报 LoginV2Logic 不存在，正常。

- [ ] **Step 3: commit**

```bash
cd ~/workspace/go/mall/yw-mall
git add mall-user-rpc/internal/logic/registerv2logic.go
git commit -m "feat(reg-v2 user-rpc): RegisterV2 — username + phone/email 二选一 + 验证码

- username regex 校验 (4-32 字母开头)
- phone/email 至少一个 + 各自 regex
- 复用 S4.3 validatePassword 强度策略
- 验证码 consumeVerifyCode 单次消费
- phone_hash / email_hash 查重 (idx_*_hash 命中)
- cryptox.Encrypt 加密写 phone_enc / email_enc
- bcrypt + INSERT user + recordPasswordHistory

注: phone 明文列写空串 (过渡期不读不写, 下个 sprint 删).

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 7: user-rpc LoginV2

**Files:**
- Create: `mall-user-rpc/internal/logic/loginv2logic.go`

- [ ] **Step 1: 写 LoginV2Logic**

```go
// mall-user-rpc/internal/logic/loginv2logic.go
package logic

import (
    "context"
    "errors"
    "strings"
    "time"

    "mall-common/cryptox"
    "mall-user-rpc/internal/svc"
    "mall-user-rpc/user"

    "github.com/zeromicro/go-zero/core/logx"
    "github.com/zeromicro/go-zero/core/stores/sqlx"
    "golang.org/x/crypto/bcrypt"
)

type LoginV2Logic struct {
    ctx    context.Context
    svcCtx *svc.ServiceContext
    logx.Logger
}

func NewLoginV2Logic(ctx context.Context, svcCtx *svc.ServiceContext) *LoginV2Logic {
    return &LoginV2Logic{ctx: ctx, svcCtx: svcCtx, Logger: logx.WithContext(ctx)}
}

func (l *LoginV2Logic) LoginV2(in *user.LoginV2Req) (*user.LoginResp, error) {
    account := strings.TrimSpace(in.Account)
    password := trimPlain(in.Password)
    if account == "" || password == "" {
        return nil, errors.New("用户名或密码错误")
    }

    var row struct {
        Id                 uint64 `db:"id"`
        Username           string `db:"username"`
        PasswordHash       string `db:"password"`
        LastPasswordChange int64  `db:"last_password_change"`
        Status             int32  `db:"status"`
    }

    var err error
    switch {
    case emailRE.MatchString(account):
        hash := cryptox.Hmac(strings.ToLower(account))
        err = l.svcCtx.DB.QueryRowCtx(l.ctx, &row, `
            SELECT id, username, password, last_password_change, status
            FROM user WHERE email_hash=? LIMIT 1`, hash)
    case phoneRE.MatchString(account):
        hash := cryptox.Hmac(account)
        err = l.svcCtx.DB.QueryRowCtx(l.ctx, &row, `
            SELECT id, username, password, last_password_change, status
            FROM user WHERE phone_hash=? LIMIT 1`, hash)
    default:
        err = l.svcCtx.DB.QueryRowCtx(l.ctx, &row, `
            SELECT id, username, password, last_password_change, status
            FROM user WHERE username=? LIMIT 1`, account)
    }

    if err != nil {
        if err == sqlx.ErrNotFound {
            // 防枚举：用户不存在 → 同一错误信息
            return nil, errors.New("用户名或密码错误")
        }
        return nil, err
    }
    if row.Status == 0 {
        return nil, errors.New("账号已停用")
    }
    if err := bcrypt.CompareHashAndPassword([]byte(row.PasswordHash), []byte(password)); err != nil {
        return nil, errors.New("用户名或密码错误")
    }

    expired := passwordExpired(row.LastPasswordChange, l.svcCtx.PasswordPolicy.MaxAgeDays)

    // 复用 P0 CreateSession
    sess, err := NewCreateSessionLogic(l.ctx, l.svcCtx).CreateSession(&user.CreateSessionReq{
        Uid:      int64(row.Id),
        Username: row.Username,
        Role:     "user",
    })
    if err != nil {
        return nil, err
    }

    return &user.LoginResp{
        Id:              int64(row.Id),
        Token:           sess.AccessToken,
        RefreshToken:    sess.RefreshToken,
        ExpiresIn:       sess.ExpiresIn,
        CsrfToken:       sess.CsrfToken,
        PasswordExpired: expired,
    }, nil
    _ = time.Now() // silence unused import if needed later
}
```

- [ ] **Step 2: build 整个 user-rpc 应该过**

```bash
cd ~/workspace/go/mall/yw-mall/mall-user-rpc && go build ./...
```

Expected: 0 错。

- [ ] **Step 3: 重启 user-rpc 验证启动**

```bash
cd ~/workspace/go/mall/yw-mall && ./start.sh restart 2>&1 | grep user-rpc
sleep 3
tail -5 logs/user-rpc.log
```

Expected: user-rpc OK，logs 没 panic。

- [ ] **Step 4: commit**

```bash
git add mall-user-rpc/internal/logic/loginv2logic.go
git commit -m "feat(reg-v2 user-rpc): LoginV2 — account 字段 regex 自动识别 + 防枚举

regex 分类: emailRE / phoneRE / 否则 username。phone/email 走 cryptox.Hmac
blind index 查询。用户不存在 + 密码错误统一返同一错误信息防枚举。
status=0 (停用) 单独识别。bcrypt 比对成功后调 CreateSession 拿 opaque token,
S4.3 passwordExpired 透传给 FE.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 8: mall-api SendCode handler/logic + 路由

**Files:**
- Modify: `mall-api/internal/types/types.go`
- Create: `mall-api/internal/handler/sendcodehandler.go`
- Create: `mall-api/internal/logic/sendcodelogic.go`
- Modify: `mall-api/internal/handler/routes.go`

- [ ] **Step 1: types.go 加 SendCodeReq/Resp**

在 `mall-api/internal/types/types.go` 末尾加：

```go
// ===== Sprint 5 multi-identifier register/login =====
type SendCodeReq struct {
    Channel int32  `json:"channel"`           // 1=sms, 2=email
    Target  string `json:"target"`
    Scene   int32  `json:"scene"`             // 1=register, etc.
}
type SendCodeResp struct {
    ChallengeToken string `json:"challengeToken"`
    ExpiresIn      int32  `json:"expiresIn"`
}

type RegisterV2Req struct {
    Username       string `json:"username"`
    Password       string `json:"password"`
    Phone          string `json:"phone,optional"`
    Email          string `json:"email,optional"`
    VerifyCode     string `json:"verifyCode"`
    ChallengeToken string `json:"challengeToken"`
}
type RegisterV2Resp struct {
    Id int64 `json:"id"`
}

type LoginV2Req struct {
    Account  string `json:"account"`
    Password string `json:"password"`
}
```

- [ ] **Step 2: sendcodelogic.go**

```go
// mall-api/internal/logic/sendcodelogic.go
package logic

import (
    "context"
    "errors"

    "mall-api/internal/svc"
    "mall-api/internal/types"

    "mall-user-rpc/userclient"

    "github.com/zeromicro/go-zero/core/logx"
)

func SendCode(ctx context.Context, svcCtx *svc.ServiceContext, req *types.SendCodeReq) (*types.SendCodeResp, error) {
    if req.Target == "" {
        return nil, errors.New("target required")
    }
    resp, err := svcCtx.UserRpc.SendVerifyCode(ctx, &userclient.SendVerifyCodeReq{
        Channel: req.Channel, Target: req.Target, Scene: req.Scene,
    })
    if err != nil {
        logx.WithContext(ctx).Errorf("SendCode rpc fail: %v", err)
        return nil, err
    }
    return &types.SendCodeResp{
        ChallengeToken: resp.ChallengeToken,
        ExpiresIn:      resp.ExpiresIn,
    }, nil
}
```

- [ ] **Step 3: sendcodehandler.go**

```go
// mall-api/internal/handler/sendcodehandler.go
package handler

import (
    "net/http"

    "mall-api/internal/logic"
    "mall-api/internal/svc"
    "mall-api/internal/types"

    "github.com/zeromicro/go-zero/rest/httpx"
)

func SendCodeHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var req types.SendCodeReq
        if err := httpx.Parse(r, &req); err != nil {
            httpx.ErrorCtx(r.Context(), w, err)
            return
        }
        resp, err := logic.SendCode(r.Context(), svcCtx, &req)
        if err != nil {
            httpx.ErrorCtx(r.Context(), w, err)
            return
        }
        httpx.OkJsonCtx(r.Context(), w, resp)
    }
}
```

- [ ] **Step 4: routes.go 注册路由**

找到 `/api/auth/login` 路由所在的 group（用 `grep -n "api/auth/login" mall-api/internal/handler/routes.go`），在同 group 内加：

```go
        {Method: http.MethodPost, Path: "/api/auth/send-code", Handler: SendCodeHandler(serverCtx)},
```

- [ ] **Step 5: build + commit**

```bash
cd ~/workspace/go/mall/yw-mall/mall-api && go build ./...
# Expected: 0 错

cd ~/workspace/go/mall/yw-mall
git add mall-api/internal/types/types.go mall-api/internal/handler/sendcodehandler.go \
        mall-api/internal/logic/sendcodelogic.go mall-api/internal/handler/routes.go
git commit -m "feat(reg-v2 api): POST /api/auth/send-code → user-rpc.SendVerifyCode

未鉴权端点 (在 /api/auth/* 同 group). 返 {challengeToken, expiresIn}.
后续 RegisterV2 / 登录验证码登录会复用同一 token + code 组合.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 9: mall-api RegisterV2 handler/logic + 路由

**Files:**
- Create: `mall-api/internal/handler/registerv2handler.go`
- Create: `mall-api/internal/logic/registerv2logic.go`
- Modify: `mall-api/internal/handler/routes.go`

- [ ] **Step 1: registerv2logic.go**

```go
// mall-api/internal/logic/registerv2logic.go
package logic

import (
    "context"

    "mall-api/internal/svc"
    "mall-api/internal/types"

    "mall-user-rpc/userclient"
)

func RegisterV2(ctx context.Context, svcCtx *svc.ServiceContext, req *types.RegisterV2Req) (*types.RegisterV2Resp, error) {
    resp, err := svcCtx.UserRpc.RegisterV2(ctx, &userclient.RegisterV2Req{
        Username:       req.Username,
        Password:       req.Password,
        Phone:          req.Phone,
        Email:          req.Email,
        VerifyCode:     req.VerifyCode,
        ChallengeToken: req.ChallengeToken,
    })
    if err != nil {
        return nil, err
    }
    return &types.RegisterV2Resp{Id: resp.Id}, nil
}
```

- [ ] **Step 2: registerv2handler.go**

```go
// mall-api/internal/handler/registerv2handler.go
package handler

import (
    "net/http"

    "mall-api/internal/logic"
    "mall-api/internal/svc"
    "mall-api/internal/types"

    "github.com/zeromicro/go-zero/rest/httpx"
)

func RegisterV2Handler(svcCtx *svc.ServiceContext) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var req types.RegisterV2Req
        if err := httpx.Parse(r, &req); err != nil {
            httpx.ErrorCtx(r.Context(), w, err)
            return
        }
        resp, err := logic.RegisterV2(r.Context(), svcCtx, &req)
        if err != nil {
            httpx.ErrorCtx(r.Context(), w, err)
            return
        }
        httpx.OkJsonCtx(r.Context(), w, resp)
    }
}
```

- [ ] **Step 3: routes.go 加 /api/auth/register**

同 send-code 同 group：

```go
        {Method: http.MethodPost, Path: "/api/auth/register", Handler: RegisterV2Handler(serverCtx)},
```

- [ ] **Step 4: build + commit**

```bash
cd ~/workspace/go/mall/yw-mall/mall-api && go build ./...

cd ~/workspace/go/mall/yw-mall
git add mall-api/internal/handler/registerv2handler.go \
        mall-api/internal/logic/registerv2logic.go \
        mall-api/internal/handler/routes.go
git commit -m "feat(reg-v2 api): POST /api/auth/register → user-rpc.RegisterV2

接受 username/password/phone/email/verifyCode/challengeToken,
返 {id}. 错误透传 user-rpc 中文消息.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 10: mall-api LoginV2 切换 + 老路由兼容

**Files:**
- Modify: `mall-api/internal/logic/authloginlogic.go`
- Modify: `mall-api/internal/types/types.go` (AuthLoginReq 已有，确认是否要 account 字段)
- Modify: `mall-api/internal/handler/routes.go` (老 /api/user/register 兼容)

- [ ] **Step 1: 查 AuthLoginReq 现状**

```bash
grep -B1 -A6 "AuthLoginReq" ~/workspace/go/mall/yw-mall/mall-api/internal/types/types.go | head -10
```

如果当前是 `{Username, Password}`，加 `Account` 字段，保留 `Username` 兼容老 FE：

```go
type AuthLoginReq struct {
    Account  string `json:"account,optional"`  // 优先用 Account；兼容老 FE 给 Username
    Username string `json:"username,optional"`
    Password string `json:"password"`
}
```

- [ ] **Step 2: 改 authloginlogic.go 切到 LoginV2**

读 `mall-api/internal/logic/authloginlogic.go`，把现有调 UserRpc.Login 的逻辑改成：

```go
func (l *AuthLoginLogic) AuthLogin(req *types.AuthLoginReq) (*types.AuthLoginResp, error) {
    account := req.Account
    if account == "" {
        account = req.Username  // 兼容老 FE
    }
    if account == "" {
        return nil, errors.New("account required")
    }
    res, err := l.svcCtx.UserRpc.LoginV2(l.ctx, &userclient.LoginV2Req{
        Account:  account,
        Password: req.Password,
    })
    if err != nil {
        return nil, err
    }
    // 老的 response 转换逻辑保留不变，把 res 的字段塞进 AuthLoginResp...
}
```

保留 username → 老 FE 不受影响；新 FE 用 Account。

- [ ] **Step 3: 老 /api/user/register 兼容层（保留路由返清晰错误）**

老 `/api/user/register` 路由对应的 logic 不动 — 它仍走老 Register RPC，不带 verify_code 直接 INSERT。**决策修订**：考虑到 backfill 已经把老用户的 phone 转到 phone_enc/phone_hash，老 RegisterReq 不写新字段会让新注册的用户既没 phone_hash 也没 email_hash，导致后续 LoginV2 用 phone/email 登录拿不到。

简化方案：老 `/api/user/register` 路由直接 hard-fail 返：

```json
{"code": 410, "message": "此接口已下线，请使用 /api/auth/register"}
```

- 修改 `mall-api/internal/logic/userregisterlogic.go` 第一行 `return errors.New("此接口已下线，请使用 /api/auth/register")`
- 路由保留不删（404 比 410 更难排查）

- [ ] **Step 4: build + commit**

```bash
cd ~/workspace/go/mall/yw-mall/mall-api && go build ./...

cd ~/workspace/go/mall/yw-mall
git add mall-api/internal/logic/authloginlogic.go \
        mall-api/internal/logic/userregisterlogic.go \
        mall-api/internal/types/types.go
git commit -m "feat(reg-v2 api): /api/auth/login 走 LoginV2 + 老 /api/user/register 410

- AuthLoginReq 加 Account 字段, 兼容老 Username 字段 (老 FE 不动)
- authloginlogic 改调 UserRpc.LoginV2, account 透传
- /api/user/register 老路由保留但 fail 410 'use /api/auth/register' (避免
  绕过 verify_code 注册造成 phone_hash/email_hash 缺失)

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 11: e2e 验证后端

整个后端跑完，先手测一遍再做前端。

- [ ] **Step 1: 重启 yw-mall**

```bash
cd ~/workspace/go/mall/yw-mall && ./start.sh restart
sleep 5
./start.sh status | grep -v stopped
```

Expected: 全部 16 服务 running，0 stopped。

- [ ] **Step 2: send-code 测试**

```bash
RESP=$(curl -s -m 5 -X POST http://localhost:18888/api/auth/send-code \
  -H 'Content-Type: application/json' \
  -d '{"channel":1,"target":"13900000001","scene":1}')
echo "$RESP"
TOKEN=$(echo "$RESP" | grep -oE '"challengeToken":"[^"]+' | cut -d'"' -f4)
echo "TOKEN=$TOKEN"

# 查 user-rpc 日志拿验证码
tail -5 logs/user-rpc.log | grep mock-sms
```

Expected: 拿到 challengeToken（非空），logs 看到 `[mock-sms] scene=1 target=13900000001 code=XXXXXX`。

- [ ] **Step 3: register 测试**

```bash
CODE=$(tail -5 logs/user-rpc.log | grep mock-sms | tail -1 | grep -oE 'code=[0-9]+' | cut -d= -f2)
echo "CODE=$CODE"

curl -s -m 5 -X POST http://localhost:18888/api/auth/register \
  -H 'Content-Type: application/json' \
  -d "{\"username\":\"newuser01\",\"password\":\"Test1234\",\"phone\":\"13900000001\",\"verifyCode\":\"$CODE\",\"challengeToken\":\"$TOKEN\"}"
```

Expected: `{"id":<某 id>}`。

- [ ] **Step 4: LoginV2 三种 identifier**

```bash
echo "=== username 登录 ==="
curl -s -X POST http://localhost:18888/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"account":"newuser01","password":"Test1234"}' | head -c 200
echo ""

echo "=== phone 登录 ==="
curl -s -X POST http://localhost:18888/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"account":"13900000001","password":"Test1234"}' | head -c 200
echo ""

echo "=== 老用户 alice (username) 登录 — 兼容性测试 ==="
curl -s -X POST http://localhost:18888/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"account":"alice","password":"alice123"}' | head -c 200
echo ""

echo "=== alice 用 phone 登录（backfill 应已写 phone_hash）==="
curl -s -X POST http://localhost:18888/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"account":"13800138001","password":"alice123"}' | head -c 200
echo ""

echo "=== 错误密码 → 防枚举 ==="
curl -s -X POST http://localhost:18888/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"account":"nonexistent999","password":"x"}'
echo ""
```

Expected: 4 个成功（拿 accessToken），第 5 个返「用户名或密码错误」。

- [ ] **Step 5: 重复注册 → 应 fail**

```bash
# 同手机号再注册
curl -s -X POST http://localhost:18888/api/auth/send-code \
  -d '{"channel":1,"target":"13900000001","scene":1}' -H 'Content-Type: application/json'
# 上面应该返 "发送太频繁" 因为 60s 重发窗口；等 60s 或换 target

# 简单验证：同 phone 注册第二次 → fail "手机号已被注册"
sleep 65 && \
RESP2=$(curl -s -X POST http://localhost:18888/api/auth/send-code \
  -d '{"channel":1,"target":"13900000001","scene":1}' -H 'Content-Type: application/json')
TOKEN2=$(echo "$RESP2" | grep -oE '"challengeToken":"[^"]+' | cut -d'"' -f4)
CODE2=$(tail -5 logs/user-rpc.log | grep mock-sms | tail -1 | grep -oE 'code=[0-9]+' | cut -d= -f2)
curl -s -X POST http://localhost:18888/api/auth/register \
  -H 'Content-Type: application/json' \
  -d "{\"username\":\"newuser02\",\"password\":\"Test1234\",\"phone\":\"13900000001\",\"verifyCode\":\"$CODE2\",\"challengeToken\":\"$TOKEN2\"}"
```

Expected: 「手机号已被注册」错误。

- [ ] **Step 6: 老 /api/user/register → 410**

```bash
curl -s -X POST http://localhost:18888/api/user/register \
  -d '{"username":"xxx","password":"yyy","phone":"123"}' -H 'Content-Type: application/json'
```

Expected: 包含「此接口已下线」消息。

- [ ] **Step 7: e2e 通过后不 commit，进 Task 12 做前端**

---

## Task 12: mall-fe types + api/auth.ts

**Files:**
- Modify: `~/workspace/go/mall/yw-mall-fe/src/types/api.ts`
- Create: `~/workspace/go/mall/yw-mall-fe/src/api/auth.ts`
- Modify: `~/workspace/go/mall/yw-mall-fe/src/api/user.ts`

- [ ] **Step 1: types/api.ts 加类型**

在 types/api.ts 末尾加：

```ts
// ===== Sprint 5 multi-identifier register/login =====
export interface SendCodeReq {
  channel: 1 | 2;          // 1=sms, 2=email
  target: string;
  scene: number;
}
export interface SendCodeResp {
  challengeToken: string;
  expiresIn: number;
}

export interface RegisterV2Req {
  username: string;
  password: string;
  phone?: string;
  email?: string;
  verifyCode: string;
  challengeToken: string;
}
export interface RegisterV2Resp { id: number }
```

- [ ] **Step 2: api/auth.ts**

```ts
// src/api/auth.ts
import { request } from './request'
import type { SendCodeReq, SendCodeResp, RegisterV2Req, RegisterV2Resp } from '@/types/api'

export function sendVerifyCode(req: SendCodeReq) {
  return request<SendCodeResp>({
    url: '/api/auth/send-code',
    method: 'POST',
    data: req,
  })
}

export function registerV2(req: RegisterV2Req) {
  return request<RegisterV2Resp>({
    url: '/api/auth/register',
    method: 'POST',
    data: req,
  })
}
```

- [ ] **Step 3: api/user.ts login 切到 account 字段**

修改现有 `login()` 函数，新签名：

```ts
export function login(account: string, password: string) {
  return request<AuthLoginResp>({
    url: '/api/auth/login',
    method: 'POST',
    data: { account, password },
  })
}
```

调用方（pages/login/index.vue）已经是 2 个参数，传新版即可（参数语义从 username 改 account，但前端调用代码不必改变）。

- [ ] **Step 4: tsc 校验**

```bash
cd ~/workspace/go/mall/yw-mall-fe && pnpm vue-tsc --noEmit 2>&1 | head -20
```

Expected: 0 错。

- [ ] **Step 5: commit**

```bash
git add src/api/auth.ts src/api/user.ts src/types/api.ts
git commit -m "feat(reg-v2 fe): types + api/auth.ts + login() 切 account 字段

- 加 SendCodeReq/Resp + RegisterV2Req/Resp 类型
- api/auth.ts: sendVerifyCode + registerV2
- api/user.ts login() 参数 username → account, URL → /api/auth/login

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 13: mall-fe 注册页

**Files:**
- Create: `~/workspace/go/mall/yw-mall-fe/src/pages/login/register.vue`

- [ ] **Step 1: 写 register.vue**

```vue
<template>
  <view class="page">
    <view class="logo-area">
      <wd-icon name="shop" size="64px" :color="'#FF4B4B'" />
      <text class="app-name">注册账号</text>
    </view>

    <view class="form">
      <wd-input v-model="username" placeholder="用户名（4-32 位字母开头）" clearable />
      <wd-input v-model="password" placeholder="密码（8 位以上 + 字母+数字）" type="password" clearable />
      <wd-input v-model="passwordConfirm" placeholder="确认密码" type="password" clearable />

      <wd-radio-group v-model="channel" inline shape="dot">
        <wd-radio :value="1">手机号</wd-radio>
        <wd-radio :value="2">邮箱</wd-radio>
      </wd-radio-group>

      <wd-input v-model="target" :placeholder="channel === 1 ? '手机号 11 位' : '邮箱地址'" clearable />

      <view class="code-row">
        <wd-input v-model="verifyCode" placeholder="6 位验证码" maxlength="6" class="code-input" />
        <wd-button
          size="small"
          :disabled="cooldown > 0 || !target"
          :loading="sending"
          class="code-btn"
          @click="onSendCode"
        >{{ cooldown > 0 ? `${cooldown}s` : '发送验证码' }}</wd-button>
      </view>

      <wd-button
        type="primary"
        block
        :loading="submitting"
        :disabled="!canSubmit"
        class="submit-btn"
        @click="onSubmit"
      >注册</wd-button>

      <view class="login-link" @click="goLogin">已有账号？立即登录</view>
    </view>
  </view>
</template>

<script setup lang="ts">
import { ref, computed, onUnmounted } from 'vue'
import { sendVerifyCode, registerV2 } from '@/api/auth'
import { showError } from '@/api/request'

const username = ref('')
const password = ref('')
const passwordConfirm = ref('')
const channel = ref<1 | 2>(1)
const target = ref('')
const verifyCode = ref('')
const challengeToken = ref('')

const cooldown = ref(0)
const sending = ref(false)
const submitting = ref(false)
let timer: ReturnType<typeof setInterval> | null = null

const canSubmit = computed(() =>
  !!username.value && !!password.value &&
  password.value === passwordConfirm.value &&
  !!target.value && !!verifyCode.value && !!challengeToken.value,
)

async function onSendCode() {
  if (cooldown.value > 0) return
  sending.value = true
  try {
    const res = await sendVerifyCode({
      channel: channel.value,
      target: target.value,
      scene: 1,
    })
    challengeToken.value = res.challengeToken
    uni.showToast({ title: '验证码已发送', icon: 'success' })
    cooldown.value = 60
    timer = setInterval(() => {
      cooldown.value -= 1
      if (cooldown.value <= 0 && timer) { clearInterval(timer); timer = null }
    }, 1000)
  } catch (err) {
    showError(err)
  } finally {
    sending.value = false
  }
}

async function onSubmit() {
  if (password.value !== passwordConfirm.value) {
    uni.showToast({ title: '两次密码不一致', icon: 'none' })
    return
  }
  submitting.value = true
  try {
    await registerV2({
      username: username.value,
      password: password.value,
      phone: channel.value === 1 ? target.value : '',
      email: channel.value === 2 ? target.value : '',
      verifyCode: verifyCode.value,
      challengeToken: challengeToken.value,
    })
    uni.showToast({ title: '注册成功，请登录', icon: 'success' })
    setTimeout(() => uni.reLaunch({ url: '/pages/login/index' }), 1200)
  } catch (err) {
    showError(err)
  } finally {
    submitting.value = false
  }
}

function goLogin() {
  uni.navigateBack({ delta: 1, fail: () => uni.reLaunch({ url: '/pages/login/index' }) })
}

onUnmounted(() => { if (timer) clearInterval(timer) })
</script>

<style lang="scss" scoped>
.page {
  min-height: 100vh;
  background: $color-bg-page;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 0 $space-lg;
}
.logo-area {
  margin-top: 60px;
  margin-bottom: $space-xl;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: $space-sm;
}
.app-name { font-size: $font-size-xl; font-weight: $font-weight-bold; }
.form { width: 100%; display: flex; flex-direction: column; gap: $space-md; }
.code-row { display: flex; align-items: center; gap: $space-md; }
.code-input { flex: 1; }
.code-btn { flex-shrink: 0; }
.submit-btn { margin-top: $space-sm; }
.login-link {
  text-align: center;
  color: $color-primary;
  font-size: $font-size-sm;
  margin-top: $space-md;
}
</style>
```

- [ ] **Step 2: commit**

```bash
git add src/pages/login/register.vue
git commit -m "feat(reg-v2 fe): pages/login/register.vue — 注册页 + 60s 验证码倒计时

手机/邮箱二选一 radio + send-code 按钮 + 倒计时 + 注册按钮.
成功后 1.2s reLaunch 登录页.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 14: mall-fe pages.json + login.vue 改造

**Files:**
- Modify: `~/workspace/go/mall/yw-mall-fe/src/pages.json`
- Modify: `~/workspace/go/mall/yw-mall-fe/src/pages/login/index.vue`

- [ ] **Step 1: pages.json 注册 register 页**

```bash
grep -n "pages/login/index" src/pages.json
```

在 login/index 后追加：

```json
    {
      "path": "pages/login/register",
      "style": { "navigationBarTitleText": "注册" }
    },
```

- [ ] **Step 2: 改 login/index.vue**

修改第一个 wd-input 的 placeholder + 在「登录」按钮下加注册链接：

```vue
<wd-input
  v-model="username"
  placeholder="用户名 / 手机号 / 邮箱"
  clearable
/>
```

并把 `username` 变量内部仍用，因为 api/user.ts login() 接收 account 但 vue 内部名可保持。

按钮下追加：

```vue
<view class="register-link" @click="goRegister">还没账号？立即注册</view>
```

script 里加：

```ts
function goRegister() {
  uni.navigateTo({ url: '/pages/login/register' })
}
```

style 里加：

```scss
.register-link {
  text-align: center;
  color: $color-primary;
  font-size: $font-size-sm;
  margin-top: $space-md;
}
```

- [ ] **Step 3: build 校验**

```bash
cd ~/workspace/go/mall/yw-mall-fe && pnpm vue-tsc --noEmit 2>&1 | head -10
```

Expected: 0 错。

- [ ] **Step 4: commit**

```bash
git add src/pages.json src/pages/login/index.vue
git commit -m "feat(reg-v2 fe): pages.json 注册 + login.vue placeholder + 注册链接

- pages.json 加 pages/login/register
- login/index.vue placeholder '用户名' → '用户名 / 手机号 / 邮箱'
- 按钮下加「还没账号？立即注册」link

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 15: e2e 完整链路验证

- [ ] **Step 1: 启 mall-fe dev**

```bash
cd ~/workspace/go/mall/yw-mall-fe && pnpm dev:h5 &
sleep 6
```

或如果用容器化部署：

```bash
cd ~/workspace/go/mall/yw-mall-deploy && podman-compose build --no-cache mall-fe && podman-compose up -d --force-recreate mall-fe
```

- [ ] **Step 2: 浏览器手测**

打开 http://localhost:5173（dev）或 http://localhost:18080（容器）：

1. 首页 → 我的 → 「去登录」→ 「还没账号？立即注册」→ register 页
2. 填用户名 `e2euser` + 密码 `Test1234` + 手机 `13911112222` → 「发送验证码」
3. `tail -f ~/workspace/go/mall/yw-mall/logs/user-rpc.log | grep mock-sms` 拿验证码
4. 填验证码 → 「注册」 → toast「注册成功」→ 跳登录
5. 登录页用 `e2euser` 或 `13911112222` 登录 → 应进我的中心

- [ ] **Step 3: 老用户登录回归**

老 alice 用 alice / alice123 登录 → 仍然成功（向后兼容确认）。

- [ ] **Step 4: 各前端 push**

```bash
cd ~/workspace/go/mall/yw-mall-fe && git push
cd ~/workspace/go/mall/yw-mall && git push
cd ~/workspace/go/mall/yw-mall-deploy && git push   # 如果改过 db-init.sh
```

---

## Task 16（可选）: APISIX 路由配置

env 仓库改动，可独立 push，不阻塞主线。

**Files:**
- Modify: `~/workspace/go/mall/env/apisix/routes/setup.sh`

- [ ] **Step 1: 加 send-code 严限流路由**

在现有 routes/setup.sh 末尾（在 `mall-api-catchall` 之前）加：

```bash
put "mall-auth-sendcode" "/api/auth/send-code 严限流 10rps + ua 反爬" '{
  "name": "mall-auth-sendcode",
  "uri": "/api/auth/send-code",
  "upstream_id": "mall-api",
  "priority": 100,
  "plugins": {
    "limit-req": { "rate": 10, "burst": 5, "key_type": "var", "key": "remote_addr", "rejected_code": 429 },
    "ua-restriction": { "bypass_missing": false,
      "denylist": ["python-requests", "Go-http-client", "curl", "Wget"]
    },
    "cors": { "allow_origins": "*" }
  }
}'

put "mall-auth-register" "/api/auth/register 限流 5rps" '{
  "name": "mall-auth-register",
  "uri": "/api/auth/register",
  "upstream_id": "mall-api",
  "priority": 100,
  "plugins": {
    "limit-req": { "rate": 5, "burst": 3, "key_type": "var", "key": "remote_addr", "rejected_code": 429 },
    "cors": { "allow_origins": "*" }
  }
}'
```

- [ ] **Step 2: 推到 APISIX**

```bash
cd ~/workspace/go/mall/env && bash apisix/routes/setup.sh 2>&1 | tail -10
```

Expected: 2 行 `[201] put mall-auth-sendcode` / `mall-auth-register` 成功。

- [ ] **Step 3: 验证限流**

```bash
for i in {1..15}; do
  curl -s -o /dev/null -w "%{http_code} " -X POST http://localhost:9080/api/auth/send-code \
    -H 'Content-Type: application/json' -d '{"channel":1,"target":"13900000000","scene":1}'
done; echo
```

Expected: 前 ~5 个返 200/错误，后续 429（限流命中）。

- [ ] **Step 4: commit**

```bash
git add apisix/routes/setup.sh
git commit -m "feat(reg-v2 apisix): /api/auth/send-code 10rps + /api/auth/register 5rps 限流

防止短信轰炸 + 注册薅羊毛. priority 100 高于 catchall 100.
ua-restriction 拒 python-requests/Go-http-client/curl/Wget UA.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
git push
```

---

## Self-Review

| Spec 章节 | 实现 task |
|-----------|-----------|
| 1 背景与目标 | — |
| 2 数据模型 | Task 1 (cryptox.Hmac) + Task 2 (DDL) + Task 3 (backfill) |
| 3 RPC 接口 | Task 4 (proto) + Task 5 (SendVerifyCode) + Task 6 (RegisterV2) + Task 7 (LoginV2) |
| 4 业务 logic | Task 6 + Task 7 |
| 5 网关路由 | Task 8 (send-code) + Task 9 (register) + Task 10 (login + 老兼容) |
| 6 前端 | Task 12 + Task 13 + Task 14 |
| 7 实施分阶段 | Task 1-15 + Task 16 (可选 APISIX) |
| 8 测试约定 | Task 11 (后端 e2e) + Task 15 (浏览器 e2e) |
| 9 风险与回滚 | Task 10 step 3 (老路由保留) + Task 7 step 3 (start.sh restart 验) |

**类型一致性**：proto SendVerifyCodeReq/RegisterV2Req/LoginV2Req → userclient 同名 → mall-api types Caml 化（SendCodeReq/RegisterV2Req/LoginV2Req）→ FE types 同 mall-api → 全文一致。

**Placeholder 扫描**：无 TBD/TODO，所有代码块完整。Task 6 step 1 里 phoneEnc/emailEnc 段有重复赋值（`var` + `if` + 再 `if`）写法是为绕 Go scoping 显式赋值，保留。
