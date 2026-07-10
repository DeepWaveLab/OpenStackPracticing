---
title: "OVN 24.03 controller (Caracal) 不跟 22.03 northd 註冊 → 沒 chassis → PortBindingFailed"
category: integration-issues
tags: [ovn, port-binding, version-mismatch, caracal, chassis]
component: ovn-controller 24.03 (from UCA caracal), ovn-central 22.03/stable charm
symptom: "VM build 進到 port bind 階段噴 `PortBindingFailed`；`ovn-sbctl list chassis` 空"
root_cause: "OVS `external_ids:ovn-match-northd-version=True` + version gap (controller 24.03 vs northd 22.03) → ovn-controller 拒絕註冊 chassis；沒 chassis Neutron 就找不到可以 bind port 的 host"
severity: critical
date: 2026-04-19
---

# OVN Controller 拒絕跟舊 northd 講話

## 症狀

Nova build log：
```
nova.exception.PortBindingFailed: Binding failed for port <uuid>,
please check neutron logs for more information.
```

OVN 視角：
```bash
$ juju ssh ovn-central/0 -- sudo ovn-sbctl list chassis
(empty)
```

ovn-controller 自己（on nova-compute LXD VM）看 log：
```
/var/log/ovn/ovn-controller.log:
WARN|controller version - 24.03.2-20.33.0-72.6 mismatch with
     northd version - 22.03.3-20.21.0-62.4
```

## 根因

兩個因素疊加：

**因素 1：版本差**
- nova-compute 2024.1/stable（Caracal）的 UCA repo 裝 ovn 24.03
- ovn-central charm 還是 22.03/stable，北向（northd）package 停在 22.03.3
- Major 版差 2（22 → 24），OVSDB schema 小有差異

**因素 2：strict version matching 開著**
```bash
$ ovs-vsctl get open . external_ids:ovn-match-northd-version
"true"
```

這個 flag 設 true 時，ovn-controller 啟動時檢查 northd 版本，不符合直接**拒絕註冊 chassis** 到 `Chassis` 表。

後果：
- Neutron 建 port 時，要指派 chassis → list chassis 找有 `ovn-bridge-mappings` 包含該 network 的 → 空 → binding 失敗
- VM 永遠不能 schedule 到任何 host

## 解法(lab 用)

關掉 version matching：

```bash
juju ssh nova-compute/2 -- '
  sudo ovs-vsctl set open . external_ids:ovn-match-northd-version=false
  sudo systemctl restart ovn-controller
'

# 等 10 秒確認 chassis 註冊
juju ssh ovn-central/0 -- 'sudo ovn-sbctl list chassis' | head -15
# 應該看到 hostname=juju-<xxx>.lxd 的 chassis
```

**⚠️ 副作用**：重啟 ovn-controller 會**清掉** `external_ids:ovn-bridge-mappings`，要補設：

```bash
juju ssh nova-compute/2 -- 'sudo ovs-vsctl set open . external_ids:ovn-bridge-mappings=physnet1:br-ex'
```

不然 chassis 有註冊但沒 bridge mapping，port bind 仍會失敗。

## 長期解

任一個可行：

| 方案 | 代價 |
|---|---|
| 升 ovn-central charm 到 24.03 對應 channel | 目前 charmhub 沒 24.03 channel (charm 22.03 是唯一)，除非切 `ovn-central-k8s` 或 upstream 自行 build |
| 降 nova-compute 側 OVN 包到 22.03 | UCA caracal 沒 22.03 package，要 pin 舊 APT source（會破壞 nova-compute 依賴） |
| **關 version matching** ← 現用 | 小版本不相容可能造成 flow 異常，lab 可接受 |

## 驗證 OVN 真的通

```bash
# 1. Chassis 列表有 nova-compute 主機
juju ssh ovn-central/0 -- 'sudo ovn-sbctl list chassis | grep hostname'
# 預期：hostname=juju-59028d-19.lxd

# 2. Bridge mapping 在 chassis 上有
juju ssh ovn-central/0 -- 'sudo ovn-sbctl list chassis | grep ovn-bridge-mappings'
# 預期：ovn-bridge-mappings="physnet1:br-ex"

# 3. VM 建完之後 logical_switch_port 的 up=true
juju ssh ovn-central/0 -- 'sudo ovn-nbctl list logical_switch_port | grep up'
# 預期：up : true
```

## 診斷速查

| 症狀 | 查哪 | 意義 |
|---|---|---|
| `PortBindingFailed` | `ovn-sbctl list chassis` | 空 = chassis 沒註冊 |
| chassis 空 | ovn-controller log `mismatch` | 版本問題 |
| chassis 有但 port 還是 bind fail | `ovn-sbctl list chassis | grep bridge-mappings` | 應有 `physnet1:br-ex` |
| 有 chassis + mapping 還是 fail | `ovn-nbctl list logical_switch_port` | `up` 應是 true |

## 預防

部署 OpenStack + OVN 之前確認兩邊 OVN 大版本一致。混版是 lab 常見，生產要避免。

若必須混版：固定在 deploy script 最後加一條：
```bash
juju ssh nova-compute/<N> -- 'sudo ovs-vsctl set open . external_ids:ovn-match-northd-version=false'
juju ssh nova-compute/<N> -- 'sudo systemctl restart ovn-controller'
juju ssh nova-compute/<N> -- 'sudo ovs-vsctl set open . external_ids:ovn-bridge-mappings=physnet1:br-ex'
```

## 相關文件

- 本專案 `docs/runbook/day-4-configure-openstack.md`
- 相關前例：[ovn-charms-caracal-release-mismatch.md](./ovn-charms-caracal-release-mismatch.md)（charm 層級的 version 問題）
- Upstream OVN docs: https://www.ovn.org/en/
