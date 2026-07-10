# Sprint 2 / Day 0: Azure VM 重建

![OpenStack-Helm 官方吉祥物](../assets/mascots/openstack-helm.png){ align=right width="100" }

## 目的

Sprint 1 teardown 之後重開一台 VM，準備給 Kolla-Ansible + Terraform 跑 Sprint 2 用。

## 跟 Sprint 1 Day 0 的差異

| | Sprint 1 | Sprint 2 |
|---|---|---|
| Ubuntu 版本 | 24.04 | **22.04**（Kolla-Ansible Caracal 主支援版） |
| Disk | OS 只有 Azure 預設 | OS 128G + **data 256G**（給 Cinder LVM VG） |
| 後續 stack | LXD + Juju | **純 Ubuntu，Docker（Kolla 裝）** |

## 指令（30 秒跑完）

```bash
az vm create \
  --subscription <subscription-id> \
  --resource-group souch-openstack \
  --name openstack-lab \
  --image Ubuntu2204 \
  --size Standard_E16s_v5 \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/juju_id_rsa.pub \
  --public-ip-sku Standard \
  --security-type TrustedLaunch \
  --enable-secure-boot false \
  --enable-vtpm false \
  --data-disk-sizes-gb 256 \
  --os-disk-size-gb 128
```

## NSG 鎖 SSH 到本機 IP

```bash
MY_IP=$(curl -s ifconfig.me)
az network nsg rule update \
  --subscription <subscription-id> \
  -g souch-openstack \
  --nsg-name openstack-labNSG \
  -n default-allow-ssh \
  --source-address-prefixes "$MY_IP/32"
```

## 本次 lab 環境

| 項目 | 值 |
|---|---|
| Subscription | <subscription-name> (`<subscription-id>`) |
| Resource group | `souch-openstack` |
| Region | japaneast |
| VM | `openstack-lab` (Standard_E16s_v5, 16 vCPU / 128 GB RAM) |
| OS | Ubuntu 22.04 LTS, kernel 6.8 |
| Public IP | `203.0.113.20` |
| Private IP | `10.0.0.4` |
| OS disk | sda 128 GB |
| Data disk | sdb 256 GB（未格式化，Day 2/3 會變 Cinder VG） |
| SSH | `ssh -i ~/.ssh/juju_id_rsa azureuser@203.0.113.20` |

## ⚠️ 本階段 Bare Metal 層的模擬

**Sprint 2 用 Azure VM 代替實體機**。未來 Sprint 3 會用 Foreman 管真 / 假 bare metal。對應關係：

| Sprint 2（本階段） | Sprint 3（未來） |
|---|---|
| 一台 Azure VM（即 Kolla AIO host） | 3-5 台 Foreman 管的 bare metal / KVM pod |
| 手動 `az vm create` | `foreman-provisioner` + PXE + Kickstart |
| SSH target = 這台 VM 本身 | SSH target = Foreman API 產出的 host list |
| 所有 OpenStack 服務都在同一台 | 分 control / network / compute / storage 多 node |

Kolla 的 `globals.yml` 在 Sprint 3 幾乎不變，只是 inventory 從 `all-in-one` 換成 `multinode`。

## 下一步

Day 1：Ansible 基礎概念 + Kolla-Ansible 工具安裝到 VM。
