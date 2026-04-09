這是後端與資料庫最核心的概念之一👇  
**CRUD 為資料的四種基本操作**

|縮寫|全名|中文|
|---|---|---|
|C|Create|新增資料|
|R|Read|查詢資料|
|U|Update|更新資料|
|D|Delete|刪除資料|


---
# 對應 SQL 指令

|CRUD|SQL|
|---|---|
|Create|`INSERT`|
|Read|`SELECT`|
|Update|`UPDATE`|
|Delete|`DELETE`|

---

# Create（新增資料）

```sql
INSERT INTO users (name, age)
VALUES ('John', 25);
```

 新增一筆使用者資料

---

# Read（查詢資料）

```sql
SELECT *
FROM users;
```

查詢所有資料

---

# Update（更新資料）

```sql
UPDATE users
SET age = 26
WHERE id = 1;
```

 修改 id = 1 的資料

---

# Delete（刪除資料）

```sql
DELETE FROM users
WHERE id = 1;
```

刪除 id = 1 的資料

---

# 超重要觀念（一定要記🔥）

## 忘記 WHERE 

```sql
UPDATE users SET age = 30;
```

 全部資料都被改掉 

```sql
DELETE FROM users;
```

 全表刪除 

---

## 正確習慣

永遠先寫：

```sql
WHERE id = ?
```

---

# CRUD 在系統中的位置

一個後端 API 基本就是 CRUD

---

## RESTful API 對應

|操作|HTTP|範例|
|---|---|---|
|Create|POST|`/users`|
|Read|GET|`/users`|
|Update|PUT / PATCH|`/users/1`|
|Delete|DELETE|`/users/1`|

---



