---
title: "nova-compute 2024.1/stable 沒寫 nova.conf [neutron] section → VM build 噴 MissingAuthPlugin"
category: integration-issues
tags: [nova-compute, neutron, caracal, auth, charm-bug]
component: nova-compute (2024.1/stable rev 827)
symptom: "VM status = ERROR，fault.message: `Build of instance ... aborted: Unknown auth type: None`"
root_cause: "nova-compute charm 沒有把 [neutron] 寫進 /etc/nova/nova.conf；neutron 的 service 帳號傳遞路徑有缺口（跟 placement 的 cloud-credentials relation 類似，但 neutron 沒對應 relation）"
severity: critical
date: 2026-04-19
---

# nova-compute [neutron] section 沒寫出來

## 症狀

```bash
$ openstack server show vm1 -c fault
{'code': 500, 'message': 'Build of instance ... aborted: Unknown auth type: None',
 'details': '... File "/usr/lib/python3/dist-packages/nova/network/neutron.py",
             line 1167, in allocate_for_instance ...'}
```

nova-compute log：
```
ERROR nova.compute.manager ... keystoneauth1.exceptions.auth_plugins.MissingAuthPlugin:
An auth plugin is required to determine endpoint URL
```

位置在 `allocate_for_instance` 試圖跟 Neutron API 講話。

## 根因

檢查 nova.conf：

```bash
$ juju ssh nova-compute/2 -- grep -E '^\[' /etc/nova/nova.conf
[DEFAULT]
[pci]
[keystone_authtoken]
[service_user]
[glance]
[api]
[libvirt]
[oslo_messaging_rabbit]
[oslo_messaging_notifications]
[notifications]
[cinder]
[oslo_concurrency]
[workarounds]
[serial_console]
[placement]      ← 這個有 ✅
[compute]
[vnc]
[wsgi]
                 ← [neutron] 完全不在！
```

placement 在 Caracal 靠 `cloud-credentials` relation 拿到 keystone service account 注入 `[placement]`。
**Neutron 沒有對應 relation** — nova-compute charm 只有：
- `cloud-compute` ↔ nova-cc（只傳 availability_zone）
- `neutron-plugin` ↔ ovn-chassis（只傳 metadata-shared-secret）

兩條都**沒帶 neutron 的 auth 資訊**。nova-cc 自己有 neutron-api relation 拿到 URL + auth，但**沒把資料 forward** 到 nova-compute。

結論：**nova-compute 2024.1/stable 是 charm bug**。缺了 [neutron] 注入，或者需要 neutron-api 的新 relation。

## 解法(workaround)

直接複製 `[placement]` 的同一組 service account 當 `[neutron]`。Nova 會從 keystone catalog 找 `network` service endpoint。

```bash
cat <<'EOF' | juju ssh nova-compute/2 -- sudo bash
python3 - <<'PY'
import re
p = '/etc/nova/nova.conf'
s = open(p).read()
m = re.search(r'(\[placement\].*?)(?=\n\[|\Z)', s, re.DOTALL)
pl = m.group(1)
neu = pl.replace('[placement]', '[neutron]', 1)
if '[neutron]' in s:
    s = re.sub(r'\[neutron\].*?(?=\n\[|\Z)', neu.rstrip(), s, count=1, flags=re.DOTALL)
else:
    s = s.rstrip() + '\n\n' + neu + '\n'
open(p, 'w').write(s)
print('patched')
PY
systemctl restart nova-compute
EOF
```

產生的 `[neutron]` 區段：
```ini
[neutron]
auth_url = http://<keystone-admin-ip>:35357
auth_type = password
project_domain_name = service_domain
user_domain_name = service_domain
project_name = services
username = nova_compute
password = <same as placement>
os_region_name = RegionOne
region_name = RegionOne
```

## 為什麼 `nova_compute` user 能 auth neutron

keystone charm 透過 `cloud-credentials`（原本給 placement）建的 `nova_compute` user 在 `services` project 有 `admin` role — 足以 call neutron API 做 port binding、network lookup 等所有 VM lifecycle 需要的操作。

## 副作用

- charm 下次跑 config-changed / upgrade-charm hook 會**重寫 nova.conf** → 這個 patch 被吃掉。需重跑。
- 可以把 patch 放成 systemd override 或 cron 確保 idempotent。

## 診斷

```bash
juju ssh nova-compute/2 -- 'grep -A 1 "^\[neutron\]" /etc/nova/nova.conf'
# 若空 → 被 charm 重寫，重跑 patch
```

## 預防

Upstream 修好前無解。建議在 deploy script 最後一步固定跑一次 patch，並把 patch 放到 `/root/fix-nova-neutron.sh` 供日後用。

## 相關文件

- 本專案 `docs/runbook/day-3-nova-compute.md`
- 本專案 `docs/runbook/day-4-configure-openstack.md`
- 類似現象前例：[nova-compute-caracal-cloud-credentials.md](./nova-compute-caracal-cloud-credentials.md)（placement 缺 relation 的版本；neutron 是同類問題但連 relation 都沒有）
