# Day 3: Nova Cloud Controller 部署

![Nova 官方吉祥物](../assets/mascots/nova.png){ align=right width="110" }

## 目的

部署 Nova 的**控制面**：API server + scheduler + conductor + novncproxy。這是 OpenStack VM 管理的指揮中心。

## 對應官方指南

[Install OpenStack § Nova Cloud Controller](https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/install-openstack.html)

## Prerequisites

- mysql-innodb-cluster, rabbitmq-server, self-signed-certificates
- keystone, glance, placement **都 active**（nova-cc 硬依賴這三個）

## Nova 架構地圖

Nova 是 OpenStack 最複雜的 project，包含多個 daemon：

```
┌────────────── Control Plane (nova-cloud-controller) ─────────────┐
│                                                                   │
│  nova-api         ← 接使用者 API call                              │
│  nova-scheduler   ← 決定 VM 開在哪台 compute（查 placement）       │
│  nova-conductor   ← 單點 DB access proxy（compute 不直接碰 DB）   │
│  nova-novncproxy  ← 把 VNC 轉成 websocket 給 horizon              │
│                                                                   │
└────────────────────────────┬──────────────────────────────────────┘
                             │ AMQP (rabbitmq)
┌────────────────────────────▼──────────────────────────────────────┐
│  nova-compute (每台 compute node 1 隻)                             │
│     └─ libvirt + KVM → 實際跑 VM                                  │
└───────────────────────────────────────────────────────────────────┘
```

---

## Step 1: Deploy + 6 條 relation

```bash
juju deploy nova-cloud-controller --channel=2024.1/stable

juju integrate nova-cloud-controller:shared-db         mysql-innodb-cluster:shared-db
juju integrate nova-cloud-controller:amqp              rabbitmq-server:amqp
juju integrate nova-cloud-controller:certificates      self-signed-certificates:certificates
juju integrate nova-cloud-controller:identity-service  keystone:identity-service
juju integrate nova-cloud-controller:image-service     glance:image-service
juju integrate nova-cloud-controller:placement         placement:placement
```

### 6 條 relation 的意義

| Relation | 做什麼 |
|---|---|
| `shared-db` | nova 自己的 DB（存 instance records, flavors, nova_cell0 等 schema） |
| `amqp` | scheduler ↔ compute 的訊息通道 |
| `certificates` | TLS |
| `identity-service` | 註冊進 keystone + 驗 user token |
| `image-service` | 問 glance 拿 image URL |
| `placement` | scheduler 查容量 → **同時解除 placement 的 `blocked` 狀態** |

### 還沒接、等之後的

- `cloud-compute` ← 等 nova-compute 部署
- `neutron-api` ← 等 neutron-api 部署
- `memcache` ← 可選（token cache 加速）
- `cinder-volume-service` ← 可選（掛 volume）
- `dashboard` ← 等 openstack-dashboard 部署

## Step 2: 等 active

```bash
juju wait-for application nova-cloud-controller --timeout=20m
```

**⏱ 10-15 分鐘**。容易卡的點：
- 需要等 placement、glance 把 service account 建好並傳給 nova
- nova-manage cell_v2 setup 步驟會耗時（建 cell0 database）

## Step 3: 驗證

```bash
source ~/openrc

# catalog 多了 2 筆 nova 相關服務
openstack service list
# 應該看到:  keystone / glance / placement / nova / nova_legacy

# nova endpoint
openstack endpoint list --service nova

# Nova daemon 狀態（scheduler / conductor 應該 up）
openstack compute service list
```

### 預期輸出

```
openstack compute service list:
| Binary         | Host              | Zone     | Status  | State |
| nova-scheduler | juju-...-10       | internal | enabled | up    |
| nova-conductor | juju-...-10       | internal | enabled | up    |
# 還沒 nova-compute daemon
```

### placement 應該解除 blocked

```bash
juju status placement
# Message 應該變 "Application Ready" 或類似，脫離 "'placement' missing"
```

---

## Gotchas

（待補 — 等部署完成觀察）

## Replay

```bash
juju deploy nova-cloud-controller --channel=2024.1/stable
juju integrate nova-cloud-controller:shared-db         mysql-innodb-cluster:shared-db
juju integrate nova-cloud-controller:amqp              rabbitmq-server:amqp
juju integrate nova-cloud-controller:certificates      self-signed-certificates:certificates
juju integrate nova-cloud-controller:identity-service  keystone:identity-service
juju integrate nova-cloud-controller:image-service     glance:image-service
juju integrate nova-cloud-controller:placement         placement:placement
juju wait-for application nova-cloud-controller --timeout=20m

source ~/openrc
openstack service list
openstack compute service list
```
