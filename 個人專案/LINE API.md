- **AI 圖片辨識**  拍照識別 → 上傳文字 → 比對商品 → 庫存異動
- **滯銷觸發條件**：時間到貨物沒賣掉 + 庫存超閾值，兩條件獨立跑
- **Multicast 推播**：依 VIP / 區域分組

---

## 認證方式

1. Webhook 路由用 `X-Line-Signature` HMAC 驗證
	負責：
	- 監聽使用者行為，只要與官方帳號產生互動
	- 接收資料、驗證簽章（HMAC）

使用場景：
	 1. 用戶將 OCR 後的文字訊息上傳，LINE 發現有新訊息時，主動把資料推送到我們的 Server
	 2. 需要創建金流服務時，可以透過介面申請一個專屬的進入點，告訴我們的 Server 往該用戶手上建立一個臨時的 **Partner ID** 和 **Merchant ID**

 2. `/api/*` 請求頭帶 LINE Login 的 Access Token  
	 負責：
	 - 後端推送消息時主動發送一個 POST 請求到 LINE 的官方服務 

```http
https://api.line.me/v2/bot/message/reply
```

使用場景：
	1. 收到出庫單的查閱請求時，伺服器主動發送請求給 **LINE 的 API 服務器**，叫它傳送 文字 或 文件 給使用者。
	2. 用戶發送付款請求時，多推一臨時證明文件 ( 專屬匯款識別碼 )給前端，後端做二次解碼驗證

```HTTP
POST /api/user/login HTTP/1.1 

Content-Type: application/json
Authorization: Bearer {line_access_token}
```

3. `/api/*` 請求頭帶自己的 Token 
	 負責：
	 - 對外的 API

使用場景：
	1. 可以先綁 LINE 需求具體了再做就好
	2. 之後要賣 API 給外面 的時候改


## 庫存服務 

![[Pasted image 20260508153330.png]]


## 入庫流程

### **入庫前：**
- 貨物到現場
- 現場人員清點數量、確認品項
- 拍攝照片，用 OCR 篩選文字訊息
- 確認批號與效期，同一批填一次
- 送出 `POST /api/inventory/in`

```json
POST /api/v1/inventory/in
```


### **API 送出後**
### 目前有想到 三 個方案

「OCR 上傳文字訊息」這個情境。如果用戶是在 **LINE 聊天室直接拍照傳圖**

```
用戶拍照 → LINE Bot 收到圖片 → 前端 OCR 辨識文字 → 解析出結構化資料 → 入庫
```


**方法一：整理好的 JSON**

```json
{
  "sku": "P001",
  "warehouse_id": 1,
  "serials": ["SN-001", "SN-002"],
  "batch_no": "BATCH-2025-05",
  "expiry_date": "2026-12-31"
}
```

後端只需要做驗證。

**方法二：正則表達式**

```php
// 假設 OCR 出來長這樣：「品項：蘋果汁 數量：24 效期：2026/12」
preg_match('/品項[：:]\s*(.+?)[\s數]/', $text, $sku);
preg_match('/數量[：:]\s*(\d+)/', $text, $qty);
preg_match('/效期[：:]\s*([\d\/]+)/', $text, $expiry);
```


**方法三：後端用 AI 解析**把 OCR 出來的文字丟給 Claude，要求回傳 JSON：

```php
$prompt = <<<PROMPT
以下是倉庫入庫單的 OCR 文字，請解析出結構化資料。
只回傳 JSON，不要其他文字。

文字內容：
{$ocrText}

回傳格式：
{
  "sku": "商品編號或名稱",
  "qty": 數量（整數）,
  "batch_no": "批號",
  "expiry_date": "YYYY-MM-DD 格式，沒有則 null"
}
PROMPT;
```

**第一，定義標準輸出格式** 告訴 AI 只能回傳固定的 JSON 結構，不能有任何前言或解釋文字。這樣後端 `json_decode` 就不會失敗。

**第二，定義信心度欄位** AI 回傳的結果裡加一個 `confidence` 欄位，讓 AI 自己評估這次解析有多確定。後端根據信心度決定是否需要人工確認：


```json
{
  "sku": "蘋果汁245ml",
  "qty": 24,
  "batch_no": "20250508",
  "expiry_date": "2026-12-31",
  "confidence": "high",
  "note": "效期原文為 2026/12，已補全為月底"
}
```

**第三，低信心度要擋住不能直接入庫** 如果 `confidence` 是 `low`，後端回傳給前端讓用戶人工確認，不能直接寫進資料庫。入庫是不可逆操作，寧可多一步確認也不要入錯。

**第四，保留原始文字** OCR 的原始文字要存進 `inventory_logs`，方便事後對帳或追查解析錯誤。


### **入庫後：**

1. 系統回傳入庫成功，庫存立即反映，並成立一張入庫單
2. 前端顯示入庫單（時間、操作人員、序號清單）供現場確認或列印
3. 如有 409，畫面上標示哪個序號重複，現場人員檢查是否重複

討論點：
	 - 409 是否整批重送，還是說可以部份入庫，回傳警告訊息手動輸入



## 出庫流程

- 先到期先出法
- **產生 FEFO 揀貨清單** 系統依商品、倉庫、數量，用 FEFO 規則計算出應該出哪些序號，列成清單給倉管。清單上顯示序號、批號、效期、儲位，讓倉管知道去哪裡拿哪件。
- 倉管照單取貨倉管拿著清單去倉庫實體取貨。這個步驟系統不介入，完全依賴倉管照單執行。
- 倉管按「確認出庫」 倉管取貨完成後，在 LINE Bot 或後台按確認，系統把這張出庫單狀態改成 `pending_approval`。此時庫存**尚未異動**，序號狀態改成 `reserved`（預留），避免同一件貨被其他出庫單搶走。
- 主管審核主管收到待審通知（LINE Push），打開審核清單看這張出庫單的序號、數量、倉庫。確認沒問題後按核准：
	 **核准** → 序號狀態從 `reserved` 改成 `sold`，寫入 `inventory_logs`，出庫完成
	 **駁回** → 序號狀態從 `reserved` 改回 `in_stock`，倉管收到通知重新處理

```json
POST /api/v1/inventory/out
```

```json
POST /api/v1/inventory/out/{id}
```




### 參考資料
- [力新國際 AI-OCR](https://www.newsoft.com.tw/ocr-service/)
- [FIFO、FEFO 一次搞定，電商倉儲不可不知的庫存管理法](https://www.og1o.com/tw/resources/articles/fifo-fefo-for-your-inventory-management-strategy)
- 
---
## 滯銷推播

![[Pasted image 20260508153432.png]]


---
## 付款

![[push_pay 1.svg]]


![[Pasted image 20260508153651.png]]







**`/api/pay/confirm` 用 GET 不用 POST**，因為這是 LINE Pay 主動回調你的 URL，LINE 會把 `transactionId` 和 `orderId` 直接帶在 query string，你沒辦法控制它用 POST。這個端點不需要 Bearer Token，改做冪等保護（同一個 `orderId` 第二次進來直接回 409，不重複扣款）。

**`/api/push/slow-stock` 同時支援手動觸發和 Cron**，手動版走 API 可以指定 groups 和覆寫條件閾值，Cron 版直接呼叫同一個 Service 但用設定檔的預設值，不過 HTTP。

**統一錯誤格式**：

```json
{
  "error": "錯誤說明文字",
  "code": "STOCK_INSUFFICIENT"
}
```

`code` 欄位給前端做多語系或特定 UI 處理用，比純看 HTTP status 更精確。




