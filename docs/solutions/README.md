# Solutions Index

Institutional knowledge from solving real problems in this project. Searchable by frontmatter tags.

## Runtime Errors

| File | Problem | Severity |
|---|---|---|
| [cirros-sshd-race-vs-ubuntu-config-drive](./runtime-errors/cirros-sshd-race-vs-ubuntu-config-drive.md) | cirros SSH refused（sshd 沒啟）／Ubuntu 無 config-drive 拿不到 key；解：Ubuntu + `--config-drive True` | 🟡 High |

## Integration Issues

| File | Problem | Severity |
|---|---|---|
| [ovn-charms-caracal-release-mismatch](./integration-issues/ovn-charms-caracal-release-mismatch.md) | OVN 22.03/stable charms 不認 caracal + SSC vs OVN tls interface 不相容 → 用 Vault 1.8 + patch charmhelpers | 🔴 Critical |
| [mysql-charm-not-compatible-with-openstack](./integration-issues/mysql-charm-not-compatible-with-openstack.md) | Canonical `mysql` 宣稱 shared-db 但實際不行 → 用 `mysql-innodb-cluster` | 🔴 Critical |
| [nova-compute-caracal-cloud-credentials](./integration-issues/nova-compute-caracal-cloud-credentials.md) | Caracal nova-compute 必須接 `cloud-credentials` 才能跟 placement 通 | 🔴 Critical |
| [nova-compute-missing-neutron-auth](./integration-issues/nova-compute-missing-neutron-auth.md) | nova.conf 缺 `[neutron]` section 導致 `Unknown auth type`；workaround 是複製 `[placement]` | 🔴 Critical |
| [ovn-version-mismatch-chassis-not-registering](./integration-issues/ovn-version-mismatch-chassis-not-registering.md) | ovn-controller 24.03 + northd 22.03 版本差 → chassis 不註冊 → PortBindingFailed | 🔴 Critical |
| [neutron-security-groups-disabled-default](./integration-issues/neutron-security-groups-disabled-default.md) | neutron-api charm `neutron-security-groups=false` 預設 → `security-group` API 404 | 🟡 High |
| [heat-missing-domain-setup-action](./integration-issues/heat-missing-domain-setup-action.md) | heat charm 寫了 stack_domain config 但不自動建 keystone domain/user，必跑 `domain-setup` action | 🟡 High |
| [magnum-api-bind-mismatch-with-haproxy](./integration-issues/magnum-api-bind-mismatch-with-haproxy.md) | magnum-api bind 127.0.0.1，但 charm 配的 haproxy backend 指 container IP → Connection reset | 🟡 High |

## 使用方式

下次遇到類似問題時 grep：
```bash
grep -rn "shared-db" docs/solutions/   # 任何 mysql 連不上
grep -rn "caracal" docs/solutions/     # 版本問題
grep -rn "MissingAuthPlugin" docs/solutions/
```
