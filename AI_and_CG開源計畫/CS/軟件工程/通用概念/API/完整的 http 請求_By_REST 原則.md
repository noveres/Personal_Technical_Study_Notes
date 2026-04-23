## RESTful API 請求包含以下主要元件：

#### 唯一資源識別符

每一筆 資料（資源）都需要一個「地址」來被找到。針對 REST 服務，伺服器通常使用統一資源定位器 **(URL)** 來執行資源識別 **俗稱「網址」** 。

 服務端對外提供功能或資料的存取入口稱為 endpoint（終點），用來表示 API 的具體網址， 在實務上由 **HTTP method 與 URL 組成**，用來表示對某個資源的操作，所以網址中不建議有動詞，盡量只用名詞，而且所用的名詞通常要與資料庫的表格名對應


#### 請求方法 ( HTTP method)

HTTP method（請求方法）用來告知伺服器對資源要進行的操作類型，例如取得、建立、更新或刪除。
#### Http 動詞

| 請求方法   | SQL 類比 | 用途              | 請求內容        | 說明             |
| ------ | ------ | --------------- | ----------- | -------------- |
| GET    | SELECT | 從伺服器取得資源（單筆或多筆） | 通常無 body    | 不應改變伺服器狀態，可被快取 |
| POST   | CREATE | 建立新資源           | 傳完整資料（body） | 每次呼叫可能產生新資源    |
| PUT    | UPDATE | 更新整個資源          | 傳完整資源（覆蓋）   | 重複請求結果相同       |
| PATCH  | UPDATE | 更新部分資源          | 傳部分欄位       | 視實作而定（可能非冪等）   |
| DELETE | DELETE | 刪除資源            | 通常無 body    | 刪多次結果相同（資源不存在） |

*注意：雖然技術上只要前後端溝通好可以不完全遵守其語意，但實務上建議依照 REST 設計原則使用，  以提升 API 的可讀性、可維護性，並與主流框架及工具相容


---
#  HTTP 請求

一個 HTTP Request 是由 請求行、請求頭、請求體組成：

![[Pasted image 20260419085414.png]]

##  請求行（Request Line）

請求行是整個http封包的第一行字串

```http
POST /api/user/login HTTP/1.1 
```

 用途：似於文章的標題，用來概括客戶端想要執行什麼行為

| 部分              | 說明          |
| --------------- | ----------- |
| POST            | HTTP Method |
| /api/user/login | Path        |
| HTTP/1.1        | 協議版本        |

備註：*HTTP  Method + Path = endpoint

---

## 請求頭（Headers）

```http
Content-Type: application/json
Authorization: Bearer token123
```

 用途：包含額外的訊息

- 以「鍵值對」（Key-Value）形式存在，用於描述請求內容、客戶端環境、認證資訊及處理參數。

- 常用於設定內容類型（Accept）、身分驗證（Authorization）、快取控制（Cache-Control）及追蹤請求來源（User-Agent）


#### 我們只需關注下面幾個請求頭即可:

1. Host: url：位址中的主機

2. User-Agent：客戶端的資訊描述

3. Content-Type：請求體的訊息是什麼格式,如果沒有請求體，通常不需要設定這個欄位

##### Content-Type 常見格式

1. application/x-www-form-urlencoded

	表示請求體的資料格式和url位址中參數的格式一樣,例如

```HTTP
loginId=admin&loginPwd=123123
```


---

2. application/json（最常用）

	表示資料為 JSON 格式

```json
{
  "loginId": "admin",
  "loginPwd": "123123"
}
```

特點：
- 結構清晰
- 支援巢狀資料
- 為目前前後端主流格式


---

 3. multipart/form-data
	
	用來表示一種特殊的請求體格式用於上傳檔案（例如圖片、影片）
	
	特點：
	- 可同時傳送文字與檔案
	- 資料會分段傳送（multipart）
	- 常用於檔案上傳功能

---

##  請求體（Body）

```json
{
  "name": "Alex",
  "email": "alex@example.com"
}
```

用途：指的是隨著HTTP 請求一同傳送的資料，這些資料不會出現在 URL 中，而是以 JSON 或其他格式（如 XML、form-data）作為請求的主體。

備註：*請求體是可以視情況省略的*，常見的情況是需要的資訊已經在 URL 內部

例如

```http
GET http://www.test.com/users?page=1&size=10
```

```http
DELETE http://www.test.com/users/1
```


---


推薦參考資料：

* [AWS 官方手冊 RESTful API](https://aws.amazon.com/tw/what-is/restful-api/)

