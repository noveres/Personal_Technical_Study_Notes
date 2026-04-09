WHERE = 篩選資料的條件，只取符合條件的資料

 基本語法

```sql
SELECT 欄位
FROM 表格
WHERE 條件;
```

---

 範例

###  基本條件

```sql
SELECT *
FROM users
WHERE age > 18;
```

找「年齡大於 18」的人

---

###  字串條件

```sql
SELECT *
FROM students
WHERE 班級 = '1 年 2 班';
```

- 字串一定要用 `' '`
- 不能用 `" "`（會被當欄位）


---

# 常用運算子

## 比較運算

|運算子|意思|
|---|---|
|`=`|等於|
|`!=` 或 `<>`|不等於|
|`>` `<` `>=` `<=`|大小比較|

---

## 邏輯運算

```sql
WHERE age > 18 AND gender = '男'
```

|關鍵字|意思|
|---|---|
|`AND`|並且|
|`OR`|或|
|`NOT`|反向|

---

##  IN（多值比對）

```sql
WHERE city IN ('台北', '高雄', '台中')
```

等同多個 OR

---

## BETWEEN（範圍）

```sql
WHERE age BETWEEN 18 AND 30
```

包含 18 和 30

---

## LIKE（模糊查詢）

```sql
WHERE name LIKE 'A%'
```


- `A%` → 開頭是 A
- `%A` → 結尾是 A
- `%A%` → 包含 A
- `A_` → 開頭 A 且接受後面為一個字元

---

##  NULL 判斷

錯誤寫法：

```sql
WHERE phone = NULL;
```

在 SQL 的世界裡，**`NULL` 並不是一個「值」，而是一種代表 未知（Unknown）的「狀態」**，不能用一般的算術比較運算子（如 `=` 或 `!=`）來比對它。

SQL 採用的是**三值邏輯**。當你介入 `NULL` 時，運算結果會有三種可能：

- **True** (真)
- **False** (假)
- **Unknown** (未知)

因此，`phone = NULL` 的結果永遠是 **Unknown**。而在 `WHERE` 子句中，只有結果為 **True** 的資料列才會被篩選出來。


正確寫法：

```sql
WHERE phone IS NULL;
```

 **`IS`** 關鍵詞是用於判斷一個欄位的**狀態**。
 `IS NULL` 就像是在問：**「這個箱子是不是空的？」** 這是一個可以直接觀察並得到「是」或「不是」的問題。

