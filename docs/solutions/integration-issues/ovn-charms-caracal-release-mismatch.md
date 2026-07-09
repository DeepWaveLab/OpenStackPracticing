---
title: "OVN 22.03/stable charms 不認識 Caracal/Bobcat release，且 tls-certificates interface 與 self-signed-certificates 不相容"
category: integration-issues
tags: [juju, openstack, ovn, caracal, bobcat, charmhelpers, tls-certificates, vault, self-signed-certificates]
component: ovn-central, ovn-chassis, neutron-api-plugin-ovn
symptom: "RuntimeError: Release caracal is not a known OpenStack release? 或 ovn-central 永遠卡 'certificates awaiting server certificate data'"
root_cause: "OVN charm 22.03/stable 內附的 charmhelpers library 只認到 yoga (2022.1)；且 SSC 新 v3 tls-certificates interface 與 OVN 舊 legacy interface schema 不相容"
severity: critical
date: 2026-04-19
---

# OVN Charms + Caracal OpenStack 整合雙坑

## Symptoms

### Pattern A：charm install hook crash
```
RuntimeError: Release caracal is not a known OpenStack release?
  File "charms_openstack/charm/core.py", line 165, in default_get_charm_instance
```
發生在：`ovn-chassis`, `neutron-api-plugin-ovn`
狀態：`juju status` 顯示 `hook failed: "install"`

### Pattern B：OVN 卡 cert 等待無限長
```
ovn-central/*  waiting  idle  "'ovsdb-peer' incomplete, 'certificates' awaiting server certificate data"
```
`self-signed-certificates/0` 收到 CSR，但 application-data 永遠空，不回簽。

## Environment When This Happens

- OpenStack charms on `2024.1/stable` (Caracal) 或 `2023.2/stable` (Bobcat)
- OVN charms on `22.03/stable` (only amd64 channel available, rev ~304/324)
- `self-signed-certificates` `1/stable` rev 588 as cert provider
- Ubuntu 22.04 (jammy)

## Root Cause

### Problem 1: Hardcoded release list

`ovn-central/ovn-chassis/neutron-api-plugin-ovn` 22.03/stable revision 的 charm 包含 `charmhelpers` library，其中：

```python
# charmhelpers/fetch/ubuntu.py
OPENSTACK_RELEASES = (
    'diablo', ..., 'yoga',   # 最後一個就是 yoga
)

# charmhelpers/contrib/openstack/utils.py
OPENSTACK_CODENAMES = OrderedDict([
    ('2011.2', 'diablo'), ..., ('2022.1', 'yoga'),  # 停在 yoga
])
```

charm 2024-11 以後才發布的 rev 還在用這份 pre-zed 的 library，完全沒更新。

當 charm 收到 relation data 或檢測本機 package 發現 release = `caracal`/`bobcat`/`zed`/`antelope` 時，validation 失敗 → `RuntimeError`。

### Problem 2: tls-certificates interface 版本分裂

`tls-certificates` 是 Juju interface 名字，但**實際協議有兩種 schema 不相容**：

| 版本 | 使用 charm | 資料放哪 | 請求欄位 |
|---|---|---|---|
| **Legacy** | OVN 22.03/stable, 舊 OpenStack charms | per-unit data bag | `certificate_name`, `sans`, `unit_name` |
| **v3 (新)** | `self-signed-certificates` 1/stable, `mysql` Canonical 8.0, modern charms | application-data bag | `certificate_signing_requests` JSON |

SSC 收到 OVN 送的 CSR（legacy 格式），不識別 → 無聲忽略，永不回簽。

## Working Solution

### Fix 1: Patch charmhelpers release list

在 charm unit 的 venv 裡直接加入新 release 字串：

```bash
# Patch OPENSTACK_RELEASES tuple
juju ssh <primary-unit> -- "sudo sed -i \"/'yoga',/a\\    'zed',\\n    'antelope',\\n    'bobcat',\\n    'caracal',\" \
    /var/lib/juju/agents/unit-<CHARM>-<N>/.venv/lib/python*/site-packages/charmhelpers/fetch/ubuntu.py"

# Patch OPENSTACK_CODENAMES dict (via Python heredoc - sed 對複雜引號易壞)
juju ssh <primary-unit> -- "sudo python3 <<'EOF'
f = '/var/lib/juju/agents/unit-<CHARM>-<N>/.venv/lib/python3.10/site-packages/charmhelpers/contrib/openstack/utils.py'
s = open(f).read()
old = \"    ('2022.1', 'yoga'),\"
new = old + \"\\n    ('2022.2', 'zed'),\\n    ('2023.1', 'antelope'),\\n    ('2023.2', 'bobcat'),\\n    ('2024.1', 'caracal'),\"
open(f, 'w').write(s.replace(old, new, 1))
EOF"

# 清 pyc cache + 重試
juju ssh <primary-unit> -- "sudo rm -rf /var/lib/juju/agents/unit-<CHARM>-<N>/.venv/lib/python3.10/site-packages/charmhelpers/{fetch,contrib/openstack}/__pycache__"
juju resolved <CHARM>/<N>
```

**注意**：subordinate charm（如 `ovn-chassis`）的 venv 在 **primary unit** 上（不是獨立 machine）。路徑：
```
/var/lib/juju/agents/unit-<subordinate-app>-<N>/.venv/...
```
要 `juju ssh <primary-unit>` 進去 patch。

Redeploy 後 patch 會被吃掉，要重跑。

### Fix 2: 用 Vault 1.8/stable 代替 SSC 給 OVN

```bash
# 拆 SSC 對 OVN 的 relation
juju remove-relation ovn-central:certificates self-signed-certificates:certificates
juju remove-relation ovn-chassis:certificates self-signed-certificates:certificates
juju remove-relation neutron-api-plugin-ovn:certificates self-signed-certificates:certificates

# 部署 Vault 1.8/stable（舊 OpenStack-charmers 風格，legacy interface）
# ⚠️ 1.15+ 是 HashiCorp K8s operator，用 v3 interface，不能用
juju deploy vault --channel=1.8/stable
juju integrate vault:shared-db mysql-innodb-cluster:shared-db

# 連到 OVN charm
juju integrate vault:certificates ovn-central:certificates
juju integrate vault:certificates ovn-chassis:certificates
juju integrate vault:certificates neutron-api-plugin-ovn:certificates

# 等到 vault workload = "Vault needs to be initialized"
juju wait-for application vault --query='status=="blocked"' --timeout=15m

# Init + Unseal (⚠️ Vault 走 HTTP，不是 HTTPS)
VAULT_IP=$(juju status vault --format=json | python3 -c "import sys,json; print(list(json.load(sys.stdin)['applications']['vault']['units'].values())[0]['public-address'])")
INIT=$(curl -s -X POST -d '{"secret_shares":1,"secret_threshold":1}' http://${VAULT_IP}:8200/v1/sys/init)
ROOT_TOKEN=$(echo "$INIT" | python3 -c "import sys,json; print(json.load(sys.stdin)['root_token'])")
UNSEAL_KEY=$(echo "$INIT" | python3 -c "import sys,json; print(json.load(sys.stdin)['keys'][0])")
echo "Save these! ROOT_TOKEN=$ROOT_TOKEN UNSEAL_KEY=$UNSEAL_KEY"
curl -s -X POST -d "{\"key\":\"${UNSEAL_KEY}\"}" http://${VAULT_IP}:8200/v1/sys/unseal

# 授權 charm + 產 CA
juju run vault/leader authorize-charm token=${ROOT_TOKEN}
juju run vault/leader generate-root-ca
```

等 30-60 秒，OVN 全部 active。

## Investigation Steps That Failed

### ❌ 嘗試用 `openstack-origin=cloud:jammy-bobcat` override release
OVN 22.03 charm（`ovn-chassis`, `ovn-central`）**沒有** `openstack-origin` config 選項。只有 `source` 或 `ovn-source`，不用於 release detection。

### ❌ `juju config nova-compute openstack-origin=cloud:jammy-bobcat` 想 in-place 降級 packages
設了沒觸發 apt downgrade，packages 保持原版本。

### ❌ 部署 `neutron-openvswitch` 避開 OVN
`neutron-openvswitch` 最新 amd64 stable 只到 `yoga/stable`（更舊），版本差太多。

### ❌ Vault 1.16/stable
Vault 1.15+ 是 HashiCorp 新 operator，用 v3 interface → 跟 OVN 一樣不相容。

## Prevention / Future Behaviour

### 部署前檢查 charm 的 charmhelpers release list

```bash
# 下載 charm source 看 OPENSTACK_RELEASES tuple
juju download <charm-name> --channel=<channel>
unzip <charm>.charm -d /tmp/charm-check
grep -A 30 "OPENSTACK_RELEASES =" /tmp/charm-check/venv/*/charmhelpers/fetch/ubuntu.py
# 如果最後一個不到你的 OpenStack release，預期會壞
```

### Cert provider 選擇規則
- 部署包含 OVN 22.03/stable 家族 → 用 `vault 1.8/stable`
- 純新式 charm（Canonical mysql, keystone 2024.1 等）→ `self-signed-certificates 1/stable` OK
- 混用時：兩個 cert provider 並存，各自 relation 對應 charm 群

### 部署 Vault 的 manual step 記得備份
`~/vault-secrets` 丟失 → Vault 整套重來。`scp` 備份到本機。

## Related

- 本專案 `docs/runbook/day-3-vault-tls.md` — 完整 replay 指令
- 本專案 `docs/runbook/day-3-network-stack.md` — OVN 部署順序
- Canonical bug report（如有）：charmhelpers 未更新至 caracal
- Upstream: https://opendev.org/openstack/charm-ovn-central
