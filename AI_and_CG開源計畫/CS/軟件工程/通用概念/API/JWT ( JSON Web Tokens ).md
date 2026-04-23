#### JSON Web Tokens ( JWT ) 就是：

> 
> 「登入成功後，伺服器發給你的一張可驗證身份的通行證或者會員卡，每次使用 API 都帶著它」
>

前幾章提到， RESTful API 是以「**Stateless 無狀態**」原則，去跟伺服器溝通，這就像去餐廳吃飯，服務生跟廚師都不用認識我們，只需要透過 JWT 這張會員卡就可以點菜

所以說 JWT 的本質**不是加密**，而是：
「一套可驗證的身份資料」，是一種 **辨別身分的憑證格式**

---
#### 如何驗證
整個驗證的流程大致會是：
1. 伺服器端在收到登入請求後驗證使用者
2. 伺服器端產生和回傳一組僅能在伺服器端被驗證的 Token
3. Token 被回傳後，存取在「客戶端」

![[Pasted image 20260419105445.png]]



---



JWT 會給我們一段加密的字串，用來代表那張卡片的身分（簽章）

![[Pasted image 20260419110342.png]]

格式長這樣（三段）：

```
Header.Payload.Signature
```

### 三個部分意義

|部分|說明|
|---|---|
|Header|使用什麼演算法（例如 HS256）|
|Payload|使用者資料（userId、角色、過期時間）|
|Signature|防止被竄改（伺服器用密鑰簽章）|

---

# 登入時 API 做了什麼（重點）

前端送登入請求

```
POST /login
Content-Type: application/json

{
  "username": "admin",
  "password": "123456"
}
```

---

## 後端驗證帳密

API 會做：

1. 查資料庫
2. 比對密碼
3. 判斷是否存在


---

## 產生 JWT

如果登入成功，後端會產生一個 token：

Payload 通常包含：

```json
{
  "userId": 123,
  "role": "admin",
  "exp": 1712345678
}
```

然後用「伺服器密鑰」簽章 → 變成 JWT

---

## Step 4：回傳給前端

```
{
  "token": "eyJhbGciOiJIUzI1NiIs..."
}
```

---

## Step 5：前端保存 Token

通常會存：

- localStorage
- 或 memory（更安全）
- 或 cookie（HttpOnly）
---

# 三、登入之後 API 怎麼用 JWT

之後每次呼叫 API 都會帶這個：

```
GET /user/profile

Authorization: Bearer <JWT>
```

---

## 後端做的事（每一次 API）

每次請求，後端都會：

### 解析 JWT

從 Payload 拆出資訊：
例如

| userId | 這是誰         |
| ------ | ----------- |
| role   | 什麼權限        |
| exp    | 這張通行證什麼時候失效 |

---

### 驗證簽章

```
確認這個是不是我發的，不是偽造的
```

---

### 3. 檢查是否過期

```
該 token 是否過期
```

---

### 4. 通過 → 放行

```
讓 API 使用
```

