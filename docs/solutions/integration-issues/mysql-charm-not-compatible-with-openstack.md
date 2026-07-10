---
title: "Canonical `mysql` charm 宣稱支援 shared-db 但實際不能跟 OpenStack 服務 integrate"
category: integration-issues
tags: [juju, openstack, mysql, mysql-innodb-cluster, shared-db, keystone]
component: mysql (Canonical 8.0/stable), keystone, glance, nova-cloud-controller, neutron-api, cinder
symptom: "keystone hook failed: 'shared-db-relation-changed' / 'Allowed_units list provided but this unit not present'"
root_cause: "Canonical 的 `mysql` charm 宣稱 provides `shared-db: mysql-shared` 但沒實作該 interface 的 relation hook；OpenStack 服務要求 `mysql-innodb-cluster` charm"
severity: critical
date: 2026-04-19
---

# `mysql` charm 不是 OpenStack 的 `mysql`

## 症狀

```bash
juju deploy mysql
juju deploy keystone --channel=2024.1/stable
juju integrate keystone:shared-db mysql:shared-db
```

約 60 秒後：
```
App         Status   Message
keystone    waiting  Allowed_units list provided but this unit not present

Unit         Workload  Message
keystone/0*  error     hook failed: "shared-db-relation-changed" for mysql:shared-db
```

## 根因

兩個不同 publisher 的 charm 都宣稱提供 `mysql-shared` interface：

| Charm | Publisher | 實際支援 interface | 用於 |
|---|---|---|---|
| `mysql` (8.0/stable rev 444) | **Canonical** | `database: mysql_client` (NEW) | 現代 K8s app / 一般資料庫需求 |
| `mysql-innodb-cluster` (8.0/stable) | **OpenStack Charmers** | `shared-db: mysql-shared` (LEGACY) | OpenStack 服務 |

`juju info mysql` 列出 `shared-db: mysql-shared` 是**誤導**— charm 的 metadata.yaml 保留了舊欄位但 reactive handler 不完整。實際整合後 application-data 永遠是空 `{}`，消費端（keystone 等）拿不到 password/db_host/allowed_units。

驗證方法：
```bash
juju show-unit mysql/0 --format=yaml | grep -A 10 "endpoint: shared-db"
# 會看到 application-data: {} 而不是包含 credentials
```

## 解法

用 `mysql-innodb-cluster`（OpenStack Charmers 出品）：

```bash
# 清掉舊 mysql（如果已部署）
juju remove-application <failing-openstack-charm> --force --no-prompt
juju remove-application mysql --destroy-storage --force --no-prompt
# 等 3-5 分鐘完全清除

# 正確部署（需要 3 unit 才能 form cluster）
juju deploy mysql-innodb-cluster -n 3 --channel=8.0/stable
juju integrate mysql-innodb-cluster:certificates self-signed-certificates:certificates
juju wait-for application mysql-innodb-cluster --timeout=20m
# 應該看到 "Cluster is ONLINE and can tolerate up to ONE failure"

# 再部署 OpenStack 服務
juju deploy keystone --channel=2024.1/stable
juju integrate keystone:shared-db mysql-innodb-cluster:shared-db
```

## 關鍵訊號

**`mysql-innodb-cluster` 特徵（正確 charm）**：
- Publisher: OpenStack Charmers
- 標籤包含 `openstack`
- `juju info mysql-innodb-cluster` 顯示 relation：
  ```
  provides:
    shared-db: mysql-shared   ← 真的 work
    db-router: mysql-router
  requires:
    certificates: tls-certificates
  ```
- 需要 `-n 3`（InnoDB Cluster quorum）

**`mysql` charm 特徵（一般 mysql，不適合 OpenStack）**：
- Publisher: Canonical
- 標籤：`cloud, databases`（注意**沒有** `openstack`）
- `juju info mysql` 雖然列出 `shared-db` 但實際不 work
- 單 unit 也能跑

## 預防

部署 OpenStack 前永遠**確認 publisher**：

```bash
juju info <charm-name> | head -5
# 看 publisher: OpenStack Charmers → 正確 OpenStack charm
# 看 publisher: Canonical（其他） → 可能不是
```

## Replay 順序

正確的 Day 3 底層 order（用 `mysql-innodb-cluster`）：

```bash
juju deploy mysql-innodb-cluster -n 3 --channel=8.0/stable
juju deploy rabbitmq-server --channel=3.9/stable
juju deploy self-signed-certificates --channel=1/stable
# 等全 active 後再部署 OpenStack 核心服務
```

## 相關文件

- 本專案 `docs/runbook/day-3-mysql-innodb-cluster.md`
- OpenStack Charms Deployment Guide: https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/install-openstack.html
