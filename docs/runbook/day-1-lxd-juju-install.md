# Day 1: LXD + Juju 安裝

## 目的

在 Azure VM 上裝 LXD（作為 Juju 的 cloud）+ Juju（deployment engine），並用 `/dev/sdb` 建 ZFS pool。

## 對應官方指南

部分對應 [Chapter 2: Install Juju](https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/install-juju.html)

**關鍵差異**：
- 指南使用 MaaS 作為 cloud → 我們用 **LXD (`localhost` cloud)**
- 指南不需要 LXD → 我們的 LXD 安裝與初始化是**新增**步驟

## Prerequisites

- Day 0 完成（VM 可 SSH，`/dev/sdb` 未格式化 512GB）

---

## Step 1: 登入 VM

```bash
ssh -i ~/.ssh/juju_id_rsa azureuser@20.210.233.133
```

所有以下指令在 VM 上執行。

---

## Step 2: 裝 LXD

**做什麼**：裝 LXD 5.21 LTS。

**為什麼**：LXD 扮演 Juju 的 cloud provider（取代 MaaS）。版本 `5.21/stable` 是 Ubuntu 24.04 配套的 LTS。

```bash
sudo snap install lxd --channel=5.21/stable
```

**預期輸出**：
```
lxd (5.21/stable) 5.21.4-aee7e08 from Canonical** installed
```

---

## Step 3: 裝 Juju

**做什麼**：裝 Juju 3.6.21（目前 3.x LTS）。

**⚠️ 重要版本決策**：
- **官方指南寫 `--channel 3.1`，但 `3.1/stable` 已不存在**
- Juju 版本政策：只有 LTS（2.9、3.6）長期維護 stable，3.1–3.5 是過渡版，被取代後降到 candidate/beta/edge
- 所以用 `3.6/stable` 取代（3.x 向後相容，charm 不受影響）

```bash
sudo snap install juju --channel=3.6/stable --classic
```

**預期輸出**：
```
juju (3.6/stable) 3.6.21 from Canonical** installed
Warning: flag --classic ignored for strictly confined snap juju
```

> `--classic` 被忽略沒關係，Juju 3.6 改成 strict confinement 後內建處理檔案路徑。

---

## Step 4: 把 azureuser 加入 lxd group

**做什麼**：讓 `azureuser` 能不用 sudo 操控 LXD（透過 `/var/snap/lxd/common/lxd/unix.socket`）。

**為什麼**：Juju bootstrap 到 LXD 時，juju client 會透過 socket 跟 LXD 溝通。root 不建議直接跑 juju client。

```bash
sudo usermod -aG lxd azureuser
```

**重要**：加入 group 後，**要重開 SSH 才生效**。目前 session 的 group list 不會更新。從 Step 5 開始重新 SSH（或用 `sg lxd` / `newgrp lxd`）。

**驗證**（重新 SSH 後）：
```bash
id azureuser
# 應該看到: groups=1000(azureuser),4(adm),24(cdrom),27(sudo),30(dip),105(lxd)
```

---

## Step 5: LXD init with Preseed（ZFS on /dev/sdb）

**做什麼**：一次性初始化 LXD，指定 storage pool 和 bridge network。

**為什麼 preseed 不用互動模式**：
- 可 replay（這個檔案本身就是 config）
- 參數明確，沒人為選錯

**為什麼 ZFS**：snapshot / CoW / 壓縮，LXD 官方推薦，效能比 dir backend 好 10 倍。

```bash
cat <<'EOF' | sudo lxd init --preseed
config: {}
networks:
- config:
    ipv4.address: auto
    ipv6.address: none
  description: ""
  name: lxdbr0
  type: bridge
storage_pools:
- config:
    source: /dev/sdb
  description: ""
  name: default
  driver: zfs
profiles:
- config: {}
  description: ""
  devices:
    eth0:
      name: eth0
      network: lxdbr0
      type: nic
    root:
      path: /
      pool: default
      type: disk
  name: default
projects: []
cluster: null
EOF
```

**YAML 解說**：

| 區塊 | 意義 |
|---|---|
| `storage_pools.source=/dev/sdb` | 整顆 disk 建 ZFS pool，名為 `default` |
| `storage_pools.driver=zfs` | ZFS backend |
| `networks.lxdbr0` | 建一條 LXD-managed bridge，LXD 自動挑沒衝突的 subnet |
| `networks.ipv6.address=none` | 關 IPv6（Azure VM 沒 IPv6，簡化） |
| `profiles.default` | 所有未指定 profile 的 container / VM 預設套用這個（綁 default pool + lxdbr0） |

**預期輸出**：（靜默，沒訊息就是成功）

---

## Step 6: 驗證 LXD

```bash
# 版本
lxd --version                # 5.21.4 LTS
juju --version               # 3.6.21-genericlinux-amd64

# Storage pool
sudo lxc storage list
# +---------+--------+---------+-------------+---------+---------+
# |  NAME   | DRIVER | SOURCE  | DESCRIPTION | USED BY |  STATE  |
# +---------+--------+---------+-------------+---------+---------+
# | default | zfs    | default |             | 1       | CREATED |
# +---------+--------+---------+-------------+---------+---------+

# Network
sudo lxc network list
# lxdbr0 bridge YES 10.254.154.1/24 none ... CREATED

# /dev/sdb 現在的分區（應該看到 sdb1 + sdb9，ZFS 的標準結構）
lsblk /dev/sdb
# sdb  512G
# ├─sdb1 512G  (資料)
# └─sdb9 8M    (ZFS 保留區)
```

---

## Gotchas

### ❌ `juju --channel=3.1/stable` 失敗

```
error: snap "juju" is not available on 3.1/stable but is available to install
       on the following channels: 3.1/candidate, 3.1/beta, 3.1/edge
```

**原因**：Juju 3.1 已 EOL 從 stable 退出，只剩 pre-release channel。
**解法**：改用 `3.6/stable`（當前 3.x LTS）。

### ⚠️ `zpool: command not found`

**原因**：LXD 透過 snap 自帶 ZFS，不用 host 的 zpool CLI。
**解法**：用 `sudo lxc storage info default` 看 ZFS pool 資訊。

### ⚠️ group 變更不即時生效

加入 `lxd` group 後，**目前的 SSH session 不會有新 group**。必須：
- 關 SSH 重登，或
- 用 `sg lxd -c '<command>'`，或
- 用 `sudo` 跑 `lxc` / `juju` 指令

---

## 完整可重跑腳本

複製以下到 VM 上一次跑完（冪等，可重跑）：

```bash
#!/bin/bash
set -e
sudo snap install lxd --channel=5.21/stable
sudo snap install juju --channel=3.6/stable --classic
sudo usermod -aG lxd azureuser
cat <<'EOF' | sudo lxd init --preseed
config: {}
networks:
- config: { ipv4.address: auto, ipv6.address: none }
  name: lxdbr0
  type: bridge
storage_pools:
- config: { source: /dev/sdb }
  name: default
  driver: zfs
profiles:
- config: {}
  devices:
    eth0: { name: eth0, network: lxdbr0, type: nic }
    root: { path: /, pool: default, type: disk }
  name: default
projects: []
cluster: null
EOF
echo "Done. lxd + juju ready."
```
