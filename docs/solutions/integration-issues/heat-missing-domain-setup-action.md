---
title: "Heat charm 預設不建 keystone domain → Stack CREATE FAILED: Authorization failed"
category: integration-issues
tags: [heat, keystone, trust, stack-domain, charm-action]
component: heat (2024.1/stable)
symptom: "Stack create 立刻 CREATE_FAILED '『Authorization failed』；heat-engine.log: `Domain admin client authentication failed: Unauthorized`"
root_cause: "heat 的 heat.conf 寫了 stack_domain_admin/stack_user_domain 的名字跟密碼，但 charm 不自動在 keystone 建 domain + user，要手動跑 action"
severity: high
date: 2026-04-19
---

# Heat charm 忘了告訴你要跑 `domain-setup` action

## 症狀

```bash
$ openstack stack create demo -t demo.yaml --wait
 Stack demo CREATE_FAILED
2026-04-19 ... [demo]: CREATE_FAILED  Authorization failed.
```

heat-engine log (`juju ssh heat/0 -- sudo tail /var/log/heat/heat-engine.log`)：
```
ERROR heat.engine.clients.os.keystone.heat_keystoneclient
  Domain admin client authentication failed:
  keystoneauth1.exceptions.http.Unauthorized:
  The request you have made requires authentication. (HTTP 401)
```

## 根因

Heat 用 Keystone **trust** 機制讓 stack owner 授權給 Heat 代做事。需要一個專用的：
- Domain `heat`（stack user / trust 放在這個獨立 domain）
- User `heat_domain_admin`（domain 裡的超級使用者）
- Role `heat_stack_user`

charm 寫了 config：
```ini
# /etc/heat/heat.conf
deferred_auth_method = trusts
stack_domain_admin = heat_domain_admin
stack_domain_admin_password = <隨機 32 字元>
stack_user_domain_name = heat
```

**但從不主動在 keystone 建這 3 樣**。每次建 stack 時 Heat 嘗試用 `heat_domain_admin` 認證失敗 → 401 → Stack create fail。

驗證：
```bash
$ openstack domain list
+--------------------+----------------+
| Name               | ...            |
+--------------------+----------------+
| admin_domain       |                |
| service_domain     |                |
| Default            |                |
+--------------------+----------------+
# 沒有 heat domain

$ openstack user list --domain heat 2>&1
No domain with a name or ID of 'heat' exists.
```

## 解法

跑 charm action：

```bash
juju run heat/leader domain-setup
```

output 會多行「No domain / user / role ... exists」，那是 action 跑完後做 post-check 時看到 **剛建好的** 東西（keystone cache/時序問題）— **誤報可忽略**。
實際驗證：

```bash
source ~/openrc
openstack domain list                           # 應有 "heat"
openstack user list --domain heat              # 應有 "heat_domain_admin"
openstack role list | grep heat_stack_user     # 應有
```

重試 stack create：
```bash
openstack stack delete demo --yes --wait 2>/dev/null
openstack stack create demo -t demo.yaml --wait
# CREATE_COMPLETE
```

## 為什麼 charm 沒自動做

這個 action 需要 `openstackclients` + admin openrc，charm 預設沒掛載這些東西進 unit（避免循環依賴：heat charm 跑之前 keystone 未必 active）。讓人手動跑是「deploy 最後一步」的設計。

類似模式的 charm action：
- `vault` 的 `authorize-charm`
- `ceph-osd` 的 `add-disk`
- `keystone` 的 `bootstrap` (舊版)

都屬於「charm 佈署 → 最後要手動觸發的初始化 action」。

## 預防

部署 script 最後一步固定跑：

```bash
juju wait-for application heat --timeout=15m
juju run heat/leader domain-setup
```

## 相關文件

- 本專案 `docs/runbook/day-5-heat.md`
- Heat deferred auth docs: https://docs.openstack.org/heat/latest/admin/auth-model.html
