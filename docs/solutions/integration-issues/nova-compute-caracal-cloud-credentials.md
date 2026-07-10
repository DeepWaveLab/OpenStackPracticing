---
title: "nova-compute 2024.1/stable 靜默 fail：缺 `cloud-credentials` relation 導致無法註冊 placement"
category: integration-issues
tags: [juju, openstack, nova-compute, placement, keystone, caracal, cloud-credentials]
component: nova-compute (2024.1/stable), placement
symptom: "juju status nova-compute 顯示 active/idle 'Unit is ready'，但 `openstack compute service list` 看不到它，`openstack hypervisor list` 空"
root_cause: "Caracal (2024.1) 起 placement 獨立於 nova，nova-compute 要直接從 keystone 拿 service account 才能認 placement；舊 zed 以前是從 nova-cc 間接繼承"
severity: critical
date: 2026-04-19
---

# nova-compute Caracal 必須接 `cloud-credentials` 才能工作

## 症狀

部署完成後 **`juju status` 一切正常**：
```
nova-compute  29.2.0  active  1  ... Unit is ready
nova-compute/0*  active  idle  ...  Unit is ready
```

**但 OpenStack CLI 看不到它**：
```bash
$ openstack compute service list
# 只有 nova-scheduler + nova-conductor，沒有 nova-compute
$ openstack hypervisor list
# (empty)
$ openstack resource provider list
# (empty)
```

nova-cc 還會卡 `"Missing relations: compute"`，即使 `cloud-compute` relation 已接。

## 根因

在 `/var/log/nova/nova-compute.log` 可以看到 **nova-compute daemon 實際一直 crash 重啟**（systemd restart loop）：

```
CRITICAL nova [...] Unhandled error:
  keystoneauth1.exceptions.auth_plugins.MissingAuthPlugin:
  An auth plugin is required to determine endpoint URL
```

發生在 `SchedulerReportClient.__init__`，試圖跟 placement API 講話時。

### 為什麼 Caracal 變嚴格

- **zed 及以前**：nova-compute 從 nova-cc 間接取得 placement 連線資訊（`cloud-compute` relation 內夾帶）
- **Caracal (2024.1) 起**：placement 完全獨立，nova-compute 要**直接拿 keystone service account** 才能自己去註冊 resource provider

charm 沒有在缺 `cloud-credentials` 時 self-block，只是靜默 crash + hook 認為一切 OK。

## 解法

加上 `cloud-credentials` relation：

```bash
juju integrate nova-compute:cloud-credentials keystone:identity-credentials
```

30-60 秒後 nova-compute charm 自動：
1. 從 keystone 拿 service account (project: services, user: nova)
2. 寫進 `/etc/nova/nova.conf` 的 `[placement]` section
3. 重啟 `nova-compute.service`

驗證：
```bash
source ~/openrc
openstack compute service list
# 應該看到 nova-compute <hostname>.lxd enabled up

openstack hypervisor list
# 1 個 QEMU hypervisor

openstack resource provider list
# 1 個 resource provider（對應 compute node）
```

## 完整部署流程(Caracal)

```bash
juju deploy nova-compute --channel=2024.1/stable \
  --constraints="virt-type=virtual-machine mem=8G cores=4 root-disk=50G"

# ⚠️ 4 條 relation，別漏任何一條
juju integrate nova-compute:cloud-compute       nova-cloud-controller:cloud-compute
juju integrate nova-compute:amqp                rabbitmq-server:amqp
juju integrate nova-compute:image-service       glance:image-service
juju integrate nova-compute:cloud-credentials   keystone:identity-credentials  # ← 關鍵

juju wait-for application nova-compute --timeout=25m
```

## 快速診斷

若看到 charm 說 active 但 CLI 找不到 compute：

```bash
juju ssh nova-compute/0 -- sudo tail -30 /var/log/nova/nova-compute.log | grep -iE "error|auth"
```
若看到 `MissingAuthPlugin` → 缺 `cloud-credentials`。

## 預防

部署 Nova 前檢查 charm 的 `requires` relations：
```bash
juju info nova-compute | grep -A 15 relations
# Caracal+ 的 charm 要 cloud-credentials: keystone-credentials
```

自動化腳本一律接足 4 條，避開驚喜。

## 相關文件

- 本專案 `docs/runbook/day-3-nova-compute.md`
- Caracal release notes: placement service full independence
