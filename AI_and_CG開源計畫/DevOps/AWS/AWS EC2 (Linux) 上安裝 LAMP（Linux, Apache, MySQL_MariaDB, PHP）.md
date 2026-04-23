
### **AWS EC2 安裝 LAMP 環境筆記**

#### **一、 系統準備與套件安裝**

1. **切換 Root 權限**：執行 `sudo -s` 切換為 root 身份，以便擁有完整權限執行後續指令
2. **安裝 Apache (HTTPD)**：執行 `yum -y install httpd` 安裝網頁伺服器 
3. **安裝 PHP 套件**：安裝 PHP 及其相關套件以支援動態語言 
4. **安裝 MariaDB 資料庫**：安裝 `mariadb-server` 與 `mariadb` 用作系統資料庫 

#### **二、 AWS 安全組 (Security Group) 防火牆設定**

需回到 AWS EC2 控制台修改「Security Group」的 **Inbound Rules** 
1. **HTTP (TCP 80)**：開放來源為 `0.0.0.0/0` (Anywhere)，讓所有人都能連上網站 
2. **ICMP (Ping)**：若需測試連線，新增 `All ICMP - IPv4` 並選擇 `Echo Request` 
3. **MariaDB (TCP 3306)**：僅開放給特定的 IP（如：My IP）進行連線
4. **SSH**：僅開放給特定的 IP（如：My IP）進行連線
    

#### **三、 啟動服務與狀態檢查**

1. **啟動 Apache**：執行 `systemctl restart httpd` 啟動服務
2. **設定開機自啟**：執行 `systemctl enable httpd` 確保主機重啟後，Apache 會自動啟動 
3. **檢查狀態**：執行 `systemctl status httpd`，看到綠色的 `active (running)` 表示運作正常 

#### **四、 建立測試網頁與驗證**

1. **建立 index.html**：在系統中建立測試網頁檔案 
2. **取得 Public IP**：在 EC2 實例的「Details」分頁中找到 `Public IPv4 address`
3. **瀏覽器驗證**：在瀏覽器貼上該 IP，若能看見網頁內容，即表示建置成功

影片來源：[AWS9- 在Linux上安裝LAMP（linux、apache、mysql、php）| AWS EC2](https://www.google.com/search?q=https://youtu.be/GsgKhk9c9v4)