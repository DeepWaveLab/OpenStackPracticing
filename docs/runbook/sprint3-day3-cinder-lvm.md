# Sprint 3 / Day 3: Cinder LVM(Sprint 1 雪恥項 #1)

> 課程定位:把 data disk 變成 Cinder 的 LVM backend,學會「已上線的 Kolla 環境增量加服務」的工作流,並完成 volume 生命週期 + boot-from-volume。
> Sprint 1 這一步死在 LXD:unprivileged container 拿不到 device-mapper ioctl 權限、udev event 進不了 namespace(solutions #22)。本次跑在真 VM,理應直通 —— 驗證這個假設。

## 原理與架構

### 1. Cinder 三元件與資料路徑

```
openstack CLI ──► cinder-api ──► cinder-scheduler ──► cinder-volume(LVM driver)
                                                          │ lvcreate -L 5G cinder-volumes
                                                          ▼
   nova-compute ◄── iSCSI target(host 上的 LIO/tgt)◄── LV /dev/cinder-volumes/volume-<uuid>
        │ 把 iSCSI LUN 接給 qemu,VM 內看到 /dev/vdb
```

關鍵理解:**control path 是 OpenStack 的,data path 是 Linux 的**(LVM + iSCSI)。AIO 下 iSCSI 是 loopback(initiator 和 target 同一台),生產 multinode 就是跨網路的同一套協定 —— 架構不變,只是距離變遠。

### 2. 增量加服務的 Kolla 工作流

1. 前置的 host 狀態自己準備(VG)—— kolla 不碰你的磁碟分割
2. `globals.yml` 開 flag
3. **重跑 `kolla-ansible deploy`**(冪等):只會新增/改動需要的容器,順帶把 nova.conf 補上 cinder 相關設定並重啟受影響容器

這就是 Day 1 說的「編排期綁定」的代價與好處:改設定 = 重跑編排,但整個過程宣告式、可重複。

### 3. 為什麼 Sprint 1 的 LXD 做不到(對照教材)

| 需求 | LXD unprivileged container | 本次(Azure VM) |
|---|---|---|
| `/dev/mapper/control` ioctl | ❌ 被 namespace 擋 | ✅ 直接是 host |
| udev event(LV 出現) | ❌ 不進 container | ✅ |
| iSCSI kernel target | ❌ 需要 host kernel 權限 | ✅ |

## 步驟

### 1. Data disk → VG(動手前先確認是對的碟!)

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE /dev/sdb   # 256G、無分割、無檔案系統
sudo blkid /dev/sdb || echo "safe"                    # 無 signature 才動手
sudo pvcreate /dev/sdb
sudo vgcreate cinder-volumes /dev/sdb                 # VG 名稱 = kolla 預設 cinder_volume_group
```

註:E16s_v5 沒有 local temp disk,所以 sdb 一定是 data disk;用有「d」的機型(Ed16s_v5)時 sdb 可能是 temp disk,**必按 size 確認**。

### 2. globals.yml + 增量 deploy

```bash
printf 'enable_cinder: "yes"\nenable_cinder_backend_lvm: "yes"\n' >> /etc/kolla/globals.yml
source ~/kolla-venv/bin/activate
kolla-ansible deploy -i ~/all-in-one    # 冪等重跑
```

### 3. Volume 生命週期 lab

```bash
source ~/demo-openrc.sh
openstack volume create --size 5 vol-data          # → available
openstack server add volume vm-ubuntu vol-data     # VM 內出現 /dev/vdb
ssh -i ~/.ssh/oslab_ed25519 ubuntu@172.24.4.111 \
  "sudo mkfs.ext4 /dev/vdb && sudo mount /dev/vdb /mnt && echo test | sudo tee /mnt/proof.txt"
openstack server remove volume vm-ubuntu vol-data  # → available
sudo lvs cinder-volumes                            # host 端看 thin LV
```

### 4. Boot-from-volume

```bash
openstack volume create --image cirros --size 2 vol-boot   # 把 image 灌進 volume
openstack server create --flavor m1.tiny --volume vol-boot \
  --network net1 --key-name oslab --wait vm-bfv            # 根碟在 Cinder,不在 hypervisor 本地
```

## Checkpoint(全數通過 2026-07-08)

| 驗證 | 判準 | 實測 |
|---|---|---|
| `openstack volume service list` | scheduler + volume@lvm-1 up | ✅ |
| attach → VM 內可用 | /dev/vdb 可 mkfs/mount/寫檔 | ✅ |
| detach | volume 回 available、資料保留在 LV | ✅ |
| boot-from-volume | vm-bfv ACTIVE | ✅ |
| host 端 | LV 出現在 cinder-volumes VG | ✅ thin LV(見下) |
| **雪恥驗證** | Sprint 1 solutions #22 的 LXD 限制在真 VM 上不存在 | ✅ 零阻力直通 |

## 踩坑 / 觀察

### 1. cinder-backup 預設被帶起來但必然 down

`enable_cinder: "yes"` 會順帶部 cinder-backup,但它需要 backup backend(Swift/Ceph/NFS),本 lab 都沒有 → 服務 down。**這不是故障**;處理:`enable_cinder_backup: "no"` + `docker rm -f cinder_backup`。教訓:kolla 的服務開關有「父子連動」,開父服務前先看它會帶起哪些子服務。

### 2. Epoxy 預設 LVM thin provisioning(觀察,非坑)

`lvs` 看到的不是 5G 的 thick LV,而是 243G 的 `cinder-volumes-pool`(thin pool)+ 掛在裡面的 thin LV(實際只佔 1.33%)。代表 volume 超賣是預設行為 —— 生產環境要監控 pool 的 Data% 而不是 VG free。

## 下一步(Day 4)

Octavia(Sprint 1 雪恥項 #2):amphora image 準備 + LB 手動建置全流程。
