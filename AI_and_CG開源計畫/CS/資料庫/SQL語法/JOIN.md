把不同資料表的資料，透過某個欄位連在一起

![[Pasted image 20260409063053.png]] 

```sql
SELECT 欄位
FROM 表1
JOIN 表2 ON 條件;
```

JOIN 類型

![[Pasted image 20260409063320.png]]

#### INNER JOIN
只取「有對應到的資料」
```sql
SELECT u.name, o.amount
FROM users u
INNER JOIN orders o ON u.id = o.user_id;
```

LEFT JOIN
左表全部保留，右表沒有就 NULL
```sql
SELECT u.name, o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
```
