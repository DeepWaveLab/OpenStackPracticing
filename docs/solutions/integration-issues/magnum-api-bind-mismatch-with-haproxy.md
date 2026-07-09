---
title: "Magnum charm: magnum-api bind 127.0.0.1 但 haproxy backend 配 container IP → API 502"
category: integration-issues
tags: [magnum, haproxy, bind, charm-bug, caracal]
component: magnum (2024.1/stable rev 70)
symptom: "`openstack coe cluster template list` 回 `RemoteDisconnected('Remote end closed connection without response')`；curl haproxy port 9511 得 `Empty reply`"
root_cause: "charm 沒在 /etc/magnum/magnum.conf 的 [api] 寫 host，Magnum default 127.0.0.1；但同一個 charm 的 haproxy config 把 backend 指到 container IP:9501，兩邊不對盤"
severity: high
date: 2026-04-20
---

# Magnum charm：bind / haproxy mismatch

## Symptom

```bash
$ openstack coe cluster template list
Unable to establish connection to http://10.254.154.240:9511/v1/clustertemplates:
  ('Connection aborted.', RemoteDisconnected('Remote end closed connection without response'))

$ curl -I http://10.254.154.240:9511/v1
curl: (52) Empty reply from server
```

但 Keystone catalog 顯示 endpoint 正確 (`http://<ip>:9511/v1`)，服務看起來 active。

## Root Cause

Magnum 在 LXD container 裡的兩個程序：

| 程序 | 實際 listen |
|---|---|
| `magnum-api` | `127.0.0.1:9501`（loopback 而已） |
| `haproxy` | `0.0.0.0:9511`（對外） |

`haproxy.cfg` 的 backend 配置：
```
backend magnum-api_admin_10.254.154.240
    server magnum-0 10.254.154.240:9501 check    ← 連 container 外部 IP
```

但 magnum-api 只 bind `127.0.0.1:9501`，不接受來自 `10.254.154.240` 的連線。
結果 haproxy 連 backend 永遠 TCP RST，client 收到 `Empty reply`。

這是 **charm 的內部配置 bug**：charm 應該讓 magnum-api bind `0.0.0.0` 或改 haproxy backend 指 `127.0.0.1:9501`，兩者選一，但現在兩邊都沒有。

## Solution

編輯 magnum.conf 加 `[api] host = 0.0.0.0`：

```bash
cat <<'EOF' | juju ssh magnum/0 -- sudo bash
sed -i '/^\[api\]/a host = 0.0.0.0' /etc/magnum/magnum.conf
grep -A 3 '^\[api\]' /etc/magnum/magnum.conf
systemctl restart magnum-api
sleep 3
ss -tlnp | grep 9501
# 期望看到 0.0.0.0:9501 而不是 127.0.0.1:9501
EOF
```

驗證：
```bash
source ~/openrc
openstack coe cluster template list -f json
# []
curl http://<magnum-ip>:9511/
# {"name": "OpenStack Magnum API", "versions": [{"id": "v1", ...}]}
```

## Diagnostic Cheatsheet

```bash
# magnum-api 有跑？
juju ssh magnum/0 -- sudo systemctl is-active magnum-api
# → active

# 但 listen 哪？
juju ssh magnum/0 -- "sudo ss -tlnp | grep magnum"
# 127.0.0.1:9501 = bug；0.0.0.0:9501 = OK

# haproxy backend？
juju ssh magnum/0 -- "sudo grep -A 5 magnum-api_admin /etc/haproxy/haproxy.cfg"

# 直連 magnum-api：
juju ssh magnum/0 -- "curl -sI http://127.0.0.1:9501/v1"
# 應該 回 405 或 200

# 透過 haproxy：
juju ssh magnum/0 -- "curl -sI http://127.0.0.1:9511/"
# 若 empty → haproxy 連不到 backend
```

## 副作用 & Prevention

- charm 下次跑 `config-changed` / `certificates-changed` hook 會<strong>覆寫 magnum.conf</strong>，patch 被吃掉
- 每次 config 改動後要手動重跑 sed + restart
- 長期解：systemd override 加 pre-exec 跑 sed；或提 charm bug 給 OpenStack Charmers

## Related

- 本專案 `docs/runbook/day-6-magnum.md`
- 類似 pattern：Magnum 跟 Heat 一樣有 `domain-setup` action 不自動跑（另一個 charm quirk）
