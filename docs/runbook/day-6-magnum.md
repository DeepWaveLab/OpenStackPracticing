# Day 6: Magnum (Container Infrastructure) 部署

![Magnum 官方吉祥物](../assets/mascots/magnum.png){ align=right width="110" }

## 目的

部署 Magnum — K8s/Swarm cluster as a service。本 lab 最終目標：從 OpenStack 一條指令得到一個 K8s cluster。

今晚先讓 Magnum API 上線，下次建 cluster template + 第一個 K8s cluster。

## Magnum 是什麼

**翻譯層**，不自己跑容器。收到「給我 3-node K8s cluster」請求 → 查你定義的 cluster template → 轉成 Heat template → 丟 Heat 去建 VM / network / router。

```
user ──▶ Magnum API ──▶ Heat ──▶ Nova / Neutron / Cinder
         │
         ├─ cluster-template（規格）
         └─ cluster（實例）
```

Magnum service type = `container-infra`。CLI `openstack coe ...`（COE = Container Orchestration Engine）。

## Prerequisites

- **Heat** active 且 domain-setup 已跑 — Magnum 所有動作都經 Heat
- keystone / mysql / rabbit / SSC 都 active

## 部署

```bash
juju deploy magnum --channel=2024.1/stable
juju integrate magnum:shared-db       mysql-innodb-cluster:shared-db
juju integrate magnum:identity-service keystone:identity-service
juju integrate magnum:amqp            rabbitmq-server:amqp
juju integrate magnum:certificates    self-signed-certificates:certificates

juju wait-for application magnum --timeout=15m
```

Charm 不直接掛 Heat — Magnum 透過 **keystone catalog** discover Heat endpoint。Heat 要先 active 不然後續 cluster 建會失敗。

## ⚠️ 必跑：`domain-setup` action

跟 Heat 一樣。建 `magnum` trustee domain + `magnum_domain_admin` user。沒跑 → cluster create 會 401。

```bash
juju run magnum/leader domain-setup
```

action output 一樣有誤報「No domain ... exists」，忽略。實際驗證：
```bash
source ~/openrc
openstack domain list | grep magnum         # magnum | Magnum trustee domain
openstack user list --domain magnum         # magnum_domain_admin
```

## ⚠️ 必補：magnum-api bind 0.0.0.0（charm bug）

charm 預設 magnum-api `[api]` 不寫 `host`，Magnum default bind `127.0.0.1`。
但 charm 同時配了 haproxy backend 指到 container IP `10.254.154.X:9501` — **兩邊不對盤**。

症狀：
```bash
$ openstack coe cluster template list
Unable to establish connection to http://<ip>:9511/v1/clustertemplates:
  ('Connection aborted.', RemoteDisconnected('Remote end closed connection without response'))
```

✅ Workaround：
```bash
cat <<'EOF' | juju ssh magnum/0 -- sudo bash
sed -i '/^\[api\]/a host = 0.0.0.0' /etc/magnum/magnum.conf
systemctl restart magnum-api
EOF
```

會被 charm config-changed hook 覆寫 — 每次配置變動後重跑。或設 systemd override。

## 驗證

```bash
source ~/openrc
openstack service list | grep magnum
# magnum | container-infra

openstack endpoint list --service magnum
# public/internal/admin 3 個都指 http://<magnum-ip>:9511/v1

openstack coe cluster template list -f json
# [] (空 list，沒 error 就對)

openstack coe cluster list -f json
# []

curl -sS http://<magnum-ip>:9511/
# {"name": "OpenStack Magnum API", "versions": [{"id": "v1", "status": "CURRENT", ...}]}
```

## 尚未完成（下一段）

要真的開一個 K8s cluster：

1. **下載 Fedora CoreOS image**（Magnum 2024.1 預設 K8s driver 用）
   - 約 900 MB，下載 10 min
2. **Upload 到 Glance**，設 `os_distro=fedora-coreos` property（Magnum driver 靠這個 property 選 driver）
3. **建 cluster template**
   ```bash
   openstack coe cluster template create k8s-small \
     --image fedora-coreos-X \
     --external-network ext_net \
     --flavor ubuntu.small \
     --master-flavor ubuntu.small \
     --coe kubernetes \
     --docker-volume-size 5 \
     --network-driver flannel
   ```
4. **建 cluster（真的開 K8s）**
   ```bash
   openstack coe cluster create my-k8s --cluster-template k8s-small --node-count 1
   ```
5. 等 15-30 分鐘，等 FCOS 開機 + cloud-init 裝 K8s node + master

可選 Octavia 讓 K8s Service `type=LoadBalancer` 能用；lab 不裝的話只能 `NodePort`/`ClusterIP`。

## Replay（整段）

```bash
juju deploy magnum --channel=2024.1/stable
juju integrate magnum:shared-db       mysql-innodb-cluster:shared-db
juju integrate magnum:identity-service keystone:identity-service
juju integrate magnum:amqp            rabbitmq-server:amqp
juju integrate magnum:certificates    self-signed-certificates:certificates
juju wait-for application magnum --timeout=15m

juju run magnum/leader domain-setup

# bind fix
juju ssh magnum/0 -- "sudo sed -i '/^\[api\]/a host = 0.0.0.0' /etc/magnum/magnum.conf && sudo systemctl restart magnum-api"

source ~/openrc
openstack coe cluster template list -f json   # []
```
