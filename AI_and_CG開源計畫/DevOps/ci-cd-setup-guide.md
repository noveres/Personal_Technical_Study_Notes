# AWS ECR + GitHub Actions CI/CD 設定指南

部署到 EC2 的 CI/CD 設定。對應 [.github/workflows/deploy.yml](../.github/workflows/deploy.yml)。

## 整體架構

```
git push (main)
     ↓
GitHub Actions
   ├── (1) OIDC 跟 AWS 拿 short-lived token
   ├── (2) docker buildx 建鏡像
   └── (3) push 到 ECR (private registry)
     ↓
GitHub Actions (deploy job)
   └── SSH 進 EC2
         └── docker compose pull (從 ECR 拉) → docker compose up -d
```

---

## 設定步驟總覽

| #   | 在哪      | 做什麼                                      | 一次性？  |
| --- | ------- | ---------------------------------------- | ----- |
| 1   | AWS ECR | 建 `bakery`、`openit` repository           | ✅ 一次  |
| 2   | AWS IAM | 建 GitHub OIDC identity provider          | ✅ 一次  |
| 3   | AWS IAM | 建 `github-actions-ecr-role`（CI 用）        | ✅ 一次  |
| 4   | AWS IAM | 建 `ec2-ecr-pull-role`（EC2 用）並掛上 EC2      | ✅ 一次  |
| 5   | EC2     | 安裝 aws CLI、設定 docker 認證                  | ✅ 一次  |
| 6   | GitHub  | 設 repo Secrets（AWS Account ID、SSH key 等） | ✅ 一次  |
| 7   | EC2     | 建 `deploy/.env`、首次拉鏡像                    | ✅ 一次  |
| 8   | 之後      | `git push main` 自動 build + deploy        | 🔁 持續 |

---

## 前置作業

開個記事本記下這些值，之後會用到：

```bash
# 在 EC2 主機（裝過 aws CLI 的話）
aws sts get-caller-identity --query Account --output text   # →  12 位帳號 ID

# 地區 / region（例：ap-northeast-1）
# 從 EC2 console 右上角看，或者執行：
aws configure get region
```

| 變數               | 範例               |
| ---------------- | ---------------- |
| AWS Account ID   | `123456789012`   |
| Region           | `ap-northeast-1` |
| GitHub username  | `your-username`  |
| GitHub repo name | `倉庫名稱`           |
| EC2 公網 IP        | `1.2.3.4`        |

---

## Step 1：建 ECR Repository

### Console 做法

AWS Console → **ECR** → 切到 region → **Create repository**：
![[Pasted image 20260521115143.png]]
![[Pasted image 20260521115108.png]]

| 欄位               | 值                          |
| ---------------- | -------------------------- |
| Visibility       | Private                    |
| Repository name  | `倉庫名稱`                     |
| Tag immutability | Disabled（讓 `latest` 可以被覆蓋） |
| Image scanning   | Enable（免費，自動掃 CVE）         |
| Encryption       | AES-256                    |

按 Create。

### CLI 做法（可選）

```bash
aws ecr create-repository --repository-name 私有庫名稱 --region 地區 --image-scanning-configuration scanOnPush=true
```

### 驗證

```bash
aws ecr describe-repositories --region 地區
```

應看到兩個 repo URL，格式：`<ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/<repo>`

---

## Step 2：建 GitHub OIDC Identity Provider

讓 AWS 信任 GitHub Actions 的身份。**全 AWS 帳號只需要建一次**（之後其他 repo 也共用）。

### Console 做法

AWS Console → **IAM** → **Identity providers** → **Add provider**：

| 欄位            | 值                                             |
| ------------- | --------------------------------------------- |
| Provider type | OpenID Connect                                |
| Provider URL  | `https://token.actions.githubusercontent.com` |
| Audience      | `sts.amazonaws.com`                           |
|               |                                               |

按 **Get thumbprint**，再 **Add provider**。

### CLI 做法

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

> Thumbprint 是 GitHub OIDC 憑證的 SHA-1，AWS 文件偶爾會更新，照他們最新文件為準。

---

## Step 3：建 GitHub Actions IAM Role

### 3-1. 建 Role

	AWS Console → **IAM** → **Roles** → **Create role**：

- Trusted entity type: **Web identity**
- Identity provider: 選剛建的 `token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`
- GitHub organization: 個人的 GitHub username
- GitHub repository: `sole`（或你 repo 的名字）
- GitHub branch: `main`

按下一步，跳過第二步，因為預設的設置限制過多
所以我們創建後，手寫設定檔案 在 3-2 。

### 3-2. 附加 Permission Policy

選 **Create inline policy**
![[Pasted image 20260521122540.png]]


切到 JSON 模式
![[Pasted image 20260521122604.png]]

貼上：

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Principal": {
				"Federated": "arn:aws:iam::974066991761:oidc-provider/token.actions.githubusercontent.com"
			},
			"Action": "sts:AssumeRoleWithWebIdentity",
			"Condition": {
				"StringEquals": {
					"token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
				},
				"StringLike": {
					"token.actions.githubusercontent.com:sub": [
						"arn:aws:ecr:ap-east-2:974066991761:repository/{ERC儲存庫名稱}",
						
					]
				}
			}
		}
	]
		}
```


"arn:aws:ecr:{地區}:{ 12 位帳號 ID }:repository/{ERC儲存庫名稱}",

	Policy name: `ecr-push-policy`，按 Create。

### 3-3. Role 命名

Role name: **`github-actions-ecr-role`**（**這個名字必須跟 [deploy.yml](../.github/workflows/deploy.yml) 第 28 行一致**）

按 Create role。

### 3-4. 修正 Trust Policy

進到剛建的 role → **Trust relationships** → **Edit trust policy**，確認 `Condition` 區塊長這樣（取代 `<github-username>` 跟 `<repo-name>`）：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                },
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:<github-username>/<repo-name>:ref:refs/heads/main"
                }
            }
        }
    ]
}
```

> ⚠️ `StringLike` + `repo:.../main` **這個條件絕對不能漏**。沒設的話，**任何 GitHub repo 都能 assume 這個 role**，等於 ECR 對全網開放（資安風險）。

---

## Step 4：建 EC2 IAM Role 並附加到 instance

### 4-1. 建 EC2 Instance Profile Role

IAM → Roles → Create role：
- Trusted entity type: **AWS service**
- Service: **EC2**
- Permission policy: 選 **`AmazonEC2ContainerRegistryReadOnly`**（AWS 內建）
- Role name: **`ec2-ecr-pull-role`**

按 Create。

### 4-2. 把 Role 附加到 EC2 instance

EC2 Console → 找 EC2 實例 instance → **Actions** → **Security** → **Modify IAM role** → 選 `ec2-ecr-pull-role` → Update。
![[Pasted image 20260521123750.png]]
![[Pasted image 20260521123716.png]]

![[Pasted image 20260521123732.png]]


> 這步驟**不需要重啟 instance**，幾秒內就生效。

---

## Step 5：EC2 上設定 docker 自動登入 ECR

### 5-1. 安裝 aws CLI

```bash
# Amazon Linux 2023 / Ubuntu 都可以這樣裝
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version          # 確認安裝成功
```

### 5-2. 測試 EC2 IAM role 通了

```bash
aws sts get-caller-identity --region ap-northeast-1
```

應該看到 EC2 role 的 ARN，類似：
```json
{
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/ec2-ecr-pull-role/i-xxxxx"
}
```

### 5-3. 測試 ECR 認證

```bash
aws ecr get-login-password --region ap-northeast-1 \
  | docker login --username AWS --password-stdin \
    YOUR_ACCOUNT_ID.dkr.ecr.ap-northeast-1.amazonaws.com
```

看到 `Login Succeeded` 就 OK。

> 預設 ECR token 有效期 12 小時。GitHub Actions 每次 deploy 會在 SSH 進來時重新登入，所以不用擔心過期問題。

---

## Step 6：GitHub Repo 設 Secrets

GitHub repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

![[Pasted image 20260521123927.png]]
![[Pasted image 20260521123959.png]]
新增以下 secrets：

| Secret 名稱        | 值                         | 用途                        |
| ---------------- | ------------------------- | ------------------------- |
| `AWS_ACCOUNT_ID` | `123456789012`（12 位帳號 ID） | 拼接 IAM role ARN 與 ECR URL |
| `EC2_HOST`       | EC2 公網 IP                 | SSH 連線目標                  |
| `EC2_USER`       | `ec2-user`                | SSH 使用者                   |
| `EC2_SSH_KEY`    | **整個 .pem 檔內容**           | SSH 認證                    |

### `EC2_SSH_KEY` 要怎麼貼

打開你本機的 .pem 檔（例：`c:\Users\YOU\my-key.pem`）整個複製，包含：

```
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA...（很多很多行）
-----END RSA PRIVATE KEY-----
```

**包含 BEGIN / END 那兩行**，貼到 secret 值欄位。

---

## Step 7：EC2 上首次部署準備

### 7-1. 建 `deploy/.env`

```bash
ssh -i key.pem ec2-user@<EC2_IP>
cd /home/ec2-user/project/sole/deploy

# 從 example 複製
cp .env.example .env

# 編輯填入實際值
nano .env
```

`.env` 內容（取代 YOUR_ACCOUNT_ID 跟 region）：
```
ECR_REGISTRY=123456789012.dkr.ecr.ap-northeast-1.amazonaws.com
```

### 7-2. 確認 docker-compose.yml 抓得到 .env

```bash
cd /home/ec2-user/project/sole/deploy
docker compose config | grep image
```

預期看到：
```
image: 123456789012.dkr.ecr.ap-northeast-1.amazonaws.com/bakery:latest
image: 123456789012.dkr.ecr.ap-northeast-1.amazonaws.com/openit:latest
```

如果還是 `${ECR_REGISTRY}` 沒展開，代表 `.env` 沒被讀到，檢查檔名與位置。

---

## Step 8：第一次跑！

### 在 GitHub 觸發 workflow

觸發條件全都定義在專案 `.github/workflows/` 目錄下 YAML 檔案的 **`on:`** 區塊中

這裡有兩種兩種方式：
- 推一個 commit 到 main：`git push origin main`
  ```yml
  on:

  push:

    branches: [main]

  workflow_dispatch:
  ```
- 手動觸發：GitHub repo → Actions → 選 workflow → Run workflow

### 觀察進度

GitHub repo → **Actions** tab，會看到正在跑的 workflow。點進去看每個 step 的 log。

![[Pasted image 20260521124332.png]]

成功的話會有兩個 job：
- ✅ `build-and-push` (約 2~5 分鐘)
- ✅ `deploy` (約 30 秒~1 分鐘)

### 在 EC2 確認

```bash
ssh -i key.pem ec2-user@<EC2_IP>
docker images | grep bakery               # 看到 ECR URL 的 image
docker compose -f /home/ec2-user/project/sole/deploy/docker-compose.yml ps
                                           # 應全部 Up
curl -I https://admin.hohosselect.com/login.php   # 200 OK
```

---

## 之後的開發流程

寫完功能：
```bash
git add .
git commit -m "feat: 新增功能 X"
git push origin main
```

→ GitHub Actions 自動 build + push + deploy → EC2 在 3~5 分鐘內跑新版本。

完全不用手動 ssh / docker save / scp。

---

## 故障排查

###  `Error: Could not assume role`（OIDC 階段）

| 可能原因 | 處理 |
|---------|------|
| trust policy 的 `repo:xxx/yyy` 寫錯 | 檢查 IAM role 的 Trust relationships |
| 沒指定 branch（`refs/heads/main`） | 確認 condition 的 sub claim |
| OIDC provider thumbprint 過期 | 從 [GitHub Actions OIDC docs](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect) 拿最新值 |

###  `pull access denied`（EC2 拉鏡像時）

| 可能原因 | 處理 |
|---------|------|
| EC2 IAM role 沒附 `AmazonEC2ContainerRegistryReadOnly` | 確認 step 4-1 |
| EC2 沒附 IAM role | 確認 step 4-2 |
| ECR 跟 EC2 不同 region | docker login 跟 compose 都用同一 region |

### SSH 步驟卡住或拒絕連線

| 可能原因 | 處理 |
|---------|------|
| `EC2_SSH_KEY` 沒包含 BEGIN/END 行 | 重新複製整個 .pem 內容 |
| EC2 Security Group 沒開 22 給 GitHub | GitHub Actions runner 是動態 IP，22 port 要對 0.0.0.0/0 開放（或用 SSM 替代 SSH） |
| EC2_HOST 是私網 IP | 改用公網 IP |

###  `docker compose pull` 一直拉不到

```bash
# 在 EC2 手動測
cd /home/ec2-user/project/sole/deploy
aws ecr get-login-password --region ap-northeast-1 \
  | docker login --username AWS --password-stdin \
    YOUR_ACCOUNT_ID.dkr.ecr.ap-northeast-1.amazonaws.com
docker compose pull
```

如果手動跑成功但 GitHub 自動跑失敗，看是不是 `${{ env.AWS_REGION }}` / `${{ secrets.AWS_ACCOUNT_ID }}` 沒展開到對的值。

---

## 安全加固（可選）

### 1. 加強 trust policy 限制

只允許 main branch + tag：
```json
"StringLike": {
    "token.actions.githubusercontent.com:sub": [
        "repo:<user>/sole:ref:refs/heads/main",
        "repo:<user>/sole:ref:refs/tags/v*"
    ]
}
```

### 2. SSH 改用 SSM Session Manager

不用開 22 port，AWS SSM agent 接手 SSH。GitHub Actions 用 `aws-actions/aws-ssm-send-command-action` 就能跑遠端指令。
- 優點：完全沒開 SSH port，IAM 控管所有存取
- 缺點：設定多一層

### 3. ECR lifecycle policy 自動清舊鏡像

設定保留最近 10 個 commit SHA tag，舊的自動刪：
```json
{
    "rules": [
        {
            "rulePriority": 1,
            "description": "Keep last 10 images by tag",
            "selection": {
                "tagStatus": "tagged",
                "tagPatternList": ["*"],
                "countType": "imageCountMoreThan",
                "countNumber": 10
            },
            "action": { "type": "expire" }
        }
    ]
}
```

放到 ECR repo → Lifecycle policy。

---

## 整理：會碰到的所有檔案

| 檔案 | 在哪 | 用途 |
|------|------|------|
| [.github/workflows/deploy.yml](../.github/workflows/deploy.yml) | repo | CI/CD 流程定義 |
| [Dockerfile](../Dockerfile) | repo 根目錄 | bakery 鏡像建置（不用改） |
| [deploy/docker-compose.yml](../deploy/docker-compose.yml) | repo + EC2 | stack 定義（image 已改用 ECR URL） |
| [deploy/.env.example](../deploy/.env.example) | repo | env 樣板（commit 進 git） |
| `deploy/.env` | **僅 EC2** | 實際 ECR registry URL（不 commit） |
| GitHub Secrets | GitHub repo settings | AWS_ACCOUNT_ID、EC2_HOST、EC2_USER、EC2_SSH_KEY |
