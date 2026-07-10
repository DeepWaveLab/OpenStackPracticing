# Kolla Octavia tenant 模式在 Ubuntu 24.04 必踩:dhclient 不存在

**Sprint 3 / Day 4(2026-07-08)· Kolla-Ansible 20.x(Epoxy)· Ubuntu 24.04 · octavia_network_type=tenant**

## 症狀

`kolla-ansible deploy` 死在:

```
TASK [octavia : Restart octavia-interface.service if required]
fatal: ... Unable to start service octavia-interface
```

`systemctl status octavia-interface`:

```
Process: ExecStart=/sbin/dhclient -v o-hm0 -cf /etc/dhcp/octavia-dhclient.conf (code=exited, status=203/EXEC)
```

`status=203/EXEC` = **執行檔不存在**。

## 根因

tenant 模式下 kolla 生成的 `octavia-interface.service` 用 `dhclient` 幫 o-hm0 要 DHCP,但 **Ubuntu 24.04(Noble)cloud image 不再內建 isc-dhcp-client**(ISC 已棄案 dhclient,Noble 全面改 systemd-networkd)。kolla Epoxy 的 unit 檔沒跟上。

o-hm0 本身其實已成功插進 br-int(`ovs-vsctl list-ports br-int` 可見),只差要 IP 這步。

## 解法

```bash
sudo apt install -y isc-dhcp-client
sudo systemctl reset-failed octavia-interface.service
sudo systemctl start octavia-interface.service
ip -br a show o-hm0        # 應拿到 lb-mgmt-subnet 的 IP(OVN 原生 DHCP 發的)
kolla-ansible deploy -i ~/all-in-one --tags octavia   # 補跑剩餘任務
```

## 教訓

1. host OS 大版升級(22.04→24.04)時,**部署工具對 host 工具鏈的隱含依賴**是地雷來源(對照 Day 1 的 dbus-python、docker SDK —— 同一類)。
2. `status=203/EXEC` 直接讀成「binary 不在」,不用進 journalctl 挖。
3. `--tags <role>` 補跑比整包 deploy 快 5-10 倍,修單一 role 的錯就用它。
