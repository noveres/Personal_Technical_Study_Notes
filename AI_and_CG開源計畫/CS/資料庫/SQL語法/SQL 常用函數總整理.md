### 什麼是 SQL 函數？
 **對資料做「計算 / 處理 / 轉換」的工具**


### 1.聚合函數
用來統計資料

| 函數        | 功能  |
| --------- | --- |
| `COUNT()` | 筆數  |
| `SUM()`   | 加總  |
| `AVG()`   | 平均  |
| `MAX()`   | 最大值 |
| `MIN()`   | 最小值 |


### 2. 字串函數
| 函數         | 功能   |
| ---------- | ---- |
| `CONCAT()` | 拼接字串 |
| `LENGTH()` | 字串長度 |
| `LOWER()`  | 轉小寫  |
| `UPPER()`  | 轉大寫  |
| `TRIM()`   | 去空白  |
### 3.數值函數
| 函數        | 功能    |
| --------- | ----- |
| `ROUND()` | 四捨五入  |
| `CEIL()`  | 無條件進位 |
| `FLOOR()` | 無條件捨去 |
### 4. 日期函數
| 函數           | 功能   |
| ------------ | ---- |
| `NOW()`      | 現在時間 |
| `CURDATE()`  | 今天   |
| `DATEDIFF()` | 日期差  |
| `DATE_ADD()` | 加時間  |
### 5.條件函數
#### CASE WHEN

```sql
SELECT name,
       CASE 
           WHEN age >= 18 THEN '成人'
           ELSE '未成年'
       END AS status
FROM users;
```


#### IF（MySQL）

```sql
SELECT IF(age >= 18, '成人', '未成年')
FROM users;
```

### 6. NULL 處理
| 函數           | 功能         |
| ------------ | ---------- |
| `IFNULL()`   | NULL → 預設值 |
| `COALESCE()` | 第一個非 NULL  |
```sql
SELECT IFNULL(phone, '無資料')
FROM users;
```