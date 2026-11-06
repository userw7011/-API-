仅适用于hddvip.com的独角数卡网对接API 。本md文件可直接发给ai 协助完成脚本
# 上游站点 API 对接教程（Dujiao-Next 串货协议 v1.0）

> 适用对象：希望自动采购本店商品的站长 / 开发者。
> 本店接口基址：`https://hddvip.com/api/v1/upstream`
> 协议版本：`1.0`　结算币种：`CNY`
> 本文所有示例均可直接运行（Python 3.8+）。

---

## 0. 它是怎么工作的

```
你的站(A)                          本店(B)
  │ 1. POST /ping 测试连通 + 查余额      │
  │ 2. GET  /products 拉商品/SKU        │
  │ 3. 用户在你站下单支付                 │
  │ 4. POST /orders 采购（扣钱包余额） ──▶ 自动发货/人工交付
  │ 5. B 回调你的 callback_url 推送交付 ◀──
  │ 6. GET  /orders/:id 轮询兜底         │
```

要点：
- **预充值模式**：采购从你的「API 对接钱包余额」扣款，余额不足会下单失败。先充值再对接。
- **自动交付**的商品（fulfillment_type=auto）秒级发货；**人工交付**（manual）需在采购时提交表单字段。
- 采购成功后**交付内容（卡密/账号）只在订单详情和回调里给**，请自行落库保存。

---

## 1. 准备

1. 在本店注册账号 → 前台「API 权限 / API 对接」→ 生成 **API Key** 和 **API Secret**。
   - Secret **只显示一次**，丢失只能重置。
   - 生成后需处于「已批准且启用」状态，账号状态正常。
2. 充值对接钱包余额（后台钱包 → 充值）。测试建议 2 元以上。
3. 确保你的服务器 **时间同步**（NTP）。签名时间戳允许偏差仅 ±60 秒。
4. 你的服务器需能访问 `https://hddvip.com`，并用 HTTPS。

---

## 2. 签名（90% 的失败都在这一步）

### 2.1 请求头

| Header | 值 |
|---|---|
| `Dujiao-Next-Api-Key` | 你的 API Key |
| `Dujiao-Next-Timestamp` | 当前 Unix 秒级时间戳（字符串） |
| `Dujiao-Next-Signature` | HMAC-SHA256 签名，hex 小写 |
| `Content-Type` | `application/json`（有 body 时） |

### 2.2 签名串

```
{METHOD}\n{PATH}\n{TIMESTAMP}\n{BODY_MD5}
```

- `METHOD`：大写，`GET` / `POST`
- `PATH`：**只到路径，不带域名、不带 `?查询参数`**
  - ✅ `/api/v1/upstream/products`
  - ❌ `/api/v1/upstream/products?page=1`
- `TIMESTAMP`：与请求头完全一致
- `BODY_MD5`：**实际发送字节**的 MD5（hex 小写）；无 body 时用空串的 MD5：
  `d41d8cd98f00b204e9800998ecf8427e`

签名 = `hex_lower(hmac_sha256(sign_string, api_secret))`

### 2.3 四个高频坑

| 坑 | 现象 | 解法 |
|---|---|---|
| 签名路径带了查询串 | `invalid_signature` | 签名用干净路径，查询参数只放 URL |
| 签名字节 ≠ 发送字节 | `invalid_signature` | 先 `body = json.dumps(...).encode()`，MD5 和发送用同一个变量 |
| 服务器时间偏差 >60s | `timestamp_expired` | 开 NTP：`timedatectl set-ntp true` |
| 请求 UA 被 Cloudflare 拦 | HTTP 403，纯文本 `error code: 1010` | UA 换成浏览器 UA（例：`Mozilla/5.0 ... Chrome/155.0.0.0 Safari/537.36`），或联系本店将该路径加入 CF 白名单 |

### 2.4 签名函数（照着用）

```python
import hashlib, hmac, time

EMPTY_MD5 = hashlib.md5(b"").hexdigest()  # d41d8cd98f00b204e9800998ecf8427e

def sign(api_secret: str, method: str, path: str, body: bytes = b"") -> tuple[str, str]:
    ts = str(int(time.time()))
    body_md5 = hashlib.md5(body).hexdigest() if body else EMPTY_MD5
    s = f"{method.upper()}\n{path}\n{ts}\n{body_md5}"
    sig = hmac.new(api_secret.encode(), s.encode(), hashlib.sha256).hexdigest()
    return ts, sig
```

---

## 3. 接口速查

Base：`https://hddvip.com/api/v1/upstream`

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/ping` | 连通性 + 余额 + 会员等级 |
| GET | `/categories` | 分类列表 |
| GET | `/products?page=1&page_size=20` | 商品列表（仅上架） |
| GET | `/products/:id` | 商品详情（含 SKU） |
| POST | `/orders` | 创建采购单（扣余额） |
| GET | `/orders/:id` | 查询采购单（含交付内容） |
| POST | `/orders/:id/cancel` | 取消（仅未交付可取消） |

统一响应：成功 `{"ok":true,...}`；失败 `{"ok":false,"error_code":"...","error_message":"..."}`。

### 3.1 关键字段

**商品 → SKU**：下单必须用 `skus[].id`（不是 `product.id`）。
`skus[].stock_status`：`unlimited / in_stock / low_stock / out_of_stock`。

**创建订单 body**：

```json
{
  "sku_id": 1,
  "quantity": 1,
  "downstream_order_no": "你的唯一单号",
  "manual_form_data": {},
  "callback_url": "https://你的域名/api/upstream/callback"
}
```

- `downstream_order_no`：**幂等键**，同一个 Key + 同单号只创建一次，重复提交返回已有订单。
- `callback_url`：公网可访问的 HTTPS 地址，**不允许内网/localhost**；不传则只能靠轮询。
- `manual_form_data`：人工交付商品必填，键为 `product_id`，值为表单字段对象。

**订单状态**：`pending_payment` → `paid` → `fulfilling` → `partially_delivered` → `delivered` → `completed`；`canceled`。

**交付**（`GET /orders/:id` 的 `fulfillment`，仅 delivered 后出现）：

```json
{
  "type": "auto",
  "status": "delivered",
  "payload": "账号----密码----2FA----Token----...",
  "delivery_data": null,
  "delivered_at": "2026-..."
}
```

---

## 4. 完整下单示例（Python，可直接运行）

```python
import hashlib, hmac, json, time, urllib.request, urllib.error

BASE   = "https://hddvip.com"
KEY    = "你的 API Key"
SECRET = "你的 API Secret"
UA     = ("Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 "
          "(KHTML, like Gecko) Chrome/155.0.0.0 Safari/537.36")
EMPTY_MD5 = hashlib.md5(b"").hexdigest()

def call(method, path, body=b"", sign_path=None):
    ts = str(int(time.time()))
    md5 = hashlib.md5(body).hexdigest() if body else EMPTY_MD5
    s = f"{method.upper()}\n{sign_path or path}\n{ts}\n{md5}"
    sig = hmac.new(SECRET.encode(), s.encode(), hashlib.sha256).hexdigest()
    req = urllib.request.Request(
        BASE + path, data=(body if method == "POST" else None), method=method,
        headers={
            "Dujiao-Next-Api-Key": KEY,
            "Dujiao-Next-Timestamp": ts,
            "Dujiao-Next-Signature": sig,
            "Content-Type": "application/json",
            "User-Agent": UA,
            "Accept": "application/json, text/plain, */*",
        })
    try:
        with urllib.request.urlopen(req, timeout=20) as r:
            return json.loads(r.read().decode())
    except urllib.error.HTTPError as e:
        return json.loads(e.read().decode())

# 1) 连通性 + 余额
print(call("POST", "/api/v1/upstream/ping"))

# 2) 拉商品（签名路径不带查询串！）
print(call("GET", "/api/v1/upstream/products?page=1&page_size=5",
           sign_path="/api/v1/upstream/products"))

# 3) 下单
body = json.dumps({
    "sku_id": 1,
    "quantity": 1,
    "downstream_order_no": "A" + str(int(time.time())),
    "callback_url": "https://你的域名/api/upstream/callback",
}, separators=(",", ":")).encode()
r = call("POST", "/api/v1/upstream/orders", body)
print(r)

# 4) 轮询取交付
if r.get("ok"):
    oid = r["order_id"]
    for _ in range(20):
        d = call("GET", f"/api/v1/upstream/orders/{oid}")
        print(d.get("status"))
        if d.get("fulfillment"):
            print(d["fulfillment"]["payload"])   # 落库保存，别丢
            break
        time.sleep(3)
```

---

## 5. 回调接入（推荐，必需）

本店在「状态变更 / 交付完成」时向你的 `callback_url` 发 `POST`，**鉴权方式与上面相同**（同一套签名算法，用你的 Key/Secret 校验）。

**回调 body**：

```json
{
  "event": "order.delivered",
  "order_id": 379,
  "order_no": "DJ...",
  "downstream_order_no": "你的单号",
  "status": "delivered",
  "fulfillment": { "type": "auto", "status": "delivered", "payload": "..." },
  "timestamp": 1792127970
}
```

**你必须做的**：

1. 校验签名（用收到的原始 body 字节算 MD5，不能用重新序列化后的 body）。
2. 按 `downstream_order_no` 找到本地订单，幂等更新（重复回调只处理一次）。
3. 返回 `{"ok": true, "message": "received"}`，否则本店视为失败。

**Flask 最小示例**：

```python
from flask import Flask, request, jsonify
import hashlib, hmac

app = Flask(__name__)
SECRET = "你的 API Secret"

@app.post("/api/upstream/callback")
def cb():
    raw = request.get_data()                       # 原始字节，别用 request.json 重序列化
    ts  = request.headers.get("Dujiao-Next-Timestamp", "")
    sig = request.headers.get("Dujiao-Next-Signature", "")
    body_md5 = hashlib.md5(raw).hexdigest()
    expect = hmac.new(SECRET.encode(),
                      f"POST\n/api/upstream/callback\n{ts}\n{body_md5}".encode(),
                      hashlib.sha256).hexdigest()
    if not hmac.compare_digest(expect, sig):
        return jsonify(ok=False, message="bad signature"), 401
    data = request.get_json(force=True)
    # TODO: 按 downstream_order_no 幂等更新本地订单并保存 payload
    return jsonify(ok=True, message="received")
```

> 注意：校验签名时，签名串里的 `PATH` 用**你自己的回调路径**（即 `callback_url` 的路径部分，如 `/api/upstream/callback`），方法为 `POST`，body 用收到的原始字节。联调时先把 `sign_string` 打印出来比对一次，一次就能对齐。

---

## 6. 错误码

| error_code | HTTP | 含义 / 处理 |
|---|---|---|
| `missing_auth_headers` | 401 | 缺签名头 |
| `invalid_timestamp` / `timestamp_expired` | 401 | 时间格式错 / 偏差超 60s |
| `invalid_signature` | 401 | 签名不匹配（见 2.3） |
| `invalid_api_key` | 401 | Key 无效/禁用 |
| `user_disabled` | 403 | 账号被封 |
| `bad_request` | 400 | 参数错 |
| `invalid_callback_url` | 400 | 回调地址非法（内网/localhost） |
| `sku_unavailable` / `product_unavailable` | 400 | SKU/商品不可用 |
| `insufficient_balance` | 402 | 钱包余额不足 → 充值 |
| `insufficient_stock` | 409 | 库存不足 |
| `payment_failed` | 200 | 扣款失败，订单已自动取消 |
| `cancel_not_allowed` | 409 | 已支付/已交付不可取消 |
| `internal_error` | 500 | 服务端错误，指数退避重试 |

---

## 7. 最佳实践

- **幂等**：`downstream_order_no` 用你自己的订单号，重试不会重复采购。
- **重试**：网络错误/5xx 用指数退避（1s→2s→4s），4xx 不要重试。
- **双保险**：回调 + 定时轮询 `GET /orders/:id`（回调可能因网络失败）。
- **落库**：交付 `payload` 到手立刻存库，别只在内存里。
- **库存前置**：下单前看 `stock_status`，售罄别让用户付钱。
- **余额监控**：定期 `ping` 查余额，低于阈值自动提醒充值。
- **取消规则**：已支付/已交付不能取消；本地退款前先调上游取消，上游拒绝则拒绝退款。

---

## 8. 联调检查清单

1. `POST /ping` 返回 `ok:true` 且余额 > 0
2. `GET /products` 能拿到 SKU ID
3. 下 1 单最小金额，状态 `paid`
4. 3~10 秒内变 `completed`，`fulfillment.payload` 有内容
5. 回调地址收到一次 POST 且验签通过（或轮询能取到）
6. 同一 `downstream_order_no` 重复提交，返回同一 `order_id`（幂等成立）

六项全过即对接完成。有报错优先看 §2.3 和 §6。
