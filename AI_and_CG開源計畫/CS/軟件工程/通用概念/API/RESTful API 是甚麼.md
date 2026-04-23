### RESTful API 是一種主流的 API 設計規範

RESTful API 通常是指 RESTful Web API 最初是作為管理複雜網路 (如網際網路) 上的通訊指導方針而建立，來支援大規模的高效能和可靠的通訊。 

> 
>  REST 的全稱是「**Representational State Transfer**」中文叫**表現層狀態轉移**
> 


### RESTful API 名稱定義

1. 資源表現層 **RE ( Representational )**
	
	* **資源**：不同應用程式向**客戶端**提供的**資料**，資源可以是任何類型的**資料**，例如在**客戶端**操作的影像、文字、數字、用戶列表等...，這些可以操作的**資料**都是一種**資源**
	
	* **表現層**：指資源呈現的具體格式，例如 後端會將**資料轉成統一的格式回傳，如JSON、XML等...** ，讓**客戶端**渲染出來，給用戶依照需求去操作資料的狀態

2. 狀態 **S ( State )** 
	狀態分為 「無狀態」和「有狀態 」  ，目前 REST 是以**無狀態**為主
	
	* **無狀態**：伺服器不會記住使用者之前的操作，每個請求都是獨立的，所以容易擴展多台 server
	
	* **有狀態**：後端會為每個使用者**建立並保存狀態**（例如 Session），Server 用 Session ID 查伺服器會儲存互動狀態，例如使用者的登入資訊、購物車內容等...
	

 3. 轉移 **T ( Transfer )**
	 
	 * Transfer 指的是資料在系統之間的傳遞方式
	 
	 * 前端發送請求（Request）後端回傳結果（Response）


###  REST 架構風格的六個原則

| **守則**        | **名詞**            |                                               |
| ------------- | ----------------- | --------------------------------------------- |
| **用戶端-伺服器分離** | Client-Server     | 前後端分離，前端管畫面，後端管資料。互不干涉，方便各自升級。                |
| **無狀態**       | Stateless         | 伺服器不記住用戶是誰。每次請求都是一個獨立的事件，要帶完整資訊 (如 Token)。    |
| **可快取**       | Cacheable         | 部分資料可以暫存前端。減少伺服器運算壓力。                         |
| **分層系統**      | Layered System    | 客戶端不需要分辨它是直接連接到伺服器，還是連接到中間層，所以請求可以經過多層轉發。     |
| **統一介面**      | Uniform Interface | 大家都用標準 HTTP 動詞 (GET, POST)。全世界工程師都能溝通。        |
| **按需代碼**      | Code on Demand    | 可選約束。伺服器可以透過發送指令（如 JavaScript）來暫時擴展或自定義客戶端的功能 |


推薦參考資料：
* [后端程序员的尊严，从学会 RESTful API 开始！](https://www.bilibili.com/video/BV1WFBXBmExs/?spm_id_from=333.337.search-card.all.click&vd_source=69f1aeabad4b9434e0b71c523349683d)
* [AWS 官方手冊 RESTful API](https://aws.amazon.com/tw/what-is/restful-api/)