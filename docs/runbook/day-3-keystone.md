# Day 3: Keystone 部署

![Keystone 官方吉祥物](../assets/mascots/keystone.png){ align=right width="110" }

## 目的

部署 OpenStack 的**第一個真正服務** — Keystone（Identity Service）。它會整合進我們的 mysql-innodb-cluster + self-signed-certificates，之後所有其他 OpenStack 服務都要跟 keystone integrate 註冊自己。

## 對應官方指南

[Install OpenStack § Keystone](https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/install-openstack.html)

原指令（2023.2 Bobcat）：
```bash
juju deploy --channel 2023.2/stable keystone
```

我們用 **2024.1 Caracal**：
```bash
juju deploy keystone --channel=2024.1/stable
```

---

## Prerequisites

- mysql-innodb-cluster 3 unit active（Day 3-mysql-innodb-cluster 完成）
- self-signed-certificates active
- rabbitmq-server 雖然在但 keystone 不需要它

---

## Step 1: Deploy + 兩條 relation

```bash
juju deploy keystone --channel=2024.1/stable
juju integrate keystone:shared-db mysql-innodb-cluster:shared-db
juju integrate keystone:certificates self-signed-certificates:certificates
```

**⏱ 5-10 分鐘**：LXD 開 container → apt install keystone+apache → config → 等 db credentials → bootstrap

### Keystone charm 的 relation 需求

| Interface | Endpoint | 對到哪個 charm |
|---|---|---|
| `mysql-shared` | `shared-db` (requires) | mysql-innodb-cluster:shared-db |
| `tls-certificates` | `certificates` (requires) | self-signed-certificates:certificates |
| `keystone` | `identity-service` (provides) | （之後 glance/nova/neutron/cinder/horizon 都接這個） |
| `keystone-credentials` | `identity-credentials` (provides) | 內部服務帳號給 ceph-radosgw 等 |

## Step 2: 等 active

```bash
juju wait-for application keystone --timeout=15m
juju status --relations
```

**預期最終狀態**：
```
keystone  25.0.0  active  1  keystone  2024.1/stable  778  no  Application Ready

keystone/0*  active  idle  7  <ip>  5000/tcp  Unit is ready
```

- port 5000 = Keystone API (HTTPS 因為有 TLS)
- port 35357（舊版管理 API）已在 2024.1 合併到 5000

---

## Step 3: 取得 admin password + 建立 openrc

Keystone charm 部署時會自動產生 admin password。

```bash
# 取 admin password（透過 charm action）
juju run keystone/leader get-admin-password
# 輸出: admin-password: eiSeeV1oich0yie0

# Keystone unit 的 IP
juju status keystone --format=json | python3 -c "
import sys, json
d = json.load(sys.stdin)
for name, u in d['applications']['keystone']['units'].items():
    print(u['public-address'])
"
```

**⚠️ 重要：port 5000 實際是 HTTP（charm 預設沒啟 TLS）**

儘管 keystone charm 有接 `certificates` relation，預設 public endpoint 仍然是 HTTP。驗證方式：
```bash
curl -sv http://10.254.154.23:5000/   # 200 OK，回 JSON version info
curl -sv -k https://10.254.154.23:5000/  # SSL WRONG_VERSION_NUMBER
```

背後 port 佔用實況：
```
port 5000   haproxy  ← public endpoint (HTTP)
port 35357  haproxy  ← admin endpoint (HTTP)
port 80     apache2  ← internal
port 4990   apache2  ← wsgi-keystone-public
port 35347  apache2  ← wsgi-keystone-admin
```

所以 openrc 的 OS_AUTH_URL 用 **http://**：

```bash
cat > ~/openrc <<EOF
export OS_AUTH_URL=http://10.254.154.23:5000/v3
export OS_USERNAME=admin
export OS_PASSWORD=eiSeeV1oich0yie0
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=admin_domain
export OS_PROJECT_DOMAIN_NAME=admin_domain
export OS_IDENTITY_API_VERSION=3
export OS_REGION_NAME=RegionOne
export OS_INTERFACE=admin
EOF
```

TLS 之後要開的話（還沒試過）：查 `juju config keystone` 裡跟 `ssl` 或 `tls` 有關的 config 欄位，打開 + reload。

## Step 4: 裝 openstack CLI + 驗證

```bash
sudo snap install openstackclients
# yoga/stable 版本 OK，跨版本相容 2024.1 keystone

source ~/openrc
openstack catalog list
openstack endpoint list
openstack service list
openstack domain list
openstack project list
openstack user list
openstack role list
```

### 實測輸出

**catalog list** — 只有 keystone 自己：
```
+----------+----------+-----------------------------------------+
| Name     | Type     | Endpoints                               |
+----------+----------+-----------------------------------------+
| keystone | identity | admin:    http://10.254.154.23:35357/v3 |
|          |          | internal: http://10.254.154.23:5000/v3  |
|          |          | public:   http://10.254.154.23:5000/v3  |
+----------+----------+-----------------------------------------+
```

**domain list** — Juju 自動建 3 個 domain：
```
| ID                               | Name           | Enabled | Description        |
| 8015cd4110...                    | admin_domain   | True    | Created by Juju    |
| a1205ec12b...                    | service_domain | True    | Created by Juju    |
| default                          | Default        | True    | The default domain |
```

**role list** — 標準 OpenStack 5 個 role：
```
Admin / manager / member / reader / service
```

之後部署 glance / placement / nova / neutron / cinder / horizon，每個都會：
1. 自動在 catalog 註冊一筆 service
2. 建立對應的 endpoint URL
3. 建立 service user（在 service_domain 下）

---

## Gotchas

### ⚠️ Self-signed cert + openstack CLI

openstack CLI 預設會驗證 CA。我們用 self-signed-certificates，所以要：
- 方法 A（快）：設 `export OS_INSECURE=true` 跳過驗證
- 方法 B（正式）：從 `self-signed-certificates/0` 撈出 CA cert，放進 `OS_CACERT`

方法 B：
```bash
juju ssh self-signed-certificates/0 'sudo cat /var/lib/juju/tools/ca.crt' > ~/ca.crt
export OS_CACERT=~/ca.crt
```

### ⚠️ Keystone port 5000 不能從 Azure VM 外直連

LXD bridge（10.254.154.x）從 Azure VM 外部連不到。要用 keystone：
- 方法 A：SSH 進 Azure VM，在 VM 內用 openstack CLI
- 方法 B：SSH tunnel `ssh -L 5000:<keystone_lxd_ip>:5000 azureuser@<azure_vm_public_ip>`

---

## Replay

```bash
juju deploy keystone --channel=2024.1/stable
juju integrate keystone:shared-db mysql-innodb-cluster:shared-db
juju integrate keystone:certificates self-signed-certificates:certificates
juju wait-for application keystone --timeout=15m
juju status --relations

# 取 admin password 和建 openrc 見 Step 3
```
