# Day 1: 第一次觀察 Juju Relation（mysql + self-signed-certificates）

## 目的

用 CLI 指令親眼觀察 `juju integrate` 做了什麼 — 兩個不認識彼此的 charm 之間如何透過 relation data bag 自動交換資料。

## Prerequisites

- Day 1 Juju bootstrap 完成（model `openstack` 存在）

## 為什麼是 mysql + self-signed-certificates

原本計畫用 `mediawiki + mysql`，但 **`mediawiki` charm 只支援 Ubuntu 11.10-14.04**，跟我們的 jammy (22.04) 不相容。

其他候選也踩了坑（詳見 §Gotchas）。最後找到的組合：

| Charm | 角色 | Base | 提供/需要 的 relation |
|---|---|---|---|
| `mysql` (Canonical, `8.0/stable`) | DB | ubuntu@22.04 | **requires `certificates` (tls-certificates)** |
| `self-signed-certificates` (Canonical Telco, `1/stable`) | 產 self-signed TLS cert | ubuntu@22.04 | **provides `certificates` (tls-certificates)** |

interface 名稱 `tls-certificates` 對上 — 可以 integrate。

---

## Step 1: 部署 mysql

```bash
juju deploy mysql
```

- 預設走 `8.0/stable` channel，revision 444
- 這是 Canonical 為 OpenStack 維護的 Charmed MySQL VM operator（跟 `mysql-innodb-cluster` 一脈相承）

⏱ 約 3 分鐘：LXD 開 container → 裝 mysql 8.0 → 初始化 → 狀態 `active`

## Step 2: 部署 self-signed-certificates

```bash
juju deploy self-signed-certificates --channel=1/stable
```

⏱ 約 1 分鐘：LXD 開 container → 裝 Python operator → 狀態 `active`

## Step 3: 等兩個都 active，觀察目前沒 relation

```bash
juju wait-for application self-signed-certificates --timeout=10m
juju status --relations
```

**預期輸出**（注意 Integrations 區塊）：

```
App                       Status  Scale  Charm                     Channel     Rev
mysql                     active      1  mysql                     8.0/stable  444
self-signed-certificates  active      1  self-signed-certificates  1/stable    588

Unit                         Workload  Agent  Machine  Public address    Ports
mysql/0*                     active    idle   0        10.254.154.162    3306,33060/tcp
self-signed-certificates/0*  active    idle   1        10.254.154.215

Integration provider  Requirer              Interface    Type  Message
mysql:database-peers  mysql:database-peers  mysql_peers  peer     ← 只有 mysql 自己的 peer
mysql:restart         mysql:restart         rolling_op   peer
mysql:upgrade         mysql:upgrade         upgrade      peer
```

**重點**：Integrations 區塊**沒有兩個 app 之間的 regular relation**。兩個 app 完全獨立運作。

## Step 4: Integrate

```bash
juju integrate mysql:certificates self-signed-certificates:certificates
```

語法：`juju integrate <app1>:<endpoint1> <app2>:<endpoint2>`

- `mysql:certificates` → mysql charm 的 `certificates` endpoint（requires 端）
- `self-signed-certificates:certificates` → SSC charm 的 `certificates` endpoint（provides 端）

**⏱ 指令本身秒回，但背後 hook 執行約 60 秒**。

## Step 5: 立刻看 status（捕捉 hook 在跑的瞬間）

```bash
juju status --relations
```

**預期**：兩邊 Agent 欄變 `executing`（hook 正在跑）：

```
Unit                         Workload  Agent      Machine
mysql/0*                     active    executing  0
self-signed-certificates/0*  active    executing  1

Integration provider                   Requirer              Interface         Type
...（原有 peer relations）
self-signed-certificates:certificates  mysql:certificates    tls-certificates  regular  ← 新增
```

**新增了 regular 類型 relation**（跟 peer 不同，是兩個獨立 app 之間的連線）。

## Step 6: 等 idle，看最終結果

```bash
juju wait-for unit mysql/0 --query='agent-status=="idle"' --timeout=10m
juju wait-for unit self-signed-certificates/0 --query='agent-status=="idle"' --timeout=5m
juju status --relations
```

兩邊都回到 `active/idle`，relation 顯示為建立完成。

## Step 7: 驗證 — 看 relation data bag 實際傳了什麼

```bash
juju show-unit mysql/0 --format=yaml
```

在輸出裡找 `endpoint: certificates`，會看到 application-data 裡包含：

```yaml
- relation-id: 3
  endpoint: certificates
  related-endpoint: certificates
  application-data:
    certificates: '[{"ca": "-----BEGIN CERTIFICATE-----\nMIID..."},
                   {"certificate_signing_request": "-----BEGIN CERTIFICATE REQUEST-----\n..."},
                   {"certificate": "-----BEGIN CERTIFICATE-----\n..."},
                   {"chain": ["-----BEGIN CERTIFICATE-----\n..."]}]'
  related-units:
    self-signed-certificates/0:
      in-scope: true
      data:
        egress-subnets: 10.254.154.215/32
        ingress-address: 10.254.154.215
        private-address: 10.254.154.215
```

**四個元素都有**：
1. **CA certificate** — SSC 產的根憑證
2. **CSR** — mysql 產的憑證簽名請求
3. **Signed certificate** — SSC 用 CA 簽 mysql CSR 的結果
4. **Chain** — 完整憑證鏈

---

## Hook 觸發時序（實測推論）

按 `juju integrate` 之後約 60 秒內：

```
t=0s   : juju integrate → relation 建立，兩 unit 狀態轉 executing
t=~5s  : self-signed-certificates 的 certificates-relation-created hook 跑
         → 產 CA → 寫到 application-data
t=~15s : mysql 的 certificates-relation-changed hook 跑
         → 讀到 CA → 生 private key + CSR → 寫回 unit-data
t=~35s : self-signed-certificates 的 certificates-relation-changed hook 再跑
         → 讀到 CSR → 用 CA 簽 → 寫回 signed cert + chain
t=~55s : mysql 的 certificates-relation-changed 再跑
         → 讀到 signed cert → 存進 /etc/mysql/tls/（charm 內部動作）
t=60s  : 兩邊 agent 回 idle
```

兩個 charm **彼此不認識對方**，只約定介面叫 `tls-certificates`。任何實作 provides/requires 同一 interface 的 charm 都能相互 integrate。

---

## Gotchas

### ❌ `juju deploy mediawiki` 失敗

```
ERROR selecting releases: charm or bundle not found for channel "stable",
base "amd64/ubuntu/22.04/stable"
available releases are:
  channel "latest/stable": available bases are: ubuntu@14.04, ubuntu@12.04, ubuntu@11.10
```

**原因**：`mediawiki` charm 最後更新是 2014 年，只支援 Trusty (14.04) 以下。
**解法**：不用它。改用現役維護中的 Canonical charm。

### ❌ `wordpress` 也不能用

`wordpress` charm 只支援 16.04/18.04，不相容 22.04。

### ❌ `postgresql-test-app` amd64 stable 不存在

```
channels: |
  latest/stable:  arm64 only
  latest/edge:    amd64 on 24.04 only
```

沒有 amd64 + 22.04 的組合。

### ✅ 可行配對

| mysql (8.0/stable) | self-signed-certificates (1/stable) |
|---|---|
| amd64, ubuntu@22.04 | amd64, ubuntu@22.04 |

透過 interface `tls-certificates` integrate。

---

## 與 12 天後續的關聯

這個 `tls-certificates` 介面**就是 OpenStack 部署裡 Vault 跟 keystone/glance/nova-* 之間在用的介面**。Day 3 部署 Vault 之後，所有 OpenStack 服務都會用這個同樣的 relation 拿 TLS 憑證。

---

## 下次用時的快速 replay

```bash
ssh -i ~/.ssh/juju_id_rsa azureuser@20.210.233.133

# 部署兩個 app
juju deploy mysql
juju deploy self-signed-certificates --channel=1/stable

# 等 active
juju wait-for application mysql --timeout=10m
juju wait-for application self-signed-certificates --timeout=10m

# 看 status（沒 relation）
juju status --relations

# Integrate
juju integrate mysql:certificates self-signed-certificates:certificates

# 看 relation 與 cert 交換
juju wait-for unit mysql/0 --query='agent-status=="idle"' --timeout=10m
juju status --relations
juju show-unit mysql/0 --format=yaml | grep -A 30 "endpoint: certificates"
```

## 清場（如果要回到空 model）

```bash
juju remove-application mysql --destroy-storage
juju remove-application self-signed-certificates
juju wait-for application mysql --query='life=="dead"' --timeout=5m 2>/dev/null || true
```
