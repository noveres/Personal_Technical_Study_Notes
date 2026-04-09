用來指定查詢的資料來源，可以是資料表、JOIN 結果或子查詢。


```sql
SELECT *
FROM users;
```

 從 `users` 表拿全部資料

---

```sql
SELECT name, email
FROM users;
```

從 `users` 表拿指定欄位

---

#  FROM 可以接什麼？

---

 單一資料表（最基本）

```sql
FROM users
```


---

## 多表（JOIN）

```sql
SELECT u.name, o.amount
FROM users u
JOIN orders o ON u.id = o.user_id;
```


---

##  子查詢
```sql
SELECT *
FROM (
    SELECT name, age
    FROM users
) AS temp;
```

- `FROM` 可以接「一個查詢結果」
- 要加別名（`AS temp`）


---

##  多表

```sql
SELECT *
FROM users, orders;
```

會變笛卡兒積（全部相乘）

---

#  圖像理解

## 單表查詢

![Image](https://images.openai.com/static-rsc-4/vwjDRh8kepNl2ftDnvfVZyCVyjYG9VfdInVyS2cl6aTHxL8THc-6shguhpLG7QY42b-gVO7ltQqs4dyV3FViIhZKcafgmFFNBKSG37v1YAkvLEm5LppSwHoOaBBw8mhGrf7EMUkmH1eLGEXDYwpYB7lcDBltP6qCdHh1iLuYhOzSPk7cAEjQF6oLHQtfEN5G?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/wzwuWYu1KYjlESF8maGo5z6uw3_cbj3U4HcFN06WYoxYoweKlPzwTh0hLupzQoWTivJrNNG2tWeHFXM5Ib4R_X15-tQbn8GdrWfG8X1uCjK-G_8buRjLlLBumqkXSTZdhhROb6WTfUeGQLks2mYLuCw_Kh-07c-cycz8efYlS6313ZRtsdye7jaqinZiwUZd?purpose=fullsize)

 從一張表取資料

---

## [[JOIN]]查詢

![Image](https://images.openai.com/static-rsc-4/qSgo2bFppx2tgcBoN_a-nxdw5gBOpGBkYF5VZ-dMAe2k5sPnpiD2Fppyrcvh8qWCIyadR_4cLCkTSn9JYTl9r3uyBwsvZAmGvn71u52BmdthgLFm2yIsAtlYWKYaZexSGwxaJLIc-LHoLl4u3MxqqWNTQSn79O3TBnzmHnYKlYPiT9sNNLl1RqomQrLaSzDv?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/-zfx17YMf8Q7SuxYaE6wQRteplFZG10DWjbZ8_27ph9l9dZcA82JyBz45_OkYoG_Oz3louTysIo8JMhG9JlYBHgE5jQVsYsp1aMJf6TAtaYl9l2Mup_u8HKZsmf6jk9X4KUBtM0gX_wd7SnNc-Zwxb2w6J_XWbzKA8SEBEnDeLFmTPc3M598DUu1JH0Evn11?purpose=fullsize)

從多張表「組合資料」

