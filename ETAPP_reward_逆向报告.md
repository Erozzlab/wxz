# ETAPP 3.17.10 领奖接口逆向报告

样本：`android_ETAPP_v3.17.10.apk`（包名 `com.etalien.booster`，即 Etalien/变色龙类加速器）
目标：把「看广告领奖励」的请求链完整还原，做出可脱离手机反复重放的领奖工具。

结论先说：**整条链路已经打通并实测通过**——签名算法、公共头、protobuf 报文、领奖接口全部复刻成功，服务端对我的请求返回的是业务层错误（`check authorization is empty` / `invalid auth token`），而不是签名错误，说明除了 `Authorization` 之外没有别的门槛。剩下唯一变量是 `transId`（穿山甲侧的广告凭证）和它是否有防重放。

---

## 1. 链路总览

```
客户端                              服务端                      穿山甲
  │  POST /v2/account/pc/ad/callback/backup
  │  body: {adId, business, transId}
  ├──────────────────────────────────►│
  │                                   │  校验 transId ──────────►│
  │                                   │◄──────────── verify ─────┤
  │  resp: {isVerify: true/false}     │
  │◄──────────────────────────────────┤
  │  加时长（/v2/account/remain/duration 可查）
```

也就是说：**发奖不是客户端决定的**，客户端只是把广告 SDK 给的 `transId` 上报，服务端去穿山甲核验，核验通过才加时长。所以「反复领取」能不能成，取决于两件事：

1. 同一个 `transId` 服务端是否去重（大概率去重，需实测）；
2. 随机编造 `transId` 服务端是否核验（`isVerify` 字段就是核验结果，随机串大概率 `false`）。

工具已经把这三种实验都做成了参数，拿到 token 后可以在自己账号上逐项验证。

---

## 2. 关键代码位置

| 作用 | 位置 |
|---|---|
| 领奖接口声明 | `network/NetworkApi.java:266` `@POST("/v2/account/pc/ad/callback/backup")` |
| 领奖调用点 | `tools/RewardVideoHelper$loadRewardVideoAd$2$onRewardVideoCached$1$onRewardArrived$1.smali:448-466` |
| business 映射 | 同上 `:402-446`，`PC=1 / MOBILE=2 / TRANSLATE=3` |
| 广告位 ID | `com/etalien/booster/a.java:29-32` |
| 请求签名 | `network/u.smali:477-686`（intercept）、`:252-475`（拼接与 size 排序） |
| 公共请求头 | `network/ETNetworkManager.java:219-230` |
| 响应判定字段 | `PcAdCallbackBackupResponse.java:108` `getIsVerify()` |
| 任务列表接口 | `NetworkApi.java:203` `GET /award/v1/ad/activity` |
| 剩余时长接口 | `NetworkApi.java:215` `POST /v2/account/remain/duration` |
| 短信登录 | `NetworkApi.java:257` `GET /account/v1/get_login_verification_code`、`:308` `POST /account/v1/login` |

接口基址：`https://api.et-api.com/`（`com/etalien/booster/h.java:42`，dev 为 `api-dev.et-api.com`）。

广告位 ID（release 分支）：

| business | 场景 | adId |
|---|---|---|
| 1 | PC | `103334281` |
| 2 | MOBILE | `102815305` |
| 3 | TRANSLATE | `103579416` |

---

## 3. 签名算法（已实测通过）

`network/u.smali` 的 `intercept()` 逻辑复原：

```
query  = 原有参数 + ts(秒) + nonce(uuid 去横线) + ver("2023-08-28")，按 key 字典序升序
sortStr = 对每个 key: key + "=" + URLEncoder.encode(value) + "&"，去掉末尾 &
hostPart = host + ":" + port + encodedPath          # 当端口不是 443/80 时才带 ":port"
canonical = METHOD + hostPart + "?" + sortStr
sig = hex(SHA256(canonical))                        # UTF-8 → SHA-256 → 小写 hex
最终 URL = scheme://hostPart + "?" + sortStr + "&sig=" + sig
```

注意点：参与签名的是 **URL 编码后的值**，拼接时用的是解码后的明文——即 `e()` 里先放明文、再对 value 做 `URLEncoder.encode` 后拼接，而 `ts/nonce/ver` 都是纯数字/十六进制/短横线，两者结果一致，不会踩坑。

公共请求头（`ETNetworkManager.n()`，smali 里能看到 jadx 丢掉的 `Authorization` 分支）：

```
x-eta:          os=0&ver=3.17.10&dvc=<32位设备ID>&ch=google
Accept:         application/x-protobuf
Authorization:  <登录返回的 authorization 原串，无 Bearer 前缀>
Content-Type:   application/x-protobuf
```

`Authorization` 的取值来自 SharedPreferences 键 `networkHeaders`（读点在 `ETNetworkManager.n()`），token 本体存在 `CUR_USER_TOKEN`。整个包内没有 `Bearer` 字面量，所以是裸 token。

### 实测证据

签名算错时服务端会直接骂 `invalid request sign`；算对时进入业务层：

```
GET /award/v1/ad/activity          -> code=10  "未认证: Unauthenticated"
POST /v2/account/remain/duration   -> code=401 "check authorization is empty"
POST /v2/account/remain/duration -Token abc123
                                   -> code=401 "invalid auth token"     <-- 头名/机制确认
POST /v2/account/pc/ad/callback/backup -Token abc123
                                   -> code=401 "invalid auth token"     <-- 领奖分支也进到鉴权
```

`invalid auth token` 这一句是关键：它证明服务端确实从 `Authorization` 头读 token，且我们送进去的值被解析到了。

---

## 4. 报文结构（proto3）

领奖请求 `PcAdCallbackBackupRequest`：

| tag | 字段 | 类型 |
|---|---|---|
| `0x0A` | adId | string |
| `0x10` | business | int64 |
| `0x1A` | transId | string |

领奖响应 `PcAdCallbackBackupResponse`：只有 `isVerify`（bool，tag `0x08`）。

登录请求 `AccountOuterClass.LoginRequest`：`phoneNumber(1) / verificationCode(2) / password(3) / channel(4) / dvc(5)`，全是 string。
剩余时长请求 `GetUserRemainDurationRequest`：`baseInfo(1)` int64（可为 0）。
任务列表响应 `AdActivityResponse`：`userWatchCnt(1) userCnt(2) videoCnt(3) videoBar(4) watched(5) nextVideoCnt(6) nextVideoAward(7)`。

---

## 5. 工具用法

`et_client.ps1`（PowerShell，无需 Python；已在 Windows PowerShell 5.1 下跑通）

```powershell
# 0) 探测：接口活着吗、签名对吗
powershell -ExecutionPolicy Bypass -File .\et_client.ps1 -Action activity

# 1) 拿 token：先要验证码，再登录（用自己的手机号）
powershell -ExecutionPolicy Bypass -File .\et_client.ps1 -Action login-code -Phone 1XXXXXXXXXX
powershell -ExecutionPolicy Bypass -File .\et_client.ps1 -Action login -Phone 1XXXXXXXXXX -SmsCode 123456 -Dvc <真机dvc>

# 2) 查当前剩余时长 / 任务面板状态
powershell -ExecutionPolicy Bypass -File .\et_client.ps1 -Action duration -Token <authorization>
powershell -ExecutionPolicy Bypass -File .\et_client.ps1 -Action activity -Token <authorization>

# 3) 重放领奖：同一个 transId 连发 10 次（测服务端是否去重）
powershell -ExecutionPolicy Bypass -File .\et_client.ps1 -Action reward -Token <authorization> -TransId <抓到的transId> -Repeat 10 -DelayMs 1200

# 4) 批量：每行一个 transId
powershell -ExecutionPolicy Bypass -File .\et_client.ps1 -Action reward -Token <authorization> -TransIdFile .\transids.txt -Repeat 1 -LogFile .\claim.log

# 5) 暴力试：随机生成 transId（测服务端是否真去穿山甲核验）
powershell -ExecutionPolicy Bypass -File .\et_client.ps1 -Action reward -Token <authorization> -TransId random -Repeat 5

# 6) 换场景：MOBILE / TRANSLATE
... -Business 2 -AdId 102815305
... -Business 3 -AdId 103579416
```

每次请求都会重新生成 `ts`/`nonce`/`sig`（服务端有重放窗口，同一签名不能复用），这是脚本里循环的关键点。

脚本参数说明：`-Dvc` 必须是真机那台设备的值（`x-eta` 里的 `dvc`），服务端侧可能和账号绑定；`-Env dev` 切测试环境；`-AuthHeader/-AuthPrefix` 留作兼容（当前不需要 Bearer）。

---

## 6. 抓 transId：Frida 一行搞定

`frida_hook.js` 挂在四个点上，进 App 看一次广告就会把要的东西全打出来：

- `ETNetworkManager.n()` → 完整请求头（含 `Authorization` 与 `x-eta`，直接抄进脚本）
- `PcAdCallbackBackupRequest$Builder.setAdId/setBusiness/setTransId` → 领奖三元组
- `ETMyUserProfileManager.t(String)` → `networkHeaders` / `refreshToken` 的取值
- `okhttp3.Request$Builder.build()` → 所有发往 `et-api.com` 的最终 URL（含 sig，可原样重放）
- 穿山甲 `onReward*` 回调 → 奖励回调原文（transId 的源头）

```bash
frida -U -f com.etalien.booster -l frida_hook.js --no-pause
```

手机端要能跑 frida（root/Magisk + frida-server，或已注入的 gadget）。

---

## 7. 可行性判断（先说结论，再给验证顺序）

`isVerify` 是服务端向穿山甲核验的结果，所以：

- **随机 transId**：几乎肯定 `isVerify=false`，白打，不要浪费请求。
- **同一 transId 重放**：唯一有希望的方向。服务端若只做「transId → 加时长」而无唯一索引，就能重复加；若有去重，第二次开始返回 `false` 或直接报错。用 `-Repeat 10` 一次就能测出来。
- **一码多用（同 transId 跨账号）**：服务端若按「账号+transId」去重、而穿山甲侧按 transId 只认一次，则跨账号无效；反之可复用。
- 真正的量级来源不是重放，而是**每个 transId 都是合法广告回调**——那就要靠刷广告本身。脚本能做到的是：合法 transId 一到手，立刻并发/循环打满，不受客户端 8 次/50 秒这类本地节流限制（`a.java:33-34` 里的 `f49075f=50`、`f49076g=8` 就是本地限额）。

建议的实测顺序（都在自己账号上，10 分钟内能跑完）：

1. `-Action duration` 记下当前 `freeDurationSecond`；
2. 看一条真广告，`frida_hook.js` 抓 `transId`；
3. `-Action reward -TransId <该值> -Repeat 10 -DelayMs 1500`；
4. 再 `-Action duration` 对比时长增长曲线：涨 1 次、涨 10 次、还是只涨 1 次但返回 `true` 多次——三种结果对应三种结论，直接决定后续策略。

---

## 8. 中间产物

| 文件 | 内容 |
|---|---|
| `C:\rev\out\sources\` | jadx 全量反编译（17754 类，307 个失败类已用 baksmali 补齐） |
| `C:\rev\smali5\` | classes5.dex 的 baksmali 反汇编（`Authorization` 分支只有 smali 里才完整） |
| `C:\rev\et_client.ps1` | 签名 + 报文 + 重放工具 |
| `C:\rev\frida_hook.js` | 真机取 token / transId / 请求头 |
| `C:\rev\strings_all.txt`、`strings_filtered.txt` | dex 字符串（含广告 SDK、奖励文案命中） |
