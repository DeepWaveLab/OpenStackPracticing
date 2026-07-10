# Day 3: Placement 部署

## 目的

部署 Placement — OpenStack 的 Resource Inventory 服務。記錄每台 compute node 的資源容量（vCPU/RAM/disk），Nova scheduler 排程時查它。

**Nova 硬依賴 placement**，所以一定要先部署 placement 才能部署 nova-cloud-controller。

## 對應官方指南

[Install OpenStack § Placement](https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/install-openstack.html)

```bash
juju deploy --channel 2024.1/stable placement
```

## Prerequisites

- mysql-innodb-cluster active
- rabbitmq-server active
- self-signed-certificates active
- keystone active

## 三朵雲對照

**placement 沒有對外的 AWS/Azure/GCP 等價 service**。內部概念都有（capacity tracking + scheduler），但不公開 API：

| 功能 | OpenStack | AWS | Azure | GCP |
|---|---|---|---|---|
| 資源 inventory API | ✅ `placement` port 8778 | ❌ 內部 | ❌ 內部 | ❌ 內部 |
| 相近概念 | Resource Provider / Traits / Allocations | Dedicated Host / Placement Groups | Dedicated Host / Proximity Placement Groups | Sole-tenant Node / Placement Policies |

詳見對話紀錄。

---

## Step 1: Deploy + 4 條 relation

```bash
juju deploy placement --channel=2024.1/stable

juju integrate placement:shared-db         mysql-innodb-cluster:shared-db
juju integrate placement:amqp              rabbitmq-server:amqp
juju integrate placement:certificates      self-signed-certificates:certificates
juju integrate placement:identity-service  keystone:identity-service
```

### Relation 意義（跟 glance 同樣模式）

| Relation | 做什麼 |
|---|---|
| `shared-db` | placement 自己的 DB（存 resource_providers, inventories, allocations, traits 四張 table）|
| `amqp` | 發 notification（容量變化事件）|
| `certificates` | TLS |
| `identity-service` | 註冊進 keystone catalog |

## Step 2: 等 placement active

```bash
juju wait-for application placement --timeout=15m
```

**⏱ 約 5-10 分鐘**（類似 glance）。

## Step 3: 驗證

```bash
source ~/openrc

# 確認 service catalog 多一筆
openstack service list
# 應該看到: keystone / glance / placement

# Placement 的 endpoints
openstack endpoint list --service placement
# 3 筆 (admin/internal/public)，port 8778

# 測試直接打 placement API（OpenStack 用的 microversion 1.x）
openstack resource provider list
# 空的（還沒部署 nova-compute）— 這就是正確結果
```

## Step 4: 確認 DB 建好

```bash
juju ssh placement/leader 'sudo grep "connection" /etc/placement/placement.conf | head -3'
# 應該看到指向 mysql-innodb-cluster 的連線字串
```

---

## 等到 nova-compute 部署後再看的事（placement 的真正用途）

nova-compute 部署後會自動向 placement 註冊自己：

```bash
# 會出現 1 筆 resource provider（對應 nova-compute 那台 machine）
openstack resource provider list

# 查這個 provider 的 inventory（VCPU / MEMORY_MB / DISK_GB）
openstack resource provider inventory list <provider-uuid>

# 查目前 allocation（哪些 VM 佔用了多少）
openstack resource provider allocation list <provider-uuid>
```

現在還沒 compute node，列表就是空的，**這是正確狀態**，不是 bug。

---

## Gotchas

### 🔴 charm 沒自動跑 DB schema migration（**確定有這雷**）

**症狀**：
- placement unit active（`juju status placement` 看起來沒事）
- port 8778 在聽
- 但 `openstack resource provider list` 回 `HTTP 500 Internal Server Error`
- `/var/log/apache2/placement_error.log` 看到：
  ```
  sqlalchemy.exc.ProgrammingError: (pymysql.err.ProgrammingError)
  (1146, "Table 'placement.traits' doesn't exist")
  ```

**原因**：charm 2024.1/stable rev 125 沒自動觸發 `placement-manage db sync`。DB 建出來了、帳密也設了，但 schema 是空的。

**解法**（手動補）：
```bash
juju ssh placement/0 -- sudo placement-manage db sync
juju ssh placement/0 -- sudo systemctl restart apache2
```

### ⚠️ Status 常駐 `blocked`「'placement' missing」

即使一切正常，charm workload status 會顯示：
```
placement/0*  blocked  idle  "'placement' missing"
```

**這不是錯**。意思是 `placement` provides 的 endpoint 還沒被消費者接。nova-cloud-controller 部署 + 整合後會自動解除。

---

## 實測驗證輸出（2026-04-19 07:10 UTC）

```
$ openstack service list
| keystone  | identity  |
| glance    | image     |
| placement | placement |   ← 新增

$ openstack endpoint list --service placement
| placement | placement | True | public   | http://10.254.154.84:8778 |
| placement | placement | True | admin    | http://10.254.154.84:8778 |
| placement | placement | True | internal | http://10.254.154.84:8778 |

$ openstack resource provider list
(empty, because no nova-compute registered yet - this is correct)
```

## Replay

```bash
juju deploy placement --channel=2024.1/stable
juju integrate placement:shared-db         mysql-innodb-cluster:shared-db
juju integrate placement:amqp              rabbitmq-server:amqp
juju integrate placement:certificates      self-signed-certificates:certificates
juju integrate placement:identity-service  keystone:identity-service
juju wait-for application placement --timeout=15m

source ~/openrc
openstack service list
openstack endpoint list --service placement
openstack resource provider list   # 應該空的
```
