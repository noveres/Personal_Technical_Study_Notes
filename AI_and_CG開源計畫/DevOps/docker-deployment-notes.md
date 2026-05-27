# Docker + EC2 + Traefik 部署學習筆記

整理自 SOLE 專案部署過程的核心概念與實作經驗。所有範例對應實際的 bakery / openit 部署架構。

---

## 1. Docker 基礎觀念

### 三個容易搞混的「名稱」

| 名稱 | 例子 | 用途 | 唯一性 |
|------|------|------|--------|
| **image name** | `bakery:latest` | 鏡像本身的識別 | 每個鏡像有自己的 ID，可有多個 tag |
| **service name** | `bakery:` (compose 第一層 key) | docker compose 內部識別 | 同一個 compose 檔內不重複 |
| **container name** | `--name bakery` | 跑起來的容器實例名 | **整台 docker host 不重複** |

實務上三個常設成一樣，記憶負擔最低。

### 鏡像建置流程

```
原始檔 + Dockerfile
       ↓ docker build
本地鏡像（image）
       ↓ docker run
容器（container）= image 的執行實例
```

每個 RUN / COPY 指令會產生一個 layer。Layer 會被 cache，只要前面 layer 沒變動就重用，這就是為什麼 Dockerfile 把「不常變動的指令」（如 apt-get install）放前面、「常變動的」（如 COPY 程式碼）放後面，能加速重 build。

---

## 2. 寫 Dockerfile（PHP + Apache 範例）

關鍵指令說明（對應本專案的 [Dockerfile](../Dockerfile)）：

```dockerfile
FROM php:8.2-apache              # 官方 PHP 鏡像，已內建 Apache + mod_php
                                  
RUN apt-get update && \           # 安裝編譯 PHP extension 需要的系統函式庫
    apt-get install -y libzip-dev libpng-dev ... && \
    rm -rf /var/lib/apt/lists/*  # 清掉 apt cache 縮小鏡像

RUN docker-php-ext-install \      # PHP 官方提供的擴充安裝工具
    pdo_mysql mbstring gd zip bcmath ...

RUN a2enmod rewrite               # 啟用 Apache mod_rewrite

# 讓 .htaccess 裡的 php_value 設定生效
RUN printf '<Directory /var/www/html>\n    AllowOverride All\n</Directory>\n' \
    >> /etc/apache2/apache2.conf

COPY . /var/www/html/             # 把專案複製進鏡像（包含 vendor/）
                                   
EXPOSE 80                         # 宣告容器內部監聽 80（純文件，沒有實際開 port）
```

### .dockerignore 的重要性

Docker build 會把整個 build context（專案資料夾）打包送給 daemon。沒設定 `.dockerignore` 的話，連 `.git`、`node_modules`、IDE 設定都會被傳，浪費時間 + 鏡像變大。

---

## 3. Docker 網路

### `127.0.0.1` 是相對的

`127.0.0.1` / `localhost` = 「我自己這台機器」，但「我」是誰要看你在哪：

```
EC2 host 的 127.0.0.1  =  EC2 自己
容器內的 127.0.0.1     =  容器自己（不是 EC2!）
```

### 容器要連到 host 上的服務怎麼辦？

兩個方法：

**方法 1：`host.docker.internal` + `--add-host`**

```yaml
# docker-compose.yml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

容器內的 `/etc/hosts` 會多一行 `172.17.0.1   host.docker.internal`，PHP 連這個名稱就能打到 EC2 host。

**方法 2：`--network host`**（不建議）

容器跟 host 共用網路 stack，`127.0.0.1` 變成 host 的 `127.0.0.1`。失去網路隔離，多個容器不能用同一 port。

### Docker bridge 網段

預設 docker0 網橋網段是 `172.17.0.0/16`。每個容器自動拿到這個網段內的 IP（如 `172.17.0.2`），**會變動**（容器重建後不一定一樣）。

---

## 4. Volumes（資料持久化）

容器砍掉重建就回到鏡像的乾淨狀態，**所有寫入會消失**。要保留的資料就要用 volume。

### 兩種 volume 寫法

| 類型 | 寫法 | 適合場景 |
|------|------|---------|
| **bind mount** | `/host/path:/container/path` | 把 host 上的特定檔案/資料夾掛進容器（如 config 檔、共享 source code） |
| **named volume** | `myvolume:/container/path` | Docker 管理的儲存空間（如使用者上傳檔、DB 資料） |

### bind mount 範例（本專案 config.json）

```yaml
volumes:
  - /home/saratony/private/bakery/config.json:/home/saratony/private/bakery/config.json:ro
  #  └─ host 路徑 ──┘                          └─ 容器內路徑 ──┘                        └─ ro = read-only
```

**為什麼掛 config.json 不放進鏡像：** DB 密碼這種敏感資訊不該打包到鏡像，否則 docker save 出去就是密碼洩漏。bind mount 讓密碼留在 host。

### named volume 範例（uploads 附件）

```yaml
services:
  bakery:
    volumes:
      - sole-uploads:/var/www/html/uploads

volumes:
  sole-uploads:        # 由 Docker 管理，實際存在 /var/lib/docker/volumes/sole-uploads/_data/
```

容器砍掉重建附件不會丟，因為 volume 是獨立物件。

### `external: true` 沿用既有 volume

```yaml
volumes:
  sole-uploads:
    external: true     # 不要自動建立，要找已存在的同名 volume
```

當 service 名稱改了（如 sole-app → bakery）但要保留舊資料時很實用。

---

## 5. MySQL 跨容器連線

### 兩個獨立的 MySQL 設定

要讓容器連到 host 上的 MySQL，需要：

1. **MySQL bind-address：** 預設只綁 `127.0.0.1`（拒絕網路連線），要改成 `0.0.0.0`
   ```bash
   sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
   # bind-address = 0.0.0.0
   sudo systemctl restart mysql
   ```

2. **MySQL 帳號授權：** 帳號是「**使用者名稱 + 來源 IP**」綁定的：
   ```sql
   CREATE USER 'saratony'@'172.17.%.%' IDENTIFIED BY '密碼';
   GRANT ALL ON sole.* TO 'saratony'@'172.17.%.%';
   FLUSH PRIVILEGES;
   ```

   `'saratony'@'127.0.0.1'` 跟 `'saratony'@'172.17.%.%'` 是**兩個不同的帳號**，密碼可以不同。

### 安全提醒

bind-address 改 `0.0.0.0` 是讓 MySQL 「願意接受所有介面的連線」，**真正擋外部的是 EC2 Security Group**。**SG 的 3306 port 絕對不能對外開放**，否則整個網際網路都能來敲 MySQL。

---

## 6. 鏡像傳輸（docker save / load）

### 流程

```
本機                                    EC2
docker build                             
docker save → .tar 或 .tar.gz       →   scp 上傳     →   docker load
                                                        
                                                        再 docker run / docker compose up
```

### 指令

**本機儲存（Linux/Mac/git bash 有 gzip）：**
```bash
docker save bakery:latest | gzip > bakery.tar.gz
```

**本機儲存（Windows cmd 沒 gzip）：**
```cmd
docker save -o bakery.tar bakery:latest
```
不壓縮，檔案會比較大（~1 GB vs ~268 MB）。

**EC2 載入（壓縮版）：**
```bash
gunzip -c bakery.tar.gz | docker load
```

**EC2 載入（未壓縮）：**
```bash
docker load -i bakery.tar
```

### 鏡像 tag 跟檔名沒關係

```bash
# 假設你執行
docker save -o whatever.tar bakery-app:latest

# load 後鏡像名是 bakery-app:latest（save 當下的 tag），不是 whatever
# 跟 .tar 檔名完全無關
```

如果鏡像 tag 對不上 compose 預期的名稱，用 `docker tag` 重貼別名（不會佔額外空間）：
```bash
docker tag bakery-app:latest bakery:latest
```

---

## 7. EC2 上的 Docker 權限

### Docker socket 權限問題

第一次跑 `docker ps` 常見錯誤：
```
permission denied while trying to connect to the Docker daemon socket
```

**原因：** `/var/run/docker.sock` 預設只給 `root` 跟 `docker` 群組，`ec2-user` 不在裡面。

**解法：** 把使用者加進 docker 群組：
```bash
sudo usermod -aG docker ec2-user
```
**必須登出再 SSH 進來**才會生效（群組 list 在 session 開始時就讀好了）。

### 安全提醒

加進 `docker` 群組 ≈ 給 root 權限（可以 `docker run -v /:/host` 讀寫整個檔案系統），所以多人共用主機要小心。EC2 自用環境沒問題。

---

## 8. EC2 Security Group

EC2 的「防火牆」概念。網際網路要能打到 EC2 的某個 port，必須**先在 SG 開放該 port**。

| Port | 用途 | 來源建議 |
|------|------|---------|
| 22 | SSH 管理 | 只開自己 IP（`x.x.x.x/32`） |
| 80 | HTTP（轉址 + Let's Encrypt 驗證） | `0.0.0.0/0` |
| 443 | HTTPS | `0.0.0.0/0` |
| 3306 | MySQL | **絕對不要開！** |

---

## 9. Docker Compose

### compose 解決什麼問題

`docker run` 一長串參數（network、volume、env、port、restart policy...）難維護。compose 用 yaml 集中定義，`docker compose up -d` 一鍵起 stack。

### 基本結構

```yaml
services:
  service-name-1:
    image: ...
    container_name: ...
    volumes: [...]
    networks: [...]
    labels: [...]
  
  service-name-2:
    ...

networks:                # 定義 network，多個 service 共用
  web:
    name: web

volumes:                 # 定義 named volume
  myvolume:
```

### 常用指令

| 指令 | 作用 |
|------|------|
| `docker compose up -d` | 啟動所有 service（背景執行） |
| `docker compose up -d <service>` | 只啟動指定 service |
| `docker compose ps` | 看 stack 內容器狀態 |
| `docker compose logs -f <service>` | 即時看 log |
| `docker compose restart <service>` | 重啟單一 service（不影響其他） |
| `docker compose down` | 停止並移除 stack（volume 預設保留） |
| `docker compose exec <service> <cmd>` | 在 service 容器內執行指令 |

---

## 10. Traefik 反向代理

### 為什麼需要反向代理

容器只監聽內部 80，外面流量要進來需要：
- 把多個域名分流到不同容器（Host header routing）
- TLS 終止（HTTPS 解密後內部走 HTTP）
- HTTP→HTTPS 轉址
- 自動申請 / 續約 SSL 憑證

Traefik 用 docker labels 配置這些，無需手刻 nginx config。

### 架構

```
Internet (80/443)
       ↓
   Traefik 容器（擁有 80/443）
       ↓ docker network
   bakery / openit 容器（內部 80，不對外）
```

### 兩個關鍵設定檔

**靜態設定 [traefik.yml](../deploy/traefik/traefik.yml)** — 容器啟動時讀取：
- entrypoints（在哪些 port 監聽：web=80、websecure=443）
- providers（從哪裡讀路由規則：docker、file）
- certificatesResolvers（ACME 設定）

**動態設定 [dynamic.yml](../deploy/traefik/dynamic.yml)** — 即時 reload：
- middlewares（轉址規則、IP allowlist 等）

### Labels 路由模式

每個要被 Traefik 服務的容器加 labels：

```yaml
labels:
  - "traefik.enable=true"                                                      # 啟用路由
  - "traefik.http.routers.bakery.rule=Host(`admin.hohosselect.com`)"            # 比對域名
  - "traefik.http.routers.bakery.entrypoints=websecure"                         # 在 443 生效
  - "traefik.http.routers.bakery.tls.certresolver=letsencrypt"                  # 用 ACME 拿憑證
  - "traefik.http.services.bakery.loadbalancer.server.port=80"                  # 轉到容器內的 80

  - "traefik.http.routers.bakery-http.rule=Host(`admin.hohosselect.com`)"       # HTTP→HTTPS 轉址 router
  - "traefik.http.routers.bakery-http.entrypoints=web"
  - "traefik.http.routers.bakery-http.middlewares=https-redirect@file"
```

### Routers / Services / Middlewares

```
Request → matches Router (by Host/Path)
              ↓ goes through Middlewares (transform/check)
              ↓
         Service (load balance to backend)
```

每個名稱（`bakery`、`bakery-http`）必須 unique，多個容器要用不同名稱。

---

## 11. Let's Encrypt ACME

### HTTP-01 Challenge 運作原理

1. Traefik 跟 Let's Encrypt 申請憑證
2. Let's Encrypt 給 Traefik 一個隨機 token
3. Traefik 把 token 放在 `http://你的域名/.well-known/acme-challenge/<token>`
4. Let's Encrypt 從外面 HTTP 訪問該 URL，若拿到 token 就證明你**控制這個域名**
5. 簽發憑證

**所以 80 port 必須能從網際網路訪問，不能只開 443。**

### staging vs production

Let's Encrypt 有 rate limit（每 7 天每域名最多 5 次失敗 / 50 個新憑證）。設定錯誤反覆失敗會被鎖。

最佳實踐：**第一次部署用 staging server 驗證流程**，沒問題再切 production。

```yaml
# traefik.yml
certificatesResolvers:
  letsencrypt:
    acme:
      caServer: https://acme-staging-v02.api.letsencrypt.org/directory  # staging
      # 切 production 就把上面這行註解掉（預設是 production）
```

切換步驟：
```bash
# 1. 編輯 traefik.yml 註解掉 caServer
# 2. 清掉 staging 憑證
echo "{}" > traefik/acme.json
chmod 600 traefik/acme.json

# 3. 重啟
docker compose restart traefik
```

### acme.json 權限

**必須 600**（只有 owner 可讀寫），否則 Traefik 拒絕啟動。

```bash
chmod 600 acme.json
```

### 90 天自動續約

Let's Encrypt 憑證效期 90 天。Traefik 在剩 30 天時自動續約，不用人為介入（前提是 acme.json 持久化、80 port 持續開放）。

---

## 12. 多 app 共存於同一 EC2

### 模式

```
同一 Traefik
   ├── admin.hohosselect.com → bakery 容器
   └── admin.openit.com.tw   → openit 容器
        ↓
   同一台 host MySQL（不同 schema）
```

### 加新 app 的步驟

1. **DNS：** 加 A record 指到 EC2 IP
2. **DB：** 建新 schema、新使用者、`GRANT ON 新schema.* TO 'user'@'172.17.%.%'`
3. **config.json：** 在 host 上新增（路徑跟既有 app 隔開）
4. **鏡像：** docker save / scp / docker load
5. **compose：** 新增 service，給唯一的 router/service labels
6. **啟動：** `docker compose up -d <新service>`

`sole-uploads`、`bakery` 等既有 service 不會被影響 — compose 只重啟「有變動」的 service。

### labels 命名要 unique

```yaml
# bakery
- "traefik.http.routers.bakery.rule=..."
- "traefik.http.services.bakery.loadbalancer.server.port=80"

# openit（router/service 名稱必須跟 bakery 不同）
- "traefik.http.routers.openit.rule=..."
- "traefik.http.services.openit.loadbalancer.server.port=80"
```

---

## 13. 故障排查對照表

| 症狀 | 可能原因 | 處理 |
|------|---------|------|
| `permission denied... docker.sock` | 沒在 docker 群組 | `sudo usermod -aG docker ec2-user` 後重新 SSH |
| `Cannot connect to MySQL: Connection refused` | bind-address 還是 127.0.0.1 | 改 my.cnf 為 0.0.0.0 後重啟 mysql |
| `Host '172.17.x.x' is not allowed` | MySQL 沒授權容器網段 | `CREATE USER ... '@'172.17.%.%'` + `GRANT` |
| `Access denied for user 'xxx'` | config.json 密碼跟 MySQL 帳號密碼不一致 | 檢查兩邊密碼 |
| `設定檔不存在: config.json` | bind mount 失敗 | 確認 host 路徑檔案存在 + 容器內路徑跟程式預期一致 |
| `Conflict. The container name is already in use` | 同名容器已存在 | `docker rm -f <名稱>` 後重跑 |
| `port is already allocated` | 別的容器或 host 服務佔用該 port | `sudo lsof -i :80` 找誰佔，或改用其他 port |
| Traefik `acme.json permission too open` | 權限不對 | `chmod 600 acme.json` |
| ACME `unable to obtain certificate` | DNS 沒生效 / 80 port 不通 | `nslookup`、外部 `curl http://域名/` |
| 觸發 Let's Encrypt rate limit | staging 沒先跑、設定錯誤反覆失敗 | 等 7 天，回 staging caServer 驗證流程 |

---

## 14. 常用指令速查

### 鏡像
```bash
docker images                        # 列出本機鏡像
docker tag <src> <dst>               # 建立別名
docker rmi <image>                   # 刪除
docker save <img> -o file.tar        # 匯出
docker load -i file.tar              # 載入
docker save <img> | gzip > file.tar.gz   # 壓縮匯出（需 git bash）
gunzip -c file.tar.gz | docker load      # 壓縮載入
```

### 容器
```bash
docker ps                            # 看執行中容器
docker ps -a                         # 含已停止的
docker logs <name> --tail 50         # 看 log
docker logs -f <name>                # 持續看 log
docker exec -it <name> bash          # 進容器 shell
docker stop <name>                   # 停止
docker rm <name>                     # 移除
docker rm -f <name>                  # 強制移除（含執行中）
```

### Compose
```bash
docker compose up -d                 # 起所有 service
docker compose up -d <service>       # 起指定 service
docker compose ps                    # 看狀態
docker compose logs -f <service>     # 看 log
docker compose restart <service>     # 重啟 service
docker compose down                  # 停整 stack
docker compose exec <service> <cmd>  # 在 service 內執行
```

### Volume / Network
```bash
docker volume ls                     # 列出 volumes
docker volume inspect <name>         # 看 volume 詳情（包含 mount path）
docker volume rm <name>              # 刪 volume（會丟資料！）
docker network ls                    # 列出 networks
docker network inspect <name>        # 看 network 詳情（含容器列表）
```

---

## 15. 本專案部署成果

最終架構：

```
admin.hohosselect.com    admin.openit.com.tw
        ↓                        ↓
        └────── EC2 SG (80/443) ─┘
                    ↓
              [Traefik :80 :443]
                    ↓ docker network "web"
        ┌───────────┴───────────┐
   [bakery :80]            [openit :80]
        ↓                        ↓
        └─── host.docker.internal ───┘
                    ↓
        [MySQL on EC2 host (3306)]
        ├── sole schema (bakery 用)
        └── openit schema (openit 用)
```

**關鍵檔案：**
- [`Dockerfile`](../Dockerfile) — bakery 鏡像建置
- [`deploy/docker-compose.yml`](../deploy/docker-compose.yml) — stack 定義
- [`deploy/traefik/traefik.yml`](../deploy/traefik/traefik.yml) — Traefik 靜態設定
- [`deploy/traefik/dynamic.yml`](../deploy/traefik/dynamic.yml) — middlewares
- `/home/saratony/private/bakery/config.json` (EC2) — bakery DB 設定
- `/home/saratony/private/openit/config.json` (EC2) — openit DB 設定
