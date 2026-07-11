---
title: "feat: Sprint 4 課程規劃 —— 服務擴充、內部原理、維運實務、水平擴展(+ Day 12 預告頁)"
type: feat
status: active
date: 2026-07-11
deepened: 2026-07-11(部署驗證作戰計畫;上游旗標與 cephadm 全查證)
---

# feat: Sprint 4 課程規劃(Day 13–24)+ Day 12 預告頁

## Overview

Sprint 3 讓一朵雲從零長出來;**Sprint 4 讓它變成一朵「像生產環境」的雲**。四個階段、12 天,延續 Day 編號(Day 13–24)與既有教學格式:

| 階段 | 主題 | 天數 | 回答的問題 |
|---|---|---|---|
| 一 | **服務擴充**(儀表板 + 儲存三部曲 + DNS + DBaaS) | 6 | 首頁「還沒拼上的積木」怎麼拼上 |
| 二 | **內部原理**(深層拆解) | 2 | 不靠 OpenStack CLI,你能不能讀懂這朵雲? |
| 三 | **維運實務**(方法論與工具) | 2 | 監控什麼、怎麼升版、怎麼備份 |
| 四 | **水平擴展**(多機部署) | 2 | **Nova/Magnum 怎麼「準備好機器讓使用者需要時去長」** |

另交付 **Day 12 頁面**:放在現有課程尾端的「下一步的地圖」,向讀者預告 Sprint 4 的完整課綱(本計畫的讀者版)。

## Problem Statement / Motivation

- Sprint 3 收在「會用」;學員問的下一層問題是「**怎麼養**」(維運)和「**怎麼長**」(擴展)——這正是 OpenStack 工程師與「跑過 lab 的人」的分界線。
- 首頁的「還沒拼上的積木」表列了 9 個未收錄服務,其中 Swift/Manila/Designate/Zun/Trove 五個是使用者指定的優先項。
- 橫向擴展是單機 lab 教不了的唯一主題——需要一次誠實的多機部署。

## 課綱設計(逐日)

### 階段一:服務擴充(Day 13–18)

> 設計主軸:**用 Ceph 當儲存骨幹**,一次解鎖物件儲存與共享檔案系統;每個服務照舊格式(定位框 → 第一次接觸 → 原理 → 步驟含實測輸出 → 驗收 checkpoint → 地雷記錄)。

| Day | 主題 | 內容與里程碑 | 已知風險/地雷預告 |
|---|---|---|---|
| **13** | **Skyline 儀表板** | `enable_skyline` 增量部署(2026-07-11 已在本 lab 實測一次過)、與 Horizon 並存對照、UI 導覽;當 Sprint 4 的暖身日 | 零已知風險——唯一「已實測通過」的一天 |
| **14** | **Ceph 基礎** | cephadm 單機 bootstrap、OSD/pool/RBD 心智模型、為什麼生產 OpenStack 的儲存標配是 Ceph(而不是 LVM/Swift) | 需要**新掛一顆 data disk**(現有 256G 已被 cinder-volumes VG 用掉);cephadm 單機要壓 replica=1 | 
| **15** | **物件儲存:RGW(Swift 的現代答案)** | Kolla 的 external Ceph 整合、RGW + Keystone 認證、**同一個服務同時講 S3 與 Swift 兩種 API** 並實測、交代「為什麼 Kolla 放棄原生 Swift」(引用部署工具圖鑑的實測矩陣) | Kolla 自 Rocky 起不部署 Ceph 本體,只做整合——這是教學點不是限制 |
| **16** | **Manila 共享檔案系統** | CephFS 後端、建 share、兩台 VM 同掛一顆、壓軸:**K8s RWX PVC**(Manila CSI 接回 Day 7-8 的 Magnum cluster——Sprint 3 的 Cinder RWO 補上另一半) | manila 與 CephFS NFS/native 模式選擇;Magnum driver 內建 manila 支援(Day 11 已考證) |
| **17** | **Designate DNS** | bind9 後端、zone/recordset 生命週期、**Neutron 整合**(port `dns_domain`、FIP 自動 A/PTR),幫 Day 4 的 LB 掛上域名收尾 | Neutron extension 設定容易漏;拿舊資源(LB)來整合是刻意設計 |
| **18** | **Trove DBaaS** | guest image 準備、**management network 佈線(本日教學重心)**、MySQL instance 生命週期、備份/還原 | **已查證(2026-07-11):image 斷代是假警報**——官方 guest image 每日發佈(查證當天的 noble qcow2)、datastore 容器在 quay.io 每日更新、上游活躍有 PTL。真雷是 **guest agent → RabbitMQ 的管理網路打通**(Kolla 不代勞,要自建 provider network + `management_networks` 覆寫);次要:官方只發 master 分支 image,備案 `TROVE_BRANCH=stable/2025.1` 自建(~30 分) |

### 階段二:內部原理(Day 19–20)

> 設計主軸:「融會貫通」不是多學新服務,是**把用過的東西拆開看**。兩天都以「不用 OpenStack CLI,改用底層工具直接操作/觀察」為驗收。

| Day | 主題 | 內容與里程碑 |
|---|---|---|
| **19** | **API 請求的完整生命週期** | 從 `server create` 追到 qemu 進程:fernet token 解剖(真的解開來看)、**request-id 跨服務追蹤**(一條 ID 串起 api→scheduler→compute 的 log)、oslo.messaging RPC 實況(`rabbitmqctl` 看佇列在動)、nova DB 哪幾張表被寫、`virsh dumpxml` / qemu 進程參數對讀。驗收:給一個 request-id,能徒手重建整條呼叫鏈 |
| **20** | **OVN 封包轉送剖析(OVN 深潛)** | Day 2 的網路魔法全部拆穿:`ovn-nbctl`/`sbctl` 讀邏輯拓樸、logical flow → OpenFlow(`ovs-ofctl dump-flows`)、**`ovn-trace` 模擬追蹤一個封包**、FIP 的 NAT 到底在哪一條 flow、安全群組怎麼變成 flow。驗收:用 ovn-trace 解釋「為什麼這個封包不通」 |

### 階段三:維運實務(Day 21–22)

| Day | 主題 | 內容與里程碑 |
|---|---|---|
| **21** | **可觀測性** | `enable_prometheus` + grafana + central logging(opensearch,Kolla 全內建);**該盯什麼**:API 延遲、agent 存活、佇列深度、磁碟水位、amphora 健康;實戰:用集中 log + request-id 重演 Day 8 的 CCM 除錯——體感「有觀測 vs 沒觀測」差多少 |
| **22** | **升版、備份與 DB 維運** | OpenStack 發布節奏與 **SLURP 升級政策**(哪些版本可以跳);`kolla-ansible upgrade` 實戰(2025.1 內 minor bump,行有餘力升 2025.2);`mariadb_backup` 與還原演練;DB 長期維護(`nova-manage db archive_deleted_rows`);維運方法論總結:變更前 checklist、回滾預案、Day 10 teardown 文件的實戰化 |

### 階段四:水平擴展(Day 23–24)——回答本 Sprint 的核心問題

> 這個階段換環境:**租 3–4 台新 VM 做真正的多機部署**,教完拆掉。這也是 Day 10「重建程序」文件第一次真正派上用場(課程呼應)。

| Day | 主題 | 內容與里程碑 |
|---|---|---|
| **23** | **多節點部署:單機設定的每一行,現在都有了意義** | multinode inventory(control/network/compute/storage 群組)、**haproxy + keepalived VIP 復活**(Sprint 3 單機時關掉的東西,現在懂為什麼存在)、部署 2 control + 2 compute、**控制面失效演練**:關掉一台 controller,API 照常回應——親眼看 HA |
| **24** | **橫向擴展實戰:雲怎麼「長」** | **正面回答「Nova/Magnum 怎麼準備好機器」**:(1) 新 compute 節點加入的完整流程:`bootstrap-servers --limit` → `deploy --limit` → `nova-manage cell_v2 discover_hosts` → placement 自動把新資源納入調度——**使用者無感,雲就是變大了**;(2) `live migration` 與 node drain(汰換機器的標準動作);(3) 容量規劃方法論:allocation ratio 生產值、host 預留、N+1 headroom、host aggregate/AZ 做資源池;(4) **Magnum 視角**:workload cluster autoscaler(Day 9)要能長,前提是雲層有容量——操作 placement API 看容量水位,對照 autoscaler 觸發 |

## Day 12 頁面(本計畫的站上交付物)

在課程主線尾端新增 **「Day 12 · 下一步的地圖:Sprint 4 預告」**:
- 定位:如 Day 11 是「後日談」,Day 12 是「預告篇」——不動手,講清楚下一階段學什麼、為什麼是這四個階段
- 內容:四個階段課程地圖(吉祥物版)、逐日一句話課表、「為什麼 Swift 變成 RGW」的說明(連到部署工具圖鑑)、多機環境的成本預告
- 首頁課表補一列 Day 12;「還沒拼上的積木」表加註「Sprint 4 預定」標記

## Technical Considerations

### Lab 環境
- **Day 13–22 沿用現有單機**(E16s_v5):需加掛一顆 data disk 給 Ceph(P15 256G,約 $44/月,用完可卸)
- **Day 23–24 多機環境**(新 RG,教完即拆):

| 拓樸選項 | 構成 | 每小時 | 2 天(20h)估算 | 能教什麼 |
|---|---|---|---|---|
| **A. 4 台(推薦)** | 2× control(E8s_v5)+ 2× compute(E8s_v5) | ~$2.43 | **~$49** | VIP failover ✓、live migration ✓、加節點 ✓ |
| B. 3 台(省) | 1× control+network + 2× compute | ~$1.82 | ~$36 | live migration ✓、加節點 ✓,**HA 只能講不能演** |

- Azure VNet 限制(無 L2 廣播)對多機的影響:Geneve 隧道(東西向)走 VNet 沒問題;provider network 限制與單機相同——課程照樣成立,並且是「公有雲上跑 IaaS」的誠實教學點

### 課程工法(沿用 Sprint 3 驗證過的)
- 每日 runbook 固定格式 + 驗收 checkpoint(判準/參考值)+ 具名地雷錨點
- mermaid 直排紀律(每列 ≤4 格)
- 高風險日的 spike 前置:Day 17/18 已於 2026-07-11 提前完成查證(見風險表),其餘日開工前照舊驗貨
- 版本一律釘 stable 分支,不追 latest

## Alternative Approaches Considered

| 方案 | 為什麼不採 |
|---|---|
| **原生 Swift 手工部署** | Kolla 2025.1 無 swift role(2026-07-10 上游實證);手工裝違背「部署工具化」的課程哲學。RGW 提供 Swift API,學員仍學到 Swift 介面,且多拿到 S3 與 Ceph——教學價值更高 |
| 多機用巢狀 VM 模擬 | 省錢但三層巢狀(VM 裡的 VM 裡的 VM)慢到失真,而且教不了真實的跨機網路——多機的意義就在「真的有網路隔著」 |
| 12 天全塞服務(不做多機) | 橫向擴展是使用者點名的核心問題,也是本 sprint 相對 Sprint 3 的最大增量,不可割 |
| 升版直上 2026.1(SLURP→SLURP) | 跨度大、風險高;先 minor 再視情況 2025.2,把 SLURP 當知識點教而非賭注 |

## Acceptance Criteria

- [ ] 12 天課綱四個階段完整,每日有明確里程碑與驗收方向
- [ ] 使用者指定的五服務全數涵蓋(Swift 以 RGW 形式,計畫內講明理由)
- [ ] 「Nova/Magnum 怎麼準備機器」在 Day 24 有正面、可操作的答案
- [ ] Day 12 預告頁上線(課程主線 + 首頁課表 + 積木表加註)
- [ ] 多機環境成本與拓樸選項給出明確數字,kickoff 可拍板
- [ ] 高風險日(Trove)有 spike 閘門與 fallback 設計
- [ ] **部署驗證階段**:Day 13–24 逐日 gate 全過(見作戰計畫),教材僅在驗證通過後撰寫

## Dependencies & Risks

兩大高風險日已於 2026-07-11 預先查證完畢(向 CI/Zuul、tarballs、quay.io、Launchpad、Gerrit 一手驗證),原本的兩個假設**都被推翻**,真雷另有其人:

| 風險 | 查證結果與對策 |
|---|---|
| ~~Trove guest image 斷代~~ → **假警報** | 官方 image 管線每日出貨(tarballs noble qcow2、quay.io datastore images 皆為查證當日更新)、上游活躍。**真雷:management network 佈線**(guest agent 連不回 RabbitMQ 5672 是 kolla+trove lab 卡最久的一關)→ Day 18 課程重心改放這裡;版本偏差備案:`TROVE_BRANCH=stable/2025.1` 自建 image |
| ~~Zun~~ → **決策:移除(2026-07-11)** | 查證結論:上游一年僅 14 個 housekeeping commit、2026.1 出生即壞無人修、kolla master 已整包移除、官方 User Survey 連提都不提(對照 Magnum 21% 生產採用)。與使用者確認後**從課綱移除**,Day 13 改為 Skyline(已實測);Zun 的退場故事保留在 Day 11 的服務表註記,當「業界怎麼淘汰技術」的一句話教材 |
| Ceph 單機資源壓力(E16s 上再跑 Ceph + 全套 OpenStack) | replica=1、只開必要 daemon;監控記憶體水位,必要時 Day 13 起關閉 Zun/Trove 等測完的服務 |
| 2025.2 upgrade 路徑未實測 | 列為 Day 22 的選做;minor upgrade 是必做保底 |
| 多機部署的未知雷(全新領域) | 這是課程價值不是風險——Sprint 3 證明了「每個地雷都能定位、能修」的路線韌性 |


## 部署驗證作戰計畫(教材前置,/ce:work 的執行腳本)

> **本節目的**:在寫任何教材之前,Day 13–24 逐日真實部署驗證——一天一關、gate 全過才前進,確保課綱不會寫出部不動的東西。教材等驗證全數通過後再補。
> 所有旗標與指令均已於 2026-07-11 向上游查證(kolla-ansible stable/2025.1 原始碼逐行比對 + Ceph 官方文件),不是推測。

### 基線(2026-07-11 實測)

- lab VM:RAM 25Gi / 125Gi 使用中(Kolla 全套 + Skyline + kind + 3 台 K8s node 全開,53 容器)——**剩 100Gi,全 Sprint 不需中途關服務**
- 磁碟:sda 128G(OS)、sdb 256G(cinder LVM 已佔)→ **Ceph 需新掛 sdc**
- Skyline 已部署並實測(= Day 13 gate 已過)

### 通用規則(每一天都適用)

1. **開工**:`kolla-genpwd` 到暫存檔 + `kolla-mergepwd` 補齊新服務密碼條目 → `kolla-ansible prechecks`
2. **Gate**:當日驗證全過才進下一天;失敗就修或標記阻塞,不帶傷前進
3. **收工**:`tar czf ~/kolla-config-day{N}.tgz /etc/kolla`(設定快照)+ 把當日完整指令與輸出記進 `~/sprint4-day{N}.log`(教材素材)
4. **高風險日前拍 Azure disk snapshot**:Day 14(Ceph 動磁碟)、Day 22(升版)前必拍
5. 記憶體預算:基線 25Gi + Ceph 6–8Gi + 監控 3–4Gi + Trove guest ~4Gi ≈ 峰值 40Gi / 125Gi ✅

### 逐日驗證規格

#### Day 13 · Skyline ✅ 已完成(2026-07-11)
`enable_skyline` → `deploy --tags skyline`,一次過;9998/9999 healthy、UI 實測正常。**待辦尾巴**:Day 21 開 Prometheus 後要連 skyline 一起 reconfigure(skyline.yaml.j2 會自動接 prometheus,監控頁才會亮)。

#### Day 14 · Ceph 基礎(cephadm 單機)
- **前置**:`az vm disk attach`(P15 256G → sdc);Azure snapshot;`apt install podman lvm2 chrony`
- **版本釘死 Tentacle 20.2.2**(Squid 兩個月後 EOL;Ubuntu 發行版套件是問題快照,不用)。curl 官方 cephadm binary
- **容器引擎必用 podman**:kolla 每次 deploy 會重啟 docker daemon,Ceph 掛 docker 下會連坐全滅;podman daemonless、各 daemon 是獨立 systemd unit
- **bootstrap**:`cephadm bootstrap --mon-ip 10.0.0.4 --single-host-defaults --skip-monitoring-stack --skip-dashboard`
  - `--skip-monitoring-stack` **必帶**:ceph 的 grafana(3000)/node-exporter(9100)/alertmanager(9093)與 Day 21 的 kolla 監控完全撞 port
- **單碟 size=1 設定**:`mon_allow_pool_size_one true`、`osd_pool_default_size 1`、`min_size 1`、`.mgr pool size 1`、`ceph health mute POOL_NO_REDUNDANCY`
- **記憶體地雷(必拆)**:cephadm autotune 會把 osd_memory_target 設成 0.7×128G÷1 OSD=幾十 GB → `ceph config set osd osd_memory_target_autotune false` + `osd_memory_target 2G` + `mds_cache_memory_limit 1G`
- OSD:`ceph orch daemon add osd $(hostname -s):/dev/sdc`(不用 --all-available-devices,同機還有 OpenStack 的碟)
- **Gate**:`ceph -s` HEALTH_OK(mute 後)、osd up 1/1、RAM 增量 ≤8Gi
- **Teardown 備援**:`cephadm rm-cluster --force --zap-osds --fsid <fsid>`

#### Day 15 · 物件儲存 RGW
- **架構確認**:kolla 的 ceph-rgw role **不部署容器**(原始碼註解明言),只做 Keystone 註冊 + 選配 LB → RGW daemon 由 cephadm 跑
- Ceph 端:`ceph orch apply rgw kolla --placement=1 --port=7480`(**預設 port 80 會撞 Horizon,必改**);`ceph config set client.rgw rgw_keystone_*` 整批(url/api_version 3/admin_user ceph_rgw/密碼取自 passwords.yml/accepted_roles/implicit_tenants/`rgw_enable_apis 's3,swift,swift_auth,admin'`)
- Kolla 端 globals:`enable_ceph_rgw: true`、**`enable_ceph_rgw_loadbalancer: "no"`(不設會因 haproxy 停用 precheck 直接 fail)**、`ceph_rgw_port: 7480`、`ceph_rgw_internal_fqdn/external_fqdn` 指向本機(haproxy off 時預設指到沒人聽的 6780)、**`update_keystone_service_user_passwords: false`**(否則每次 reconfigure 重設密碼、token 失效)
- 密碼:merge `ceph_rgw_keystone_password`;`deploy --tags ceph-rgw`
- **Gate**:`openstack endpoint list --service object-store` 有 `/swift/v1`;`openstack container create t1` 成功;S3 API 用 s3cmd 或 aws cli 對 7480 驗一筆

#### Day 16 · Manila(CephFS native)
- Ceph 端:`ceph fs volume create manila_fs`(單機 MDS 記得 `standby_count_wanted 0`);client key **只需** `mon 'allow r' mgr 'allow rw'`(Wallaby+ caps)
- Kolla 檔案(**照原始碼不照文件——文件有 bug**):keyring 放 `/etc/kolla/config/manila/ceph.client.manila.keyring`(不是文件寫的 manila-share/ 子目錄);`/etc/kolla/config/manila/ceph.conf` 用 `ceph config generate-minimal-conf` 產生後**把行首 tab 刪掉**(kolla 的 ini parser 會壞)
- globals:`enable_manila: "yes"`、`enable_manila_backend_cephfs_native: "yes"`、`manila_cephfs_filesystem_name: manila_fs`;注意 `ceph_manila_keyring` 這類舊變數 2025.1 已移除,檔名由 `ceph_cluster`+`ceph_manila_user` 推導
- share type 是 DHSS=False:`openstack share type create default_share_type False`
- **Gate**:`share service list` 的 manila-share up;建 share → export location 可取;兩台 VM 同掛讀寫(K8s RWX 留給教材日示範,驗證日做到 VM 層即可)

#### Day 17 · Designate(bind9)
- globals:`enable_designate: "yes"`(bind9 是預設 backend,kolla 自帶容器);`designate_ns_record` 是 **list**;`neutron_dns_domain` **必須以 `.` 結尾且 ≠ openstacklocal**(最常見翻車點)
- Neutron 整合**全自動**(`neutron_dns_integration` 預設=enable_designate;extension driver、[designate] auth section 都由模板代勞)——但要 **`deploy --tags designate,neutron,nova`** 讓跨服務設定生效
- 單節點不需 coordination(多 worker 才要 valkey);port 53/5354/953 佔用先 prechecks
- **Gate**:建 zone → `openstack network set --dns-domain` → 開 VM → `recordset list` 出現 A 記錄 → `dig -p 5354 @<dns_interface_ip>` 解得到

#### Day 18 · Trove(重心:management network)
- globals:`enable_trove: "yes"`;guest-agent 覆寫放 `/etc/kolla/config/trove/trove-guestagent.conf`(原始碼確認的合法路徑)
- **主戰場——guest → RabbitMQ 打通**:guest-agent 的 `transport_url` 直指內部 RabbitMQ(10.0.0.4:5672)。本 lab 方案:management network 用 ext-net(172.24.4.0/24,host 內可路由到 10.0.0.4)→ `/etc/kolla/config/trove.conf` 覆寫 `management_networks = <ext-net id>` + `management_security_groups`(放行 egress 5672);不通再評估獨立 provider 網
- image:Plan A 下載 `trove-master-guest-ubuntu-noble.qcow2`(每日出貨,1.4G)+ `openstack datastore version create ... --image-tags trove,mysql`;RPC 版本偏差就切 Plan B:`TROVE_BRANCH=stable/2025.1 ./trovestack build-image ubuntu jammy false ubuntu`
- datastore 容器可覆寫 `docker_image = quay.io/openstack.trove/mysql` 避開 Docker Hub 限流;**備份存 object-store = 吃 Day 15 的 RGW swift endpoint**(依賴鏈剛好)
- **Gate**:`database instance create` → ACTIVE → 連進 MySQL 寫一筆 → `database backup create` 成功落到 RGW
- **本日是全 Sprint 最可能吃掉兩天的一關**;卡死 fallback:記錄卡點、降級為「部署 + 已知限制」教材

#### Day 19–20 · 原理雙日(無部署)
驗證 = 演練腳本能跑:fernet token 解碼、request-id 跨 log 追蹤(`grep req-xxx /var/log/kolla/*/*.log`)、`rabbitmqctl list_queues` 看 RPC、`virsh dumpxml`;OVN 側 `ovn-nbctl show`/`ovn-sbctl lflow-list`/`ovn-trace`(容器內工具已在)。**Gate**:兩天各自的示範腳本從頭到尾跑通並存檔

#### Day 21 · 可觀測性
- globals:`enable_prometheus`、`enable_grafana`、`enable_central_logging`(自動連動 opensearch + dashboards);**用 stable/2025.1 最新版部署**(20.4.0 修了 dashboards logrotate 與 retention timeout)
- 記憶體:opensearch heap 預設 1g(RSS ~2G),整套 +3–4Gi;prometheus 是 **9091 不是 9090**、有 basic auth(admin/`prometheus_password`)
- **連動**:同日把 skyline 一起 reconfigure(接 prometheus);ceph 端可設 `prometheus_ceph_mgr_exporter_endpoints` 把 ceph mgr 指標拉進來(mgr prometheus module port 9283)
- **Gate**:`/api/v1/targets` 全 up;9200 cluster health green(單節點 yellow 可接受,記錄);grafana 3000 / dashboards 5601 登得進;fluentd 有 log 進 opensearch

#### Day 22 · 升版與備份(前拍 snapshot)
- **系列內更新的正確姿勢**(operating-kolla 原文):pip 升級 kolla-ansible@stable/2025.1 → `install-deps` → `pull`(同 tag 抓新 digest)→ prechecks → **`deploy`(不是 `upgrade`——upgrade 是跨系列用)**
- **已知變更**:20.4.0 起 MariaDB `innodb_log_file_size` 預設 96MB→2GB(磁碟與 recovery 時間);`--limit` 不要用在更新
- 備份:`enable_mariabackup: "yes"` → `reconfigure -t mariadb` → **`kolla-ansible mariadb-backup`(CLI 是 dash,文件內文的底線寫法是殘留錯誤)**;落點 docker volume `mariadb_backup`
- **Gate**:更新後全容器 healthy + `openstack service list` 正常;備份檔存在;**還原演練**(臨時容器 mbstream→prepare→stop mariadb→copy-back→start)走一遍成功
- 2025.2 跨系列升級:時間允許才做,做前再拍一次 snapshot

#### Day 23 · 多節點部署(新環境)
- 依 kickoff 拓樸(建議 A:2 control + 2 compute,E8s×4);新 RG、同 VNet subnet、hostnames、SSH key 佈署
- globals 關鍵差異:`kolla_internal_vip_address` 改成**網段上未使用的 IP**(keepalived 漂移);**移除** `enable_haproxy: "no"` 覆寫;inventory 用官方 multinode sample 起手(**務必 diff 新版 sample**,群組是 control/network/compute/monitoring/storage + :children 映射)
- **Gate**:VIP 在、關掉一台 controller API 仍可用(HA 演練)、開一台 VM 落在指定 compute
#### Day 24 · 橫向擴展
- 加節點(官方文件逐字驗證過):`bootstrap-servers --limit <new>` → `pull --limit` → `deploy --limit`;**cell discover 自動**(nova-cell role 的 discover_computes 會代跑,不用手動 nova-manage)
- **Gate**:`openstack compute service list` 新節點 up;`hypervisor list` 可見;live migration 成功;node drain(disable service + migrate)演練;placement API 看到容量變化
- 收工:整套多機環境 teardown(Day 10 程序實戰)

### 依賴與順序鎖

- Day 15、16 依賴 Day 14(Ceph);Day 18 的備份依賴 Day 15(RGW)
- Day 21 必須在 Day 22 之前(升版過程要有觀測)
- Day 23–24 獨立新環境,與單機 lab 無耦合,可視 VM 交期並行準備

## 開工前決策(kickoff)

1. **多機拓樸**:A(4 台,能演 HA failover,+$13)或 B(3 台)?——建議 A
2. **Sprint 4 何時開跑**(影響 lab 重啟與 Ceph disk 掛載時點)
3. Day 編號延續(Day 13–24,建議)或另起 S4-Day 1–12?
4. Day 12 預告頁要在 Sprint 4 開跑前先上,或跟著 Sprint 4 一起?——建議先上(它本身就是給讀者的預告)

## Sources & References

### Internal
- 服務支援實證:`docs/deployment-tools.md`(2026-07-10 上游查證:Kolla 2025.1 無 Swift、有 manila/designate/zun/trove role)
- 未收錄服務定位:`docs/runbook/sprint3-day11-openstack-service-map.md`(Manila-Magnum 整合、各服務 AWS 對應)
- 重建程序(Day 23 直接引用):`docs/runbook/sprint3-day10-terraform-teardown.md`
- 課程格式標準:Sprint 3 全部 runbook + 本 repo 既有教學寫作紀律
- Sprint 3 課程計畫(格式前例):`docs/plans/2026-07-07-feat-sprint3-kolla-magnum-capi-azure-lab-course-plan.md`

### External(執行期查證,不預先信任)
- Kolla-Ansible external Ceph 整合、multinode 部署、upgrade 指南(docs.openstack.org,執行當日對 stable/2025.1 查證)
- cephadm 單機部署;Trove guest image 現況(Day 18 spike 時查)
- OpenStack SLURP release 政策(releases.openstack.org)
