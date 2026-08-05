 Atlas 是專門用來管理資料庫結構的現代方案，
 官網與文件連結如下：

- **官方網站：** [https://atlasgo.io](https://atlasgo.io/)
    
- **官方文件：** [https://atlasgo.io/docs](https://atlasgo.io/docs)
    
- **GitHub 專案：** [https://github.com/ariga/atlas](https://github.com/ariga/atlas)

##  Atlas 超快速上手指南 (MySQL / MariaDB 範例)

### 第一步：安裝 Atlas CLI

在終端機（Terminal）執行安裝指令：

- **macOS / Linux:**
    
    ``` Bash
    curl -sSf https://atlasgo.sh | sh
    ```
    
- **Windows (PowerShell):**
    

    ```    PowerShell
    iid | iex (Invoke-WebRequest https://atlasgo.sh?install=windows)
    ```
    

### 第二步：定義想要的資料庫結構 (`schema.sql`)

在的專案目錄下，直接放一個**理想中最新、最乾淨**的資料表定義檔。例如建立一個 `schema.sql`：

```SQL
-- schema.sql (理想狀態)
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone VARCHAR(20) NULL -- 這是新加的欄位
);
```

### 第三步：讓 Atlas 自動通靈，生成 Migration 與回滾檔！

現在，我們不需要自己寫 `ALTER TABLE` 語法，也不用自己寫 `DROP COLUMN` 的回滾檔。

只需要執行 `atlas migrate diff` 指令，告訴 Atlas：**「請比對我目前的實體資料庫，跟我的 `schema.sql` 有什麼差，並幫我自動寫好遷移檔！」**



```Bash
atlas migrate diff add_phone_to_users \
  --dir "file://migrations" \
  --to "file://schema.sql" \
  --dev-url "docker://mysql/8/dev"
```

> **什麼是 `--dev-url`**
> Atlas 它會在後台自動起一個暫時的 Docker MySQL 容器（或指定一個空資料庫），在裡面默默模擬執行、精準計算出差異 SQL，完全不會弄髒正在使用的開發資料庫！

#### 執行後得到的成果：

執行完後，Atlas 會自動在 `migrations/` 資料夾下生成以下檔案：

- **`20260714120000_add_phone_to_users.up.sql`** (自動寫好 `ALTER TABLE users ADD COLUMN phone...`)
    
- **`20260714120000_add_phone_to_users.down.sql`** (**自動寫好回滾 DDL** `ALTER TABLE users DROP COLUMN phone;`!)
    
- **`atlas.sum`** (自動生成的校驗檔，防止隊友亂改檔案，確保一致性)
    

### 第四步：一鍵套用變更 (Apply)

當想要同步最新欄位到本地資料庫時，只需執行：

```Bash
atlas migrate apply \
  --dir "file://migrations" \
  --url "mysql://root:password@localhost:3306/your_db"
```

Atlas 會在資料庫自動建立一張追蹤表，並將還沒跑過的 `up.sql` 套用進去。

### 第五步：發生 Bug？一鍵回滾 (Rollback)!

如果程式部署上去發現有問題，想要把最後一次套用的欄位復原（拔除）：



```Bash
atlas migrate rollback \
  --dir "file://migrations" \
  --url "mysql://root:password@localhost:3306/your_db"
```

Atlas 就會自動執行剛才寫好的 `down.sql`，安全地把 `phone` 欄位拔掉，讓資料庫毫無無痛退回上一步！

## 為什麼這個工具很適合？

1. **不用寫 PHP 代碼：** 不需要把 `parseStatements` 改成靜態還是介面，直接把資料庫變更交給系統級的 CLI 工具處理。
    
2. **Git 零衝突：** 因為生成的檔案都帶有 `20260714xxxxxx` 這種精準到秒的時間戳記，隊友同時加欄位也不會撞檔。
    
3. **終結手寫 SQL：** 只要修改 `schema.sql`，前後比對和回滾的 DDL 語法全由 Atlas 自動寫到好。