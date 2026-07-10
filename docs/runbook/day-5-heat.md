# Day 5: Heat (Orchestration) 部署

![Heat 官方吉祥物](../assets/mascots/heat.png){ align=right width="110" }

## 目的

部署 Heat，讓你能用 YAML 模板（HOT）一次建一整批 OpenStack 資源。
**Heat 是 Magnum 的底層依賴** — Magnum 把「建 K8s cluster」請求轉成 Heat stack 模板，實際動作都是 Heat 做。

## 對應官方指南

Heat 不在官方 Install OpenStack 的必裝清單（歸為 optional service）。  
Upstream doc: https://docs.openstack.org/heat/latest/

## 3 個核心概念

| 概念 | 類比 |
|---|---|
| Template（HOT YAML） | Dockerfile / Terraform `.tf` |
| Stack（實例） | container / Terraform state |
| Resource | 單個 OpenStack 物件（server、network、router…） |

## 部署

```bash
juju deploy heat --channel=2024.1/stable
juju integrate heat:shared-db       mysql-innodb-cluster:shared-db
juju integrate heat:identity-service keystone:identity-service
juju integrate heat:amqp            rabbitmq-server:amqp
juju integrate heat:certificates    self-signed-certificates:certificates

juju wait-for application heat --timeout=15m
```

約 5 分鐘 active，port 8000（heat-api-cfn）+ 8004（heat-api）。

## ⚠️ 必跑 charm action：domain-setup

charm 預設**不自動建** Heat 要用的 keystone domain + user，直接建 stack 會：

```
Stack CREATE FAILED: Authorization failed.
# heat-engine.log: Domain admin client authentication failed: Unauthorized
```

跑 action：

```bash
juju run heat/leader domain-setup
```

會建：
- Domain `heat`（給 stack 使用者獨立放）
- User `heat_domain_admin`（Heat 代操作的「domain 超級使用者」）
- Role `heat_stack_user`

action output 會有幾行誤報 "No domain ... exists"（charm 在建完後檢查時看到剛建的東西）— 無視。
實際可用 `openstack domain list` 驗證。

## 驗證

### 1. Catalog / endpoint
```bash
source ~/openrc
openstack service list | grep -E 'heat|orchestration'
# 應該看到：
# heat      orchestration
# heat-cfn  cloudformation
```

### 2. 跑一個最小 stack
```bash
cat > ~/demo-stack.yaml <<'EOF'
heat_template_version: 2018-08-31
description: Demo
resources:
  demo_net:
    type: OS::Neutron::Net
    properties: { name: heat_demo_net }
  demo_subnet:
    type: OS::Neutron::Subnet
    properties:
      name: heat_demo_subnet
      network: { get_resource: demo_net }
      cidr: 10.99.99.0/24
      dns_nameservers: [8.8.8.8]
outputs:
  subnet_cidr:
    value: { get_attr: [demo_subnet, cidr] }
EOF

openstack stack create demo-stack -t ~/demo-stack.yaml --wait
# 應該看到 CREATE_COMPLETE

openstack stack resource list demo-stack
# 列出 demo_net + demo_subnet，狀態 CREATE_COMPLETE

openstack stack output show demo-stack subnet_cidr
# 10.99.99.0/24

openstack network show heat_demo_net   # Neutron 真的有這個
```

清掉：
```bash
openstack stack delete demo-stack --yes --wait
```

## ⚠️ 地雷：snap openstackclient 看不到 /tmp

本 lab 的 `openstackclients` snap 是 strict confinement，`/tmp` 看得見但在私有 namespace — `cat > /tmp/x.yaml` 寫到 host `/tmp`，snap 讀不到。

症狀：
```
$ openstack stack create demo -t /tmp/demo.yaml
<urlopen error [Errno 2] No such file or directory: '/tmp/demo.yaml'>
```

✅ **正解**：template 放 `~/`，snap 看得到。

## Replay（整段）

```bash
juju deploy heat --channel=2024.1/stable
juju integrate heat:shared-db       mysql-innodb-cluster:shared-db
juju integrate heat:identity-service keystone:identity-service
juju integrate heat:amqp            rabbitmq-server:amqp
juju integrate heat:certificates    self-signed-certificates:certificates
juju wait-for application heat --timeout=15m

juju run heat/leader domain-setup    # ← 必跑

source ~/openrc
openstack stack list   # 應該是空（OK）
```

## Magnum 進度

Heat ✅ → 下一步：Magnum（可選：Octavia，給 K8s LoadBalancer Service 用）
