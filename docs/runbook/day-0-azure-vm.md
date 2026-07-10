# Day 0: Azure VM 建置

![OpenStack Charms 官方吉祥物](../assets/mascots/openstack-charms.png){ align=right width="90" }

## 目的

在 Azure 上開一台 `Standard_E16s_v5` VM，作為整個 OpenStack 學習的實驗環境。

## 對應官方指南

**無**。官方指南預設你已經有 MaaS + 物理機；我們用 Azure VM 替代這整層。

---

## Step 1: 註冊必要的 Resource Provider

**做什麼**：在目標 subscription 啟用 `Microsoft.Compute` / `Network` / `Storage` 三個 RP。

**為什麼**：新的 / 少用的 subscription 沒預先註冊 `Microsoft.Compute`（查 quota 會回傳空陣列）。Azure Portal 建資源會自動註冊，但我們走 CLI，必須手動。

```bash
SUB="<subscription-id>"

az provider register --namespace Microsoft.Compute --subscription "$SUB" --wait
az provider register --namespace Microsoft.Network --subscription "$SUB" --wait
az provider register --namespace Microsoft.Storage --subscription "$SUB" --wait
```

**驗證**：

```bash
az provider list --subscription "$SUB" \
  --query "[?namespace=='Microsoft.Compute' || namespace=='Microsoft.Network' || namespace=='Microsoft.Storage'].{RP:namespace, State:registrationState}" \
  -o table
```

預期：三列都 `Registered`。

---

## Step 2: 檢查 vCPU Quota

**做什麼**：確認 `Standard ESv5 Family vCPUs` 額度 ≥ 16（E16s_v5 吃 16 vCPU）。

```bash
az vm list-usage --location japaneast --subscription "$SUB" -o json | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
relevant = [i for i in data if 'ESv5' in i['name']['value'] or i['localName'] in ('Total Regional vCPUs', 'Standard ESv5 Family vCPUs')]
for i in relevant:
    print(f\"{i['localName']:50} {i['currentValue']:>4} / {i['limit']:>4}\")
"
```

**實測結果（<subscription-name>）**：

```
Total Regional vCPUs                                  0 /   50
Standard ESv5 Family vCPUs                            0 /   50
```

50/50 空閒，足夠。

---

## Step 3: 建 Resource Group

```bash
az group create \
  --name souch-openstack \
  --location japaneast \
  --subscription "$SUB"
```

---

## Step 4: 建 VM

**做什麼**：開 E16s_v5 Ubuntu 24.04 VM，讓 az CLI 自動順便建 VNet/Subnet/NSG/Public IP。

**關鍵選擇**：
- `--security-type TrustedLaunch --enable-secure-boot false --enable-vtpm false`
  - 不能用 `--security-type Standard`（該 feature 未註冊）
  - 改用 TrustedLaunch 但關掉 Secure Boot 與 vTPM，對 LXD 最友善
- 鎖 image URN：`Canonical:ubuntu-24_04-lts:server:latest`（Gen2 image，預設相容 E-series v5）
- `--storage-sku Premium_LRS`：OS disk 用 Premium SSD

```bash
az vm create \
  --subscription "$SUB" \
  --resource-group souch-openstack \
  --name openstack-lab \
  --location japaneast \
  --size Standard_E16s_v5 \
  --image Canonical:ubuntu-24_04-lts:server:latest \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/juju_id_rsa.pub \
  --public-ip-sku Standard \
  --public-ip-address-allocation static \
  --os-disk-size-gb 128 \
  --storage-sku Premium_LRS \
  --security-type TrustedLaunch \
  --enable-secure-boot false \
  --enable-vtpm false \
  --nsg-rule SSH \
  --accept-term
```

**實測輸出**：
```
ResourceGroup    PowerState    PublicIpAddress    Location
---------------  ------------  -----------------  ----------
souch-openstack  VM running    20.210.233.133     japaneast
```

---

## Step 5: 收緊 NSG（SSH 只允許本機 IP）

**做什麼**：預設 `--nsg-rule SSH` 會開 22 port 給 0.0.0.0/0，太開放。限制成當下本機 IP。

```bash
MY_IP=$(curl -s https://api.ipify.org)

az network nsg rule update \
  --subscription "$SUB" \
  --resource-group souch-openstack \
  --nsg-name openstack-labNSG \
  --name default-allow-ssh \
  --source-address-prefixes "$MY_IP"
```

> 換網路（家 / 咖啡廳）→ 同樣指令重跑一次即可

---

## Step 6: 掛 Data Disk（給 LXD 用）

**做什麼**：掛一顆 512GB Premium SSD，之後 LXD 會把它當 ZFS pool 的底層。

**為什麼不用 OS disk 的剩餘空間**：隔離學習環境，砍重來時只重建 data disk，OS 保留。

```bash
az vm disk attach \
  --subscription "$SUB" \
  --resource-group souch-openstack \
  --vm-name openstack-lab \
  --name openstack-lab-data \
  --new \
  --size-gb 512 \
  --sku Premium_LRS
```

---

## Step 7: 驗證 VM

```bash
ssh-keyscan -t ed25519,rsa -H 20.210.233.133 >> ~/.ssh/known_hosts 2>/dev/null

ssh -i ~/.ssh/juju_id_rsa azureuser@20.210.233.133 '
  echo "=== vCPU ==="; nproc
  echo "=== RAM ==="; free -h | head -2
  echo "=== nested virt (vmx count) ==="; egrep -c "(vmx|svm)" /proc/cpuinfo
  echo "=== block devices ==="; lsblk
'
```

**實測結果**：

```
=== vCPU === 16
=== RAM === 125Gi total
=== nested virt === 32  (每個 core 都有 vmx → KVM 可用)
=== block devices ===
sda  128G  (OS, sda1 / mounted)
sdb  512G  (Data disk, 未格式化 — 給 LXD)
```

---

## Gotchas

### ❌ `--security-type Standard` 失敗

```
ERROR: The value 'Standard' is not available for property 'securityType' until
the feature Microsoft.Compute/UseStandardSecurityType is registered on subscription
```

**解法**：改用 `--security-type TrustedLaunch` + `--enable-secure-boot false --enable-vtpm false`，效果等同 Standard，不用等 feature 註冊。

### ❌ 一開始查 quota 回傳 `[]`

**原因**：`Microsoft.Compute` RP 沒註冊，查詢根本沒資料可回。
**解法**：Step 1 先註冊。

---

## 日常操作

### 停機省錢
```bash
az vm deallocate -g souch-openstack -n openstack-lab --subscription "$SUB"
```

### 開機
```bash
az vm start -g souch-openstack -n openstack-lab --subscription "$SUB"
```

### 成本估算
- 開機中：$1.216/hr（Linux pay-as-you-go, japaneast）
- 停機中：只收 disk 錢（128GB + 512GB Premium SSD ≈ $73/月）
