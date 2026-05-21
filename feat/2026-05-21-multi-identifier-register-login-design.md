# 多 identifier 注册 / 登录 + PII 加密

> 状态：设计已批准，待实施
> 日期：2026-05-21
> 涉及 repo：yw-mall (mall-common / mall-user-rpc / mall-api) · yw-mall-fe
> 工作量预估：后端 1.5d + 前端 0.5d + 验证 0.5d，共 ~2.5 人天

---

## 1. 背景与目标

### 现状

- 注册：只支持 `username + password + phone`（phone 字段非 unique、不能登录）
- 登录：只支持 `username + password` 一种方式
- `user` 表：`username` UNIQUE，`phone` 明文非 unique，**无 email 字段**
- S4 已引入 `mall-common/cryptox` 包（AES-256-GCM 字段加解密，`v1:` 前缀双读），但 user 表尚未使用
- S4.1 已有 Mock SMS（写 logx 的 `[mock-sms] code=...`）+ challenge_token Redis 握手范式

### 目标

1. 注册时 username 必填 + phone/email 至少一个；填了 phone 或 email 必须通过验证码握手
2. 登录支持 `username | phone | email` 三种 identifier，前端单字段、后端 regex 自动识别
3. phone / email 列级 AES 加密 + HMAC blind index，满足 PIPL 数据安全要求

### 非目标

- 不接真实 SMS / SMTP 服务（mock 同 S4.1 路子，写 logx）
- 不动 admin SPA（admin 不开放注册）
- 不做"忘记密码"流程（独立 sprint）

---

## 2. 数据模型

### DDL 增量（`mall-user-rpc/sql/user_multi_id_v2.sql`）

```sql
USE mall_user;

ALTER TABLE `user`
  ADD COLUMN `email`      VARCHAR(128) NOT NULL DEFAULT '',   -- 过渡期保留明文显示用
  ADD COLUMN `phone_enc`  VARCHAR(512) NOT NULL DEFAULT '',   -- AES-256-GCM(phone), cryptox v1:
  ADD COLUMN `email_enc`  VARCHAR(512) NOT NULL DEFAULT '',
  ADD COLUMN `phone_hash` CHAR(64) NOT NULL DEFAULT '',       -- HMAC-SHA256(phone) hex (64 char)
  ADD COLUMN `email_hash` CHAR(64) NOT NULL DEFAULT '';

-- 空串允许多行（业务侧：未绑定 phone/email 的用户 hash 为空）
-- MySQL 9.6 不支持 partial unique index，靠业务侧约束 + 应用层校验
-- 改为普通 index 用于查询，应用层在 register/bind 时显式 SELECT 一次确认未占用
ALTER TABLE `user`
  ADD INDEX `idx_phone_hash` (`phone_hash`),
  ADD INDEX `idx_email_hash` (`email_hash`);
```

**说明**：
- 不上 UNIQUE 因为空串重复合法。改用普通索引 + 应用层在 INSERT 前 SELECT 锁
- 老 `phone` 明文列保留过渡期不读不写，下个 sprint 删
- `user_address.phone` 是独立字段，不在本次范围

### Backfill（`mall-user-rpc/cmd/backfill_pii/main.go`）

```go
// 1. SELECT id, phone FROM user WHERE phone != '' AND phone_enc = '';
// 2. for each: enc = cryptox.Encrypt(phone); hash = cryptox.Hmac(phone)
// 3. UPDATE user SET phone_enc=?, phone_hash=? WHERE id=?
// 4. 完成后 LOG total/migrated/skipped 计数
```

幂等：用 `phone_enc = ''` 作为「未迁移」标记。

### cryptox 扩展

```go
// mall-common/cryptox/cryptox.go 增量
func Hmac(plain string) string {
    if plain == "" {
        return ""   // 空串特殊：避免给所有空 phone 用户造一个相同 hash
    }
    mac := hmac.New(sha256.New, hmacKey)
    mac.Write([]byte(plain))
    return hex.EncodeToString(mac.Sum(nil))
}

// hmacKey: MALL_FIELD_ENCRYPTION_KEY 前 32 字节，跟 AES key 共享 — key rotation 时一起换
```

---

## 3. RPC 接口

### proto 增量（`mall-common/proto/user/user.proto`）

```protobuf
// === 新增 RPC ===
rpc SendVerifyCode(SendVerifyCodeReq) returns (SendVerifyCodeResp);
rpc RegisterV2(RegisterV2Req) returns (RegisterResp);
rpc LoginV2(LoginV2Req) returns (LoginResp);

message SendVerifyCodeReq {
  int32  channel = 1;   // 1=sms, 2=email
  string target  = 2;   // 手机号 / 邮箱地址
  int32  scene   = 3;   // 1=register, 2=login, 3=bind, 4=change-password (本次只用 1)
}
message SendVerifyCodeResp {
  string challenge_token = 1;   // 32 字节 base64url
  int32  expires_in      = 2;   // 300 (秒)
}

message RegisterV2Req {
  string username        = 1;
  string password        = 2;
  string phone           = 3;   // optional, 至少 phone/email 一个非空
  string email           = 4;
  string verify_code     = 5;   // 6 位数字
  string challenge_token = 6;   // 来自 SendVerifyCode
}

message LoginV2Req {
  string account  = 1;   // username / phone / email 三选一
  string password = 2;
}
```

老 `Register` / `Login` RPC 保留不动（mall-api 层做兼容转发）。

### 验证码握手（仿 S4.1 MFA challenge_token）

**Redis key**：`verify:{scene}:{target}` → `{"code":"482919","token":"abc...","sent_at":1716268800}`，TTL 300s。

**限频规则**（Redis）：
- 同 `target` 60 秒内不能重复发：检查 `sent_at` 差值
- 同 `IP` 1 小时 10 次：复用 S4.2 `login_fail:` 命名空间逻辑（新 scope `send_code`）

**消费**：register 时 logic 先 `GET verify:1:{target}`，比对 code + token，匹配则 DEL（单次消费）。

**Mock 实现**：
- channel=sms → 复用 S4.1 `SmsSend(ctx, target, code)` 但参数从 admin_id 改 target
- channel=email → 新 `EmailSend(ctx, target, code)`：`logx.Infof("[mock-email] target=%s code=%s", target, code)`，结构同 SMS

---

## 4. 业务 logic

### user-rpc RegisterV2 流程

```
1. 校验 username 唯一（SELECT WHERE username=?）
2. 校验 password 强度（复用 S4.3 PasswordPolicy validate）
3. 至少 phone/email 一个非空 — 都空 → error
4. 对每个非空 identifier:
   a. SELECT 检查 hash 未占用 (idx_phone_hash / idx_email_hash)
   b. 从 Redis 弹 verify_code，校验 + challenge_token 比对，单次消费
5. 计算 phone_enc / phone_hash / email_enc / email_hash
6. bcrypt(password)
7. INSERT INTO user (username, password, email, phone_enc, email_enc, phone_hash, email_hash, last_password_change, create_time, ...)
   保留 phone 明文列空串（过渡期不写）
8. 返 RegisterResp{id}
```

### user-rpc LoginV2 流程

```
1. account 字段 regex 分类：
   - emailRE  = `^[^\s@]+@[^\s@]+\.[^\s@]+$`
   - phoneRE  = `^\d{11}$` (中国手机号；可后续扩展国际)
2. 按类型查询：
   - email   → hash := cryptox.Hmac(strings.ToLower(account)); SELECT WHERE email_hash=?
   - phone   → hash := cryptox.Hmac(account); SELECT WHERE phone_hash=?
   - else    → SELECT WHERE username=?
3. 用户不存在 OR bcrypt 不匹配 → 统一返 "用户名或密码错误"（防枚举）
4. 复用 S4.2 失败锁定逻辑 (MarkLoginFail / ClearLoginFail，scope=user)
5. 复用 P0 CreateSession 创建 opaque token + refresh + csrf
6. 返 LoginResp（含 S4.3 password_expired）
```

---

## 5. 网关路由（mall-api）

| 路由 | 方法 | logic | 备注 |
|------|------|-------|------|
| `/api/auth/send-code` | POST | 新 SendCodeLogic | 不需要 auth |
| `/api/auth/register` | POST | 新 RegisterV2Logic（调 user-rpc.RegisterV2）| 取代 `/api/user/register` 的新功能位置 |
| `/api/user/register` | POST | 兼容层：未带 verify_code 返 `verify_code required` | 不删除，老 FE 调时给清晰错误 |
| `/api/auth/login` | POST | **改写**：调 user-rpc.LoginV2 | 当前实现只支持 username，改为单 identifier |

**APISIX 路由**：现有 `/api/auth/*` 已被 mall-user catchall 覆盖，需在 `env/apisix/routes/setup.sh` 显式加 `/api/auth/send-code` 路由配较严限流（10rps + ua-restriction）。但这是 env 仓库改动，可分批做（不阻塞主线）。

---

## 6. 前端（mall-fe）

### 新页 `pages/login/register.vue`

```
[用户名 wd-input]
[密码 wd-input password]
[确认密码 wd-input password]

类型 [手机号 ▼] (radio)
   ↓
[手机号 wd-input + 发送验证码 button]
[6 位验证码 wd-input maxlength=6]

[注册 wd-button primary]
```

- 「发送验证码」点后变 「60s 后重发」（倒计时）
- mock 邮件/短信时 toast 提示「验证码已发送（开发模式：查看 user-rpc logx）」

### 改 `pages/login/index.vue`

- 单输入框 placeholder 从「用户名」改「用户名 / 手机号 / 邮箱」
- 底部加「还没账号？立即注册」link → 跳 register

### api 增量

```ts
// src/api/auth.ts (新文件)
export function sendVerifyCode(channel: 1|2, target: string, scene: number)
export function registerV2(req: RegisterReq)
export function loginV2(account: string, password: string)  // 旧 login() 别名换内部 path
```

### pages.json 注册新页

```json
{ "path": "pages/login/register", "style": {"navigationBarTitleText":"注册"} }
```

---

## 7. 实施分阶段

| 阶段 | 内容 | 文件数 | 工作量 |
|------|------|--------|--------|
| 1 | cryptox.Hmac + DDL `user_multi_id_v2.sql` + start.sh/db-init 注册迁移 + backfill 脚本 | 4 | 0.25d |
| 2 | proto 加 SendVerifyCode/RegisterV2/LoginV2 + 重生成 pb.go + userclient | 3 | 0.25d |
| 3 | user-rpc 3 个新 logic + sendcodehelpers.go (Mock email) | 5 | 0.5d |
| 4 | mall-api 兼容层 + 3 个新 handler + LoginV2 切换 | 5 | 0.25d |
| 5 | mall-fe register.vue + login.vue 改 + api/auth.ts | 3 | 0.5d |
| 6 | e2e 验证 + commit/push 3 仓库 | — | 0.5d |
| 7 | env/apisix/routes/setup.sh 加 send-code 路由 + commit | 1 | 0.25d |

---

## 8. 测试约定

**最小 e2e**（手工）：

1. `curl POST /api/auth/send-code {channel:1,target:"13800000001",scene:1}` → 拿 `challenge_token`，user-rpc logx 看到 `[mock-sms] target=13800000001 code=482919`
2. `curl POST /api/auth/register {username:"newuser",password:"Test1234!",phone:"13800000001",verify_code:"482919",challenge_token:"..."}` → 拿到 user id
3. `curl POST /api/auth/login {account:"newuser",password:"Test1234!"}` → 200
4. `curl POST /api/auth/login {account:"13800000001",password:"Test1234!"}` → 200（phone 登录）
5. `curl POST /api/auth/login {account:"wrong",password:"x"}` → 401 message="用户名或密码错误"（防枚举验证）
6. 同 phone 重复 register → 应 fail "手机号已注册"
7. 老用户 alice 用 username 登录 → 仍 200（向后兼容验证）

**回归项**：S4 全部端点（MFA / IP / KYC / OpLog / 改密）+ 9 个原 e2e 端点。

---

## 9. 风险与回滚

| 风险 | 缓解 |
|------|------|
| backfill 跑挂导致老 alice 登录失败 | backfill 独立脚本 + 完成后 phone_hash != '' 才算迁移；alice 用 username 登录不依赖 phone_hash |
| LoginV2 实现错误把 alice 锁出 | 旧 Login RPC 不删，mall-api `/api/user/login` 仍可用，FE 出问题切回 |
| 邮件 mock 上线被误用 | logx 打 `[mock-email]` 前缀；监控告警关键字过滤；生产配 env `MOCK_EMAIL_ENABLED=false` 强制失败避免误发 |
| HMAC key 跟 AES key 共享 = key rotation 同步成本 | 接受。Sprint 5/6 上 KMS 时再分离 |
| 防枚举返回模糊错误增加用户体验摩擦 | 接受。安全 > UX；前端可对常见输入做 client-side hint |

**回滚**：
- 后端：revert RegisterV2/LoginV2 logic 即可，DDL 加列不影响老代码
- 数据：phone/email 明文 + 加密双写过渡期，删 enc/hash 列不影响业务

---

## 10. 与现有功能的关系

- **复用** S4.2 失败锁定（MarkLoginFail / ClearLoginFail，scope="user"）
- **复用** S4.3 PasswordPolicy（强度校验）
- **复用** S4.1 SmsSend 模式实现 EmailSend
- **复用** P0 CreateSession（opaque token）
- **不影响** admin 登录链路（admin 仍走 `/admin/v1/login` + S4.1 MFA）
- **预留** L1.1 短信验证码登录（本次的 SendVerifyCode 已支持 scene=2，logic 接 LoginV3 时直接复用）

---

## 11. 决策记录

| 决策 | 选择 | 否决项 |
|------|------|--------|
| 注册流程 | 二步 challenge token 握手 | 一步同接口 / 三步分接口 |
| 登录端点 | 单端点 regex 自动识别 | type 字段 / 三个端点 |
| PII 存储 | AES + HMAC blind index | 明文 unique / 只加密 email |
| 邮件验证 | 6 位验证码（同短信 UI 模型）| 邮件激活链接（需真 SMTP） |
| 验证码发送 | Mock logx（复用 S4.1 模式）| 接 SES/SendGrid |
| Backfill | 独立脚本 + 手动执行 | 懒加载在 login 时迁 |
| 兼容老 `/api/user/register` | 保留路由返清晰错误 | 直接删 |
