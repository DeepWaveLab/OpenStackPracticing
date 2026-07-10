---
title: "neutron-api charm 預設關掉 core security-group extension 導致 404"
category: integration-issues
tags: [neutron, security-group, charm-config, ovn, openstack]
component: neutron-api (2024.1/stable)
symptom: "`openstack security group create` 回 404 / `security-group` extension 不在 API extension list"
root_cause: "neutron-api charm config `neutron-security-groups` 預設 false → ml2_conf.ini 寫 `enable_security_group = False` → neutron server 根本沒掛 core extension"
severity: high
date: 2026-04-19
---

# Neutron Security Groups 預設關掉（charm quirk）

## 症狀

```bash
$ openstack security group create lab_sg
ResourceNotFound: 404: Client Error for url:
http://10.254.154.116:9696/v2.0/security-groups?...
The resource could not be found.
```

```bash
$ openstack extension list --network -f value -c Alias | grep security
security-groups-default-rules
# ↑ 只有 default-rules extension，沒有 core 的 security-group！
```

## 根因

charm 的 config option `neutron-security-groups` **預設 false**，導致：

```ini
# /etc/neutron/plugins/ml2/ml2_conf.ini
enable_security_group = False
```

Neutron server 啟動時跳過 core security-group extension。API 路由 `/v2.0/security-groups` 根本沒註冊 → 404。

## 解法

```bash
juju config neutron-api neutron-security-groups=True
# 等 ~60s 讓 config-changed hook 寫檔 + 重啟 neutron
```

驗證：
```bash
source ~/openrc
openstack extension list --network -f value -c Alias | grep security
# 應該多出 "security-group"
openstack security group create test_sg && openstack security group delete test_sg
```

## 為什麼 charm 預設關

歷史包袱。OpenStack Charms 原本配 `neutron-openvswitch` + 傳統 iptables firewall，部分使用者不要 firewall（用外部 fw），就預設關。OVN 時代 security group 透過 OVN ACL 實作，其實**應該**預設開才對，但 charm 沒跟著改。

## 預防

每次部署 OpenStack，跟著 mysql/keystone 一起把這條加進去：

```bash
juju deploy neutron-api --channel=2024.1/stable --config neutron-security-groups=True
# 或之後補
juju config neutron-api neutron-security-groups=True
```

## 相關文件

- 本專案 `docs/runbook/day-4-configure-openstack.md`
- Upstream neutron-api charm: https://opendev.org/openstack/charm-neutron-api
