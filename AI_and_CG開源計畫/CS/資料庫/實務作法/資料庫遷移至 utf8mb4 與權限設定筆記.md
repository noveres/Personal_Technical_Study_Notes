	
將資料庫環境升級為 `utf8mb4，並建立新的資料庫 `bakery_utf8mb4`。
## 選擇「建立新資料庫」而非「原地修改」

1. 安全性高：保留原始資料庫，若轉換過程出錯可隨時恢復。

2. 資料完整性：避免原地轉換時因編碼衝突導致資料損毀。

3. 環境潔淨：確保新建立的物件從基礎就使用正確的編碼設定。

## 操作步驟

#### SSL 進入 主機後

1. 建立資料庫

```SQL
CREATE DATABASE bakery_utf8mb4 
CHARACTER SET utf8mb4 
COLLATE utf8mb4_unicode_ci;
```


2. 設定權限 (解決常見 ERROR 1133)

| **Host 設定** | **適用範圍**       | **優先級** |
| ----------- | -------------- | ------- |
| `localhost` | 僅本機 socket 連線  | 最高      |
| `127.0.0.1` | 僅 IP 127.0.0.1 | 次之      |
| `172.%.%.%` | 匹配 172 開頭的 IP  | 普通      |
| `%`         | **所有 IP**      | **最低**  |


在授權前，需先確認 MySQL 使用者表中的 Host 設定。

```sql
SELECT user, host FROM mysql.user WHERE user = 'saratony';
```


如果已經有 `'saratony'@'%'`，則無需額外建立 `'127.%'`。

```sql
-- 給予已存在的 '%' 帳號權限
GRANT ALL PRIVILEGES ON bakery_utf8mb4.* TO 'saratony'@'%';
-- 給予 'localhost' 權限
GRANT ALL PRIVILEGES ON bakery_utf8mb4.* TO 'saratony'@'localhost';
-- 刷新權限
FLUSH PRIVILEGES;
```


### 遷移時 選擇完整匯出

- **Create Dump in a Single Transaction (self-contained file only)**：這會啟動一個單一交易來執行備份，確保備份文件中的資料在時間點上是一致的（即在備份期間資料沒有被修改）。
- **Include Create Schema**：勾選此項會在匯出的 SQL 檔案中包含 `CREATE DATABASE` 或 `CREATE SCHEMA` 的指令，以便在還原時自動建立資料庫結構


  ## 1. 做後處理

直接用 `mysqldump` 匯出的原始 SQL 檔，若要拿到不同主機、不同帳號的環境去匯入，會踩到三類問題。三者都不會在自家環境出現，但會在「換環境部署」時整批爆炸：

### 1.1 `CREATE DATABASE` 帶 `latin1` 預設字元集

mysqldump 加上 `--databases` 旗標時，會在檔頭寫入：

```sql
CREATE DATABASE  IF NOT EXISTS `xxx` /*!40100 DEFAULT CHARACTER SET latin1 COLLATE latin1_swedish_ci */;
USE `xxx`;
```

問題：

- 目標環境如果 DB 還沒建，會被建成 `latin1` 預設字元集。
- 雖然底下每張 `CREATE TABLE` 都明確寫 `DEFAULT CHARSET=utf8mb4`，table 本身是 utf8mb4 沒錯。
- 但**未來 `ALTER TABLE` 加新欄位時若沒指定 CHARSET，會繼承 DB 層的 `latin1`**，中文寫進去就變亂碼。
- 這是長期慢性問題，初期不會發作，等到某次線上加欄位之後才被發現，回頭修很痛。

### 1.2 物件帶有 `DEFINER` 子句

`CREATE PROCEDURE` / `CREATE FUNCTION` / `CREATE TRIGGER` / `CREATE VIEW` / `CREATE EVENT` 五種物件被匯出時，會帶上建立者資訊：

```sql
CREATE DEFINER=`saratony`@`127.0.0.1` PROCEDURE `all_bom_products`() ...
```

問題：

- 目標環境如果沒有 `saratony@127.0.0.1` 這個 MySQL 使用者，匯入會失敗或需要超級權限。
- 跨主機部署時 `127.0.0.1` 這個來源主機限定也會變成限制，常常匯入後 procedure 雖然建出來但任何帳號都呼叫不到（`The user specified as a definer ... does not exist`）。
- 不同 client / server charset 設定下，反引號裡的使用者名稱有時會解析失敗。

### 1.3 預設不會匯出 routines / events

`mysqldump` 預設**不會**匯出 PROCEDURE / FUNCTION / EVENT，要明確加 `--routines --events` 才會帶。TRIGGER 預設會匯，但如果有人手動加 `--skip-triggers` 也會掉光。

問題：

- 第一次匯出常常用 `mysqldump db > dump.sql` 不帶旗標，匯出來的檔案表面上看起來正常，**沒人會發現裡面是空的**（沒有錯誤訊息，只有商業邏輯整批失蹤）。
- 等到部署上線發現 BOM 計算、用料需求、登入流程整批壞掉才會回頭追，浪費很多時間。

---

## 2. `*.nodefiner.sql` 是什麼

是「**已做過後處理、可以直接匯入新環境**」的成品檔。命名來自三項處理中最顯眼的一項（去除 DEFINER），但實際上至少包含以下處理：

| 處理項目                                 | 處理方式                                  |
| ------------------------------------ | ------------------------------------- |
| 刪掉檔頭的 `CREATE DATABASE ... latin1`   | 移除前兩行（含 `USE \`xxx`;`）                |
| 清掉所有 `DEFINER=\`使用者`@`主機`` 子句        | 全域字串取代為空                              |
| 確認 routines / triggers / events 都有匯出 | 重匯時加 `--routines --events --triggers` |

`sql/openit.nodefiner.sql`（2026-05-04 從 `bakery` 來源產生的成品）是這個慣例的範例。

---

## 3. 這次處理紀錄（2026-05-14）

### 來源檔
原始狀態：
- 43,188,269 bytes / 4,351 行
- 73 張 table + 資料
- 23 個 PROCEDURE，**全部都 `DEFINER=\`saratony`@`127.0.0.1``**
- 0 個 TRIGGER（缺）
- 0 個 VIEW / FUNCTION / EVENT
- 檔頭含 `CREATE DATABASE \`bakery_ite` ... latin1`與`USE `bakery_ite`;`

### 處理動作

用 `sed` 在單一指令裡完成兩件事：

- `1,2d` ：刪掉第 1、2 行（`CREATE DATABASE` 與 `USE`）。
- `s/.../...//g` ：把所有 `DEFINER=\`saratony`@`127.0.0.1`` 子句（注意前面有個空白）取代為空字串。

### 輸出檔

`sql/bakery_ite.nodefiner.sql`

驗證結果：

|檢查項|處理前|處理後|
|---|---|---|
|檔案大小|43,188,269 B|43,183,075 B|
|行數|4,351|4,349|
|`DEFINER=` 出現次數|23|**0**|
|`^CREATE PROCEDURE` 開頭的行|0|**23**|
|開頭第一行|`CREATE DATABASE ...`|`-- MySQL dump 10.13 ...`|

處理前後 procedure 從 `CREATE DEFINER=\`saratony`@`127.0.0.1` PROCEDURE `xxx`()`變成`CREATE PROCEDURE `xxx`()`，與`openit.nodefiner.sql` 的格式一致。

---

## 4. 已知缺項：4 個 TRIGGER

來源（`bakery` DB）有以下 4 個 trigger，本次 `bakery_ite` 來源**完全沒有**：


```sql
check_bom_detail_before_insert    BEFORE INSERT ON bom_detail check_bom_detail_before_update    BEFORE UPDATE ON bom_detail check_bom_header_before_insert    BEFORE INSERT ON bom_header check_bom_header_before_update    BEFORE UPDATE ON bom_header
```

可能原因：

1. `bakery_ite` 資料庫上根本沒建這些 trigger（最可能）。
2. 匯出時加了 `--skip-triggers`（少見）。

**影響**：BOM 資料完整性檢查會掉光，bom_detail / bom_header 表沒有資料驗證 trigger，前端錯資料能直接寫進去。

**確認方法**：到 `bakery_ite` 對應的 DB 上執行：


```sql
SHOW TRIGGERS FROM bakery_ite;
```


若回傳空集合，代表來源 DB 本來就沒這些 trigger；要的話需要從 `bakery` DB 抓出 trigger 定義後手動建立。

---

## 5. 旗標說明：

|旗標|作用|
|---|---|
|`--routines`|匯出 PROCEDURE / FUNCTION（預設不匯）|
|`--events`|匯出 EVENT（預設不匯）|
|`--triggers`|明確要求匯出 TRIGGER（預設已啟用，明寫避免被人關掉）|
|`--no-create-db`|**不要寫 `CREATE DATABASE ... latin1` 開頭**，省掉手動刪兩行|
|`--single-transaction`|InnoDB 表用一致性快照匯出，避免鎖表|
|`--default-character-set=utf8mb4`|連線字元集設成 utf8mb4，避免中文字被截斷或轉碼錯誤|

匯出後仍需做最後一步：清掉 DEFINER。


## 關鍵知識點

• MySQL 權限匹配：
  • 當多個 Host 匹配時，MySQL 會選擇「最精確」的設定。
  • `localhost` (最精確) > `127.0.0.1` > `172.%` > `%` (最模糊)。
• 權限錯誤 (1133)：
  • 出現此錯誤通常是因為你在 `GRANT` 時指定了不存在的 `user@host` 組合。
  • 請務必先執行 `SELECT user, host FROM mysql.user;` 確認目標帳號是否存在。
• 關於 `%`：
  • 在 MySQL 中，`%` 為萬用字元，代表所有來源主機。擁有 `%` 權限的帳號，原則上已經涵蓋了所有 IP 連線。
