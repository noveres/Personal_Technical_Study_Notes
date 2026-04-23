
### 1. 登入與選擇區域 (Region)

1. **登入 AWS 控制台**：搜尋並進入 **EC2** 服務介面。
2. **選擇資料中心**：在頁面右上角選擇地理位置較近的區域。以台灣使用者為例，建議選擇 **東京 (Tokyo)**，連線速度較快。

>
>連結 ： https://aws.amazon.com/tw/
>


![[Pasted image 20260423075440.png]]
### 2. 啟動虛擬機實例 (Launch Instance)

在左側選單點選 **Instances (實例)**，接著點擊右上角的 **Launch instances (啟動實例)**

1. 搜尋並選擇 EC2
![[Pasted image 20260423075846.png]]


2. 那顆橘色的 "啟用實例"

![[Pasted image 20260423075926.png]]


### 3. 設定基本資訊

![[Pasted image 20260423080116.png]]

- **名稱與標籤 (Name)**：為伺服器取一個辨識名稱（例如：`l00`）。
    
- **作業系統映像檔 (AMI)**：
    
    - 建議選擇 **Amazon Linux**。這是 AWS 針對其硬體優化過的系統，效能較佳。
        
    - 版本建議選擇最新的 **Amazon Linux 2023 AMI**。
        
- **實例類型 (Instance Type)**：
    
    - 免費帳號建議選用 **t2.micro**。
        
    - 規格：1 核 CPU、1GB 記憶體。
        
    - 免費方案通常提供每月 750 小時的免費額度，持續 12 個月。
        

### 4. 設定登入金鑰 (Key Pair)

1. 點擊 ：Create new key pair
![[Pasted image 20260423080703.png]]

 ![[Pasted image 20260423080856.png]]

- 金鑰是連線至伺服器的「鑰匙」，非常重要。
    
- 點選 **Create new key pair (建立新金鑰對)**。
    
- **格式選擇**：
    
    - 若使用 Mac 或 Linux 的內建終端機，選 **.pem**。
        
    - 若 Windows 使用者打算使用 **PuTTY** 工具連線，建議直接選 **.ppk** 格式。
        
- 下載後請妥善保存，金鑰遺失將無法連入伺服器
	*兩個金鑰可以互相轉換，但我們還是盡量一開始就選對*
  
 個人推薦連線工具：
	
1. **WinSCP**（Windows Secure Copy）是一款在 Windows 環境下非常經典且受歡迎的**開源圖形化檔案傳輸客戶端**。

### 5. 網路與防火牆設定 (Network Settings)

![[Pasted image 20260423081102.png]]

- **Auto-assign public IP**：設為 **Enable**。若要架設網站，必須有公開 IP 讓外界連線。
    
- **安全性群組 (Security Group)**：即防火牆設定。
    
    - **SSH 規則**：預設開啟 Port 22。
        
    - **安全性**：將來源 IP 從「Anywhere (0.0.0.0/0)」改為 **「My IP」**。這樣僅限當前IP可以進入這台虛擬機，防止駭客攻擊。
        
     _註：若未來更換網路環境導致 IP 變動，需回此處更新規則。_
        

### 6. 儲存硬碟設定 (Storage)

- 預設通常為 8GB，可依需求調整（例如：10GB）。
    
- 硬碟格式建議選擇 **gp3**（效能與價格的平衡點）。
    

### 7. 完成啟動與連線

- 點擊 **Launch instance**，等待約 2 分鐘。
    
- 回到 EC2 首頁，點擊左側欄位。返回實例列表，確認實例狀態顯示為 **Running** 且狀態檢查皆為綠色。
    ![[Pasted image 20260423081658.png]]
- **取得公開 IP**：在實例詳情 (Details) 中複製 **Public IPv4 address**。
    
- **進行連線**：
  1. 使用連線工具（如 PuTTY），輸入 IP 並載入剛才下載的金鑰即可連入伺服器。
  2. 或是進入實例，選擇連線，可以叫出瀏覽器的小黑窗口

	![[Pasted image 20260423081938.png]]
