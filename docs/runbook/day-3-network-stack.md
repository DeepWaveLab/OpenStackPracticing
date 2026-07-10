# Day 3: Network Stack（OVN + Neutron）部署

![Neutron 官方吉祥物](../assets/mascots/neutron.png){ align=right width="110" }

## 目的

部署 OpenStack 網路層 — 4 個 charm、13 條 relation 最複雜的一段。完成後可以建 virtual network / router / floating IP / security group。

## 對應官方指南

[Install OpenStack § Neutron](https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/install-openstack.html)

## Prerequisites

- nova-cloud-controller active（neutron-api 要接它）
- nova-compute active（ovn-chassis subordinate 要附著在它上面）
- mysql / rabbitmq / keystone / self-signed-certificates 都 active

---

## ⚠️ Channel / Base 相容性地雷

2026-04 當下實測：

| Charm | 可用 amd64 channel | Base | 備註 |
|---|---|---|---|
| ovn-central | `22.03/stable` rev 304 | **ubuntu@20.04 only** | amd64 沒 jammy 版，要強制 focal |
| neutron-api | `2024.1/stable` rev 650 | ubuntu@22.04 ✓ | 對齊其他 OpenStack charm |
| neutron-api-plugin-ovn | `2023.2/stable` rev 163 | ubuntu@22.04 ✓ | 2024.1/stable 只 arm64，退一版 |
| ovn-chassis | `22.03/stable` rev 324 | ubuntu@22.04 ✓ | subordinate, 走 nova-compute 的 base |

**混 base 部署是正常**的 — ovn-central 跑 focal，其他跑 jammy。OVN 內部的 OVSDB 協定對 host OS 版本不敏感。

---

## Step 1: 4 個 Charm 部署（**都要 Bobcat**）

⚠️ 見下方 Gotchas：**neutron-api 必須是 2023.2/stable，2024.1 會壞 OVN plugin**。

```bash
juju deploy ovn-central             --channel=22.03/stable --base=ubuntu@20.04
juju deploy neutron-api             --channel=2023.2/stable                 # ← bobcat
juju deploy neutron-api-plugin-ovn  --channel=2023.2/stable
juju deploy ovn-chassis             --channel=22.03/stable

# ovn-central 要 3 unit 才能 operate
juju add-unit ovn-central -n 2
```

⚠️ **nova-compute 也得是 bobcat**：
```bash
# 如果已經部署 nova-compute 2024.1/stable（caracal），要先清掉重建
juju remove-application nova-compute --force --destroy-storage --no-prompt
# 等清乾淨後
juju deploy nova-compute --channel=2023.2/stable \
  --constraints="virt-type=virtual-machine mem=8G cores=4 root-disk=50G"
```

## Step 2: 13 條 Relation

```bash
# ovn-central 自己只要 cert
juju integrate ovn-central:certificates self-signed-certificates:certificates

# neutron-api 核心 5 條
juju integrate neutron-api:shared-db         mysql-innodb-cluster:shared-db
juju integrate neutron-api:amqp              rabbitmq-server:amqp
juju integrate neutron-api:certificates      self-signed-certificates:certificates
juju integrate neutron-api:identity-service  keystone:identity-service
juju integrate neutron-api:neutron-api       nova-cloud-controller:neutron-api

# neutron-api-plugin-ovn (subordinate) 綁 neutron-api + 連 OVN
juju integrate neutron-api-plugin-ovn:neutron-plugin  neutron-api:neutron-plugin-api-subordinate
juju integrate neutron-api-plugin-ovn:ovsdb-cms       ovn-central:ovsdb-cms
juju integrate neutron-api-plugin-ovn:certificates    self-signed-certificates:certificates

# ovn-chassis (subordinate) 綁 nova-compute + 連 OVN + mq/cert
juju integrate ovn-chassis:nova-compute    nova-compute:neutron-plugin
juju integrate ovn-chassis:ovsdb           ovn-central:ovsdb
juju integrate ovn-chassis:amqp            rabbitmq-server:amqp
juju integrate ovn-chassis:certificates    self-signed-certificates:certificates
```

### 13 條的心智模型

```
                    self-signed-certificates (cert provider)
                             │
          ┌──────────┬───────┼───────┬──────────────┐
          │          │       │       │              │
      ovn-central   neutron-api      neutron-api-plugin-ovn   ovn-chassis
          │           │ └── mysql     │                        │  └── rabbitmq
          │           │     rabbitmq  │                        │
          │           │     keystone  │                        │
          │           └── nova-cc     │                        │
          │                            │                        │
          └─ ovsdb-cms ─────── 連到 ─┘                        │
          │                                                     │
          └─ ovsdb ─────────────────── 連到 ───────────────────┘
                                              │
                                              │ nova-compute (subordinate 附著於此)
                                              │
                                              └ neutron-plugin ↔ ovn-chassis:nova-compute
```

### 關鍵概念

- **subordinate**（`neutron-api-plugin-ovn`, `ovn-chassis`）不佔獨立 LXD machine，**附著**在 primary（分別是 neutron-api 和 nova-compute）身上
- **ovsdb vs ovsdb-cms**：
  - `ovsdb`（給 chassis 用）= Southbound DB，存實際流表
  - `ovsdb-cms`（給 plugin 用）= Northbound DB，存邏輯拓撲
  - neutron 在 Northbound 寫，chassis 從 Southbound 讀 + 實作

## Step 3: 等 ovn-central + neutron-api active

```bash
juju wait-for application ovn-central --timeout=20m
juju wait-for application neutron-api --timeout=20m
```

**⏱ 20-30 分鐘**：
- ovn-central focal container 開機 + 裝 ovsdb-server + 形成 Raft cluster（1 unit 也要跑 cluster setup）
- neutron-api 裝 python-neutron + 5 條 relation 交換
- subordinate 附著完成 + 自己的 3 條 relation

## Step 4: 驗證

```bash
source ~/openrc

# catalog 多了 neutron
openstack service list

# neutron 的 agent 應該都 up
openstack network agent list

# 空 network list（還沒建）
openstack network list
```

### 預期 `network agent list`

```
Host                  Binary                   State   Alive  Availability Zone
juju-...-12           neutron-ovn-metadata     up      :-)    —
juju-...-11.lxd       networking-ovn           up      :-)    —   (ovn-chassis subordinate 在 nova-compute 上)
```

---

## Step 5: 建第一個 network（確認 OVN 運作）

```bash
# 建 external network (flat, 對應 LXD bridge)
openstack network create --external --share \
  --provider-network-type flat \
  --provider-physical-network physnet1 \
  ext_net

# 建 external subnet (LXD lxdbr0 預設 10.254.154.0/24 沒法用，要另外建 bridge)
# 或用 vxlan/geneve 封裝 - 之後 VM 之間 L2
```

**這裡會有 LXD bridge ↔ OVN physnet 對接的地雷**，等真的動手才詳細記。

---

## Gotchas

### 🔴 **CRITICAL**：SSC vs OVN tls-certificates interface 不相容

**症狀**：ovn-central 卡 "certificates awaiting server certificate data"，SSC 不回 CSR。

**根因**：
- `self-signed-certificates` 1/stable 用 **新 v3 tls-certificates interface**（application-data bag）
- OVN 22.03/stable 用 **舊 legacy interface**（per-unit data bag with `certificate_name`/`sans`/`unit_name`）
- 同名 interface，不同 schema

**解法**：部署 **Vault 1.8/stable**（OpenStack-charmers 舊版，支援 legacy interface）。見 [day-3-vault-tls.md](./day-3-vault-tls.md)。

Vault 1.15+ 是新 HashiCorp 架構，**不能用**。

### 🔴 OVN 22.03/stable charms 不支援 "caracal" release

**症狀**（neutron-api-plugin-ovn 或 ovn-chassis）：
```
RuntimeError: Release caracal is not a known OpenStack release?
```
hook "install" failed。

**根因**：
- `ovn-central`、`ovn-chassis`、`neutron-api-plugin-ovn` amd64 版本 channels 最新是 `22.03/stable` 或 `2023.2/stable`（2024-11 到 2025-04 間更新的 revision）
- 這些 charm 的 `charms_openstack` 庫寫死的 OpenStack release list 只到 bobcat，**完全不認識 "caracal"**
- 當 OpenStack charm（nova-compute, neutron-api）是 2024.1/stable（caracal），relation data 或 package 版本帶 caracal 字樣，OVN charm 讀到就 crash

**可用的 amd64 channels**（實測 2026-04）：
| Charm | 最新 amd64 channel |
|---|---|
| `ovn-central` | `22.03/stable` rev 304 (**只 focal 20.04**) |
| `ovn-chassis` | `22.03/stable` rev 324 (focal + jammy) |
| `neutron-api-plugin-ovn` | `2023.2/stable` rev 163 (**stuck at bobcat**) |
| `neutron-api` | `2024.1/stable` ✓ 但用 bobcat plugin 要降 |

**解決方案**：**neutron layer + nova-compute 必須降到 bobcat（2023.2/stable）**。keystone/glance/placement/nova-cc 可以留在 caracal（cross-version 有支援）。

具體做法：
1. `juju deploy neutron-api --channel=2023.2/stable` （不用 2024.1）
2. `juju deploy neutron-api-plugin-ovn --channel=2023.2/stable`
3. `juju remove-application nova-compute --force --destroy-storage`
4. `juju deploy nova-compute --channel=2023.2/stable --constraints=...`（LXD VM 要重建）
5. `juju deploy ovn-chassis --channel=22.03/stable`
6. 所有 relations 重接

**不要試的路**：
- ❌ `juju config nova-compute openstack-origin=cloud:jammy-bobcat`（in-place 降級）：charm 不會真的 apt downgrade，packages 維持 caracal
- ❌ `juju deploy ovn-chassis --config openstack-origin=...`：ovn-chassis **沒有** openstack-origin config

### ⚠️ ovn-central 要 3 unit

```
Charm requires peers to operate, add more units. A minimum of 3 is required for HA
```

解法：`juju add-unit ovn-central -n 2`（從 1 補到 3）。

## Replay

```bash
# Deploys
juju deploy ovn-central             --channel=22.03/stable --base=ubuntu@20.04
juju deploy neutron-api             --channel=2024.1/stable
juju deploy neutron-api-plugin-ovn  --channel=2023.2/stable
juju deploy ovn-chassis             --channel=22.03/stable

# 13 integrations (見上文)

juju wait-for application ovn-central --timeout=20m
juju wait-for application neutron-api --timeout=20m
source ~/openrc
openstack service list
openstack network agent list
```
