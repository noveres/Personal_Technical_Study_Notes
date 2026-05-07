OIDC Identity Provider 本身**沒有 permission policy**（它只是一個「信任設定」，告訴 AWS：「我承認這個外部 OIDC 提供者發的 token」）

### Console 操作

1. **登入 AWS Console**（用 root 或有 IAM 全權的 user，不需要切 region — IAM 是 global）
    
2. 左上搜尋 **IAM** → 進 IAM 服務
    
3. 左側選單 → **Identity providers** → 右上 **Add provider**
    
4. 填表單：

| 欄位            | 值                                           |
| ------------- | ------------------------------------------- |
| Provider type | OpenID Connect                              |
| Provider URL  | https://token.actions.githubusercontent.com |
| Audience      | sts.amazonaws.com                           |

5. 按 **Get thumbprint** 按鈕 — AWS 會自動抓 GitHub 的憑證指紋（顯示一串 hex）
    
6. 按 **Add provider**
    

完成後在 Identity providers 列表會看到一筆 `token.actions.githubusercontent.com`。

### 常見疑問

**Q1：要不要選 region？** 不用。IAM 全 global，Identity Provider 建立後所有 region 共用。

**Q2：要不要設 permission policy？** **不用**。IdP 沒有 policy，只是信任聲明。權限是在 **IAM Role** 上掛，下一步（Step 3）才會做。

**Q3：建立後要怎麼驗證？**

```bash
# 在你本機（如果有 AWS CLI）
aws iam list-open-id-connect-providers
```

預期看到 `token.actions.githubusercontent.com` 對應的 ARN：

```json
{
    "OpenIDConnectProviderList": [
        {
            "Arn": "arn:aws:iam::974066991761:oidc-provider/token.actions.githubusercontent.com"
        }
    ]
}
```

這個 ARN 之後 Step 3 的 trust policy 會用到