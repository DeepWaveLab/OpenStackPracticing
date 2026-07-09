# Sprint 3 / Day 0: Azure Lab 環境建置

> 課程定位:本章目標是建出一台**能跑 nested virtualization 的乾淨 lab VM**,並把成本 guardrails 一次到位。
> 完整計畫見 `docs/plans/2026-07-07-feat-sprint3-kolla-magnum-capi-azure-lab-course-plan.md`。

## 原理與架構

### 1. 為什麼是 E16s_v5

Kolla-Ansible AIO + Magnum CAPI 的記憶體帳(Day 8 全套同時跑的尖峰):

| 元件 | RAM 預估 |
|---|---|
| Kolla 全服務(core + Octavia + Barbican + Magnum) | ~24 GB |
| kind management cluster(CAPI/CAPO controllers) | ~6 GB |
| K8s workload cluster nested VM(CP ×1 + worker ×2, 各 4 GB) | 12 GB |
| Octavia amphora ×2 | ~3 GB |
| host OS + buffer | 其餘 |

E16s_v5 = 16 vCPU / 128 GB(Intel Ice Lake),**支援 nested virtualization** —— Nova 才能用 KVM 加速開 VM,否則退回 qemu 純模擬慢 3–5 倍。這也是 Sprint 2 驗證過的同款規格。

### 2. Azure 網路約束(整個 lab 的架構前提)

Azure VNet **沒有 L2 廣播、擋 MAC/IP spoofing**。後果:

- Neutron 的 external/provider network **出不了這台 host**,floating IP 只在 host 內部有效
- 從 Mac 存取 lab 裡的資源一律走 **SSH tunnel / sshuttle**
- 這不是缺陷,是公有雲上跑 IaaS lab 的既定架構;生產環境的 external network 會接真實的 provider VLAN

### 3. Security type:為什麼堅持 Standard

Azure 新政策預設強制 TrustedLaunch。TrustedLaunch 與 nested virt 的相容性有版本地雷,lab 沒必要冒險 —— 直接用 `--security-type Standard`。**坑**:新訂閱要先註冊 feature `Microsoft.Compute/UseStandardSecurityType` 才准用 Standard(見下方踩坑)。

### 4. 成本 guardrails 三件套

| 機制 | 值 | 作用 |
|---|---|---|
| auto-shutdown | 每日 14:00 UTC(台北 22:00) | 忘了關機的保險絲;**只會 deallocate,早上要自己開機** |
| budget alert | RG 範圍 US$150/月;80% 實際 + 100% 預測寄信 | 帳單面警報 |
| tags | `purpose=lab owner=souch sprint=3` | Cost Analysis 一眼歸戶 |

牌價(japaneast, 2026-07-07):E16s_v5 $1.216/hr;P10 $22.67/月 + P15 $43.72/月(**磁碟關機也計費**)。每天 10 小時 ≈ **US$101/週**。

## 步驟

### 0. 前置確認

```bash
az account list -o table          # 確認 <subscription-id>(Azure AI Services)在列
ls ~/.ssh/juju_id_rsa.pub         # SSH key 存在
# quota 免申請:japaneast ESv5 family 50 vCPU、用量 0(az vm list-usage 確認過)
```

### 1. Resource group

```bash
az group create -n souch-openstack-sprint3 -l japaneast \
  --subscription <subscription-id> \
  --tags purpose=lab owner=souch sprint=3
```

### 2. 註冊 Standard security type feature(第一次會踩)

```bash
az feature register --namespace Microsoft.Compute --name UseStandardSecurityType \
  --subscription <subscription-id>
az provider register -n Microsoft.Compute \
  --subscription <subscription-id> --wait
```

### 3. 開 VM(從這行開始計費)

```bash
az vm create \
  --subscription <subscription-id> \
  -g souch-openstack-sprint3 -n openstack-lab \
  --image Ubuntu2404 --size Standard_E16s_v5 \
  --security-type Standard \
  --admin-username azureuser --ssh-key-values ~/.ssh/juju_id_rsa.pub \
  --public-ip-sku Standard \
  --os-disk-size-gb 128 --data-disk-sizes-gb 256 \
  --tags purpose=lab owner=souch sprint=3
```

### 4. NSG 鎖 SSH 到本機 IP

```bash
MY_IP=$(curl -s ifconfig.me)
az network nsg rule update \
  --subscription <subscription-id> \
  -g souch-openstack-sprint3 --nsg-name openstack-labNSG \
  -n default-allow-ssh --source-address-prefixes "$MY_IP/32"
```

### 5. auto-shutdown

```bash
az vm auto-shutdown \
  --subscription <subscription-id> \
  -g souch-openstack-sprint3 -n openstack-lab --time 1400   # UTC → 台北 22:00
```

### 6. Budget alert(az CLI 沒有含通知的指令,走 az rest)

```bash
az rest --method PUT \
  --url "https://management.azure.com/subscriptions/<subscription-id>/resourceGroups/souch-openstack-sprint3/providers/Microsoft.Consumption/budgets/sprint3-lab-budget?api-version=2023-05-01" \
  --body '{"properties":{"category":"Cost","amount":150,"timeGrain":"Monthly",
    "timePeriod":{"startDate":"2026-07-01T00:00:00Z","endDate":"2026-12-31T00:00:00Z"},
    "notifications":{
      "actual80":{"enabled":true,"operator":"GreaterThan","threshold":80,"contactEmails":["you@example.com"],"thresholdType":"Actual"},
      "forecast100":{"enabled":true,"operator":"GreaterThan","threshold":100,"contactEmails":["you@example.com"],"thresholdType":"Forecasted"}}}}'
```

### 7. 驗證

```bash
ssh -i ~/.ssh/juju_id_rsa azureuser@203.0.113.10
sudo apt-get update -qq && sudo apt-get install -y cpu-checker
sudo kvm-ok
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT
```

## Checkpoint(全數通過 2026-07-07)

| 驗證 | 判準 | 實測 |
|---|---|---|
| `sudo kvm-ok` | `KVM acceleration can be used` | ✅ |
| `nproc` / `free -h` | 16 / 125Gi | ✅ |
| `lsblk` | sda 128G(/)、sdb 256G(未格式化,Day 3 做 cinder-volumes VG) | ✅ |
| `lsb_release -d` | Ubuntu 24.04 LTS | ✅ 24.04.4 |
| 既有 3 個 `*-deepwave-central-services` RG | 零接觸 | ✅ |

## 本次 lab 環境總表

| 項目 | 值 |
|---|---|
| Subscription | Azure AI Services(`<subscription-id>`) |
| Resource group | `souch-openstack-sprint3`(japaneast) |
| VM | `openstack-lab`(Standard_E16s_v5, 16 vCPU / 128 GB, security type **Standard**) |
| OS | Ubuntu 24.04.4 LTS |
| Public IP | `203.0.113.10`(SSH 限 `198.51.100.1/32`,換網路要重跑步驟 4) |
| Private IP | `10.0.0.4` |
| Disk | OS sda 128 GB(P10)+ data sdb 256 GB(P15,未格式化) |
| SSH | `ssh -i ~/.ssh/juju_id_rsa azureuser@203.0.113.10` |
| 每日操作 | 早上 `az vm start -g souch-openstack-sprint3 -n openstack-lab`;晚上 22:00 自動關機 |

## 踩坑

### UseStandardSecurityType feature gate(新)

`az vm create --security-type Standard` 直接吃 `BadRequest: The value 'Standard' is not available for property 'securityType' until the feature Microsoft.Compute/UseStandardSecurityType is registered`。Azure 2024 起新訂閱預設鎖 TrustedLaunch,Standard 要 feature opt-in + provider 重註冊(步驟 2)。Sprint 2 當時用 TrustedLaunch + 關 secure boot 繞過,沒撞到這顆。

## 下一步(Day 1)

Kolla-Ansible 原理 + AIO 部署 core 服務 —— 見課程計畫 Day 1;開工前先重讀 Sprint 1 的 `docs/solutions/` 當先修教材。
