---
title: "feat: Sprint 3 — Kolla-Ansible + Magnum CAPI Azure lab 與完整課程"
type: feat
status: active
date: 2026-07-07
---

# Sprint 3 — Kolla-Ansible + Magnum (CAPI driver) Azure Lab 與完整課程

## Overview

在 Azure <tenant> tenant 的 **Azure AI Services 訂閱（`<subscription-id>`）** 開一個獨立 resource group,建一台 lab VM,用 **Kolla-Ansible** 部署 OpenStack(core + Octavia + Barbican),再用 **Magnum + Cluster API driver** 開出能跑 workload 的 K8s cluster。全程寫成課程:每個 Day 一份 runbook(原理 + 架構 + 可照抄的步驟 + checkpoint),沿用本 repo 既有的 `docs/runbook/` / `docs/solutions/` 慣例。

**預估成本:推薦情境約 US$101/週(約 NT$3,000),詳見成本章節。**

## 與 Sprint 1 / Sprint 2 的關係(重要:本計畫推翻兩個舊決定)

| 舊決定 | 出處 | 本次翻案理由 |
|---|---|---|
| 「Sprint 2 直接走 OpenStack-Helm,不走 Kolla-Ansible」 | `docs/sprint2-course-plan.md` | Kolla-Ansible 是社群主流(reflection.md 自己的市佔表:30-35%),Ansible paradigm 值得補;OSH 的 K8s 兩層除錯負擔在 Sprint 2 已體感過 |
| 「Magnum 不用再碰,2026+ 產業走 CAPO」 | `docs/reflection.md` §給未來的你 | 當時結論基於**舊 heat driver**(FCOS image 爛掉、flannel URL 失效 = Sprint 1 day-7 卡點)。新的 **magnum-cluster-api driver 底層就是 CAPO**——學它等於同時學 Magnum API 面和 CAPO 實作面,兩個結論其實相容 |

本 Sprint 直接針對 Sprint 1 的三個未竟項目雪恥:

1. **Octavia** — Sprint 1 敗因是 Canonical charm amd64 斷代(非技術問題);Kolla 的 Octavia 支援完整。
2. **Cinder LVM** — Sprint 1 敗因是 LXD container 的 device-mapper 限制;本次直接跑在 Azure VM host 上,data disk 做 `cinder-volumes` VG。
3. **Magnum workload** — Sprint 1 敗因是 heat driver 內嵌的 2019-2021 image URL 全數失效;本次改用 CAPI driver,node image 由 CAPI 生態維護。

Sprint 2(OSH 路線)留下的 `sprint2-*` runbook 保留為歷史紀錄,不再續作;其 Azure VM 已 teardown(已確認兩個訂閱目前都沒有任何 VM)。

## 目標架構

```
┌──── Azure VM Standard_E16s_v5（japaneast, 16 vCPU / 128 GB）────┐
│  Ubuntu 24.04 LTS（Day 0 驗證 kolla 支援矩陣後定案）              │
│                                                                  │
│  Kolla-Ansible AIO（每服務一個 Docker container, host network）   │
│    ├─ Core: Keystone / Glance / Nova(KVM) / Neutron(OVN)         │
│    │        / Cinder(LVM← data disk VG) / Horizon                │
│    ├─ Magnum 前置: Octavia(amphora) / Barbican                   │
│    └─ Magnum API + conductor（含 CAPI driver）                    │
│                                                                  │
│  CAPI management cluster（kind, 跑在同一台 host 的 Docker 上）     │
│    └─ cluster-api + CAPO + magnum driver 的 controllers          │
│                                                                  │
│  Nova 的 nested VMs（/dev/kvm）                                   │
│    ├─ K8s workload cluster: control-plane ×1 + worker ×2         │
│    └─ Octavia amphora ×2（API LB + Service LB）                   │
└──────────────────────────────────────────────────────────────────┘
        ▲ SSH tunnel / sshuttle（Azure VNet 無 L2,external net 僅 host 內部）
┌── Mac 本機 ──────────────────────────────┐
│  openstack CLI / kubectl / Terraform      │
└───────────────────────────────────────────┘
```

## Azure 資源清單與規格

| 資源 | 規格 | 備註 |
|---|---|---|
| Resource group | `souch-openstack-sprint3`(japaneast) | 訂閱內既有 3 個 `*-deepwave-central-services` RG **完全不碰**;lab 資源全部隔離在新 RG,teardown = 刪 RG 一鍵清空 |
| VM | `Standard_E16s_v5`(16 vCPU / 128 GB, Intel Ice Lake, 支援 nested virt) | 與 Sprint 2 同款、已驗證過的規格。128 GB 的餘裕來源:Kolla 服務 ~24 GB + kind mgmt cluster ~6 GB + nested VM(4+4+4 GB)+ amphora ×2 ~3 GB + buffer |
| OS disk | 128 GB Premium SSD(P10) | |
| Data disk | 256 GB Premium SSD(P15) | Day 3 做成 `cinder-volumes` VG |
| Public IP | Standard static ×1 | NSG 只開 22,SSH 鎖 source IP(沿用 sprint2-day-0 做法) |
| 安全型態 | `--security-type Standard` | Sprint 2 用 TrustedLaunch;為避免 nested virt 相容性問題本次改 Standard,Day 0 以 `kvm-ok` 驗證 |
| 成本 guardrail | auto-shutdown 排程 + RG budget alert(US$150/月訊號線)+ `purpose=lab` tag | Day 0 一併建立 |

**配額已確認**:japaneast ESv5 family 50 vCPU、目前用量 0,E16s_v5 不需申請提額。

## 成本估計(japaneast 即時牌價,2026-07-07 由 Azure Retail Prices API 查得)

單價:E16s_v5 $1.216/hr(Spot $0.2247/hr)、E8s_v5 $0.608/hr、P10 $22.67/月、P15 $43.72/月、Public IP ~$0.005/hr。磁碟與 IP 在 VM 關機(deallocate)時**仍計費**。

| 情境 | VM 用法 | VM/週 | 磁碟+IP/週 | **合計/週** |
|---|---|---|---|---|
| **推薦:上班時段開機** | E16s_v5,10h/天 ×7(auto-shutdown) | $85.1 | $16.1 | **≈ $101(NT$3,000)** |
| 省錢版 | E8s_v5(64 GB),10h/天 ×7 | $42.6 | $16.1 | ≈ $59(NT$1,800);RAM 較緊,Day 8 全套同時跑會逼近上限 |
| 全速版 | E16s_v5 24/7 | $204.3 | $16.1 | ≈ $220(NT$6,600) |
| 進階省錢 | E16s_v5 **Spot** 24/7 | $37.8 | $16.1 | ≈ $54;有 eviction 風險,`kolla-ansible deploy` 中被收走要重來,建議熟練後才用 |

課程全程預估 **2–3 週**(比照 Sprint 1 的 9 天節奏),推薦情境總成本約 **US$200–300**。

## 課程結構(Day 0–10)

每天一份 `docs/runbook/sprint3-dayN-<topic>.md`,固定格式:**原理與架構**(為什麼這樣設計、元件關係圖)→ **步驟**(可照抄指令 + 預期輸出)→ **Checkpoint**(驗證指令與判準)→ **踩坑**(即時記錄,重大者另立 `docs/solutions/`)。

| Day | 主題 | 原理重點 | Lab / Checkpoint |
|---|---|---|---|
| 0 | Azure lab 環境 | nested virt、Azure VNet 無 L2 的網路約束、成本 guardrails | RG + VM + NSG + auto-shutdown + budget;`kvm-ok` 通過 |
| 1 | Kolla-Ansible 原理 + AIO 部署 | Kolla image vs kolla-ansible、globals.yml/inventory/merge 機制、與 Sprint 1 charm relation model 對照 | core 服務 deploy 成功;`openstack service list` 全綠 |
| 2 | OpenStack 資源全流程 | project/quota、image、OVN 網路模型(對照 Sprint 1 已學的 logical router/switch)、SG | 純 CLI 開出可 SSH 的 VM(cirros + Ubuntu) |
| 3 | Cinder LVM(雪恥 #1) | LVM backend、attach 資料路徑、為何 Sprint 1 的 LXD 做不到 | data disk → VG → volume attach/detach、boot from volume |
| 4 | Octavia(雪恥 #2) | amphora 模型、LB 資料面/控制面、amphora image 準備 | 手動建 LB → listener → pool,打通兩台 VM 後端 |
| 5 | Barbican + Heat | secret/cert 存放、Magnum 的 cert-manager 依賴(對照 Sprint 1 的 x509keypair fallback)、HOT 複習 | secret 存取 + 一個小 Heat stack |
| 6 | Cluster API 原理 + management cluster | CAPI/CAPO 架構、Machine/MachineDeployment、driver 兩案比較與定案(見技術決策) | kind + clusterctl init + CAPO 就緒 |
| 7 | Magnum + CAPI driver 整合(雪恥 #3) | driver 註冊機制、Kolla magnum image 客製(pip 加裝 driver)、ClusterTemplate labels | `openstack coe cluster template create` 成功 |
| 8 | E2E:開 cluster 跑 workload | 三層互動:Magnum→CAPI→CAPO→Nova/Octavia/Cinder | cluster CREATE_COMPLETE;kubectl 部 app;`Service type=LoadBalancer` 拿到 Octavia LB;PVC 由 Cinder CSI 供裝 |
| 9 | Day-2 operations | cluster 升版(CAPI rolling)、node group、cluster-autoscaler、監控 | 一次 K8s 小版升級 + autoscale 觸發 |
| 10 | Terraform 接管 + teardown 演練 + 回顧 | terraform-provider-openstack(Sprint 1 未竟)、重建速度驗證 | HCL 管 network/VM/LB;`kolla-ansible destroy` → 重部;寫 `sprint3-reflection.md` |

## 技術決策

| 決策 | 選擇 | 理由 / 待驗證 |
|---|---|---|
| OpenStack release | **2025.1 Epoxy(SLURP)** | 文件與踩坑資料最多的近期穩定版;2026.1 較新但 kolla 生態通常慢半拍。Day 0 confirm kolla-ansible 對 Ubuntu 24.04 的支援矩陣後定案 |
| Magnum driver | 傾向 **`magnum-cluster-api`(Vexxhost)**,備案 `magnum-capi-helm`(StackHPC) | 前者文件與社群案例較多(Atmosphere 同源);但需客製 magnum image(pip 安裝 driver)。Day 6 做 30 分鐘 spike 後定案,兩案的安裝差異寫進 runbook 當教材 |
| 部署工具 | Kolla-Ansible(quay.io 官方 image,不自建 registry) | AIO 單機不需要 local registry |
| K8s 版本 | 依 driver 支援矩陣選(預期 1.30–1.32 區間) | Day 6 spike 時從 driver release notes 確認 |

## System-Wide Impact(訂閱層面)

- **隔離**:所有資源限縮在新 RG;唯一共用面是訂閱層級的 regional vCPU quota(50),lab 佔 16,不影響既有 `*-deepwave-central-services` RG。
- **費用歸屬**:`purpose=lab` / `owner=souch` tag + RG budget alert,帳單面一眼可分。
- **失敗回收**:任何時候 `az group delete` 即完全回收;唯一殘留是本 repo 的文件(這正是目的)。

## Risks & Mitigations

| 風險 | 影響 | 對策 |
|---|---|---|
| nested virt 不可用(security type / SKU 因素) | Nova 只能 qemu 純模擬,慢 3–5 倍 | VM 用 Standard security type;Day 0 `kvm-ok` 不過就重開 VM(30 秒級操作,sprint2-day-0 已驗證) |
| Kolla magnum image 需客製才能裝 CAPI driver | Day 7 卡關 | Day 6 spike 先驗證;備案:`docker exec` 進 magnum 容器 pip 安裝做 PoC,課程註明非正規做法 |
| Azure VNet 無 L2 → Neutron external network 出不了 host | floating IP 只能 host 內打通 | 架構已按此設計(SSH tunnel / sshuttle);寫進 Day 0 原理章,當成 Azure lab 的教學重點而非缺陷 |
| MTU(VXLAN/Geneve overhead) | VM 內網路詭異斷流 | `global_physnet_mtu`/network MTU 設 1450,Day 1 globals.yml 直接帶入 |
| RAM 壓力(選 E8s_v5 時) | Day 8 全套同跑 OOM | 推薦 E16s_v5;若選省錢版,worker 減為 1 台 |
| Spot eviction | 部署中斷 | 預設不用 Spot;若用,限 Day 2 之後的「可中斷」時段 |
| 舊坑重現 | — | Sprint 1 的 22 條 solutions 是本課程的先修教材,Day 1 開場先重讀 |

## Acceptance Criteria

- [x] RG 建立且含 auto-shutdown、budget alert、tags;既有 3 個 RG 零接觸(2026-07-07,runbook: `sprint3-day0-azure-vm.md`;`kvm-ok` 亦通過)
- [ ] `kvm-ok` 通過,Kolla AIO(core + Octavia + Barbican + Magnum)deploy 成功
- [x] 純 CLI 完成 VM/network/volume/LB 全流程各至少一次(Day 2 VM/network、Day 3 volume、Day 4 LB round-robin + OVN provider 對照,2026-07-08)
- [ ] Magnum(CAPI driver)開出的 cluster:`kubectl get nodes` 全 Ready、LoadBalancer Service 取得 Octavia LB、PVC 綁定 Cinder volume — **三項全過才算雪恥完成**
- [ ] 每個 Day 有 runbook(原理 + 步驟 + checkpoint),新坑寫入 `docs/solutions/`
- [ ] 週成本落在所選情境 ±20% 內(以 Azure Cost Analysis 驗證)
- [ ] `sprint3-reflection.md` 完成

## Day 0 首批指令(規劃參考,尚未執行)

```bash
az group create -n souch-openstack-sprint3 -l japaneast \
  --subscription <subscription-id> \
  --tags purpose=lab owner=souch sprint=3

az vm create --subscription <subscription-id> \
  -g souch-openstack-sprint3 -n openstack-lab \
  --image Ubuntu2404 --size Standard_E16s_v5 \
  --security-type Standard \
  --admin-username azureuser --ssh-key-values ~/.ssh/juju_id_rsa.pub \
  --public-ip-sku Standard \
  --os-disk-size-gb 128 --data-disk-sizes-gb 256

az vm auto-shutdown -g souch-openstack-sprint3 -n openstack-lab \
  --time 1400 # UTC = 台北 22:00,早上手動開機
# + NSG 鎖 SSH source IP(沿用 sprint2-day-0-azure-vm.md 做法)
# + az consumption budget(US$150/月 alert)
```

## Sources & References

### Internal
- `docs/reflection.md` — Sprint 1 回顧、22 條坑、部署工具市佔表
- `docs/sprint2-course-plan.md` — 被本計畫取代的 OSH 決定與其理由
- `docs/runbook/sprint2-day-0-azure-vm.md` — VM 建置指令基底(本次改 security type 與 RG/訂閱)
- `docs/runbook/day-6-magnum.md`、`day-7-magnum-k8s-cluster.md` — Sprint 1 Magnum 卡點的第一手紀錄

### External
- [Kolla-Ansible 官方文件](https://docs.openstack.org/kolla-ansible/latest/)
- [vexxhost/magnum-cluster-api](https://github.com/vexxhost/magnum-cluster-api) 與 [文件站](https://vexxhost.github.io/magnum-cluster-api/)
- [magnum-capi-helm release notes](https://docs.openstack.org/releasenotes/magnum-capi-helm/unreleased.html)(備案 driver)
- [Satish Patel: OpenStack Magnum CAPI 實作紀錄](https://satishdotpatel.github.io/openstack-magnum-capi/)(Kolla + CAPI 的社群實例)
- [Magnum 2026.1 User Guide](https://docs.openstack.org/magnum/2026.1/user/index.html)
- 價格:Azure Retail Prices API(2026-07-07 查詢,japaneast)
