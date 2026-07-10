# Day 3: MySQL InnoDB Cluster 部署（取代 Day 1 的 `mysql`）

![OpenStack Charms 官方吉祥物](../assets/mascots/openstack-charms.png){ align=right width="90" }

## 目的

部署 OpenStack 專用的 `mysql-innodb-cluster` charm，取代 Day 1 用於學習 relation 概念的 `mysql` charm。

## ⚠️ 重要 Gotcha：`mysql` charm ≠ `mysql-innodb-cluster` charm

### 現象

Day 3 第一次嘗試 `juju integrate keystone:shared-db mysql:shared-db`，keystone hook 在 60 秒內 fail：

```
keystone/0*  error  hook failed: "shared-db-relation-changed" for mysql:shared-db
keystone   waiting  "Allowed_units list provided but this unit not present"
```

### 根本原因

兩個都宣稱提供 `mysql-shared` interface 的 charm，實際實作完全不同：

| Charm | Publisher | 真正工作的 interface | OpenStack 相容? |
|---|---|---|---|
| `mysql` (8.0/stable) | Canonical | `database: mysql_client`（新） | ❌ |
| `mysql-innodb-cluster` (8.0/stable) | OpenStack Charmers | `shared-db: mysql-shared`（舊） | ✅ |

`mysql` charm 的 `juju info` 雖然列出 `shared-db: mysql-shared`，但它**沒有實際實作這個 interface 的 relation-changed hook**。整合後 relation data bag 是空的：

```yaml
# juju show-unit mysql/0（錯誤狀態下）
endpoint: shared-db
application-data: {}          # ← 應該要有 password/db_host/allowed_units
related-units:
  keystone/0:
    data:                      # 只有 keystone 送來的 (username/hostname/database)
      database: keystone
      hostname: 10.254.154.140
      username: keystone
```

keystone 等著拿 `allowed_units` 等回傳資料，等不到就報 "Allowed_units list provided but this unit not present"。

### 結論

**OpenStack 部署 100% 用 `mysql-innodb-cluster`，不要用 `mysql`。**

---

## 對應官方指南

[Install OpenStack § MySQL InnoDB Cluster](https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/install-openstack.html)

```bash
juju deploy -n 3 --to lxd:0,lxd:1,lxd:2 --channel 8.0/stable mysql-innodb-cluster
```

我們的 LXD 版簡化（不指定 `--to`，讓 Juju 自動開新 container）：

```bash
juju deploy -n 3 --channel=8.0/stable mysql-innodb-cluster
```

---

## Step 1: 清掉舊的 mysql + 失敗的 keystone

```bash
juju remove-application keystone --force --no-prompt
juju remove-application mysql --destroy-storage --force --no-prompt

# 等 unit 真的消失（可能 1-2 分鐘）
while juju status --format=tabular 2>&1 | grep -qE "^keystone|^mysql "; do
  echo "still removing..."; sleep 20
done
juju status
```

## Step 2: 部署 mysql-innodb-cluster（3 unit）

```bash
juju deploy mysql-innodb-cluster -n 3 --channel=8.0/stable
```

**⏱ 10-15 分鐘**：
1. LXD 開 3 個新 container（machine 4, 5, 6）
2. 每個 unit 裝 MySQL 8.0 + MySQL Router
3. Group Replication 配置
4. Cluster 形成（elect primary）

### Relation（peer）觀察

部署後立刻看 `juju status --relations`，會看到：

```
mysql-innodb-cluster:cluster      mysql-innodb-cluster:cluster      mysql-innodb-cluster  peer  joining
mysql-innodb-cluster:coordinator  mysql-innodb-cluster:coordinator  coordinator            peer  joining
```

兩條 peer relation：
- `cluster` — InnoDB Cluster group replication
- `coordinator` — 協調 leader election 用

`joining` 會變成穩定狀態等各 unit ready。

## Step 3: 整合 TLS

```bash
juju integrate mysql-innodb-cluster:certificates self-signed-certificates:certificates
```

跟 Day 1 的 `mysql` 一樣是 `tls-certificates` interface。

## Step 4: 等全部 active

```bash
juju wait-for application mysql-innodb-cluster --timeout=20m
juju status
```

**預期最終狀態**：

```
App                   Status  Scale  Charm                 Channel
mysql-innodb-cluster  active    3/3  mysql-innodb-cluster  8.0/stable

Unit                      Workload  Message
mysql-innodb-cluster/0*   active    Unit is ready: Mode: R/W, Cluster is ONLINE...
mysql-innodb-cluster/1    active    Unit is ready: Mode: R/O, Cluster is ONLINE...
mysql-innodb-cluster/2    active    Unit is ready: Mode: R/O, Cluster is ONLINE...
```

`/0*` 是 leader 也是 R/W primary。
`/1`, `/2` 是 R/O replica。

---

## Step 5: 驗證 Cluster

```bash
# 從 leader 查 cluster status
juju ssh mysql-innodb-cluster/leader '
  sudo mysqlsh --uri clusteradmin@localhost \
    --password=$(sudo cat /var/lib/mysql/mysql-cluster-admin-password) \
    -- cluster status
'
```

應該看到 3 個 member 都 ONLINE。

---

## Step 6: 記錄關鍵 credentials 取得方式

OpenStack service（keystone, glance, nova-cc...）通過 `shared-db` relation 自動拿到 db credentials，不用手動建 user/db。

但若要手動進 mysql：

```bash
# 從 leader 拿 root password
juju ssh mysql-innodb-cluster/leader 'sudo cat /var/lib/mysql/mysql.passwd'
# 連線
juju ssh mysql-innodb-cluster/leader 'sudo mysql -u root -p<password>'
```

---

## Gotchas

### ❌ 用 `mysql` charm（Canonical 8.0/stable）整合 OpenStack

就是本文開頭那段「hook failed: shared-db-relation-changed」。

**正確做法**：改用 `mysql-innodb-cluster`（publisher: OpenStack Charmers）。

### ⚠️ 3 unit 記憶體占用

每個 mysql-innodb-cluster unit 大概吃 2-4GB RAM。3 unit 約 6-12GB。
我們 Azure VM 128GB RAM，輕鬆。但如果你用較小的 VM 要注意。

### ⚠️ Cluster 初始化順序

MySQL InnoDB Cluster 需要 quorum（3/2/1 不同保護級別）：
- 3 unit：安全，允許 1 個 unit 掛掉不影響
- 2 unit：不建議
- 1 unit：不是真正的 cluster

Juju charm 預設會等 3 unit 都起來才 elect primary，所以整個過程要等最慢的 unit。

---

## Replay

```bash
juju deploy mysql-innodb-cluster -n 3 --channel=8.0/stable
juju integrate mysql-innodb-cluster:certificates self-signed-certificates:certificates
juju wait-for application mysql-innodb-cluster --timeout=20m
```
