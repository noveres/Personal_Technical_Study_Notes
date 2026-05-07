 SOLE 部署健康檢查指南

對應腳本：[`deploy/healthcheck.sh`](../deploy/healthcheck.sh)

每次部署完自動跑、或手動驗證部署狀態時用。涵蓋 10 個面向，從 `.env` 設定一路檢查到 TLS 憑證。

---

## 用法

```bash
cd /home/ec2-user/project/sole/deploy
./healthcheck.sh             # 跑全部
./healthcheck.sh --quick     # 跳過 DNS / 外部 HTTPS 測試（DNS 沒生效時用）
./healthcheck.sh --verbose   # 顯示每個 check 的細節輸出（失敗時 debug 用）
./healthcheck.sh --help      # 看用法
```

退出碼：
- `0` → 全通過或只有警告
- `1` → 至少一個檢查失敗

---

## 三種狀態

| 符號 | 意義 | 處理 |
|------|------|------|
| ✓ 綠色 | 通過 | 沒事 |
| ! 黃色 | 警告 | 還能用，但建議修（如 staging 憑證、DNS 沒生效） |
| ✗ 紅色 | 失敗 | 必須修，腳本 exit 1，CI 會 fail |

---

## 10 個檢查項目

### [1] 前置條件

| 檢查 | 失敗常見原因 |
|------|-------------|
| `deploy/.env` 存在 | EC2 上沒建 `.env`，從 `.env.example` 複製 |
| `ECR_REGISTRY` 變數格式合法 | 拼錯 account ID 或 region |
| `traefik/acme.json` 存在 | 沒 `touch` 過、或被 `rm` 掉 |
| `acme.json` 權限是 600 | 用 `chmod 600 traefik/acme.json` 修 |
| `acme.json` **不是資料夾** | Docker 在 host 沒檔時會建成資料夾，要 `sudo rm -rf` 後重建 |

### [2] Compose 設定

| 檢查 | 失敗常見原因 |
|------|-------------|
| `docker compose config` 解析 OK | YAML 語法錯，看錯誤訊息 |
| 鏡像 URL 含 ECR 前綴 | `.env` 沒被讀到 → `${ECR_REGISTRY}` 展開成空字串，image 變成 `/bakery:latest` |

### [3] Container 狀態

檢查 `traefik` / `bakery` / `openit` 三個容器都是 running。

失敗時跑 `docker compose logs <service>` 看死因。常見：
- 鏡像拉不下來（ECR token 過期或 IAM role 沒掛）
- volume mount 失敗（host 路徑不存在）
- 容器啟動指令錯誤

### [4] Traefik 啟動 log

掃 Traefik log 找這些**不該出現的關鍵字**：

| 關鍵字 | 含義 | 修正 |
|------|------|------|
| `permissions ... are too open` | acme.json 權限太寬 | `chmod 600 traefik/acme.json` |
| `nonexistent certificate resolver` | router labels 提到的 resolver 在 traefik.yml 找不到 | 確認 `traefik.yml` 有 `certificatesResolvers.letsencrypt` 區塊 |
| `HTTP challenge is not enabled` | acme resolver 沒設 httpChallenge | 加上 `httpChallenge.entryPoint: web` |

也檢查**該出現的成功訊息**：
- `Starting provider *acme.Provider` → ACME 機制起來了
- `Configuration loaded from file` → 靜態設定有讀到

### [5] DB 連線

從 bakery / openit 容器內跑 PHP 連 DB。**腳本會根據錯誤訊息給對應提示：**

| 錯誤訊息 | 意義 | 修正 |
|---------|------|------|
| `Access denied for user '...'` | 帳號存在但密碼錯 | `ALTER USER 'xxx'@'yyy' IDENTIFIED BY '<config.json 裡的密碼>'` |
| `Host '...' is not allowed to connect` | MariaDB 沒授權該網段 | `CREATE USER 'xxx'@'172.%.%.%'` + `GRANT` |
| `Connection refused` | MariaDB 沒跑、或 bind-address 仍是 127.0.0.1 | 改 my.cnf 為 `0.0.0.0` 後 `systemctl restart mariadb` |
| `設定檔不存在: config.json` | bind mount 失敗 | 確認 host 路徑檔案存在、容器內路徑跟 PHP 預期一致 |

### [6] 容器內部 Apache

從容器內 `curl localhost/login.php`。**繞過 Traefik，純測 Apache + PHP**。

| 結果 | 含義 |
|------|------|
| 200/301/302/401/403 | Apache + PHP 正常（即使 401/403 代表程式有跑，只是要登入） |
| 000 / 連不上 | 容器內 Apache 沒啟動，看 `docker compose logs <service>` |
| 500 | PHP 程式錯誤，看 PHP error log |

### [7] Traefik 路由（繞 DNS）

```bash
curl -k -I -H "Host: admin.hohosselect.com" https://localhost/login.php
```

**用 Host header 假扮域名**，Traefik 看 header 路由。**不需要 DNS 生效**。

| 結果 | 含義 |
|------|------|
| 200/301/302 | Traefik 認得域名 + 路由到對的容器 + TLS 握手成功 |
| 404 | Host rule 沒 match，檢查 `docker-compose.yml` 的 `Host(\`...\`)` 規則 |
| 503 | Traefik 找不到 backend service，labels 設定錯 |
| 連不上 | Traefik 沒在 443 監聽 |

### [8] DNS 解析（`--quick` 跳過）

用 `dig @8.8.8.8` 查兩個域名，繞過本機 DNS cache。

| 結果 | 含義 |
|------|------|
| 回 EC2 IP | DNS 已生效 ✅ |
| NXDOMAIN | DNS 還沒生效，等或檢查 A record |

### [9] 外部 HTTPS（`--quick` 跳過）

真的打 `https://admin.hohosselect.com/login.php`。**完整模擬使用者**。

DNS 沒生效時這項會 fail，但只是警告（不是錯誤）。

### [10] TLS 憑證（`--quick` 跳過）

抓憑證 issuer 看是哪種：

| Issuer | 狀態 |
|--------|------|
| Let's Encrypt R3/R10/R11 | ✅ 正式憑證，瀏覽器會綠鎖 |
| `(STAGING) Pretend Pear` | ⚠️ Staging 憑證，瀏覽器會警告但加密 OK，**驗證流程通過後切 production** |
| 拿不到 | ⚠️ DNS 沒通或 TLS 握手失敗 |

---

## 手動執行各項檢查

腳本是把下面這些指令包起來。如果**不想跑整個腳本**、或想單獨 debug 某一塊，可以挑對應指令貼到 EC2 SSH 跑。

> 全部假設你已經 `cd /home/ec2-user/project/sole/deploy`

### 對應 [1] 前置條件

```bash
# .env 存在且有 ECR_REGISTRY
test -f .env && grep ECR_REGISTRY .env

# acme.json 存在、是檔案、權限是 600
ls -la traefik/acme.json
stat -c '%a %n' traefik/acme.json    # 應印 600 traefik/acme.json
```

### 對應 [2] Compose 設定

```bash
# 解析 compose 並看 image URL（變數有展開應看到完整 ECR URL）
docker compose config | grep image
```

### 對應 [3] Container 狀態

```bash
docker compose ps
```

### 對應 [4] Traefik 啟動 log

```bash
# 不該出現的訊息（看到代表有問題）
docker compose logs traefik 2>&1 | grep -iE "permissions.*are too open|nonexistent certificate resolver|HTTP challenge is not enabled"

# 該出現的訊息
docker compose logs traefik 2>&1 | grep "Starting provider \*acme.Provider"
docker compose logs traefik 2>&1 | grep "Configuration loaded from file"

# 看完整 ACME 流程
docker compose logs traefik 2>&1 | grep -iE "acme|certificate" | tail -10
```

### 對應 [5] DB 連線

```bash
# bakery
docker compose exec -T bakery php -r "
require '/var/www/html/database.php';
try {
    Database::getInstance();
    echo \"OK\n\";
} catch (Exception \$e) {
    echo \"FAIL: \" . \$e->getMessage() . \"\n\";
}"

# openit
docker compose exec -T openit php -r "
require '/var/www/html/database.php';
try {
    Database::getInstance();
    echo \"OK\n\";
} catch (Exception \$e) {
    echo \"FAIL: \" . \$e->getMessage() . \"\n\";
}"
```

### 對應 [6] 容器內部 Apache

```bash
docker compose exec bakery curl -sI http://localhost/login.php | head -3
docker compose exec openit curl -sI http://localhost/login.php | head -3
```

### 對應 [7] Traefik 路由（用 Host header 繞 DNS）

```bash
curl -k -I -H "Host: admin.hohosselect.com" https://localhost/login.php
curl -k -I -H "Host: admin.openit.com.tw" https://localhost/login.php
```

### 對應 [8] DNS 解析

```bash
# 用 Google DNS 查（避開本機 cache）
dig nslookup @8.8.8.8 admin.hohosselect.com
dig nslookup @8.8.8.8 admin.openit.com.tw

# 也可以用 Cloudflare
dig nslookup @1.1.1.1 admin.hohosselect.com
```

### 對應 [9] 外部 HTTPS

```bash
# 從 EC2 自己打域名（DNS 沒通會 fail）
curl -k -I --max-time 10 https://admin.hohosselect.com/login.php
curl -k -I --max-time 10 https://admin.openit.com.tw/login.php
```

### 對應 [10] TLS 憑證

```bash
# 看憑證 issuer
echo | openssl s_client -servername admin.hohosselect.com -connect admin.hohosselect.com:443 2>/dev/null \
  | openssl x509 -noout -issuer -subject -dates

echo | openssl s_client -servername admin.openit.com.tw -connect admin.openit.com.tw:443 2>/dev/null \
  | openssl x509 -noout -issuer -subject -dates
```

預期看到（DNS 通且憑證已拿到）：
```
issuer=C = US, O = Let's Encrypt, CN = R3
subject=CN = admin.hohosselect.com
notBefore=Apr 30 ...
notAfter=Jul 29 ...   ← 90 天有效，到期前 30 天 traefik 會自動續約
```

---

## 一次跑全部（懶人版 one-liner）

```bash
echo "=== [1] 前置條件 ===" && test -f .env && grep ECR_REGISTRY .env && stat -c '%a %n' traefik/acme.json && \
echo "=== [2] Compose ===" && docker compose config | grep image && \
echo "=== [3] Containers ===" && docker compose ps && \
echo "=== [4] Traefik log ===" && docker compose logs traefik 2>&1 | grep -iE "starting provider \*acme|configuration loaded" && \
echo "=== [5] bakery DB ===" && docker compose exec -T bakery php -r "require '/var/www/html/database.php'; try { Database::getInstance(); echo \"OK\n\"; } catch (Exception \$e) { echo \"FAIL: \".\$e->getMessage().\"\n\"; }" && \
echo "=== [5] openit DB ===" && docker compose exec -T openit php -r "require '/var/www/html/database.php'; try { Database::getInstance(); echo \"OK\n\"; } catch (Exception \$e) { echo \"FAIL: \".\$e->getMessage().\"\n\"; }" && \
echo "=== [6] Apache internal ===" && docker compose exec bakery curl -sI http://localhost/login.php | head -1 && docker compose exec openit curl -sI http://localhost/login.php | head -1 && \
echo "=== [7] Traefik routing ===" && curl -k -sI -H "Host: admin.hohosselect.com" https://localhost/login.php | head -1 && curl -k -sI -H "Host: admin.openit.com.tw" https://localhost/login.php | head -1 && \
echo "=== [8] DNS ===" && dig nslookup @8.8.8.8 admin.hohosselect.com && dig nslookup @8.8.8.8 admin.openit.com.tw && \
echo "=== [9] HTTPS external ===" && curl -k -sI --max-time 10 https://admin.hohosselect.com/login.php | head -1 && curl -k -sI --max-time 10 https://admin.openit.com.tw/login.php | head -1
```

---

## 怎麼跟 CI/CD 整合

[`.github/workflows/deploy.yml`](../.github/workflows/deploy.yml) 已經串好：

1. **scp step** — 把最新 `healthcheck.sh` 同步到 EC2
2. **ssh step 最後** — 自動執行 healthcheck.sh
3. **healthcheck 失敗** → SSH step exit 1 → deploy job 紅色

每次 `git push main` 都會自動跑這個 healthcheck。**deploy job 變綠 = 部署 + 健康檢查都通過**。

---

## 常見失敗劇本

### 劇本 A：第一次部署，DNS 還沒設

```
[1] 前置條件        ✓ 全綠
[2] Compose 設定    ✓ 全綠
[3] Container 狀態  ✓ 全綠
[4] Traefik log     ✓ 全綠
[5] DB 連線         ✓ 全綠
[6] 容器內 Apache   ✓ 全綠
[7] Traefik 路由    ✓ 全綠（用 Host header 測）
[8] DNS 解析        ! 全黃（NXDOMAIN）
[9] 外部 HTTPS      ! 全黃
[10] TLS 憑證       ! 全黃（拿不到）

結果：✓ 14 通過  ! 6 警告  ✗ 0 失敗
```
**這是好結果** — 部署本身完美，只差 DNS。等 DNS 生效後 [8][9][10] 會自動轉綠。

### 劇本 B：`.env` 弄丟

```
[1] 前置條件
  ✗ deploy/.env 不存在
    → 從 .env.example 複製並填入實際值

[2] Compose 設定
  ✗ 鏡像 URL 沒有 ECR 前綴（變數沒展開）
```

**修法：** 在 EC2 上 `cp .env.example .env && nano .env` 填入實際 `ECR_REGISTRY`。

### 劇本 C：DB 密碼跟 MariaDB 不同步

```
[5] DB 連線
  ✗ bakery → DB 認證失敗
    → config.json 的密碼跟 MariaDB 對不上，ALTER USER 重設
  ✗ openit → DB 認證失敗
```

**修法：**
```sql
ALTER USER 'saratony'@'172.%.%.%' IDENTIFIED BY '<config.json 真實密碼>';
FLUSH PRIVILEGES;
```

### 劇本 D：Compose 換 subnet 後 GRANT 不對

```
[5] DB 連線
  ✗ bakery → DB host 不被允許
    → MariaDB 沒有對應網段的 GRANT，或 skip-name-resolve 沒生效
```

**修法：** 用 `docker network inspect web` 看實際 subnet，補對應 `GRANT`。或乾脆授權 `172.%.%.%` 涵蓋所有 docker subnet。

### 劇本 E：traefik 重啟沒生效

```
[4] Traefik 啟動 log
  ✗ Traefik log 出現: nonexistent certificate resolver
```

**修法：** `docker compose restart` 不夠（static config 不會重讀），要：
```bash
docker compose stop traefik
docker compose rm -f traefik
docker compose up -d traefik
```

---

## 為什麼有些是「警告」不是「失敗」

刻意設計給**漸進式部署**用。一個還沒切到 production 的環境：
- DNS 沒生效 → 警告（不該卡部署）
- 外部 HTTPS 連不上 → 警告（DNS 沒通的副作用）
- 還在 staging 憑證 → 警告（提醒記得切，但目前能跑）

這些黃色項目**不會讓 CI 變紅**，但會在 log 裡明顯標示，提醒之後處理。

---

## 想加新的檢查項目

`deploy/healthcheck.sh` 最後一段是 summary，新檢查在前面加 section 即可：

```bash
section 11 "新檢查"

if <條件>; then
  ok "新項目通過"
elif <警告條件>; then
  warn "新項目有問題" "提示如何修"
else
  fail "新項目失敗" "提示如何修"
fi
```

`ok` / `warn` / `fail` 三個 helper 會自動處理計數跟顏色。
