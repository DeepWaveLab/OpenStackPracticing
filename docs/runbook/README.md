# OpenStack 學習 Runbook

> 此目錄記錄**實際執行過且驗證成功**的指令。與 `docs/plans/` 的差異：
>
> - `docs/plans/` = 要學什麼（計畫）
> - `docs/runbook/` = 實際跑過的指令（replay 時照抄即可）

## 環境資訊

| 項目 | 值 |
|---|---|
| Azure subscription | `<subscription-name>` (id: `<subscription-id>`) |
| Resource group | `souch-openstack` |
| Region | `japaneast` |
| VM | `openstack-lab` (Standard_E16s_v5, Ubuntu 24.04) |
| VM Public IP | `20.210.233.133` |
| VM Private IP | `10.0.0.4` |
| SSH key | `~/.ssh/juju_id_rsa` |
| SSH user | `azureuser` |

## 目錄

1. [Day 0: Azure VM 建置](./day-0-azure-vm.md) ✅
2. [Day 1: LXD + Juju 安裝](./day-1-lxd-juju-install.md) ✅
3. [Day 1: Juju bootstrap + 第一個 model](./day-1-juju-bootstrap.md) ✅
4. [Day 1: 第一次觀察 Juju Relation](./day-1-first-relation.md) ✅（用 mysql + self-signed-certificates）
5. [Day 2: OpenStack Charm 架構地圖](./day-2-openstack-charm-map.md) ✅（18 charm + 部署順序）
6. [Day 3: RabbitMQ 部署](./day-3-rabbitmq.md) ✅（amqp broker + TLS）
7. [Day 3: MySQL InnoDB Cluster](./day-3-mysql-innodb-cluster.md) ✅（⚠️ 含 mysql charm 不相容 OpenStack 的踩坑紀錄）
8. [Day 3: Keystone](./day-3-keystone.md) ✅（第一個 OpenStack 服務 + openrc 設定，注意 port 5000 是 HTTP）
9. [Day 3: Glance](./day-3-glance.md) ✅（Image Registry，file backend，cirros upload 成功）
10. [Day 3: Placement](./day-3-placement.md) ✅（Resource Inventory；⚠️ charm 有 db sync bug，手動補）
11. [Day 3: Nova Cloud Controller](./day-3-nova-cloud-controller.md) ✅（Nova 控制面，scheduler + conductor up）
12. [Day 3: Nova Compute](./day-3-nova-compute.md) ✅（Hypervisor，LXD VM + KVM；⚠️ 必須接 `cloud-credentials`）
13. [Day 3: Network Stack (OVN + Neutron)](./day-3-network-stack.md) ✅（4 charm + 13 relation；⚠️ SSC interface 不合 → 要 Vault）
14. [Day 3: Vault TLS for OVN](./day-3-vault-tls.md) ✅（Vault 1.8/stable；OVN legacy interface 解藥）
15. [Day 3: Horizon (openstack-dashboard)](./day-3-horizon.md) ✅（Web UI，走 HTTP + SSH tunnel）
16. [Day 4: Configure OpenStack — 開第一台 VM](./day-4-configure-openstack.md) ✅（Ubuntu VM end-to-end SSH 通）
17. [Day 5: Heat (Orchestration)](./day-5-heat.md) ✅（Magnum 前置，demo stack 驗證）
18. [Day 6: Magnum (Container Infra)](./day-6-magnum.md) ✅（API 上線 + domain-setup）
19. [Day 7: Magnum K8s Cluster 實建](./day-7-magnum-k8s-cluster.md) 🟡（2 nodes Ready；workload 卡在 Magnum upstream 舊 image URL）
20. [Day 8: Storage 嘗試與 LXD 限制](./day-8-storage-attempts.md) ❌（Ceph / Cinder LVM 在 LXD container 撞 kernel 權限，沒跑完）
21. [Sprint 回顧 + 下一步規劃](../reflection.md) 📝 整個 9 天總結

## ✅ 2026-04-19 nova-compute 重建完成

- `nova-compute` 從 2023.2/stable 升級到 2024.1/stable（Caracal 一致）
- v66/v67 版本錯位問題解決，`openstack compute service list` 看到 daemon up
- QEMU hypervisor up，resource provider 有 4 VCPU / 7956 MB / 48 GB
- 詳見 [day-3-nova-compute.md §第二次部署](./day-3-nova-compute.md#第二次部署caracal-升級2026-04-19)

## 尚未完成

- **Configure OpenStack** — external network、flavor、security group、keypair、first VM 開機
- **Ceph / Cinder**（刻意跳過，Glance file backend 已夠；未來 Magnum 要 persistent volume 再補）

## 快速重登 VM

```bash
ssh -i ~/.ssh/juju_id_rsa azureuser@20.210.233.133
```

## 省錢指令

```bash
# 停機（保留 disk，compute 不收費）
az vm deallocate -g souch-openstack -n openstack-lab \
  --subscription <subscription-id>

# 開機
az vm start -g souch-openstack -n openstack-lab \
  --subscription <subscription-id>

# 查狀態
az vm show -g souch-openstack -n openstack-lab -d --query powerState -o tsv \
  --subscription <subscription-id>
```

⚠️ Public IP 是 **Standard static**，deallocate 不會變。但若你換本機網路，NSG 的 SSH 來源要更新（見 Day 0 §NSG 重設）。
