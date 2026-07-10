# Day 3: RabbitMQ 部署

![OpenStack Charms 官方吉祥物](../assets/mascots/openstack-charms.png){ align=right width="90" }

## 目的

部署 `rabbitmq-server`（AMQP broker），讓後面的 OpenStack 服務可以透過 `amqp` relation 拿 broker credentials。並整合 TLS（`certificates` relation）。

## 對應官方指南

[Install OpenStack § RabbitMQ](https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/install-openstack.html) — 原生步驟之一

## 為什麼需要 RabbitMQ

幾乎每個 OpenStack service（keystone 是例外之一）都透過 RabbitMQ 做非同步訊息。例如：
- `nova-cloud-controller` 收到「建 VM」API call → 發訊息到 rabbitmq → 對應 compute node 的 `nova-compute` 拿到訊息 → 實際開 VM
- `neutron-api` 收到「建 port」 → 發訊息 → `neutron-agent` 處理

沒有 rabbitmq，OpenStack control plane 無法驅動 compute/network worker。

---

## Step 1: 確認 charm 版本

```bash
juju info rabbitmq-server | head -40
```

**可用 channel（2026-04）**：
- `3.9/stable` rev 246 — ✅ amd64 + ubuntu@22.04（我們用這個）
- `3.12/stable` — s390x + 24.04 only
- `latest/stable` — 空的

決策：`3.9/stable`。

**Relations**：
- provides: `amqp: rabbitmq` ← 之後 OpenStack 服務跟這接
- requires: `certificates: tls-certificates` ← 接 self-signed-certificates（或 vault）
- requires: `ha: hacluster` — optional，我們不做 HA

## Step 2: 部署

```bash
juju deploy rabbitmq-server --channel=3.9/stable
```

**預期輸出**：
```
Deployed "rabbitmq-server" from charm-hub charm "rabbitmq-server",
revision 246 in channel 3.9/stable on ubuntu@22.04/stable
```

## Step 3: 等 active

```bash
juju wait-for application rabbitmq-server --timeout=10m
```

**經歷階段**：
- `waiting` — LXD container 還沒起來
- `maintenance` — 安裝 erlang + rabbitmq-server package
- `active` with message "Unit is ready"

**⏱ 約 3-4 分鐘**。

**驗證**：
```bash
juju status
```

預期：
```
rabbitmq-server  3.9.27  active  1  rabbitmq-server  3.9/stable  246  no  Unit is ready
rabbitmq-server/0*  active  idle  2  10.254.154.29  5672,15672/tcp  Unit is ready
```

- port **5672** = AMQP（服務走這個）
- port **15672** = Management UI（HTTP）

## Step 4: 整合 TLS

**做什麼**：把 `rabbitmq-server:certificates` 接到 `self-signed-certificates:certificates`。rabbitmq 收到 cert 後會把 AMQP port 升級成 TLS。

```bash
juju integrate rabbitmq-server:certificates self-signed-certificates:certificates
```

**⏱ 約 30-60 秒**（跟 mysql 的 cert 交換過程一樣）。

```bash
juju wait-for unit rabbitmq-server/0 --query='agent-status=="idle"' --timeout=5m
```

## Step 5: 驗證

```bash
juju status --relations
```

Integrations 區塊應該新增一行：
```
self-signed-certificates:certificates  rabbitmq-server:certificates  tls-certificates  regular
```

---

## 此時的 Model 狀態

**3 個 LXD container（不含 controller）+ 3 個 regular integration**：

```
Apps:
  mysql (8.0/stable)                    — machine 0
  self-signed-certificates (1/stable)   — machine 1
  rabbitmq-server (3.9/stable)          — machine 2

Integrations:
  mysql:certificates             ← self-signed-certificates
  rabbitmq-server:certificates   ← self-signed-certificates
```

---

## Gotchas

### ⚠️ `waiting` 階段很久

rabbitmq-server 進入 active 前會在 `waiting` / `maintenance` 卡 2-3 分鐘安裝 erlang。正常現象，不用急。

### ⚠️ rabbitmq-server:cluster peer relation 顯示 "joining"

部署單 unit 時會看到：
```
rabbitmq-server:cluster  rabbitmq-server:cluster  rabbitmq-ha  peer  joining
```

「joining」表示 cluster peer relation 等待其他 peer（但沒有）。**不影響運作**。Scale 到 3 unit 時才會變成實際 cluster。

---

## Replay 快速指令

```bash
juju deploy rabbitmq-server --channel=3.9/stable
juju wait-for application rabbitmq-server --timeout=10m
juju integrate rabbitmq-server:certificates self-signed-certificates:certificates
juju wait-for unit rabbitmq-server/0 --query='agent-status=="idle"' --timeout=5m
juju status --relations
```
