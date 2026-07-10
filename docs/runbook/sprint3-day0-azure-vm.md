# Day 0:準備 Azure Lab 環境

> 今天還不碰 OpenStack——先把地基打好:在 Azure 開一台**支援巢狀虛擬化**的 VM 當實驗室,並且把成本控管(自動關機、預算警報)一次設好,確保這個 lab 不會變成月底的帳單驚喜。

!!! abstract "你在課程的哪裡"
    - **今天**:租一台夠大的 Azure VM。規格、安全類型、網路限制每一項都有理由,原理章會逐一解釋。
    - **今天之後**:接下來十一天的所有東西——整朵 OpenStack 雲、裡面開的每台虛擬機——全部跑在這一台 VM 上。它每晚 22:00 會自動關機幫你省錢,早上記得自己開。

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

Azure 新政策預設強制 TrustedLaunch。TrustedLaunch 與巢狀虛擬化的相容性有版本地雷,lab 沒必要冒險——直接用 `--security-type Standard`。但要注意:新訂閱必須先註冊 feature `Microsoft.Compute/UseStandardSecurityType` 才准用 Standard,忘了會在開 VM 時被拒絕(步驟 2 就是在做這件事;完整錯誤訊息見文末的地雷記錄)。

### 4. 成本 guardrails 三件套

| 機制 | 值 | 作用 |
|---|---|---|
| auto-shutdown | 每日 14:00 UTC(台北 22:00) | 忘了關機的保險絲;**只會 deallocate,早上要自己開機** |
| budget alert | RG 範圍 US$150/月;80% 實際 + 100% 預測寄信 | 帳單面警報 |
| tags | `purpose=lab owner=souch sprint=3` | Cost Analysis 一眼歸戶 |

牌價(japaneast, 2026-07-07):E16s_v5 $1.216/hr;P10 $22.67/月 + P15 $43.72/月(**磁碟關機也計費**)。每天 10 小時 ≈ **US$101/週**。

## 步驟

### 0. 前置確認

確認三件事:要用的訂閱存在、SSH 金鑰存在、目標區域的 vCPU 配額夠開 16 核:

```bash
az account list -o table          # 你要用的訂閱在清單裡
ls ~/.ssh/juju_id_rsa.pub         # SSH 公鑰存在(沒有就先 ssh-keygen)
az vm list-usage -l japaneast -o table | grep -i esv5   # ESv5 家族配額 ≥ 16(不足要先申請)
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

## 驗收 checkpoint

在 VM 上逐項執行,**全部符合判準才算完成今天**;不符合就回對應步驟排查:

| 驗證指令 | 你應該看到 |
|---|---|
| `sudo kvm-ok` | `KVM acceleration can be used`——巢狀虛擬化可用,這是整個 lab 的前提 |
| `nproc` 與 `free -h` | 16 核心、約 125 GiB 記憶體 |
| `lsblk` | sda 128G 掛在 `/`;sdb 256G **保持未格式化**(Day 3 會把它做成 Cinder 的儲存池) |
| `lsb_release -d` | Ubuntu 24.04 LTS |

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

## 地雷記錄

### 地雷 1:`--security-type Standard` 被 Azure 拒絕

**症狀**:`az vm create --security-type Standard` 直接失敗,錯誤訊息是 `BadRequest: The value 'Standard' is not available for property 'securityType' until the feature Microsoft.Compute/UseStandardSecurityType is registered`。

**根因**:Azure 從 2024 年起,新訂閱預設強制 TrustedLaunch;要用 Standard 必須先註冊 feature 並重新註冊 provider——也就是步驟 2 存在的原因。如果你跳過了步驟 2,回去補跑再重試 `az vm create` 即可。

## 下一步

一台乾淨的、支援巢狀虛擬化的 VM 已經就緒。[Day 1](sprint3-day1-kolla-aio-core.md) 我們安裝部署工具 Kolla-Ansible,把 OpenStack 的五個核心服務部上去——今天這台空白的主機,明天就會是一朵能登入的雲。

如果你想先了解「為什麼選 Kolla-Ansible、之前兩條路線怎麼失敗的」,可以先讀[前兩次嘗試](../previous-attempts.md);想直接動手也完全沒問題,Day 1 開頭會給你需要的所有背景。
