# Day 1: Juju Bootstrap + 建立 openstack Model

## 目的

Bootstrap 一個 Juju controller 到 LXD（取代 MaaS），並建立之後部署 OpenStack 用的 `openstack` model。

## 對應官方指南

[Chapter 2: Install Juju § Bootstrap + add-model](https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/install-juju.html)

**差異表**：

| 指南（MaaS） | 我們（LXD） |
|---|---|
| `juju add-cloud maas-one -f maas-cloud.yaml` | **跳過**（LXD 是 Juju 內建 cloud，cloud 名 = `localhost`） |
| `juju add-credential maas-one -f maas-creds.yaml` | **跳過**（LXD 本機 socket，無需 credential）|
| `juju bootstrap --bootstrap-series=jammy --constraints tags=juju maas-one maas-controller` | `juju bootstrap --bootstrap-series=jammy localhost openstack-controller` |
| `juju add-model --config default-series=jammy openstack` | **完全照抄** |

## Prerequisites

- Day 1 LXD + Juju 安裝完成（[day-1-lxd-juju-install.md](./day-1-lxd-juju-install.md)）
- `azureuser` 已經在 `lxd` group 且**重新 SSH 過**（或新 session）

---

## Step 1: Bootstrap Juju controller

**做什麼**：要 LXD 給 Juju 開一台 container，把 jujud agent 裝上去，讓它變成 controller。

**為什麼 `--bootstrap-series=jammy`**：對齊 OpenStack 2023.2 charm 支援的 Ubuntu 版本（22.04）。charm 在 `2023.2/stable` channel 預期 host OS 是 jammy。

**為什麼 controller 名字叫 `openstack-controller`**：純命名；未來如果要 bootstrap 其他 cloud（例如另一個 LXD、或 AWS），名稱分開好管理。

```bash
juju bootstrap --bootstrap-series=jammy localhost openstack-controller
```

**⏱ 時間**：約 3-5 分鐘（首次要下載 552MB LXD image）。

**關鍵輸出階段**：

```
1. Looking for packaged Juju agent version 3.6.21 for amd64
2. acquiring LXD image - Retrieving image: rootfs: 100% (~50MB/s)
3. Creating container → juju-<uuid6>-0 (arch=amd64)
4. Installing Juju agent on bootstrap instance
5. Attempting to connect to 10.254.154.xxx:22 (via lxdbr0)
6. Bootstrap complete, controller "openstack-controller" is now available
```

**驗證**：

```bash
juju controllers
```

預期（範例）：
```
Controller             Model  User   Access     Cloud/Region         Models  Nodes    HA  Version
openstack-controller*  -      admin  superuser  localhost/localhost       1      1  none  3.6.21
```

重點：
- `*` = 目前預設 controller
- `Models: 1` = 自動建的 `controller` model（管 controller 自己）
- `Nodes: 1` = 1 個 LXD container 在跑

```bash
lxc list
```

預期：
```
| juju-<uuid6>-0 | RUNNING | 10.254.154.xxx (eth0) | | CONTAINER | 0 |
```

---

## Step 2: 建立 openstack model

**做什麼**：在 controller 下建一個 namespace 叫 `openstack`，之後所有 OpenStack 相關 charm 部署都在這裡面。

**為什麼要分 model**：Juju 的 model 類似 Kubernetes 的 namespace — 隔離應用，權限分組。一個 controller 可以管無數個 model。

**`--config default-series=jammy`**：這個 model 裡新開的 machine 預設用 Ubuntu 22.04。

```bash
juju add-model --config default-series=jammy openstack
```

**⏱ < 5 秒**。只是在 controller 的 MongoDB 裡建一個 entry，不會叫 LXD 開機器。

**驗證**：

```bash
juju models
```

預期：
```
Model       Cloud/Region         Type  Status     Machines  Units  Access  Last connection
controller  localhost/localhost  lxd   available         1      1  admin   just now
openstack*  localhost/localhost  lxd   available         0      -  admin   never connected
```

重點：
- `openstack*` — 星號在這，所有後續 `juju` 指令預設對 openstack model
- `0 Machines` — 驗證「model 不是馬上吃資源」的懶加載原則

```bash
juju status
```

預期：
```
Model      Controller            Cloud/Region         Version  SLA          Timestamp
openstack  openstack-controller  localhost/localhost  3.6.21   unsupported  16:27:50Z

Model "admin/openstack" is empty.
```

---

## 學到的核心概念

### Bootstrap 做了什麼

一個 `juju bootstrap` 指令背後：
1. 跟 cloud（LXD）要一台 machine
2. 在 machine 上裝 jujud 和 MongoDB
3. 從此這台 machine 就是 controller，所有 juju client 指令都打到這裡

**對照 Terraform**：`terraform init` 下載 provider 但不開資源；`juju bootstrap` 會**立即開一個 container**。

### Juju 的層級關係

```
Controller (jujud + MongoDB)
├─ Model: controller  (自動建，管 controller 自己)
└─ Model: openstack   (我們建的，還空著)
     └─ (future) Applications / Units / Machines / Relations
```

### 懶加載

- `juju bootstrap` → 開 1 個 container（controller）
- `juju add-model` → 0 個 container
- `juju deploy <charm>` → 開 N 個 container（依 charm 需求）

---

## Gotchas

### ⚠️ Group 沒生效就 bootstrap 會失敗

如果前一步 `usermod -aG lxd azureuser` 後沒重開 SSH session 就 bootstrap，會看到 permission denied on LXD socket。

**解法**：重開 SSH session（新的 bash shell 會讀 `/etc/group`）。

### ⚠️ `--bootstrap-series=jammy` vs host OS noble

Host（Azure VM）是 Ubuntu 24.04 (noble)，但 controller container 跑 22.04 (jammy)。這不衝突——LXD container 可以跑任何 Ubuntu 版本。

---

## 現狀總結

做完這兩步後：
- **1 個 LXD container**：`juju-<uuid6>-0`（controller）
- **Controller**：`openstack-controller` in `localhost/localhost`
- **Models**：`controller`（自動）+ `openstack*`（我們的）
- **資源**：controller 約佔 500MB disk，小量 RAM
- **下一步**：在 `openstack` model 裡部署 charm

---

## 下次用時的快速 replay

```bash
ssh -i ~/.ssh/juju_id_rsa azureuser@20.210.233.133

# 如果 controller 已存在，跳過 bootstrap；否則：
juju bootstrap --bootstrap-series=jammy localhost openstack-controller

# 如果 model 已存在，跳過；否則：
juju add-model --config default-series=jammy openstack

juju status
```
