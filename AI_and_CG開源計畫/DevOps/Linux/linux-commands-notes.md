# Linux 指令學習筆記

整理 SOLE 專案部署過程中實際用到的 Linux / bash 指令。所有範例對應你在 EC2 上真的跑過的場景。

---
## 1. 檔案系統與導航

### 基本導航

```bash
pwd                          # Print Working Directory：印當前所在路徑
cd /home/ec2-user/           # Change Directory：切換目錄
cd ~                         # ~ 是家目錄縮寫（等同 /home/<你>）
cd ..                        # 上一層
cd -                         # 回上次的目錄
```

### 列檔案

```bash
ls                           # 簡單列檔名
ls -l                        # long format（含權限、大小、時間）
ls -la                       # 含隱藏檔（. 開頭）
ls -lh                       # human-readable 大小（KB/MB/GB 而非 bytes）
ls -la *.tar                 # 萬用字元，列所有 .tar 檔
```

`-l` 輸出範例解讀：
```
-rw-------  1 root   root  428  Apr 27 09:15  acme.json
│           │ │      │     │    │             │
│           │ │      │     │    │             └─ 檔名
│           │ │      │     │    └─ 修改時間
│           │ │      │     └─ 大小（bytes）
│           │ │      └─ 群組
│           │ └─ 擁有者
│           └─ 連結數
└─ 權限與類型（- = 一般檔，d = 資料夾，l = symlink）
```

### 建立與移動

```bash
mkdir foo                    # 建資料夾
mkdir -p a/b/c               # 連同中間缺的層一起建（非常常用）
touch acme.json              # 建空檔（或只更新時間戳）
mv file1.txt /other/path/    # 移動或重新命名
cp file1.txt file2.txt       # 複製
rm file.txt                  # 刪檔
rm -rf folder/               # 強制遞迴刪除（**極危險**，無 trash 可救）
```

### 找檔案

```bash
find / -name "bakery*" 2>/dev/null         # 從根目錄找名字以 bakery 開頭的東西
find /home -name "*.json"                  # 只在 /home 下找
find . -type f -size +100M                 # 當前目錄下找超過 100MB 的檔
```

> `2>/dev/null` 是把錯誤訊息丟掉（不然 find 會印一堆「無權限」訊息），詳見「資料流」章節。

---

## 2. 檢視與編輯檔案

### 看檔案內容

```bash
cat config.json              # 印出整個檔
head -30 file.log            # 印前 30 行（預設 10 行）
tail -50 file.log            # 印最後 50 行
tail -f /var/log/syslog      # 持續追蹤新增內容（很常用看 log）
less file.log                # 分頁瀏覽（可前後翻、搜尋 /keyword）
                             #   q 離開、空白鍵下一頁
```

### 找字串

```bash
grep "ERROR" file.log                       # 列出含 ERROR 的行
grep -i "error" file.log                    # 不分大小寫
grep -r "TODO" .                            # 遞迴搜尋當前目錄
grep -E "error|warning" file.log            # extended regex（用 |）
docker ps | grep bakery                     # 配合 pipe 過濾其他指令的輸出
ls *.tar | grep bakery                      # 找名字含 bakery 的 tar 檔
```

### 編輯檔案

```bash
nano file.txt                # 簡單易用的編輯器
                             #   Ctrl+O 存檔
                             #   Ctrl+X 離開
                             #   Ctrl+W 搜尋
                             #   Ctrl+K 刪整行

vi file.txt / vim file.txt   # 進階編輯器（學習曲線較陡，但功能強）
                             #   i 進入插入模式
                             #   Esc 回到指令模式
                             #   :wq 存檔離開
                             #   :q! 不存檔強制離開
```

### 寫入檔案（不開編輯器）

```bash
echo "hello" > file.txt              # > 蓋寫（檔案會被清空再寫入）
echo "world" >> file.txt             # >> 追加（保留原有內容）
echo "{}" > traefik/acme.json        # 你部署時清空 acme.json 用過

# Heredoc：寫多行
cat > /tmp/note.txt <<'EOF'
第一行
第二行
EOF
                                     # 'EOF' 加單引號表示「裡面 $ 不展開」
                                     # 不加引號 EOF 內 $VAR 會被替換成變數值

# tee：同時寫檔 + 印到螢幕（搭配 sudo 寫需要 root 權限的位置）
sudo tee /home/saratony/private/bakery/config.json > /dev/null <<'EOF'
{
    "database": { ... }
}
EOF
                                     # > /dev/null 把 stdout 丟掉（不然 tee 會印一份到螢幕）
```

> 為什麼要 tee 不用 echo `>`？因為 `sudo echo "x" > /file` 的 `>` 是**目前 shell** 做的（沒有 root 權限），不是 echo 做的。`sudo tee` 才是 tee 程式以 root 身份寫檔。

---

## 3. 權限管理

### 看權限

```bash
ls -l file.txt
# -rw-r--r--  1 ec2-user ec2-user  ...  file.txt
#  ↑↑↑↑↑↑↑↑↑
```

權限格式 `rwxrwxrwx` 分三段：

| 段 | 對誰 | 例子 |
|----|-----|------|
| 1~3 | owner（擁有者） | `rw-` = 讀寫不執行 |
| 4~6 | group（群組） | `r--` = 只讀 |
| 7~9 | others（其他人） | `r--` = 只讀 |

數字寫法：`r=4, w=2, x=1`，加總：
- `rw-` = 4+2 = 6
- `rwx` = 4+2+1 = 7
- `r--` = 4

所以：
- `chmod 600` = `rw-------`（只有 owner 能讀寫）→ acme.json、SSH key、密碼檔
- `chmod 644` = `rw-r--r--`（owner 讀寫，其他人只讀）→ 一般網頁檔
- `chmod 755` = `rwxr-xr-x`（owner 全權，其他人讀+執行）→ script、資料夾
- `chmod 700` = `rwx------`（只有 owner）→ 私人資料夾

### 改權限

```bash
chmod 600 acme.json                          # 你部署 Traefik 時用過（必須是 600）
chmod +x deploy.sh                           # 加 execute 權限
chmod -R 755 /var/www/html                   # 遞迴改整個資料夾
```

### 改擁有者

```bash
sudo chown www-data:www-data /var/www/html   # owner=www-data, group=www-data
sudo chown -R www-data /var/www/html         # 遞迴只改 owner
```

---

## 4. sudo 與使用者管理

### sudo

```bash
sudo <指令>                 # 用 root 身份執行
sudo -i                    # 切換到 root shell（之後指令都是 root）
sudo -u www-data <指令>     # 用其他使用者身份執行
exit                       # 離開 root shell
```

### 群組

```bash
groups                              # 看自己屬於哪些群組
groups ec2-user                     # 看特定使用者的群組
sudo usermod -aG docker ec2-user    # 把 ec2-user 加進 docker 群組
                                    #   -a = append（不加 -a 會把舊群組蓋掉，慘）
                                    #   -G = 副群組
# 加完群組必須登出再 SSH 進來才會生效
exit
ssh ec2-user@...                    # 重新登入

# 不想登出的暫時方案
newgrp docker                       # 切換群組（只影響當前 shell）
```

---

## 5. 資料流（>、|、2>&1）

Linux 每個程式有三條資料流：
- **stdin** (0)：標準輸入
- **stdout** (1)：標準輸出（一般訊息）
- **stderr** (2)：標準錯誤輸出（錯誤訊息）

### 重新導向

```bash
cmd > file.txt              # stdout 寫入 file（蓋寫）
cmd >> file.txt             # stdout 追加到 file
cmd 2> errors.log           # stderr 寫入 errors.log
cmd > out.log 2>&1          # stdout 跟 stderr 都寫到 out.log
cmd 2>/dev/null             # 把 stderr 丟掉（/dev/null 是黑洞）
cmd > /dev/null 2>&1        # 完全靜音
```

### Pipe

`|` 把前一個指令的 stdout 接到下一個指令的 stdin：

```bash
docker ps | grep bakery                     # docker ps 輸出餵給 grep 過濾
docker save bakery:latest | gzip > out.gz   # save 輸出 → gzip 壓縮 → 寫檔
gunzip -c file.tar.gz | docker load         # 解壓 → 餵給 docker load
ls -la | head -5                            # 看前 5 個
cat /var/log/syslog | tail -50 | grep ERROR  # 多層 pipe
```

### 連續執行多指令

```bash
cmd1 ; cmd2                  # 不論 cmd1 成功與否都跑 cmd2
cmd1 && cmd2                 # cmd1 成功才跑 cmd2（**最常用**）
cmd1 || cmd2                 # cmd1 失敗才跑 cmd2

docker stop bakery && docker rm bakery   # 你部署時用過
docker stop sole-app 2>/dev/null; docker rm sole-app 2>/dev/null  # 失敗也繼續
```

---

## 6. 壓縮與解壓縮

### tar / gzip

| 指令 | 作用 |
|------|------|
| `gzip file` | 壓縮 file → file.gz（原檔消失） |
| `gunzip file.gz` | 解壓縮 file.gz → file |
| `gzip -c file > out.gz` | 壓縮但保留原檔 |
| `gunzip -c file.gz > file` | 解壓但保留 .gz |

實際使用：
```bash
# 壓縮 docker save 的輸出（git bash）
docker save bakery:latest | gzip > bakery.tar.gz

# EC2 上解壓並 docker load
gunzip -c bakery.tar.gz | docker load
```

### tar（多檔打包）

```bash
tar cvf archive.tar folder/        # 建 .tar（c=create, v=verbose, f=file）
tar xvf archive.tar                # 解壓 .tar
tar czvf archive.tar.gz folder/    # 建 .tar.gz（z=gzip）
tar xzvf archive.tar.gz            # 解壓 .tar.gz
```

> docker save 出來的 `.tar` 是特殊格式，**用 `tar xvf` 解出來不會變成可用的鏡像**，必須用 `docker load`。

---

## 7. 系統與服務管理

### systemctl（systemd）

```bash
sudo systemctl status mysql           # 看服務狀態（running? stopped? failed?）
sudo systemctl start mysql            # 啟動
sudo systemctl stop mysql             # 停止
sudo systemctl restart mysql          # 重啟（你改 bind-address 後用過）
sudo systemctl reload mysql           # reload 設定（如果服務支援，比 restart 溫和）
sudo systemctl enable mysql           # 開機自動啟動
sudo systemctl disable mysql          # 取消開機自動啟動

sudo journalctl -u mysql -n 50        # 看服務 log（最近 50 行）
sudo journalctl -u mysql -f           # 持續追蹤
```

### 程序

```bash
ps aux | grep mysql                  # 看程序
top                                  # 即時系統監控（按 q 離開）
htop                                 # 比 top 漂亮（可能要 sudo apt install）

kill <PID>                           # 結束程序（送 SIGTERM，溫和）
kill -9 <PID>                        # 強制結束（送 SIGKILL，當卡死才用）
```

### Port

```bash
sudo lsof -i :80                     # 看誰佔用 80 port
sudo lsof -i :3306                   # MySQL port
sudo netstat -tulpn | grep :80       # 另一種寫法（可能要 sudo apt install net-tools）
```

---

## 8. 網路工具

### DNS 查詢

```bash
nslookup admin.hohosselect.com       # 簡單查詢，回 IP
dig admin.hohosselect.com            # 進階查詢，含詳細記錄
dig +short admin.hohosselect.com     # 只回答案，安靜
dig admin.hohosselect.com @8.8.8.8   # 指定用 Google DNS 查
```

### HTTP 測試

```bash
curl https://admin.hohosselect.com/login.php       # 抓內容印出來
curl -I https://...                                # 只看 response headers（你常用）
curl -k https://...                                # 忽略 SSL 憑證錯誤（測 staging 用）
curl -v https://...                                # verbose，看完整握手過程
curl -L https://...                                # 跟隨 redirect
curl -X POST -d "key=value" https://...            # 送 POST 帶資料
curl -H "Host: admin.openit.com.tw" https://...    # 自訂 header（測 Traefik 路由用）

# 你部署時實際用過的
curl -k -I https://admin.hohosselect.com/login.php
```

### TLS / SSL 檢查

```bash
# 看憑證 issuer 跟有效期
echo | openssl s_client -servername admin.hohosselect.com -connect admin.hohosselect.com:443 2>/dev/null \
  | openssl x509 -noout -issuer -subject -dates
```

### SSH

```bash
ssh -i ~/.ssh/key.pem ec2-user@1.2.3.4              # 用 key 登入
ssh -i key.pem ec2-user@1.2.3.4 'ls -la'            # 不進 shell，跑完指令就退出
ssh -i key.pem -p 2222 user@host                    # 指定 port

# Key 權限要對（太寬會被拒）
chmod 600 ~/.ssh/key.pem
```

### SCP（透過 SSH 傳檔）

```bash
# 從本地傳到 EC2
scp -i key.pem local-file.txt ec2-user@1.2.3.4:/home/ec2-user/

# 從 EC2 抓回本地
scp -i key.pem ec2-user@1.2.3.4:/home/ec2-user/file.log ./

# 整個資料夾（你部署 deploy/ 用過）
scp -i key.pem -r deploy/ ec2-user@1.2.3.4:/home/ec2-user/project/sole/
```

---

## 9. 套件管理（Ubuntu / Debian）

```bash
sudo apt update                       # 更新套件清單（必先做）
sudo apt upgrade                      # 升級已安裝套件
sudo apt install nano                 # 安裝
sudo apt remove nano                  # 移除
sudo apt search keyword               # 搜尋
apt list --installed                  # 列已安裝
apt list --installed | grep mysql     # 過濾
```

> Amazon Linux 用 `yum` 或 `dnf`，CentOS/RHEL 也是。指令類似但不同套件管理器。

---

## 10. MySQL CLI

雖然不算 Linux 指令本身，但你部署時大量用到：

```bash
sudo mysql -u root -p                              # 用 root 登入（會問密碼）
sudo mysql -u root -p -e "SHOW DATABASES;"         # 跑單一指令就退出
sudo mysql -u root -p -e "SELECT user, host FROM mysql.user;"
```

進去後常用：
```sql
SHOW DATABASES;                                     -- 列所有 schema
USE sole;                                           -- 切換 schema
SHOW TABLES;                                        -- 看 table
DESC users;                                         -- 看 table 結構
SELECT user, host FROM mysql.user;                  -- 看所有 MySQL 帳號

CREATE DATABASE openit CHARACTER SET utf8mb4;       -- 建 schema
CREATE USER 'openit'@'172.17.%.%' IDENTIFIED BY 'pwd';
GRANT ALL ON openit.* TO 'openit'@'172.17.%.%';
FLUSH PRIVILEGES;
EXIT;                                               -- 或 \q 離開
```

---

## 11. 編輯系統設定（你的部署實例）

### MySQL bind-address

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
# 找到 bind-address 那行，改成 0.0.0.0
sudo systemctl restart mysql

# 檢查
sudo grep bind-address /etc/mysql/mysql.conf.d/mysqld.cnf
```

### Hosts 檔（如果需要本機 DNS override）

```bash
sudo nano /etc/hosts
# 加一行：
# 1.2.3.4   admin.example.com
```

---

## 12. shell 常用快捷鍵

| 快捷鍵 | 作用 |
|--------|------|
| `Tab` | 自動補完檔名/指令 |
| `Ctrl+C` | 中斷當前程式 |
| `Ctrl+D` | 結束輸入 / 登出 shell |
| `Ctrl+L` | 清螢幕（同 `clear` 指令） |
| `Ctrl+R` | 反向搜尋歷史指令（強！） |
| `Ctrl+A` / `Ctrl+E` | 跳到行首 / 行尾 |
| `Ctrl+U` | 刪除游標前所有字 |
| `Ctrl+K` | 刪除游標後所有字 |
| `Ctrl+W` | 刪除前一個單字 |
| `↑` / `↓` | 上一個 / 下一個歷史指令 |
| `!!` | 重跑上一個指令 |
| `sudo !!` | 用 sudo 重跑上一個指令（很常用） |
| `history` | 看歷史指令清單 |

---

## 13. 你部署過程的指令時間軸

按你實際操作的順序整理：

### 階段 A：EC2 環境準備
```bash
# Docker 權限
sudo usermod -aG docker ec2-user
exit                                                # 重新 SSH 才生效
groups                                              # 確認有 docker

# MySQL 設定
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf       # bind-address 改 0.0.0.0
sudo systemctl restart mysql
sudo mysql -u root -p
# > CREATE USER 'saratony'@'172.17.%.%' IDENTIFIED BY '...';
# > GRANT ALL ON sole.* TO 'saratony'@'172.17.%.%';
# > FLUSH PRIVILEGES;

# Config 檔
sudo mkdir -p /home/saratony/private/bakery
sudo tee /home/saratony/private/bakery/config.json > /dev/null <<'EOF'
{ ... }
EOF
sudo chmod 600 /home/saratony/private/bakery/config.json
```

### 階段 B：傳檔
```bash
# 本機（Windows cmd 或 git bash）
docker save -o bakery.tar bakery:latest
scp -i key.pem bakery.tar ec2-user@<EC2-IP>:/home/ec2-user/project/sole/
scp -i key.pem -r deploy/ ec2-user@<EC2-IP>:/home/ec2-user/project/sole/
```

### 階段 C：載入鏡像
```bash
# EC2 上
cd /home/ec2-user/project/sole/
ls -la *.tar                                        # 確認檔案
docker load -i bakery.tar
docker images | grep bakery
docker tag bakery-app:latest bakery:latest         # 修正 tag
```

### 階段 D：啟動 stack
```bash
cd /home/ec2-user/project/sole/deploy/
touch traefik/acme.json
chmod 600 traefik/acme.json
docker stop sole-app 2>/dev/null; docker rm sole-app 2>/dev/null
docker compose up -d traefik bakery
docker compose ps
docker compose logs traefik --tail 30
```

### 階段 E：驗證
```bash
curl -k -I https://admin.hohosselect.com/login.php
nslookup admin.hohosselect.com
docker compose exec bakery php -r "..."
```

---

## 14. 常用指令速查卡

```bash
# 找東西
find / -name "x*" 2>/dev/null
grep -r "keyword" .
ls -la | grep keyword

# 看 log
tail -f /var/log/...
docker logs -f <container>
journalctl -u <service> -f

# 改權限
chmod 600 file
chmod 755 dir
chown -R user:group dir

# 服務
sudo systemctl restart <name>
sudo systemctl status <name>

# 網路
nslookup <domain>
curl -I <url>
sudo lsof -i :<port>

# 快速開個檔
nano <file>

# 批次執行
cmd1 && cmd2                # cmd1 成功才跑 cmd2
cmd1 ; cmd2                 # 不管成不成功都跑

# 安靜執行
cmd 2>/dev/null             # 不顯示錯誤
cmd > /dev/null 2>&1        # 完全靜音

# 重跑
!!                          # 上一個指令
sudo !!                     # 加 sudo 重跑
Ctrl+R                      # 搜尋歷史
```

---

## 15. 進階學習方向

- **shell scripting：** `.sh` 檔、變數、迴圈、function、`#!/bin/bash` shebang
- **cron：** 排程任務（如每天備份 DB）
- **rsync：** 比 scp 強大，支援增量同步、resume
- **ssh tunnel：** 透過 SSH 包一條加密通道（如本機連 RDS）
- **vim：** 學會基礎使用，比 nano 強大
- **環境變數：** `~/.bashrc`、`export VAR=value`
- **package management：** apt/yum/dnf 進階用法、PPA
- **iptables / firewalld：** 主機防火牆
- **screen / tmux：** 長時間 SSH 任務不怕斷線
