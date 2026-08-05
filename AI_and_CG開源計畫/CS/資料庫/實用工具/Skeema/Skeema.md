要上手 **Skeema** 其實非常直覺，因為它不像其他 Migration 工具需要你學習一堆指令或宣告語法，它**只跟純 SQL 檔案打交道**。

以下為您整理一份**最實用的 Skeema 快速上手與多人協作教學**

---

## 1. 安裝 Skeema CLI

Skeema 是一個用 Go 寫成的單一執行檔，安裝非常簡單：

* **macOS (使用 Homebrew):**
```bash
brew install skeema/tap/skeema

```


* **Linux:**
```bash
curl -sSL https://skeema.io/install.sh | sh

```


* **Windows (Scoop 或直接手動下載):**
```powershell
scoop bucket add skeema https://github.com/skeema/scoop-bucket.git
scoop install skeema
```

**備註**：Windows 要付費，所以使用

## 在 Windows 透過 WSL / Docker 免費使用 Skeema

繞過 Windows 原生執行檔的限制**即可。因為 Skeema 的 **Linux/macOS 版本是 100% 免費且開源的**，我們可以利用 Windows 的虛擬化環境來執行它。
### 途徑 A：使用 WSL2 (Windows Subsystem for Linux) — 最推薦

WSL2 是 Windows 內建的 Linux 子系統，運作效率極高，且能直接存取 Windows 本地的檔案。

1. 在 Windows 搜尋「啟用或關閉 Windows 功能」，勾選「適用於 Linux 的 Windows 子系統」並重啟。
    
2. 在 Microsoft Store 安裝 **Ubuntu**。
    
3. 打開 Ubuntu 終端機，像在 Linux 一樣直接用 `brew` 或官方腳本安裝免費的 Skeema：
    
    
    ```Bash
    # 使用 apt 或是直接下載編譯好的 Linux 免費版
    wget https://github.com/skeema/skeema/releases/download/v1.11.1/skeema_1.11.1_linux_amd64.tar.gz
    tar -xvf skeema_1.11.1_linux_amd64.tar.gz
    sudo mv skeema /usr/local/bin/
    ```
    
4. 現在你就可以在 WSL 裡免費快樂地使用 `skeema diff` 與 `skeema push` 了！它一樣可以連線到你 Windows 本機或遠端的資料庫。
    

---

## 2. 第一步：初始化專案（從既有資料庫匯出）

如果資料庫已經有一些資料表，你不需要自己動手寫 DDL。Skeema 可以**一鍵把當前結構全部拉下來變成 SQL 檔案**。

### 執行初始化指令

```bash
skeema init --host 127.0.0.1 --port 3306 --user root --password your_password --schema your_db_name -d database_schema

```

> **參數說明：**
> * `-d database_schema`：指定要把 SQL 檔案導出到哪一個資料夾。
> 

執行完後，你會在 `database_schema` 資料夾下看到：

1. **`.skeema`**：Skeema 的環境設定檔，裡面記錄了你的資料庫連線資訊。
2. **一堆 `.sql` 檔案**：資料庫裡有多少張表，就會自動生成多少個 `table_name.sql`，裡面只含最純粹的 `CREATE TABLE` 語法。

`.skeema` 設定檔內容大約長這樣：

```ini
generator=skeema
host=127.0.0.1
port=3306
user=root
password=your_password
schema=your_db_name
```

---

## 3. 日常開發：修改、比對與套用

有了這套設定後，未來不論是你還是隊友，要修改資料庫欄位時，再也不用連進資料庫手動去點 GUI 了。

### 步驟 A：直接在編輯器修改 `.sql`

打開你本地專案中的 SQL 檔案。例如，你想在 `users.sql` 裡加上 `phone` 欄位與一個 Trigger（免費版 Skeema 完整支援 Trigger 與 Stored Procedures！）：

```sql
-- database_schema/users.sql
CREATE TABLE `users` (
  `id` int NOT NULL AUTO_INCREMENT,
  `name` varchar(100) NOT NULL,
  `email` varchar(100) NOT NULL,
  `phone` varchar(20) DEFAULT NULL, -- 新增這一行
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 還可以順便在下方直接加上 Trigger！
CREATE TRIGGER `before_insert_users` BEFORE INSERT ON `users` FOR EACH ROW
SET NEW.name = TRIM(NEW.name);

```

### 步驟 B：比對差異 (Diff)

修改完檔案後，在該資料夾目錄下執行：

```bash
skeema diff

```

Skeema 會自動去撈你 `.skeema` 設定的實體資料庫，並與你修改後的 `.sql` 檔案做比對。它會在終端機上印出它**自動幫你計算好的 DDL 語句**：

```sql
-- On host 127.0.0.1:3306, db your_db_name:
-- Altering table users
ALTER TABLE `users` ADD COLUMN `phone` varchar(20) DEFAULT NULL;
-- Creating trigger before_insert_users
CREATE TRIGGER `before_insert_users` BEFORE INSERT ON `users` FOR EACH ROW SET NEW.name = TRIM(NEW.name);

```

### 步驟 C：安全套用 (Push)

如果確認 `diff` 印出的 SQL 完全正確，直接執行：

```bash
skeema push

```

Skeema 就會把這些變更真正寫入你的資料庫中，此時你的本機資料庫欄位與 Trigger 就同步完成了！

---

## 4. 多人協作與 Docker 環境配置

當項目是在 Docker 容器化環境運行，而資料庫是在實體主機、或者另一個 Docker 容器時，建議在專案中這樣配置：

### Git 版本控制結構

將 `.skeema` 檔案與所有的 `.sql` 檔案直接 Commit 進 Git 倉庫中。

> ⚠️ **安全提示：**
> 為了避免把資料庫密碼（`password`）提交到 Git，請在 `.skeema` 中把 `password` 刪除，改在每位開發者的電腦本機，將密碼設定為環境變數 `SKEEMA_PASSWORD`，或者在執行指令時用 `-p` 互動式輸入。

```

當隊友早上開工 `git pull` 拿到你修改後的 `.sql` 檔案後，他只需要在本地目錄輸入：

```bash
skeema push
```

