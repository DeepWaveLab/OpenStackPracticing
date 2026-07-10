---
title: "cirros 0.6 在 OVN 上 sshd 沒啟動；Ubuntu 需 --config-drive 避開 metadata race"
category: runtime-errors
tags: [cirros, ubuntu, cloud-init, config-drive, ovn, sshd, metadata]
component: Nova VM boot, cloud-init, OVN metadata agent
symptom: "cirros：SSH Connection refused；Ubuntu 無 config-drive：SSH Permission denied (publickey)"
root_cause: "OVN 程式化 logical flow 比 VM boot 慢；cirros 早期放棄 DHCP 就不啟 sshd；Ubuntu 用 169.254.169.254 metadata 拿 key，race 輸就拿不到"
severity: high
date: 2026-04-19
---

# cirros sshd race / Ubuntu metadata race（OVN lab 常見）

## 症狀

### cirros 0.6 image
```
$ ssh cirros@<floating-ip>
ssh: connect to host <fip> port 22: Connection refused
```

但 `ping` 通、`tcpdump` 看到 VM 有 IP 跑 HTTP 到 metadata。TCP SYN 到 VM → VM 回 RST。

### Ubuntu 22.04 image（沒加 --config-drive）
```
$ ssh ubuntu@<floating-ip>
ubuntu@<fip>: Permission denied (publickey).
```

console log：
```
Cloud-init ... finished ... Datasource DataSourceNone.
Used fallback datasource
ci-info: no authorized SSH keys fingerprints found for user ubuntu.
```

## 根因

OpenStack + OVN 啟 VM 時，logical_flow 程式化要 ~10-30 秒。期間：
- VM 已 boot、kernel up
- 但 VM 發 DHCP / HTTP 到 metadata 可能收不到 reply（OVN flow 還沒 program 好）

### cirros 的脆弱點

cirros init script `/etc/init.d/rc.sysinit`：
1. 啟 dhcpcd，立刻檢查 "is there carrier?"
2. 若無 carrier → 進 "offline" 分支
3. **offline 分支裡根本不啟 sshd**（cirros 假設 offline 就不需要 ssh）
4. 晚一點背景的 dhcpcd 成功拿到 IP，但 init flow 已經過去，sshd 永遠不會啟

### Ubuntu 的脆弱點

cloud-init 啟動時會試 multiple datasources，包括 OpenStack 的 `http://169.254.169.254/openstack/`。若 OVN metadata agent 還沒 program 好 ARP flow，這個 URL 不通：
- cloud-init 標記為 `DataSourceNone`（fallback）
- 就不 inject SSH keypair
- sshd 雖啟，但 authorized_keys 空 → Permission denied

## 解法

### 最佳解：使用 Ubuntu image + `--config-drive True`

```bash
openstack server create \
  --image jammy --flavor ubuntu.small \
  --network tenant_net --key-name lab_key --security-group lab_sg \
  --config-drive True \   # ← 關鍵
  vm2 --wait
```

`--config-drive True` 讓 Nova 把 metadata（含 keypair、user-data）打包成 **ISO 檔掛載到 VM** 的 /dev/sr0。cloud-init 優先從 config drive 讀，**不 rely on 169.254.169.254 HTTP**，避開 race。

### 不能用 cirros 的原因

cirros 沒支援 cloud-init，拿 key 是用舊版 `curl metadata` 模式。即使加 config-drive 它也不會讀。**cirros 基本上不適合 OVN 環境 end-to-end 測試**。

## 本專案實測結果

| Image | config-drive | 結果 |
|---|---|---|
| cirros 0.6 | - | SSH Connection refused（sshd 沒啟動） |
| jammy | False（預設） | SSH Permission denied（cloud-init fallback） |
| jammy | True | ✅ SSH 成功 |

## 檢查自己踩到哪個雷

```bash
# 確認 VM 網路本身通
ping <floating-ip>  # 通 = OVN NAT 沒問題

# TCP 握手有沒有到 sshd
nc -zv <floating-ip> 22
# "Connection refused" → sshd 沒啟（cirros 狀況）
# "succeeded" → sshd 跑，但可能沒拿 key

# 從 console 看 cloud-init 結果
openstack console log show <vm> | grep -iE 'cloud-init|datasource|authorized'
# "DataSourceNone" / "no authorized SSH keys" → metadata race（Ubuntu 無 config-drive）
```

## 預防

**部署 script 最後一步一定加 `--config-drive True` 給 Ubuntu image**。任何 cloud-init 為基礎的 image（Ubuntu / CentOS / Rocky / Debian）都適用。

若不能改 script，可以設 Glance image property：
```bash
openstack image set jammy --property img_config_drive=mandatory
# 之後不指定 --config-drive 也會強制啟用
```

## 為什麼官方指南 Jammy 沒踩到

官方文件針對的是 MaaS 實體部署，bare metal 的網卡 carrier 穩定、OVN flow 先啟再 boot VM，race window 很小。我們在 Azure VM 裡跑 LXD VM 套 nested KVM，每層都慢一點，race window 大很多，所以一定中。

## 相關文件

- 本專案 `docs/runbook/day-4-configure-openstack.md`
- cirros upstream issue: https://github.com/cirros-dev/cirros
- cloud-init datasource docs: https://cloudinit.readthedocs.io/en/latest/topics/datasources/openstack.html
