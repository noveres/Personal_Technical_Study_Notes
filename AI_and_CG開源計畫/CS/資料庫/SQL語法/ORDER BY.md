```sql
SELECT 欄位
FROM 表格
ORDER BY 欄位 排序方式;
```

|關鍵字|意思|
|---|---|
|`ASC`|由小到大（預設）|
|`DESC`|由大到小|

---

多欄位排序

```sql
SELECT *
FROM users
ORDER BY age DESC, name ASC;
```

排序邏輯：

1. 先看 age（大→小）
2. age 一樣 → 再看 name（A→Z）

---

## 進階用法

###  用欄位位置排序

```sql
SELECT name, age
FROM users
ORDER BY 2;
```

依第 2 個欄位（age）排序  
可讀性差，不建議

---

###  用計算欄位排序

```sql
SELECT name, salary * 12 AS yearly
FROM employees
ORDER BY yearly DESC;
```

---

### NULL 排序（進階）

```sql
ORDER BY age DESC NULLS LAST;
```

把 NULL 放最後（PostgreSQL）

---

**ORDER BY 用來對查詢結果排序，常搭配 LIMIT 跟 WHERE 做後端分頁，且預設為ASC (由小到大 )。**
