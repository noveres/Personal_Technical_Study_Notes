## 入庫 endpoint
```
POST http://localhost:9090/api/v1/inventory/Create.php
Content-Type: application/json
```

## 出庫 endpoint
```
POST http://localhost:9090/api/v1/inventory/out.php
Content-Type: application/json
```


3種主要情況 完整 
### body

```json
{
  "destinations": [
    {
      "note": "台南美術館",
      "warehouse_id": 5,
      "items": [
        { "sku": "BG030001", "planned_qty": 12, "actual_qty": 12 },
        { "sku": "BG030002", "planned_qty": 12, "actual_qty": 12 },
        { "sku": "BG030003", "planned_qty": 6,  "actual_qty": 6  },
        { "sku": "BG030004", "planned_qty": 10, "actual_qty": 5  }
      ]
    },
    {
      "note": "嘉義新光三越",
      "warehouse_id": 6,
      "items": [
        { "sku": "BG030005", "planned_qty": 15, "actual_qty": 15 },
        { "sku": "BGXXXX99", "planned_qty": 30, "actual_qty": 20 }
      ]
    },
    {
      "note": "高雄漢神巨蛋（全錯）",
      "warehouse_id": 5,
      "items": [
        { "sku": "WRONG_A",  "planned_qty": 8, "actual_qty": 8 },
        { "sku": "WRONG_B",  "planned_qty": 4, "actual_qty": 4 }
      ]
    }
  ]
}
```

預期回 207：
- destination 1 → 1 張入庫單、4 個品項全寫
- destination 2 → 1 張、1 個品項寫、`BGXXXX99` 進 note 與 unresolved
- destination 3 → 1 張、無明細，2 個品項都進 note 與 unresolved

```json
{
    "language": "zh-TW",
    "code": 207,
    "success": true,
    "resultMessage": "部分品項 SKU 未識別，已建單但需審核人員補正",
    "systemStatusCode": "PARTIAL_SUCCESS",
    "totalDataCount": 3,
    "totalPageCount": 1,
    "resultObject": [
        {
            "id": "IN202605130046",
            "name": "IN202605130046",
            "date": "20260513",
            "total_planned": 40,
            "total_actual": 35,
            "unresolved": []
        },
        {
            "id": "IN202605130047",
            "name": "IN202605130047",
            "date": "20260513",
            "total_planned": 15,
            "total_actual": 15,
            "unresolved": [
                {
                    "sku": "BGXXXX99",
                    "planned_qty": 30,
                    "actual_qty": 20
                }
            ]
        },
        {
            "id": "IN202605130048",
            "name": "IN202605130048",
            "date": "20260513",
            "total_planned": 0,
            "total_actual": 0,
            "unresolved": [
                {
                    "sku": "WRONG_A",
                    "planned_qty": 8,
                    "actual_qty": 8
                },
                {
                    "sku": "WRONG_B",
                    "planned_qty": 4,
                    "actual_qty": 4
                }
            ]
        }
    ]
}
```


---
### body
1. 全部資料有效（HTTP 200 / code 0）
```json
{
  "destinations": [
    {
      "note": "台南美術館",
      "warehouse_id": 5,
      "items": [
        { "sku": "BG030001", "planned_qty": 12, "actual_qty": 12 },
        { "sku": "BG030002", "planned_qty": 10, "actual_qty": 5  },
        { "sku": "BG030003", "planned_qty": 6,  "actual_qty": 6  }
      ]
    },
    {
      "note": "嘉義新光三越",
      "warehouse_id": 6,
      "items": [
        { "sku": "BG030004", "planned_qty": 15, "actual_qty": 15 },
        { "sku": "BG030005", "planned_qty": 30, "actual_qty": 20 }
      ]
    }
  ]
}

```

### Responses

```json
 {
    "language": "zh-TW",
    "code": 0,
    "success": true,
    "resultMessage": "Success",
    "systemStatusCode": "OK",
    "totalDataCount": 1,
    "totalPageCount": 1,
    "resultObject": [
        {
            "id": "IN202605130045",
            "name": "IN202605130045",
            "date": "20260513",
            "total_planned": 1,
            "total_actual": 1,
            "unresolved": []
        }
    ]
}
}
```


1. 部分 SKU 無效（ HTTP 207）
```json
{
  "destinations": [
    {
      "note": "台南美術館",
      "warehouse_id": 5,
      "items": [
        { "sku": "BG030001", "planned_qty": 8, "actual_qty": 8 },
        { "sku": "BGxxxxxx", "planned_qty": 5, "actual_qty": 3 }
      ]
    }
  ]
}

```

3. 全部 SKU 無效（ HTTP 207、父單仍建、明細為空）
```json
{
  "destinations": [
    {
      "note": "台南美術館",
      "warehouse_id": 5,
      "items": [
        { "sku": "BGxxxxxx", "planned_qty": 8, "actual_qty": 8 },
        { "sku": "BGxxxxxx", "planned_qty": 5, "actual_qty": 3 }
      ]
    }
  ]
}

```

4. `destinations` 空（預期 HTTP 400 `EMPTY_DESTINATIONS`）

```json
{
  "destinations": []
}
```

---

5. `warehouse_id` 不存在（預期 HTTP 400 `INVALID_WAREHOUSE`）

```json
{
  "destinations": [
    {
      "note": "錯誤倉庫",
      "warehouse_id": 9999,
      "items": [
        { "sku": "BG030001", "planned_qty": 10, "actual_qty": 10 }
      ]
    }
  ]
}
```

---

 6. `qty` < 0（HTTP 400 `INVALID_QTY`） 如果 `qty` = 0 還是會讓他建空表單，但我有寫判斷過濾掉 = 0 的資料 
    

P.s ： 我有紀錄那些資料為 0，放到 ，這段我怕被多問先註解掉，之後他們有要求再加 
```php
if (!empty($filteredItems))
{
$filteredText = '記錄在實體單上但數量標記為 0 的資料共 '.count($filteredItems) . ' 項: ';
 foreach ($filteredItems as $it) {
               $filteredText .= ($it['sku'] ?? '') . ', ';
               }
              $noteAdditions[] = rtrim($filteredText, ', ');
}

```

	
	
```json
{
  "destinations": [
    {
      "note": "qty 錯誤",
      "warehouse_id": 5,
      "items": [
        { "sku": "BG030001", "planned_qty": 0, "actual_qty": 0 }
      ]
    }
  ]
}
```

---

7. `items` 空（預期 HTTP 400 `EMPTY_DESTINATIONS`）

```json
{
  "destinations": [
    {
      "note": "沒明細",
      "warehouse_id": 5,
      "items": []
    }
  ]
}
```

---



## 可用的真實 SKU

`BG030001`~`BG030017` 
- BG030001 倫敦貝果
- BG030002 巧克力倫敦貝果
- BG030003 油漬番茄乳酪貝果
- BG030004 藍莓乳酪貝果
- BG030005 羅勒青醬培根貝果
- BG030006 麻辣燙火腿貝果
- ...

## 可用的 

常用：
- `5` = 三倉
- `6` = 走道倉
- `12` = 不良品庫
- `13` = 門市出貨虛擬倉
- `34` = 嘉義新光三越快閃櫃
