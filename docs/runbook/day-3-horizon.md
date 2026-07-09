# Day 3: Horizon (openstack-dashboard) 部署

## 目的

補完 Install OpenStack 章第 10 個應用：**OpenStack 的 Web UI**。

## 對應官方指南

[Install OpenStack § openstack-dashboard](https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/install-openstack.html) — 第 10 個部署應用。

## Horizon 是什麼

- 純 Django web app，打在 Apache 後面
- 背後 call Keystone / Nova / Neutron / Glance REST API — **它自己沒資料**
- AWS 對照：Management Console
- CLI 的 `openstack` 指令能做的，Horizon 都能做（但反之未必，有些細節只 CLI 看得到）

## Prerequisites

- mysql-innodb-cluster active
- keystone active
- vault active（或 self-signed-certificates 也可）

## Charm Relations

| Relation | 對端 | 作用 |
|---|---|---|
| `shared-db` | `mysql-innodb-cluster` | Django session、偏好設定存這 |
| `identity-service` | `keystone` | 登入驗證（SSO token） |
| `certificates` | `vault` | HTTPS 用（備著，不開 `enforce-ssl` 走 HTTP） |

## 部署指令

```bash
juju deploy openstack-dashboard --channel=2024.1/stable
# Revision 728 實測

juju integrate openstack-dashboard:shared-db       mysql-innodb-cluster:shared-db
juju integrate openstack-dashboard:identity-service keystone:identity-service
juju integrate openstack-dashboard:certificates    vault:certificates

juju wait-for application openstack-dashboard --timeout=15m
```

約 4-5 分鐘 active，ports `80,443/tcp` 開放。

## 驗證

```bash
# LXD 內 IP 查法
juju status openstack-dashboard --format=yaml | grep public-address

# 本次 IP = 10.254.154.136
curl -sSLI http://10.254.154.136/horizon
# HTTP/1.1 302 Found
# Location: http://10.254.154.136/horizon/auth/login/?next=/horizon/
# Server: Apache/2.4.52 (Ubuntu)
```

Body 可看到 `<title>Login - OpenStack Dashboard</title>`。

## 從本機開瀏覽器

LXD 的 `10.254.x.x` 不對外，要 SSH tunnel：

```bash
# Mac 上
ssh -i ~/.ssh/juju_id_rsa -L 8080:10.254.154.136:80 azureuser@20.210.233.133

# 瀏覽器：
http://localhost:8080/horizon
```

## 登入

| 欄位 | 值 |
|---|---|
| Domain | `admin_domain` |
| User | `admin` |
| Password | 從 `~/openrc` 的 `OS_PASSWORD` |

本次：`eiSeeV1oich0yie0`（keystone charm 自產）。

## HTTPS 未啟用（刻意）

- `enforce-ssl` config 預設 `false`，charm 只開 Apache HTTP
- 443 port 在 `juju status` 顯示是 charm 宣告，不是 Apache 真的 bind TLS
- Vault 的 cert 已經拿到 `/etc/apache2/ssl/` 底下，要啟用：
  ```bash
  juju config openstack-dashboard enforce-ssl=True
  juju config openstack-dashboard secret="$(openssl rand -hex 16)"
  ```
  但實驗室沒有 FQDN，curl 會 name mismatch → 保持 HTTP 比較單純。

## 之後能用 Horizon 做

官方 Configure OpenStack 章 step 5 "Dashboard access" 會用它：
- 看 instance list / console（VNC）
- 看 network topology 圖
- 看 hypervisor 使用率

但第一次 create network / instance 照例用 CLI（指令比點擊精確）。

## Replay

```bash
juju deploy openstack-dashboard --channel=2024.1/stable
juju integrate openstack-dashboard:shared-db mysql-innodb-cluster:shared-db
juju integrate openstack-dashboard:identity-service keystone:identity-service
juju integrate openstack-dashboard:certificates vault:certificates
juju wait-for application openstack-dashboard --timeout=15m

# tunnel + 瀏覽器
ssh -L 8080:$(juju exec --unit openstack-dashboard/0 'unit-get private-address'):80 azureuser@<vm-ip>
```
