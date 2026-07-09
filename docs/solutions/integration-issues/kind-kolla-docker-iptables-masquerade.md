# kind on Kolla host:所有 image pull i/o timeout —— Kolla 把 docker iptables 關了

**Sprint 3 / Day 6(2026-07-09)· Kolla-Ansible(Epoxy)host + kind v0.32.0 / Docker 29.6.1 / Ubuntu 24.04**

## 症狀

- 在跑著 Kolla-Ansible 的同一台 host 上 `kind create cluster` 成功,control-plane Ready
- 但 kind cluster 內**所有 pod 卡 `ImagePullBackOff`**(cert-manager、ORC、之後的 CAPI/CAPO):

```
Failed to pull image "quay.io/jetstack/cert-manager-webhook:v1.20.2":
  failed to resolve reference ...: dial tcp 100.58.143.218:443: i/o timeout
```

- 連帶 `clusterctl init` 死在 `Waiting for cert-manager to be available... context deadline exceeded`
- **關鍵對比**:host 自己 `curl -sI https://quay.io/v2/` 秒回 401(通),Kolla 的 45 個容器也都是從 quay.io 拉的 —— 所以 host 對外沒問題,問題只在 kind 這層

## 排查(逐步排除,別跳結論)

1. **host 連 quay OK** → 問題在 kind node → internet 這條路
2. kind node 連**任意外部 IP**(`/dev/tcp/1.1.1.1/443`)也 FAIL → **非 quay 專屬**,是整條 egress 斷
3. TCP **handshake 就失敗**(不是傳到一半斷)→ **排除 MTU**(MTU 只會斷大封包,SYN 是小封包會過)
4. `ip_forward=1`、`iptables -P FORWARD ACCEPT` 都正常 → 不是 forward 被擋
5. `iptables -t nat -S POSTROUTING`:**只有 Kolla 的 `-s 172.24.4.0/24 -o eth0 -j MASQUERADE`,沒有 kind 網段(172.17.0.0/16)的** → 容器封包帶 private source 出 eth0,Azure 直接丟
6. `cat /etc/docker/daemon.json` → 真相:

```json
{ "bridge": "none", "ip-forward": false, "iptables": false, ... }
```

## 根因

**Kolla-Ansible 刻意把 docker 設成 `iptables:false`(+ `ip-forward:false`、`bridge:none`)。** 因為 Kolla 容器全走 **host network**,網路由 neutron/OVN 自己管,不要 docker 去動 iptables/NAT。

但 **kind 是正常的 bridge 網路容器**,它的對外連線靠 docker 幫忙建 `MASQUERADE`(SNAT 成 host IP)。`iptables:false` 之下 docker 不建這條 → kind node 的封包帶著 `172.17.x` 私網來源出 eth0 → 上游丟棄 → 一切 image pull `i/o timeout`。

> 附帶次要因素:kind 的 docker network 帶 IPv6 ULA(`fc00::/64`)且有 v6 default route,`getent hosts quay.io` 會先回 AAAA,Azure 又無 IPv6 egress,更拖慢。IPv4 修好後 Happy Eyeballs 走 v4 即可,非必修。

## 解法(外科手術,不動 Kolla)

**不要**把 daemon.json 改成 `iptables:true` 或全域重啟 docker 網路 —— 那會讓 docker 重寫整台 host 的防火牆,可能弄壞正在跑的 OpenStack(OVN、neutron、Octavia、45 個容器)。只加一條**只針對 kind 網段**的 MASQUERADE,比照 Kolla 既有那條的模式:

```bash
sudo iptables -t nat -A POSTROUTING -s 172.17.0.0/16 -o eth0 -j MASQUERADE
# 驗證
docker exec capi-mgmt-control-plane bash -c 'cat </dev/null >/dev/tcp/1.1.1.1/443 && echo OK'
```

### 持久化(VM 每天 auto-shutdown/reboot)

用獨立的 systemd oneshot,開機冪等補上(check-then-add,不會重複):

```ini
# /etc/systemd/system/kind-masquerade.service
[Unit]
Description=MASQUERADE for kind mgmt-cluster egress (workaround Kolla docker iptables=false)
After=docker.service
Wants=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash -c "iptables -t nat -C POSTROUTING -s 172.17.0.0/16 -o eth0 -j MASQUERADE 2>/dev/null || iptables -t nat -A POSTROUTING -s 172.17.0.0/16 -o eth0 -j MASQUERADE"

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now kind-masquerade.service
```

## 教訓

- **在 OpenStack/Kolla host 上跑任何「正常 bridge」的 docker 工作負載(kind、compose)都會撞這顆** —— Kolla 的 `iptables:false` 是全域前提,不是 kind 的錯。
- 排查網路「連不上」先分層:host 通不通 → 容器通不通 → 任意 IP 還是特定 IP → handshake 斷還是傳輸斷(分辨 firewall/NAT vs MTU)。一路排除比猜快。
- `172.17.0.0/16` 是本機 docker 給 `kind` network 配到的網段;若 docker 配到別的(如 172.18.x),MASQUERADE 的 `-s` 要對應改。用 `docker network inspect kind -f '{{range .IPAM.Config}}{{.Subnet}}{{end}}'` 確認。
