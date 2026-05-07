
### 1. 採購與供應商管理 (Procurement & Supplier)

處理從詢價、採購到進貨品質檢驗的流程。

|**表名**|**用途描述**|
|---|---|
|`suppliers`|供應商基本資料。|
|`supplier_contacts`|供應商聯絡人資訊。|
|`supplier_contracts`|與供應商簽訂的合約管理。|
|`supplier_evaluations`|供應商績效評估紀錄。|
|`supplier_price_catalog`|供應商報價清單/價格目錄。|
|`rfq` / `rfq_quotes`|詢價單 (Request for Quotation) 與相關報價紀錄。|
|`purchase_requests` / `_items`|內部採購申請單及其明細。|
|`purchase_orders` / `_items`|正式採購訂單 (PO) 及其明細。|
|`iqc_inspections`|進料品質檢驗 (Incoming Quality Control) 紀錄。|
|`iqc_defects`|進料檢驗發現的缺陷/不良品紀錄。|
|`iqc_returns`|檢驗不合格導致的退貨紀錄。|

### 2. 生產與配方管理 (Manufacturing & BOM)

核心生產邏輯，定義產品如何被製造。

|**表名**|**用途描述**|
|---|---|
|`bom_header` / `_detail`|物料清單 (Bill of Materials) 的標頭與組成零件明細。|
|`recipe_cards` / `_formula`|生產配方與配方公式（常見於食品或化工行業）。|
|`daily_production`|每日生產進度/產量紀錄。|
|`production_conditions`|生產環境或機台參數設定（如溫度、壓力等）。|
|`schedule_headers` / `_items`|生產排程計劃與細項。|

### 3. 庫存與倉儲管理 (Inventory & Warehouse)

監控物料流向、存量與調撥。

|**表名**|**用途描述**|
|---|---|
|`warehouses` / `factories`|倉庫與工廠的基礎設施定義。|
|`inventory`|目前庫存總量/現貨表。|
|`inventory_transactions`|庫存異動明細（所有入庫、出庫的歷史紀錄）。|
|`inventory_adjustments`|庫存盤點後的盈虧調整。|
|`inventory_transfers`|倉庫間的庫存移轉紀錄。|
|`inventory_retail_buffer`|零售端的緩衝庫存設定。|
|`inventory_retail_cache`|零售端庫存快取（用於提升查詢效能）。|
|`inbound_orders` / `outbound_orders`|入庫單與出庫單。|
|`transfer_orders`|庫存調撥指令單。|

### 4. 銷售與零售門店 (Sales & POS)

處理終端訂單、門店營運與促銷。

|**表名**|**用途描述**|
|---|---|
|`orders` / `order_details`|一般銷售訂單及其明細。|
|`orders_templates`|常用訂單模板（用於快速下單）。|
|`pos_order_header` / `_line_item`|POS 系統產生的零售訂單與品項。|
|`pos_payment_record`|POS 訂單的付款紀錄（信用卡、現金等）。|
|`stores` / `stores_products`|門店清單以及各門店販售的產品。|
|`store_business_hours`|門店營業時間設定。|
|`store_purchase_orders`|門店向總部或工廠發起的採購申請。|
|`promotion_master` / `promotions_stores`|促銷活動主檔及適用門店。|

### 5. 物流配送 (Logistics & Delivery)

負責產品離開倉庫後的運輸過程。

|**表名**|**用途描述**|
|---|---|
|`delivery_plans`|配送計畫排程。|
|`delivery_assignment`|配送任務指派（指派給司機或車隊）。|
|`delivery_tracking`|物流即時追蹤狀態紀錄。|

### 6. 權限與流程控制 (Permissions & State Machine)

系統的安全與工作流 (Workflow) 管理。

|**表名**|**用途描述**|
|---|---|
|`users` / `store_users`|系統使用者與門店特定使用者的資料。|
|`permission_roles` / `_groups`|權限角色與群組定義。|
|`permission_pages` / `_roles`|各頁面功能對應的角色權限設定。|
|`permission_users_roles`|使用者與角色的關聯表。|
|`state_machine` / `state_definition`|狀態機定義（控制單據如「待審核->已核准」的邏輯）。|
|`state_transition` / `_log`|狀態轉換規則及操作歷史紀錄。|
|`status_map`|狀態代碼與顯示文字的對照表。|

### 7. 基礎資料與系統紀錄 (Basic Data & Logs)

系統運行的支撐資料。

|**表名**|**用途描述**|
|---|---|
|`products` / `categories`|產品主檔與分類定義。|
|`site` / `system_settings`|站點資訊與全域系統參數。|
|`options`|系統下拉選單或配置選項。|
|`access_logs` / `error_logs`|使用者操作日誌與系統錯誤日誌。|
|`upload_files`|系統上傳的附件、圖片檔案管理紀錄。|
|`short_cuts`|使用者個人化的快捷選單設定。|

---

### 總結分析

1. **複雜度高**：這是一個具備 **State Machine (狀態機)** 的系統，意味著訂單或流程（如採購、生產）有非常嚴謹的審核與狀態切換機制。
    
2. **零售導向**：包含 `pos_`、`store_`、`inventory_retail_buffer` 等表，顯示該企業除了後端生產，還有前端零售門店營運。
    
3. **生產與品質並重**：有完整的 `BOM`、`Recipe` 以及 `IQC` (進料檢驗)，說明對生產配方管理與原材料品質有高度要求。
   
   
   整理如下：

## 供應商管理

| 頁面 | 說明 | 主要資料表 |
|------|------|-----------|
| suppliers.php | 供應商列表 | `suppliers`, `supplier_contacts`, `purchase_orders` |
| suppliers_edit.php | 供應商新增/編輯 | `suppliers` |
| suppliers_evaluation.php | 供應商評鑑 | `supplier_evaluations`, `suppliers`, `users` |
| suppliers_contracts.php | 供應商合約管理 | `supplier_contracts`, `suppliers`, `options` |
| apis/suppliers.php | 供應商 API | `suppliers` |

## 報價管理

| 頁面                               | 說明      | 主要資料表                                                      |
| -------------------------------- | ------- | ---------------------------------------------------------- |
| rfq.php                          | 詢報價列表   | `rfq`, `rfq_quotes`, `products`                            |
| rfq_edit.php                     | 詢價單編輯   | `rfq`, `rfq_quotes`, `suppliers`, `products`               |
| rfq_compare.php                  | 報價比價分析  | `rfq`, `rfq_quotes`, `suppliers`, `supplier_price_catalog` |
| suppliers_price_catalog.php      | 供應商價格目錄 | `supplier_price_catalog`, `suppliers`, `products`          |
| suppliers_price_catalog_edit.php | 報價新增/編輯 | `supplier_price_catalog`, `suppliers`, `products`D         |
| suppliers_price_history.php      | 報價歷史    | `supplier_price_catalog`, `suppliers`, `products`          |

## 請購單管理

| 頁面                         | 說明        | 主要資料表                                                                               |
| -------------------------- | --------- | ----------------------------------------------------------------------------------- |
| purchase_requests.php      | 請購單列表     | `purchase_requests`, `purchase_request_items`, `users`                              |
| purchase_requests_edit.php | 請購單編輯     | `purchase_requests`, `purchase_request_items`, `products`, `supplier_price_catalog` |
| purchase_orders.php        | 採購訂單列表    | `purchase_orders`, `purchase_order_items`, `suppliers`                              |
| purchase_orders_edit.php   | 採購訂單編輯    | `purchase_orders`, `purchase_order_items`, `products`, `supplier_price_catalog`     |
| apis/convert_pr_to_po.php  | 請購轉採購 API | `purchase_requests`, `purchase_orders`, `purchase_order_items`                      |

## 驗收管理

| 頁面                    | 說明          | 主要資料表                                                                           |
| --------------------- | ----------- | ------------------------------------------------------------------------------- |
| iqc.php               | 來料檢驗單列表     | `iqc_inspections`, `purchase_orders`, `suppliers`, `iqc_defects`, `iqc_returns` |
| iqc_detail.php        | 檢驗單明細/編輯    | `iqc_inspections`, `iqc_defects`, `purchase_order_items`, `products`            |
| iqc_returns.php       | 不良品退貨處理     | `iqc_returns`, `iqc_inspections`, `suppliers`（**檔案尚未建立**）                       |
| apis/get_po_items.php | 取得採購單品項 API | `purchase_orders`, `purchase_order_items`, `products`                           |

---

**核心資料表共 17 張**，四個模組之間透過 `suppliers`、`products`、`purchase_orders` 互相關聯。


BUG
purchase_requests_edit.php
單位無法修改


 purchase_orders_edit.php 

 付款條件自動帶出會出現 <br /><b>Deprecated</b>:  htmlspecialchars(): Passing null to parameter #1 ($string) of type string is deprecated in <b>/var/www/html/purchase_orders_edit.php</b> on line <b>730</b><br />
 __________________________________________________________________
 
  purchase_orders_edit.php 
  數量 可以小數 但按加時優先給整數
   __________________________________________________________________
  iqc_detail.php
  物料明細（選填） 代出的值 功能 不明顯
 
 
 ID 要限制
 ![[Pasted image 20260507120555.png]]
 