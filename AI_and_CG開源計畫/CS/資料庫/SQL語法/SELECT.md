SELECT * 意思是從表格中**選取資料**全部欄位的資料

```sql
SELECT * FROM 表格名稱 ; 
```

*表格名稱可以視情況加｢ " "」號依照資料庫軟體，可加可不加

| 資料庫        | 寫法                              |
| ---------- | ------------------------------- |
| MySQL      | `` `table_name` ``              |
| PostgreSQL | `"table_name"`                  |
| SQL Server | `[table_name]` 或 `"table_name"` |
| Oracle     | `"TABLE_NAME"`                  |
慣用寫法如上

如果要查特定欄位
SELECT 後方添加欄位名稱
```sql
SELECT 姓名, 班級, 成績 FROM students;
```
