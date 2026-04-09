LIMIT  限制結果筆數( **要拿幾筆資料** )
OFFSET 從第幾筆開始跳過


```sql
SELECT *
FROM 表格
LIMIT 數量 OFFSET 起始位置;
```

**「跳過前面幾筆，然後取後面幾筆」**
## 範例

取前 10 筆

```sql
SELECT *
FROM users
LIMIT 10;
```


### 跳過前 10 筆，再取 10 筆

```sql
SELECT *
FROM users
LIMIT 10 OFFSET 10;
```


---

## 分頁公式（超重要🔥）

第 N 頁（每頁 pageSize 筆）

```text
OFFSET = (頁數 - 1) * pageSize
```

---

### 📌 範例：每頁 10 筆

|頁數|OFFSET|SQL|
|---|---|---|
|第 1 頁|0|`LIMIT 10 OFFSET 0`|
|第 2 頁|10|`LIMIT 10 OFFSET 10`|
|第 3 頁|20|`LIMIT 10 OFFSET 20`|

##  實務上要搭配 ORDER BY 進行排序

```sql
SELECT *
FROM users
ORDER BY id
LIMIT 10 OFFSET 10;
```

### OFFSET 很大時

```sql
LIMIT 10 OFFSET 100000
```

 DB 要先掃 10 萬筆再丟掉，效率太差 
 所以我們直接使用「條件語句」取代 OFFSET

```sql
SELECT *
FROM users
WHERE id > 100000
LIMIT 10;
```

