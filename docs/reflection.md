# OpenStack 學習 Sprint — 完整回顧與下一步規劃

本文寫於 2026-04-21，lab teardown 前。濃縮 9 天的實際所得 + 誠實列出卡住的地方 + 給未來的你做決定。

## Sprint 時程實際執行

| Day | 主題 | 完成度 |
|---|---|---|
| 0 | Azure VM 建置 | ✅ |
| 1 | LXD + Juju bootstrap | ✅ |
| 2 | OpenStack Charm 架構地圖 | ✅ |
| 3 | Install OpenStack 15 服務 | ✅ 11/15（Ceph / Cinder / Cinder-Ceph / Ceph-radosgw 刻意跳過） |
| 4 | Configure OpenStack（開 Ubuntu VM + SSH + 出網） | ✅ |
| 5 | Heat (Orchestration) | ✅ |
| 6 | Magnum API | ✅ |
| 7 | Magnum 建 K8s cluster | 🟡 2 nodes Ready；workload 被 Magnum upstream 舊 image URL 卡住 |
| 8 | Octavia LBaaS | ❌ **Canonical charm amd64 斷代**（2024.1/stable 只 publish ppc64el） |
| 8.b | Ceph + Cinder（改 LVM） | ❌ **LXD container 不支援 storage kernel 操作**（device-mapper / loop udev） |

**6.5 / 8 成功**。剩下失敗的 1.5 個都不是 OpenStack 本身問題，是 Canonical charm 生態 + LXD 限制。

## 確實學到的東西（與部署工具無關，換家用也通用）

### OpenStack 概念
- **Keystone**：service catalog、service user、trust、domain-scoped token、OS_AUTH_URL port 5000 是 HTTP
- **Nova**：conductor / scheduler / compute 分工、placement 獨立成服務（Caracal 起）、cellsv2、metadata 機制
- **Neutron + OVN**：logical router / switch / port、localnet / localport、metadata agent 透過 unix socket 與 haproxy 串接、DHCP options 推路由
- **Glance**：image format、os_distro property 對 Magnum 的意義、file vs Ceph backend
- **Placement**：resource provider、inventory、allocation_ratio、trait
- **Heat**：HOT template、stack、resource、trust-based deferred auth
- **Magnum**：cluster template vs cluster、driver 模型（Heat template + bash 腳本）、keystone trust domain
- **Horizon**：純 Django web，catalog 都從 keystone 拿

### 業界真實地雷（非文件層級）
1. `mysql` charm vs `mysql-innodb-cluster` — 同名不同貨
2. SSC v3 vs OVN legacy tls-certificates interface 不相容
3. Placement charm 不跑 db sync
4. Caracal 起 nova-compute 需要 `cloud-credentials` relation 給 placement auth
5. OVN 22.03 charmhelpers library 停在 yoga，不認 caracal
6. Horizon charm 預設 `SESSION_COOKIE_SECURE=True`，HTTP tunnel 登入會被踢
7. Neutron flat network 預設關，要 `juju config neutron-api flat-network-providers=physnet1`
8. OVN chassis charm 需要真實 NIC 才建 br-ex（dummy interface 不收）
9. Neutron security-group extension 預設關
10. nova-compute charm 缺 `[neutron]` section（無對應 relation 傳 auth info）
11. OVN 24.03 controller vs 22.03 northd 版本檢查預設嚴格
12. cirros sshd race in OVN 環境
13. Ubuntu VM 要 `--config-drive True` 繞開 metadata race
14. Heat + Magnum charm 不自動建 keystone domain（手動跑 `domain-setup` action）
15. snap openstackclient strict confinement 看不到 `/tmp`
16. Magnum API bind 127.0.0.1 但 haproxy 指 container IP（charm internal bug）
17. Magnum cert-manager 預設 Barbican（沒 Barbican → 改 `x509keypair`）
18. Metadata shared secret 在不同 charm 間不同步
19. FCOS 40+ 移除 Docker，Magnum 2024.1 driver 腳本還 sed `/etc/sysconfig/docker`
20. Magnum driver manifest 寫死 `quay.io/coreos/flannel-cni:v0.3.0`（已被 deprecated）
21. Canonical charmhub octavia / cinder-lvm / manila / barbican 等在 amd64 上 2024.1/stable 斷代
22. LXD unprivileged container 無法跑 Ceph OSD（udev namespace）或 Cinder LVM（dm ioctl 權限）

**這 22 條都是業界操作者會遇到的真實問題**，docs 層面不會告訴你，踩過才知道。

## 確實不適合這種 lab 的東西（老實寫）

### 1. Canonical charm 在 amd64 的 publish gap
| 服務 | 2024.1 amd64 狀態 |
|---|---|
| octavia, octavia-diskimage-retrofit, octavia-dashboard | ❌ 只 ppc64el/s390x |
| cinder-lvm | ❌ 只 s390x |
| manila, manila-ganesha | ❌ |
| barbican | ❌ |
| neutron-api-plugin-ovn 2024.1 | ❌（退到 2023.2 才有 amd64） |

amd64 上**最後一個 charm 完整的 OpenStack release = Yoga (2022.1)**，但 Yoga 已 upstream EOL。

### 2. LXD container 本質限制
- nova-compute 要 LXD VM（需 `/dev/kvm`）
- ceph-osd 要 LXD VM 或實體機（udev event / 直接 block device）
- cinder LVM 要 LXD VM 或 privileged container（device-mapper ioctl）
- horizon / keystone / glance / placement 等 API 服務 → LXD container OK

### 3. Magnum driver 老舊
- 內嵌 YAML 的 image URL 多半是 2019-2021 年的 quay.io/coreos 名字空間（已 relocate）
- bash 腳本假設 FCOS 有 Docker（但 FCOS 40+ 只有 cri-o）
- k8s-keystone-auth webhook image 拉不到 → apiserver webhook fail → SA token 發不出 → workload 卡

## 產業現實（綜合 7 天踩雷後的 bird's eye view）

### 部署工具市佔（2026 約估）
| 工具 | 市佔 | 適合 |
|---|---|---|
| Kolla-Ansible | 30-35% | **社群主流**，新 deployment 的預設 |
| 自訂 fork | 20-25% | 雲廠商（OVH、阿里、騰訊、華為） |
| Red Hat OpenStack Platform | 15-20%（下滑） | RH 企業客戶 |
| **Canonical Charmed OpenStack（本次）** | **10-15%** | **電信 NFV 為主** |
| OpenStack-Ansible | 5-10% | 純開源 / DevOps |
| OpenStack on K8s (RHOSP 18 / Atmosphere) | 5-10% | **成長中**，RH 新一代客戶 |

### 「一條 IaC 管全棧」的現實
2026 年沒有成熟方案，只有三條**都有取捨**的路：

| 路線 | 優 | 劣 |
|---|---|---|
| MaaS + Juju + Terraform（本次） | 有 provider 串整條 | charm publish 斷代、release 慢 |
| MaaS + Kolla-Ansible + Terraform | arch 完整、release 新 | 部署是兩段式，Ansible 不是 Terraform |
| Metal3 + OpenStack-Helm + Terraform (k8s provider) | 未來式 gitops 一條龍 | 學習負擔大、成熟度低 |

## 學習目標重新盤點

你原始 goals：
1. 學 OpenStack 概念 → ✅ **達成**
2. 用 MaaS + Juju + Terraform 一條 IaC → 🟡 **部分達成**（concept 學到，charm gap 讓生產路不完整）
3. 跑到 Magnum → ✅ **API + cluster 起來**，workload 被 upstream bug 卡

### 達成的 knowledge gain（可寫履歷）
- **OpenStack 架構全景**（IaaS / CaaS / SDN 三層）
- **Juju charm relation model**（與 K8s Operator / Helm 類似思維）
- **Bare metal / LXD container / LXD VM 三種隔離的場景選擇**
- **Nova / OVN / Ceph / Cinder 的實作層限制**（LXD container 不能跑 storage；FCOS 跟 K8s 的 driver tight coupling）
- **產業部署工具生態**（為什麼 Kolla 是主流、為什麼 RH 在推 K8s 版）

### 下一步 3 種分支建議

#### A. 繼續 OpenStack 深化（留在這條線）
**做**：從已經會 OpenStack concept 的位置往下挖
- 學**官方部署工具 Kolla-Ansible**（1-2 週），踩一次不同 paradigm
- 練**Terraform openstack-provider**（VM / network / LB 管理）
- 學 OVN 內部（flow 程式化、policy routing、流表分析）

**適合**：想當 OpenStack operator / 做私雲 architect

#### B. 跳到 K8s 生態（往上疊）
**做**：保留 OpenStack 當基礎設施，學 K8s 生態
- K8s 基本（Pod / Deployment / Service / Ingress / PVC）
- CAPI + CAPO（K8s 管 K8s cluster，OpenStack 當底層）
- GitOps（ArgoCD / Flux）

**適合**：想做 cloud platform engineer / SRE

#### C. 往基礎設施 IaC 走（往下深）
**做**：Terraform / OpenTofu / Pulumi 深入
- 多 provider（AWS / Azure / GCP / OpenStack / vSphere）
- State 管理、module 設計、CI/CD 整合
- Policy as Code（OPA / Sentinel）

**適合**：想做 platform / DevOps lead

### 本次 sprint 的收尾

| 檔案 | 內容 |
|---|---|
| `docs/runbook/day-0..day-7-*.md` | 每一步實跑過的指令 |
| `docs/solutions/` | 22 個 integration / runtime 地雷的詳細 doc |
| `docs/plans/` | 原始計畫書 |
| `/Users/souch_hsu/Downloads/configure-openstack-lxd-lab.html` | 完整的 HTML 教學文件 |
| 本檔 `docs/reflection.md` | 回顧 + 下一步 |

**這些檔案 teardown 後也都保留**（git 在本機 Mac）。

## 給未來回來看這份文件的你

1. **不用把 Octavia / Ceph 失敗當作失敗**。你已經證明 charm gap + LXD container 限制是**真的**，不是你的錯。
2. **下次要跑 production / 完整 lab**，請選 **MaaS + Kolla-Ansible on real hosts（或 LXD VM 模擬）**。
3. **OpenStack 本體概念你懂了**，這是最重要的。換工具只是換詞彙。
4. **Magnum 不用再碰**，2026+ 產業走 CAPO。
5. **TerraForm + Juju 這條路** 在本 lab 沒完整驗證，但 provider 都存在，只要繞過 charm publish gap 就能做。

---

_Sprint 實跑 9 天，踩 22 個雷，部了 15 個服務，3 個 VM，2 個 K8s node。這段路沒白走。_
