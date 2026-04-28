EC2 遠端連線簡短手冊，涵蓋了指令列（SSH）以及圖形化檔案傳輸（WinSCP）的操作步驟。

---

## **AWS EC2 遠端連線操作手冊**

### **一、 指令列連線 (Terminal / PowerShell)**
適合執行指令、安裝套件與環境設定。

1.  **開啟終端機**：進入金鑰檔案（`.pem`）所在的資料夾。
2.  **執行指令**：
   ```bash
ssh -i "SteamTrain_linux000_SSH_KEY.pem" ec2-user@ec2-43-213-64-157.ap-east-2.compute.amazonaws.com
   ```

* **使用者名稱**：Amazon Linux 預設為 `ec2-user`（非 `root`）。
* **金鑰對**："SteamTrain_linux000_SSH_KEY.pem"
* **IP 地址**：若公有 DNS 太長，也可直接使用公有 IP。
  
  每次重啟後 IP 都會變動，請去連線畫面找
  
  ![[Pasted image 20260424100748.png]]

---

### **二、 WinSCP 檔案傳輸 (GUI)**
適合上傳專案程式碼（如 Angular 編譯後的 `dist` 檔）或下載 Log 紀錄。

1.  **新建站台**：
    * **檔案通訊協定**：選取 **SFTP**。
    * **主機名稱**：輸入公有 IP (例如 `43.213.27.39`)。
    * **使用者名稱**：輸入 `ec2-user`。
    * **密碼**：留空。

2.  **匯入金鑰**：
    * 點擊 **「進階 (Advanced)」** 按鈕。
    * 在左側選單選擇 **SSH** > **驗證 (Authentication)**。
    * 在「私鑰檔案」欄位點擊 `...`，選取你的 `SteamTrain_linux000_SSH_KEY.pem`。
    * *註：WinSCP 會提示將 `.pem` 轉換為 `.ppk` 格式，請點擊「確定」並儲存。*

3.  **儲存與連線**：
    * 儲存站台名稱以便下次快速進入，點擊「登入」即可看到伺服器目錄。

---

### **三、 常見注意事項**

* **權限問題**：上傳檔案到 `/var/www/html` 等系統目錄時，若遇到 `Permission denied`，請先傳到 `/home/ec2-user`，再回 SSH 使用 `sudo cp` 移動檔案。
* **安全組設定**：如果連不上，請檢查 AWS Console 的 Security Group 是否已開啟 **Port 22**（SSH）與 **Port 80/443**（網頁服務）。
* **切換 root**：登入後若需最高權限，請執行：
    ```bash
    sudo -i
    ```
