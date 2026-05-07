## 為什麼需要授權「網段」？

### 問題的根源：容器有自己的 IP

![[Pasted image 20260507101907.png]]

## host.docker.internal 是什麼

`host.docker.internal` 是 Docker 提供的一個**特殊 DNS 名稱**，意思是「**容器外面那台 host 主機**」。

```
容器內：
  127.0.0.1               → 容器自己 ❌
  host.docker.internal    → EC2 host ✅ ← 這才是 MySQL 跑的地方
```

當你在 `docker run` 加上 `--add-host=host.docker.internal:host-gateway`，Docker 就會在容器內部的 `/etc/hosts` 自動寫一行：

```
172.17.0.1   host.docker.internal
```

`172.17.0.1` 是 docker0 網橋的 IP，從容器看出去就是 host。所以 PHP 連 `host.docker.internal` 會被解析成 `172.17.0.1` → 走出容器 → 找到 EC2 host 上的 MySQL。

---

## 三種選項的比較

|寫法|在本機 Windows 跑 PHP|在容器裡跑 PHP|
|---|---|---|
|`127.0.0.1`|✅ 連到本機 MySQL|❌ 連到容器自己（沒 MySQL）|
|`host.docker.internal`|❌ 不存在這個名稱|✅ 連到 host MySQL|
|EC2 內網 IP（如 `10.0.1.5`）|❌ 不通|✅ 連到 host MySQL|

兩邊都沒辦法用同一個 host 名稱通用，所以**容器版本的 config.json 跟本機開發版本必須是不同檔案**。

- 本機 Windows：`C:/Project/.env/config.json` → host=`127.0.0.1`（你現有的，不動）
- EC2 容器：`/home/saratony/private/bakery/config.json` → host=`host.docker.internal`（新建的）

## 為什麼不直接寫 EC2 的內網 IP

例如你 EC2 內網是 `10.0.1.5`，理論上容器寫這個也能連。但有兩個缺點：

1. **不可攜**：換一台 EC2，IP 就變了，config.json 又得改
2. **多了一跳網路繞行**：流量會走 EC2 的網卡再回到自己，不如 docker0 網橋直接

`host.docker.internal` 是寫死的名稱，不論你部署到哪台 EC2 都一樣，container 內直接走網橋，這就是它最大的好處