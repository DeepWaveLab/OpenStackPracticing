# Day 3: Glance 部署

## 目的

部署 Glance — OpenStack 的 Image Registry。存 VM image（Ubuntu/CirrOS/...），Nova 開 VM 時從這裡拉。

## 對應官方指南

[Install OpenStack § Glance](https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/install-openstack.html)

原指令：
```bash
juju deploy --channel 2023.2/stable glance
```

我們鎖 `2024.1/stable` 對齊 keystone。

## Prerequisites

- mysql-innodb-cluster active
- rabbitmq-server active
- self-signed-certificates active
- keystone active（glance 要註冊進它的 catalog）

## 為什麼跳過 Ceph 影響不大

生產部署：glance 把 image 存在 Ceph（大檔分散儲存）。
我們環境：沒 Ceph，glance 自動 fallback 到 **file backend**（image 存在 container 本地 `/var/lib/glance/images/`）。

file backend 限制：
- 單台 glance unit 的容量 = 它所在 LXD container 的 disk 配額
- image 不能在 glance 間共享（但我們只有 1 個 glance unit）
- 不適合 production，學習夠用

---

## Step 1: Deploy + 4 條 relation

```bash
juju deploy glance --channel=2024.1/stable
juju integrate glance:shared-db         mysql-innodb-cluster:shared-db
juju integrate glance:amqp              rabbitmq-server:amqp
juju integrate glance:certificates      self-signed-certificates:certificates
juju integrate glance:identity-service  keystone:identity-service
```

### 4 條 relation 的意義

| Relation | 做什麼 |
|---|---|
| `shared-db ↔ mysql-innodb-cluster` | Glance 自己的 DB（存 image 記錄、owner、size、checksum 等 metadata） |
| `amqp ↔ rabbitmq-server` | 發 notification（image 建/改/刪時通知其他服務，像 search 或 audit） |
| `certificates ↔ self-signed-certificates` | TLS cert（慣例接）|
| `identity-service ↔ keystone` | **關鍵**。glance 透過這條：<br>1. 向 keystone 註冊自己 → `openstack service list` 會多一筆<br>2. 拿到 service account → 可以向 keystone 驗 user token |

## Step 2: 等 glance active

```bash
juju wait-for application glance --timeout=15m
```

**⏱ 5-10 分鐘**：
1. LXD 開 container
2. apt install glance-api + python deps
3. config-changed（設 file backend）
4. 4 條 relation 的 changed hook 觸發（拿 DB/MQ/keystone/cert credentials）
5. glance-manage db_sync（建 image tables in mysql）
6. Apache + glance-api 啟動（port 9292）

## Step 3: 驗證（keystone 裡多了 glance service）

```bash
source ~/openrc
openstack service list
# 應該多一筆:  glance / image

openstack endpoint list --service image
# 應該看到 3 筆 glance endpoints (admin/internal/public)
```

## Step 4: 上傳 cirros image 測試

CirrOS 是極小的測試用 Linux image（~15MB），專為 OpenStack demo 設計。

```bash
# 下載
wget http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img \
     -O ~/cirros.img

# 上傳
openstack image create \
  --disk-format qcow2 \
  --container-format bare \
  --public \
  --file ~/cirros.img \
  cirros

# 驗證
openstack image list
# 應該看到 cirros  active
```

## Step 5: 驗證 image 真的存在 glance container 裡

```bash
juju ssh glance/leader 'sudo ls -la /var/lib/glance/images/'
# 會看到一個 uuid 檔名的檔案，size ≈ 15MB
```

---

## Gotchas

### ⚠️ glance 跟 keystone 的 identity-service relation 順序

如果 glance 的 `identity-service` relation 在 keystone 還沒 fully active 時建立，glance 會卡在 `waiting`。解法：先確保 keystone active 再建 glance。（我們是這麼做的，沒遇到這坑。）

### ⚠️ file backend 不支援多 glance unit

如果以後 scale glance 到 2+ unit，file backend 會出問題（每個 unit 的本地 disk 不互通）。scale 時必須先切 Ceph。

### ⚠️ 實測耗時 ≈ 27 分鐘（比預期慢）

hook 執行階段看起來有內部 retry 或 wait。單 unit 部署時若很久沒動靜，看 `juju debug-log --include glance` 確認沒 error 就繼續等。

---

## 實測驗證輸出（2026-04-19 06:56 UTC）

```
$ openstack service list
| keystone | identity |
| glance   | image    |         ← 新增

$ openstack endpoint list --service image
| glance | image | True | internal | http://10.254.154.203:9292 |
| glance | image | True | admin    | http://10.254.154.203:9292 |
| glance | image | True | public   | http://10.254.154.203:9292 |

$ openstack image create --disk-format qcow2 --container-format bare --public --file ~/cirros.img cirros
id: 5a3c23a0-2a2a-4431-8a2e-d36411cb9adb
status: queued → active

$ openstack image list
| 5a3c23a0-... | cirros | active |

$ juju ssh glance/leader 'sudo ls -lah /var/lib/glance/images/'
-rw-r----- 1 glance glance 21M   5a3c23a0-2a2a-4431-8a2e-d36411cb9adb
                                 ^^^ 實體檔，file backend 確認運作
```

---

## Replay

```bash
juju deploy glance --channel=2024.1/stable
juju integrate glance:shared-db         mysql-innodb-cluster:shared-db
juju integrate glance:amqp              rabbitmq-server:amqp
juju integrate glance:certificates      self-signed-certificates:certificates
juju integrate glance:identity-service  keystone:identity-service
juju wait-for application glance --timeout=15m

source ~/openrc
openstack service list
openstack endpoint list --service image

wget http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img -O ~/cirros.img
openstack image create --disk-format qcow2 --container-format bare --public \
  --file ~/cirros.img cirros
openstack image list
```
