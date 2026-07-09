# Day 8: Storage 嘗試與 LXD 限制

## 目的

本 sprint 最後想補上 block storage（Cinder + Ceph 或 LVM backend）。這裡記**踩的限制**，不是成功步驟。結論：**LXD unprivileged container 不適合跑 storage service**。

## 對應官方指南

[Install OpenStack § Ceph OSD / Ceph Monitor / Cinder](https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/install-openstack.html)

## 嘗試 1：Ceph (ceph-mon + ceph-osd reef/stable)

### 指令
```bash
juju deploy -n 3 ceph-mon --channel=reef/stable
juju deploy -n 3 ceph-osd --channel=reef/stable --storage osd-devices=loop,20G,1
juju integrate ceph-osd:mon ceph-mon:osd
```

### 卡在哪：loop device 不夠

```
Unit       Status     Message
ceph-osd/0 attaching  losetup: cannot find an unused loop device: exit status 1
```

Host 預設只建 `/dev/loop0-7`（前 7 個已被 snap 佔）。需要手動加：
```bash
for i in $(seq 8 31); do sudo mknod -m 0660 /dev/loop$i b 7 $i; done
```

LXD container 不會自動看到新 loop device，還要 bind-mount：
```bash
for M in <osd-machine-ids>; do
  CT=juju-59028d-$M
  for L in 8 9 10 11 12 13 14 15 16; do
    sudo lxc config device add $CT loop$L unix-block source=/dev/loop$L path=/dev/loop$L
  done
  sudo lxc config device add $CT loopctl unix-char source=/dev/loop-control path=/dev/loop-control
done
```

juju 的 loop 儲存 backing file 路徑：`/var/lib/juju/storage/loop/volume-<machine>-<idx>`。要手動 `truncate -s 20G` 建，juju 自己不會重試。

### 最後還是死：ceph-volume 等 udev event

```
unit.ceph-osd/0.juju-log: Skipping osd devices previously processed by this unit: []
unit.ceph-osd/0.mon-relation-changed: WARNING: Device /dev/loop8 not initialized in udev database even after waiting 10000000 microseconds.
```

**Root cause**：LXD unprivileged container 的 udev 跑在 host，container 內看不到 udev event。ceph-volume 等 event 等到天荒地老。

**產業解**：ceph-osd 放 LXD VM 或實體機，不是 LXD container。

## 嘗試 2：Cinder + 內建 LVM backend

### 跳 cinder-lvm subordinate charm 的原因
```bash
juju info cinder-lvm | grep 2024
# 2024.1/stable: s390x  ubuntu@22.04
# 沒 amd64，amd64 最新是 yoga/stable（2022）
```

Canonical 在 amd64 上 cinder-lvm 也斷代。改用 cinder charm 的 `[LVM]` 內建 driver。

### 手動建 LVM VG
```bash
# 在 cinder container 內
sudo truncate -s 20G /var/lib/cinder/cinder-volumes.img
sudo losetup /dev/loop17 /var/lib/cinder/cinder-volumes.img
sudo pvcreate /dev/loop17
sudo vgcreate cinder-volumes /dev/loop17
```

VG 建好（有 udev warning 但不 block）。

### 加 [LVM] section 到 cinder.conf
charm 寫了 `enabled_backends = LVM` 跟 DEFAULT section 的 `volume_group=cinder-volumes`，但**沒建 [LVM] section**（Cinder 新版要求分 backend section）：

```ini
[LVM]
volume_backend_name = LVM
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
lvm_type = default
target_protocol = iscsi
target_helper = tgtadm
```

重啟 cinder-volume，service 註冊 up ✓（`openstack volume service list` 看到 `cinder-volume up`）。

### 卡在哪：lvcreate 失敗

```bash
openstack volume create --size 5 test-vol
# status = error
```

log：
```
Command: lvcreate -n volume-xxx cinder-volumes -L 5g
Stderr: device-mapper: version ioctl on failed: Permission denied
         Incompatible libdevmapper 1.02.175 and kernel driver (unknown version).
         striped: Required device-mapper target(s) not detected in your kernel.
```

**Root cause**：LXD unprivileged container 沒有 device-mapper ioctl 權限（kernel 層 deny）。bind-mount `/dev/mapper/control` 無助。

**解法**：
1. `lxc config set <ct> security.privileged=true` — 弱化 container 隔離（production 不會這樣）
2. 把 cinder 部到 LXD VM — redeploy cinder 到 VM，加 `--constraints virt-type=virtual-machine`
3. 用外部 Ceph / SAN，cinder 只做 control plane

## 業界實務對照

**為什麼 Canonical production 從不把 ceph / cinder 放 LXD container**：
- **OpenStack Charms 官方 reference architecture** 都基於 MaaS + 實體機
- 偶爾用 LXD container 也只限 API 服務（keystone / nova-api / neutron-api），**storage / compute 一律不進 container**
- 本 lab 把 cinder deploy 到 LXD container 是 Juju 預設行為，沒人 override 就會這樣，**是 lab-specific 的狀況，不是 production pattern**

## 本次 sprint 收尾的妥協

- 不解 LVM/Ceph in LXD 限制（要弱化安全性或 redeploy 到 VM）
- Cinder control plane（API / scheduler）**有 active 且註冊到 catalog**
- 沒能 provision 實際 volume

## 3 種可能的完整路徑（留給下次）

| 路徑 | 方法 |
|---|---|
| A. Cinder 到 LXD VM | `juju add-machine --constraints="virt-type=virtual-machine mem=4G cores=2 root-disk=30G"`<br>`juju deploy cinder --to <machine> ...`<br>Relation 重接 |
| B. Cinder privileged container | `lxc config set <ct> security.privileged=true`<br>快但弱化安全性 |
| C. External Ceph | 另外架 Ceph cluster（MAAS / 真 bare metal），從 OpenStack 這邊只部 cinder-ceph / glance-ceph subordinate 接外部 Ceph |

**C 是 production 標準做法**（儲存跟計算分離）。

## 學到的通用規則

**LXD container 適合 vs 不適合 OpenStack 服務**：

| 服務 | container 適合？ |
|---|---|
| keystone / horizon / glance-api / placement / heat-api / magnum-api | ✅ 適合 |
| nova-api / nova-cc / nova-scheduler / nova-conductor | ✅ 適合 |
| neutron-api | ✅ 適合 |
| rabbitmq / mysql / vault / self-signed-certs | ✅ 適合 |
| **nova-compute（KVM）** | ❌ 要 VM |
| **ovn-chassis（OVN data plane）** | 🟡 能跑但常要 bind-mount 修 |
| **ceph-osd** | ❌ 要 VM 或實體機（udev / block device） |
| **cinder-volume LVM** | ❌ 要 VM 或實體機（device-mapper ioctl） |

**這是 Canonical 文件沒明寫、但踩過才知道的事實**。

## Related

- 本專案 `docs/reflection.md` — 整個 sprint 的總回顧
- `docs/runbook/day-3-nova-compute.md` — nova-compute 就用了 LXD VM（同樣限制的範例）
