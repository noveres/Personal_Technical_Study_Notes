LINE bot 拍照 → 前端 OCR → 前端產出結構化 JSON（含多個 destination、`planned_qty` / `actual_qty` 兩個量、NG 段、SUM 段）。

這份 spec 給前端對接
關鍵設計：
- **不新建任何表**：現有 schema（`inbound_orders` + `order_details` + `inventory_adjustments`）全部夠用
-  **多 destination → 多張單**：每個地點拆 1 張 `inbound_orders`（草稿），一張 `inbound_orders` 只能對應 1 個倉庫（warehouse_id 是單一欄位，不是陣列）。兩個不同的 warehouse_id，不可能塞進同一張 `inbound_orders row`。所以後端會 INSERT 2 列：
- **NG 分離**：另開 1 張 `inventory_adjustments`（`Damage` + `Draft`）
- **SUM 不寫 DB**：彙總後端即時 `SUM ... GROUP BY sku` 計算
- **草稿不扣帳**：不碰 `inventory_transactions`、不動 `inventory.on_hand_qty`，等專人過帳
- **視角假設入庫**：同一筆物流，但對發送方是出庫、對接收方是入庫。把 `inbound_orders` 換成 `outbound_orders` 即可

## API endpoint

```http
POST /api/v1/inbound-orders
```

一個 request 可帶多個 destination + NG，後端自動拆成多張 `inbound_orders` + 視情況 1 張 `inventory_adjustments`。「批次」是實作細節，URL 不暴露。

## Request body

``` json
{
  "destinations": [
    {
      "note": "台南美術館",
      "warehouse_id": 1,
      "items": [
        { "sku": "BG030101", "planned_qty": 12, "actual_qty": 12 },
        { "sku": "BG030109", "planned_qty": 10, "actual_qty": 5  }
      ]
    },
    {
      "note": "嘉義新光三越",
      "warehouse_id": 2,
      "items": [
        { "sku": "BG030101", "planned_qty": 15, "actual_qty": 15 },
        { "sku": "BG030112", "planned_qty": 30, "actual_qty": 20 }
      ]
    }
  ],
}
```

### 欄位規則

| 欄位                                | 型別     | 必填  | 說明                                                                                                |
| --------------------------------- | ------ | :-: | ------------------------------------------------------------------------------------------------- |
| `destinations`                    | array  |  ✓  | 至少 1 個地點，每個地點產出 1 張 inbound_orders                                                                |
| `destinations[].destination_name` | string |  ✓  | 顯示用名稱（如「台南美術館」），會塞進 `inbound_orders.note` 給人對單                                                    |
| `destinations[].warehouse_id`     | int    |  ✓  | 對應 `warehouses` 表，前端 mapping 後送                                                                   |
| `destinations[].items`            | array  |  ✓  | 至少 1 筆；qty=0 的行建議省略                                                                               |
| `items[].name`                    | string |  ✓  | OCR 解析出的商品名稱。後端 `SELECT sku FROM products WHERE product_name = :name` 精準 match；match 不到整張單 reject |
| `items[].planned_qty`             | number |  ✓  | → `order_details.planned_quantity`                                                                |
| `items[].actual_qty`              | number |  –  | → `order_details.processed_quantity`；不送視為 0                                                       |
| `ng_items`                        | array  |  –  | NG 異常品，產出 1 張 `inventory_adjustments`                                                             |
| `ng_items[].warehouse_id`         | int    |  ✓  | NG 發生倉庫                                                                                           |
| `ng_items[].name`                 | string |  ✓  | 同 `items[].name` 規則                                                                               |
| `ng_items[].qty`                  | number |  ✓  | NG 數量                                                                                             |
| `ng_items[].reason`               | string |  –  | NG 原因，預設 `"NG"`                                                                                   |
| `note`                            | string |  –  | 整批備註，會塞進每張單                                                                                       |

## Response 200

```json
{
  "language": "zh-TW",
  "code": 0,
  "success": true,
  "resultMessage": "Success",
  "systemStatusCode": "OK",
  "totalDataCount": 3,
  "totalPageCount": 1,
  "resultObject": [
    { "type": "inbound",    "id": "IN-20260511-0001",  "note": "台南美術館",     "item_count": 13 },
    { "type": "inbound",    "id": "IN-20260511-0002",  "note": "嘉義新光三越",  "item_count": 13 }
  ]
}
```

`totalDataCount` 是這次建出的單據總數。

## Error response

```json
{
  "language": "zh-TW",
  "code": 400,
  "success": false,
  "resultMessage": "...",
  "systemStatusCode": "...",
  "totalDataCount": 0,
  "totalPageCount": 0,
  "resultObject": []
}
```

| HTTP | `systemStatusCode` | 情境 |
|---|---|---|
| 401 | `UNAUTHORIZED` | token 無效 |
| 400 | `EMPTY_DESTINATIONS` | destinations 空陣列 |
| 400 | `INVALID_WAREHOUSE` | warehouse_id 不存在 |
| 400 | `NAME_NOT_MATCHED` | 某筆 `name` 在 `products.product_name` 找不到精準 match（resultObject 帶 `[{destination, name, line}]`，可一次列所有未 match 的行給前端修正）|
| 400 | `INVALID_QTY` | qty ≤ 0 或非數字 |
| 500 | `INTERNAL_ERROR` | DB 例外 |


## 單號規則

- 入庫：`IN-YYYYMMDD-NNNN`（NNNN 當日 4 碼流水）
- 調整：`ADJ-YYYYMMDD-NNNN`
- 流水序產生需在 transaction 內、用 `SELECT COUNT(*) ... FOR UPDATE` 避免併發碰撞


## 出庫 API
```HTTP
POST /api/v1/outbound-orders
```

Request / Response 結構**完全相同**（destinations + items + ng_items + note），下面只列差異：

|項目|入庫|出庫|
|---|---|---|
|Endpoint|`POST /api/v1/inbound-orders`|`POST /api/v1/outbound-orders`|
|單頭表|`inbound_orders`|`outbound_orders`|
|`order_type` 預設|`'Purchase'`|`'Sale'`|
|來源/去向欄位|`source_type='Supplier'`|`destination_type='Customer'`|
|`destination_name` 語意|顯示用倉庫/收貨方名稱|送貨目的地名稱（客戶/分店），會塞進 `outbound_orders.shipping_address`|
|`warehouse_id` 語意|目標入庫倉庫 ID|**來源**發貨倉庫 ID|
|單號前綴|`IN-`|`OUT-`|
|入帳方向（過帳時）|`inventory.on_hand_qty +=`|`inventory.on_hand_qty -=`|
|NG 段（同 `inventory_adjustments`）|用 `Damage` 表破損入庫|用 `Damage` 表破損出庫 — 一樣，**NG 邏輯共用**|
|Response `resultObject[].type`|`'inbound'`|`'outbound'`|

### Request 範例（出庫）

```json
{
  "destinations": [
    {
      "note": "台南美術館",
      "warehouse_id": 1,
      "items": [
        { "sku": "BG030101", "planned_qty": 12, "actual_qty": 12 },
        { "sku": "BG030109", "planned_qty": 10, "actual_qty": 5  }
      ]
    },
    {
      "note": "嘉義新光三越",
      "warehouse_id": 1,
      "items": [
        { "sku": "BG030101", "planned_qty": 15, "actual_qty": 15 },
        { "sku": "BG030112", "planned_qty": 30, "actual_qty": 20 }
      ]
    }
  ],
}
```


注意：出庫情境通常**多個 destination 共用同一個來源倉庫**（如同一個中央倉發貨給多分店），上面範例兩個 destination 的 `warehouse_id` 都是 `1`。前端視業務決定。

### Response 範例（出庫）

```json
{
  "language": "zh-TW",
  "code": 0,
  "success": true,
  "resultMessage": "Success",
  "systemStatusCode": "OK",
  "totalDataCount": 3,
  "totalPageCount": 1,
  "resultObject": [
    { "type": "outbound",   "id": "OUT-20260511-0001", "note": "台南美術館",    "item_count": 2 },
    { "type": "outbound",   "id": "OUT-20260511-0002", "note": "嘉義新光三越", "item_count": 2 }
  ]
}
```


### 出庫專屬 schema 細節

- `outbound_orders.order_status` enum 是 `Pending|Picking|Shipped|Completed|Cancelled`（跟入庫不同，多了 `Picking`、`Shipped`），草稿仍用 `'Pending'`
- `outbound_orders.shipping_address` 存 `destination_name`，方便送貨人員看
- `outbound_orders.customer_id` 暫時 NULL（系統無 customers 表）
  
  
## 用到的現有表

| 表                               | 用途                                                      |
| ------------------------------- | ------------------------------------------------------- |
| `inbound_orders` (行 616)        | 入庫單頭，狀態用既有 `Pending` 當草稿                                |
| `order_details` (行 1036)        | 出入庫共用明細，已含 `planned_quantity` + `processed_quantity` 兩量 |
| `inventory_adjustments` (行 692) | NG 異常單，已有 `Damage` type + `Draft` status                |
| `inventory_transactions`        | 草稿階段**不寫**，過帳時才寫                                        |
| `inventory`                     | 草稿階段**不動** on_hand_qty                                  |
