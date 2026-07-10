# Sprint 2 課程規劃：OpenStack-Helm on K8s

## 決定

**Sprint 2 直接走 OpenStack-Helm**，不走 Kolla-Ansible。

理由：
- 使用者已會 K8s / Helm / kubectl — Ansible 學習成本可省
- OpenStack-Helm 的編排本質跟 K8s 原生能力對齊
- Sprint 3+ 要走 Foreman + OpenStack-Helm / Metal3，技能堆疊一致

## 架構（Sprint 2 本階段）

```
┌──── Azure VM E16s_v5（單節點 lab）─────────┐
│                                             │
│  kubeadm single-node K8s cluster            │
│    ├─ CNI: Cilium 或 Calico                 │
│    ├─ Storage: local-path-provisioner       │
│    ├─ LB:  MetalLB (L2 mode)                │
│    └─ Helm v3                               │
│                                             │
│  openstack-helm-infra 部的東西：             │
│    ├─ MariaDB (statefulset)                 │
│    ├─ RabbitMQ                              │
│    ├─ Memcached                             │
│    ├─ Etcd                                  │
│    ├─ libvirt DaemonSet (privileged)        │
│    └─ (optional) Ceph / OVN                 │
│                                             │
│  openstack-helm 部的 OpenStack 服務：        │
│    Keystone / Glance / Placement / Horizon  │
│    Nova (api + compute DaemonSet)           │
│    Neutron (server + OVN DaemonSet)         │
│    Heat / Cinder / Octavia / Magnum         │
│                                             │
└─────────────────────────────────────────────┘
        │ OpenStack API (via MetalLB VIP)
        ▼
┌──── Mac 本機 ──────────────────────┐
│  Terraform + openstack provider     │
│  HCL 建 VM / network / LB 等租戶資源│
└─────────────────────────────────────┘
```

單節點 AIO 等價版，之後 Sprint 3 分成 multi-node + Foreman 管 bare metal。

## 課表（課程主體 8 步，前面有 pre-req 環境準備）

### Pre-req 環境（Day 0，不計入課程編號）

**課前先備好**（使用者已會 K8s，這段是環境準備不是課程）：

- 沿用 Azure VM E16s_v5 / Ubuntu 22.04 / 128 GB RAM / 256 GB data disk（已完成）
- Public IP：`203.0.113.20`
- VM 清掉先前 Kolla 殘留（已完成 2026-04-21）
- `kubeadm init` 起 single-node K8s
- 移除 control-plane taint（讓 pod 能排程）
- 裝 CNI（Cilium 或 Calico）
- 裝 Helm v3
- 裝 MetalLB + L2 pool（給 OpenStack 服務 VIP）
- 裝 local-path-provisioner（PV 存 `/var/lib/...`）
- 驗證：`kubectl get nodes` Ready、test PVC 可 bind

### 課程 (1) — openstack-helm-infra
- Clone `openstack-helm-infra` repo
- values override 寫一份本地 `env-values.yaml`
- 順序部：
  1. openvswitch / ovn（network 底層）
  2. mariadb（cluster，3 replica）
  3. rabbitmq
  4. memcached
  5. etcd
  6. libvirt（給 nova-compute 用）
- 每個 chart 完成後 `kubectl get pods -n openstack` 驗證

### 課程 (2) — openstack-helm 核心服務
- Clone `openstack-helm` repo
- 部：
  1. keystone（identity）
  2. glance（image）
  3. placement（resource tracking）
  4. horizon（UI）
- 生成 admin openrc + `openstack service list`
- Horizon 透過 MetalLB VIP 打
- 上一個 cirros / Ubuntu image 進 Glance

### 課程 (3) — Compute + Network
- 部 nova（api 是 Deployment、compute 是 privileged DaemonSet with `/dev/kvm`）
- 部 neutron（server Deployment + OVN DaemonSet on hostNetwork）
- 設 OVN bridge mapping（跟 Sprint 1 類似邏輯但在 K8s 裡）
- 建 external network + tenant network
- 開第一台 VM 驗證

### 課程 (4) — 進階服務
- Heat（orchestration）
- Cinder（LVM backend，data disk 256 GB 切給 `cinder-volumes` VG）
- Octavia（LBaaS）
  - Amphora image prep
  - OpenStack-Helm 的 Octavia chart 比 Kolla 更完整
- Magnum（K8s CaaS）

### 課程 (5) — Terraform 接手
- 本機 Mac 裝 Terraform
- `terraform-provider-openstack` 寫 provider config
- 用 HCL 建：
  - keypair / network / router / secgroup / instance / floating-ip
- `terraform apply` → SSH 進 VM

### 課程 (6) — Octavia LoadBalancer 實戰
- Terraform + Octavia 建 LB + listener + pool + member
- Sprint 1 最大遺憾這裡補完

### 課程 (7) — Magnum K8s cluster with workload
- 建 cluster template + cluster
- 看 workload 能不能真的跑（Sprint 1 卡在 upstream image URL）

### 課程 (8) — Foreman 未來對應文件 + 回顧
- 不實跑 Foreman
- Runbook 每個章節補「Sprint 3 Foreman 時這裡會變 X」對應表
- Sprint 2 回顧文件

### （選配）Teardown 演練

## 預期會踩的雷（預防接種）

跟 Sprint 1 比 Sprint 2 會踩到不同的雷：

### 1. **MetalLB L2 跟 Azure 網路衝突**
Azure VM 的 private subnet 不讓你隨便宣告 IP。MetalLB L2 mode 要選 VM 內部 lo 或 dummy interface 的 IP 範圍，不能佔 Azure subnet IP。

### 2. **OVN 在 K8s DaemonSet 裡 privileged + hostNetwork**
- DaemonSet spec 要對
- host 的 OVS userspace 要裝（chart 或 Ansible 處理）
- debug 時要跨 K8s + OVS 兩邊看 flow

### 3. **Cinder 在單節點 K8s 上的 LVM backend**
- K8s DaemonSet 要 privileged + hostPath `/dev`
- `/dev/mapper/control` ioctl 權限
- host 上要先把 data disk 做成 `cinder-volumes` VG

### 4. **openstack-helm chart 版本漂移**
- 兩個 repo（osh + osh-infra）的 commit hash 要對齊
- Chart 的 default image tag 常常落後，要在 values 覆蓋

### 5. **Keystone DB migration 卡住**
- chart 用 K8s Job 跑 `keystone-manage db_sync`
- 順序錯了 mariadb 還沒 ready 就跑 → 卡
- 要等 mariadb ready probe 通過才能部 keystone

### 6. **Horizon VIP 不能自動給 DNS 名**
- MetalLB 給 IP，沒給 DNS
- 可以自己改 `/etc/hosts` 或裝 CoreDNS override

### 7. **Octavia amphora image**
- Chart 預設拉 `docker.io/openstackhelm/openstack-helm-amphora`
- 網路慢要 pre-pull
- 有些 chart 版本 image tag 錯誤，要自己 build

### 8. **Magnum driver 老舊問題（跟 Sprint 1 類似）**
- openstack-helm 用同一批 Kolla image，Magnum driver 依然是 upstream 那套
- 預期 flannel / k8s-keystone-auth 的 image URL 還是會壞
- **但 K8s 版 chart 可能會更新頻率高一點**

## 課程設計原則

1. **每步驟完成 → 寫 runbook `sprint2-dayN-*.md`**
2. **每天結束 → 更新 HTML 教學**
3. **使用者已會 K8s**：不用解釋 Pod / Deployment / Service / Probe / CRD
4. **使用者 Ansible 新手**：本 Sprint 不碰 Ansible 了
5. **Kolla 相關** 只出現在「vs OpenStack-Helm」對照表
6. **Bare metal 層被 Azure VM 代替**：每章節標未來 Sprint 3 Foreman 時這裡會變啥

## 歷史檔案處理

Day 0 runbook（sprint2-day-0-azure-vm.md）= 保留，環境相同。

Day 1 runbook（sprint2-day-1-ansible-kolla-install.md）= 改成 "Day 1 歷史紀錄：原本走 Kolla 路線的殘留，Sprint 2 後來 pivot 到 OpenStack-Helm"。不改為正式 Day 1。

**正式的 Sprint 2 Day 1** = K8s + Helm + infra prep，新寫。

## 下次動作點

實際動手時從 **Sprint 2 Day 1 = K8s bootstrap** 開始：
1. SSH 進 VM
2. Cleanup 舊的 kolla-venv + /etc/kolla
3. `kubeadm init`
4. 逐步上 CNI / MetalLB / Helm / local-path-provisioner
5. 寫 runbook `sprint2-day-1-k8s-helm-bootstrap.md`
6. 更新 HTML
