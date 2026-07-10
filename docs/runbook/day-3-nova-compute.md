# Day 3: Nova Compute 部署

## 目的

部署 Nova 的**運算節點**：libvirt + KVM 跑在這，**實際開 VM 的地方**。這也是**第一個必須跑在 LXD VM（不是 container）的 charm**，因為需要 KVM 硬體虛擬化權限。

## 對應官方指南

[Install OpenStack § Nova Compute](https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/install-openstack.html)

## Prerequisites

- nova-cloud-controller active（它等著 nova-compute 接 `cloud-compute` relation）
- rabbitmq-server active
- glance active

---

## ⚠️ 為什麼 nova-compute 必須用 LXD VM

nova-compute 透過 libvirt + KVM 跑 VM。KVM 需要：
- 存取 `/dev/kvm` device（/dev/kvm 是 KVM 的 ioctl interface）
- 硬體虛擬化支援（nested virt）
- mount cgroup v1/v2

LXD **container** 基於 kernel namespace 共用 host kernel，**不給** `/dev/kvm`。
LXD **VM** 是完整 qemu VM，有 nested KVM，可以跑 `/dev/kvm`。

## Azure VM 內的 3 層 virtualization

```
Azure Hyper-V (外層)
    └── Azure VM (Standard_E16s_v5, nested virt enabled)
           └── LXD VM (KVM inside Azure VM, nested × 2)
                   └── Instance (KVM inside LXD VM, nested × 3)
```

E-series v5 的 VM 都支援這個。

---

## Step 1: Deploy 帶 constraints

```bash
juju deploy nova-compute \
  --channel=2024.1/stable \
  --constraints="virt-type=virtual-machine mem=8G cores=4 root-disk=50G"
```

### Constraints 解讀

| Constraint | 意義 |
|---|---|
| `virt-type=virtual-machine` | **關鍵**：告訴 LXD 這個 machine 要用 VM，不是 container |
| `mem=8G` | LXD VM 分 8GB RAM |
| `cores=4` | LXD VM 配 4 vCPU |
| `root-disk=50G` | root disk 50GB（預設 10GB 不夠跑 KVM + image cache） |

## Step 2: Integrate 4 條 relation

**⚠️ 一定要 4 條，別漏**：

```bash
juju integrate nova-compute:cloud-compute       nova-cloud-controller:cloud-compute
juju integrate nova-compute:amqp                rabbitmq-server:amqp
juju integrate nova-compute:image-service       glance:image-service
juju integrate nova-compute:cloud-credentials   keystone:identity-credentials
```

### Relation 意義

| Relation | 做什麼 |
|---|---|
| `cloud-compute ↔ nova-cloud-controller` | 雙向通道：nova-cc 發指令給 compute，compute 回報狀態。**這條會解除 nova-cc 的 blocked 狀態** |
| `amqp ↔ rabbitmq` | 接 nova scheduler 發的訊息（開 VM 指令） |
| `image-service ↔ glance` | 從 glance 拉 image 到本地 |
| `cloud-credentials ↔ keystone:identity-credentials` | **關鍵**：nova-compute 拿 service account 跟 placement API 驗 token。**漏接會 CRASH**（見 Gotchas） |

### 尚未接、等 neutron/ovn 部署後才接

- `neutron-plugin` ← 由 `ovn-chassis`（subordinate charm）提供

### 不接的

- `certificates` — nova-compute **不需要** TLS（內部走 AMQP，不對外開 HTTP API）
- `ceph`, `ceph-access` — 沒部署 Ceph
- `ephemeral-backend` — 選配
- `lxd` — 不是「給 LXD cloud 用」，是 nova 自己的 LXD driver（舊架構），我們不用
- `ironic-api`, `nova-vgpu`, `secrets-storage`, `nova-ceilometer` — 選配

## Step 3: 等 nova-compute active

```bash
juju wait-for application nova-compute --timeout=25m
```

**⏱ 15-25 分鐘**（LXD VM 比 container 慢很多）：
1. LXD 建 VM disk image (1-2 min)
2. VM 開機 + cloud-init (3-5 min)
3. Juju agent install (1-2 min)
4. apt install nova-compute + libvirt + qemu-kvm (~10 min)
5. config-changed + 3 relation-changed hooks (~3 min)

## Step 4: 驗證 compute node 已加入

```bash
source ~/openrc

# nova-compute daemon 註冊到 nova-cc
openstack compute service list
# 應該多一筆 nova-compute daemon

# 列出所有 hypervisor
openstack hypervisor list
# 應該看到 1 台 hypervisor（我們的 nova-compute unit）

# placement 現在該有 resource provider 了
openstack resource provider list
# 對應 nova-compute 的 resource provider

# inventory
openstack resource provider inventory list <provider-uuid>
# 看到 VCPU / MEMORY_MB / DISK_GB 的容量
```

### 預期結果

```
openstack compute service list:
| nova-scheduler | juju-...-10 | internal | enabled | up    |
| nova-conductor | juju-...-10 | internal | enabled | up    |
| nova-compute   | juju-...-11 | nova     | enabled | up    |   ← 新增

openstack hypervisor list:
| 1  | juju-...-11 | QEMU | <ip> | up |

openstack resource provider list:
| <uuid> | juju-...-11 | 1 | 1 |
```

## Step 5: nova-cc 狀態應該解除 blocked

```bash
juju status nova-cloud-controller
# workload 應該變 active
```

---

## ⚠️ 尚未完成：還不能開 VM

即使 nova-compute active，**還不能真的 `openstack server create`**，因為：

1. **沒 network** — neutron-api + ovn-central 還沒部署，沒法建 network/subnet
2. **沒 security group / keypair 初始化**
3. **沒 flavor 定義**（要建）

下一個 phase：部署 neutron stack，才能開第一台 VM。

---

## Gotchas

### 🔴 漏接 `cloud-credentials` → nova-compute CRASH（實測遇到）

**症狀**：
- `juju status` 顯示 nova-compute `active / idle / "Unit is ready"`（**騙人**）
- 但 `openstack compute service list` **找不到 nova-compute daemon**
- `openstack hypervisor list` 空的
- `openstack resource provider list` 空的
- nova-cc 仍保持 `blocked: "Missing relations: compute"`（即使我們接了 cloud-compute）

**診斷**：
```bash
juju ssh nova-compute/0 -- sudo tail -30 /var/log/nova/nova-compute.log
```
會看到：
```
CRITICAL nova [...] Unhandled error:
  keystoneauth1.exceptions.auth_plugins.MissingAuthPlugin:
  An auth plugin is required to determine endpoint URL
```

**原因**：2024.1 Caracal 起，placement 完全獨立於 nova，nova-compute 要自己向 placement 註冊 resource provider。需要 keystone service account（透過 `cloud-credentials` relation）來 auth。舊版（zed/yoga 之前）nova-compute 從 nova-cc 間接繼承，新版必須直接接 keystone。

**修正**：
```bash
juju integrate nova-compute:cloud-credentials keystone:identity-credentials
```

等 30 秒，nova-compute 自動寫 `/etc/nova/nova.conf` `[placement]` section + 重啟 → 自動註冊。

### ⚠️ `discover_hosts` 看 "Found 0 unmapped computes"

這不是 error。意思是 nova-compute **已經 map 過** — 所以 unmapped = 0。有 compute 但都 map 完了。

### ⚠️ LXD VM 比 container 慢 3-5 倍

**Container** 5-10 分鐘；**LXD VM** 15-25 分鐘（整台 VM 開機、cloud-init、snap refresh、apt install kvm...）。不要在 LXD VM 階段急著打斷。

## Replay

```bash
juju deploy nova-compute --channel=2024.1/stable \
  --constraints="virt-type=virtual-machine mem=8G cores=4 root-disk=50G"
juju integrate nova-compute:cloud-compute       nova-cloud-controller:cloud-compute
juju integrate nova-compute:amqp                rabbitmq-server:amqp
juju integrate nova-compute:image-service       glance:image-service
juju integrate nova-compute:cloud-credentials   keystone:identity-credentials  # 別漏！
juju wait-for application nova-compute --timeout=25m

source ~/openrc
openstack compute service list     # 應該看到 nova-compute daemon up
openstack hypervisor list           # 應該看到 1 台 hypervisor
openstack resource provider list    # 應該看到 1 個 provider
```

---

## 第二次部署：Caracal 升級（2026-04-19） {#nova-caracal-redeploy}

### 為什麼重做

第一次部署是 `2023.2/stable`（Bobcat），`nova-compute` 的 RPC 版本 v66，跟 `nova-cloud-controller` 2024.1/stable（Caracal, v67）**版本相差 2 個**，跨版相容只容忍 1 個 → 直接 `ServiceTooOld` 拒絕 join cluster。手工 patch `_check_minimum_version` 不成功。唯一解：整個 nova-compute 重建成 Caracal 一致版。

### 步驟

```bash
# 1. 移除舊 nova-compute（同時會拉掉 subordinate ovn-chassis unit）
juju remove-application nova-compute --force --destroy-storage --no-prompt
# 等 80-90 秒完全清掉（LXD VM 銷毀）

# 2. 部署 Caracal 版
juju deploy nova-compute --channel=2024.1/stable \
  --constraints="virt-type=virtual-machine mem=8G cores=4 root-disk=50G"

# 3. 重接 4 條 relation
juju integrate nova-compute:cloud-compute       nova-cloud-controller:cloud-compute
juju integrate nova-compute:amqp                rabbitmq-server:amqp
juju integrate nova-compute:image-service       glance:image-service
juju integrate nova-compute:cloud-credentials   keystone:identity-credentials

# 4. 重接 ovn-chassis subordinate（只需 nova-compute relation；ovsdb/amqp/certs 是 app-level 保留）
juju integrate ovn-chassis:nova-compute nova-compute:neutron-plugin

# 5. 等 LXD VM 起來 + ovn-chassis hook 會 install fail（charmhelpers 不認 caracal）
# 大約 10 分鐘後會看到：ovn-chassis/N workload:error "hook failed: install"
```

### Patch ovn-chassis charmhelpers（重跑 /tmp/patch-ovn-chassis.sh）

LXD VM 重建 → subordinate venv 也是全新 → patch 會被吃掉，必須重做。
詳細 patch script 見 [solutions/integration-issues/ovn-charms-caracal-release-mismatch.md](../solutions/integration-issues/ovn-charms-caracal-release-mismatch.md)。

本次使用的 unit ID 是 `ovn-chassis/3`（隨機），script 需要 subordinate unit 路徑：
```bash
UNIT_DIR=/var/lib/juju/agents/unit-ovn-chassis-3
```

跑法：
```bash
cat /tmp/patch-ovn-chassis.sh | juju ssh nova-compute/2 -- sudo bash
juju resolved ovn-chassis/3
# 30-60 秒後 ovn-chassis active
```

### 驗證結果（實跑）

```
$ openstack compute service list
| 4a5c2cf1-f3a8-42bc-a853-ddc44e42da1d | nova-compute  | juju-59028d-19.lxd | nova     | enabled | up  |

$ openstack hypervisor list
| 2 | juju-59028d-19.lxd | QEMU | 10.254.154.221 | up |

$ openstack resource provider inventory list <RP_UUID>
| VCPU      | 4    |
| MEMORY_MB | 7956 |
| DISK_GB   | 48   |
```

### ⚠️ juju status 假陽性

完成後 `juju status nova-compute` 仍顯示 `blocked: Services not running that should be: nova-api-metadata`。這是**假陽性**：
- ovn-chassis 故意 mask `nova-api-metadata.service`（OVN 用 ovn-metadata-agent 自己處理 metadata）
- nova-compute charm 不知道 OVN 的存在，還在 check 舊 service
- **實際 nova-compute daemon 正常跑**，control plane 也看到它

不影響開 VM。後續如要消掉 warning：`juju config nova-compute enable-live-migration=false`（讓 charm 不要 enforce metadata service）或忽略。

