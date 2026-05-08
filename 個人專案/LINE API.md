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


入庫流程

```json
POST /api/v1/inventory/
```



---
## 滯銷推播

![[Pasted image 20260508153432.png]]


---
## 付款

![[push_pay 1.svg]]


![[Pasted image 20260508153651.png]]






好，先把所有 API 端點、Request / Response 格式全部定義清楚，之後實作就不會跑掉。幾個設計決策說明一下：

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

要繼續哪塊的實作？










---

## 注意事項

**AI 辨識準確度**的關鍵在於 `ai_description` 欄位的品質。建議商品上架時讓 AI 預先描述一次存入 DB，例如顏色、形狀、包裝特徵，辨識時比對這份描述會比純比圖準確很多。

**LINE 圖片存取**要注意：Webhook 拿到的是 `message.id`，要用 `https://api-data.line.me/v2/bot/message/{id}/content` 加 Channel Access Token 才能下載原圖，不是直接的圖片 URL。

**Multicast 費用**：LINE 的 Multicast 每則算推播則數，分組推播前建議先確認方案的推播額度。

需要我繼續展開 LINE Pay confirm 流程的細節，或是 AI 辨識信心度不足時的人工確認機制嗎？