# Day 14:Ceph 基礎 —— 在同一台主機立起儲存骨幹

> 今天不碰 OpenStack 本體,改立一套 **Ceph**——生產環境 OpenStack 的儲存標配。它會成為 Day 15(物件儲存)與 Day 16(共享檔案系統)的共同地基。單機跑 Ceph 有一串「必須預先知道」的調校,本章全部寫清楚;照著做,一小時內 `HEALTH_OK`。

!!! abstract "你在課程的哪裡"
    - **Day 3 的儲存**:Cinder 用主機上的 LVM——單機夠用,但生產環境很少這樣做。
    - **今天**:用 **cephadm** 在同一台主機部署一套最小 Ceph 叢集(mon + mgr + osd)。今天只立骨幹,還不接 OpenStack。
    - **今天之後**:Day 15 在上面開物件儲存(RGW),Day 16 開共享檔案系統(CephFS)。

## 第一次接觸 Ceph?先讀這段

### 為什麼生產環境的答案是 Ceph

![Ceph 官方標誌](../assets/logos/ceph.png){ align=right width="110" }

Day 3 的 LVM 方案有個天花板:硬碟綁在單一主機上,主機掛了資料就下線。**Ceph 是一套分散式儲存系統**:把許多主機的許多硬碟組成一個大池子,資料自動複製多份、自動修復,而且**一套系統同時提供三種儲存介面**——區塊(給 Cinder 當後端)、物件(S3/Swift API)、檔案(共享檔案系統)。這就是為什麼業界不再為物件儲存單獨維護一套 Swift(這段歷史在 [Day 12](sprint3-day12-sprint4-preview.md) 講過):養一套 Ceph,三種儲存全有了。

順帶一提,Ceph 的吉祥物是章魚 🐙——多隻觸手同時做事,跟它的架構挺相配。

### 最小心智模型:三種守護程序

```mermaid
flowchart TB
    subgraph brain["控制面(小而關鍵)"]
        direction LR
        MON["mon:叢集地圖與投票<br/>(誰活著、資料在哪)"] ~~~ MGR["mgr:管理與編排<br/>(cephadm 的大腦)"]
    end
    subgraph data["資料面(可以一直加)"]
        direction LR
        OSD1["osd:一顆硬碟一個 osd<br/>真正存資料的人"] ~~~ POOL["pool:邏輯儲存池<br/>(設定複本數的單位)"]
    end
    brain ==>|"指揮資料放置"| data
```

生產環境:mon×3(投票要奇數)、osd 幾十到幾千顆。我們的 lab:全部一份,擠在同一台主機——**這正是今天所有特殊調校的原因**。

### cephadm 與 Kolla 的分工

Kolla-Ansible 自 2020 年起就**不部署 Ceph 本體**,只做「接上一套現成 Ceph」的整合——部署 Ceph 的官方工具是 **cephadm**(Ceph 自帶的編排器)。所以今天的角色分工:cephadm 管 Ceph、Kolla 管 OpenStack,兩者同機共存。

一個關鍵決定:**cephadm 底下用 podman,不用 docker**。因為 Kolla 每次 deploy 都可能重啟主機的 docker daemon——Ceph 若也掛在 docker 下,mon/osd 會被連坐重啟;podman 沒有常駐 daemon,每個 Ceph 守護程序是獨立的 systemd 服務,跟 Kolla 完全解耦。(這個選擇在本章結尾的重開機測試會得到回報。)

## 開始之前

- [x] 一顆**全新的空硬碟**給 Ceph 專用。本課環境:在 Azure 加掛一顆 256G Premium 磁碟,VM 內出現為 `/dev/sdc`:
  ```bash
  # 在你的電腦上執行
  az vm disk attach -g <資源群組> --vm-name <VM名> --name ceph-data --new --size-gb 256 --sku Premium_LRS
  ```
- [x] 建議先拍一張磁碟 snapshot(動儲存屬於高風險操作):`az snapshot create …`
- [x] 確認 `lsblk` 看得到新磁碟且**沒有任何分割**

## 步驟

### 步驟 1:安裝前置與 cephadm

```bash
sudo apt-get install -y podman lvm2 chrony
cd ~
curl --silent --remote-name --location https://download.ceph.com/rpm-20.2.2/el9/noarch/cephadm
chmod +x cephadm
./cephadm version
```

```text
cephadm version 20.2.2 (...) tentacle (stable)
```

版本選擇有講究:**釘 Tentacle 20.2.2**。前一代 Squid 即將停止維護,而 Ubuntu 發行版倉庫裡的 ceph 套件是有已知問題的預覽快照——一律用官方下載的 cephadm(它是不挑發行版的獨立執行檔,`el9` 路徑在 Ubuntu 上照用)。

### 步驟 2:bootstrap(三個旗標都有理由)

```bash
sudo ./cephadm bootstrap \
  --mon-ip 10.0.0.4 \
  --single-host-defaults \
  --skip-monitoring-stack \
  --skip-dashboard
```

- `--single-host-defaults`:把「資料要分散在多台主機」的預設放寬到單機。
- `--skip-monitoring-stack`:**必帶**。Ceph 自帶的 grafana(3000)、node-exporter(9100)、alertmanager(9093)與 Day 21 要部署的 Kolla 監控**完全同 port**——現在跳過,未來才不會對撞。
- `--skip-dashboard`:lab 省資源;要看狀態用 CLI 就夠。

結尾看到 `Bootstrap complete.` 即成功(約 2–3 分鐘,含拉 Ceph 容器映像檔)。

### 步驟 3:單機調校(每一條都在拆一顆雷)

```bash
sudo ./cephadm shell -- bash -c "
ceph config set global mon_allow_pool_size_one true
ceph config set global osd_pool_default_size 1
ceph config set global osd_pool_default_min_size 1
ceph config set mon osd_pool_default_size 1
ceph config set osd osd_memory_target_autotune false
ceph config set osd osd_memory_target 2G
ceph config set mds mds_cache_memory_limit 1G
"
```

前四條:只有一顆硬碟,資料只能存一份(`size=1`)——生產環境絕不這樣,lab 明知故犯並知道代價。
後三條是**本章最重要的一顆雷的預拆**:cephadm 有記憶體自動調整,公式是「總記憶體 × 0.7 ÷ OSD 數」——在 128 GB、單 OSD 的機器上,它會把 OSD 的記憶體目標推到**幾十 GB**,直接跟 OpenStack 搶記憶體。關掉自動調整、鎖 2 GB。

### 步驟 4:加入 OSD

指名硬碟,不用「自動吃掉所有可用磁碟」的懶人選項——這台主機上還有 OpenStack 的磁碟:

```bash
sudo ./cephadm shell -- ceph orch daemon add osd $(hostname -s):/dev/sdc
```

```text
Created osd(s) 0 on host 'openstack-lab'
```

約 30 秒後 `ceph osd stat` 應顯示 `1 osds: 1 up, 1 in`。

### 步驟 5:處理必然出現的告警

```bash
sudo ./cephadm shell -- ceph health detail
```

單機 + 與 OpenStack 共存,會看到兩種告警,都要「知情處理」而不是無視:

1. `POOL_NO_REDUNDANCY`——size=1 的必然產物,知情靜音:
   ```bash
   sudo ./cephadm shell -- ceph health mute POOL_NO_REDUNDANCY --sticky
   ```
2. `CEPHADM_REFRESH_FAILED`——見[地雷 1](#mine-1),同樣知情靜音。

處理完:

```bash
sudo ./cephadm shell -- ceph -s
```

```text
    health: HEALTH_OK
    mon: 1 daemons, quorum openstack-lab
    mgr: openstack-lab.xxxx(active), standbys: openstack-lab.yyyy
    osd: 1 osds: 1 up, 1 in
    usage: 28 MiB used, 256 GiB / 256 GiB avail
```

## 驗收 checkpoint

逐項驗證,**全部符合判準才算完成今天**。「本課環境的結果」欄是我們實測的參考值:

| 驗證 | 判準 | 本課環境的結果 |
|---|---|---|
| `ceph -s` | HEALTH_OK(兩項知情靜音已記錄) | 符合 |
| OSD | 1 up / 1 in,容量等於新磁碟 | 256 GiB 可用 |
| 記憶體 | 部署後增量 ≤ 8 GiB | 實測只多 ~1 GiB(OSD 空載) |
| Kolla 不受影響 | `openstack service list` 正常、容器數不變 | 13 服務 / 53 容器,無感 |
| 守護程序歸屬 | `ceph orch ps` 全部由 systemd+podman 管理 | 符合(與 kolla 的 docker 零耦合) |

## 地雷記錄

### 地雷 1:裝置掃描被 Cinder 的 thin-pool 嗆到 {#mine-1}

**症狀**:`ceph health` 出現 `CEPHADM_REFRESH_FAILED: failed to probe daemons or devices`,詳情裡是一長串 `blkid: error: /dev/cinder-volumes/...pool_tmeta: No such file or directory`。

**根因**:cephadm 定期做全機磁碟盤點(`ceph-volume inventory`),掃到 Day 3 建立的 Cinder thin-pool 的**內部中繼 LV** 時讀不到裝置節點而失敗。這是「OpenStack 與 Ceph 同機共存」特有的碰撞——各自獨立時都不會發生。

**處置**:功能無損——加 OSD 走的是指名路徑(步驟 4),不依賴盤點。知情靜音並記錄:`ceph health mute CEPHADM_REFRESH_FAILED --sticky`。

### 地雷 2:記憶體 autotune 會吃掉幾十 GB {#mine-2}

已在步驟 3 預拆。若忘了關,`osd_memory_target` 會被自動推到「總記憶體 × 0.7 ÷ OSD 數」——單 OSD 的大記憶體主機上是災難。驗證:`ceph config get osd osd_memory_target` 應回 `2147483648`(2G)。

### 地雷 3:告警讀到舊值——health check 有快取 {#mine-3}

**症狀**:明明 `ceph config get mon osd_pool_default_size` 已回 1,`ceph -s` 卻持續顯示 `OSD count 1 < osd_pool_default_size 2`。

**根因**:健康檢查的評估值有快取,改設定後不會立即重算。

**解法**:`ceph mgr fail` 讓 standby mgr 接手,告警立即依新值重評消失。「改了設定但告警沒變?先讓 mgr 換手再說」是張好用的牌。

### 地雷 4:調校與 pool 誕生的時序 {#mine-4}

bootstrap 剛完成時叢集裡**零個 pool**(第一個 pool `.mgr` 要等首顆 OSD 上線才誕生),此時對 `.mgr` pool 的操作會回 `unrecognized pool`。無需補救——只要 `osd_pool_default_size=1` 在 pool 誕生**之前**設好,新 pool 就會以正確參數出生。教訓:**先調 default、再加 OSD**,順序就是一切。

## 彩蛋:重開機的兩種命運

本課環境的 VM 每晚自動關機。隔天開機後實測:**Ceph 全自動復原**——`HEALTH_OK` 秒回、靜音設定都在,零人工介入。對照 Day 6 的 kind(重開機後要人工檢查、換 port 還要重配 kubeconfig),這就是「每個守護程序都是 systemd 服務」的架構紅利,也是 podman 選擇的回報。

## 附錄:整套拆除

Lab 結束或想重來:

```bash
FSID=$(sudo ./cephadm shell -- ceph fsid)
sudo ./cephadm rm-cluster --force --zap-osds --fsid $FSID
```

`--zap-osds` 會把資料碟上的 Ceph 痕跡一併清除。

## 下一步

儲存骨幹立起來了,但現在它跟 OpenStack 還是兩個世界。[Day 15](sprint4-day15-object-storage-rgw.md) 蓋第一座橋:在 Ceph 上開 **RGW 物件儲存**,讓你的雲同時擁有 S3 與 Swift 兩種 API。

---

*Ceph 標誌為 Ceph 專案之官方資產,此處作社群教學用途。*
