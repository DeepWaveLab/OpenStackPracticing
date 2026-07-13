---
title: "feat: Sprint 5 課程規劃——把雲當成生意經營(計量、計費、SLA、治理稽核)+ Day 26 預告章"
type: feat
status: active
date: 2026-07-13
---

# Sprint 5 課程規劃:把雲當成生意經營 + Day 26 預告章

## 目標

規劃課程的下一個 Sprint(**Day 27–35,9 章**),並產出 **Day 26 預告章**(比照 Day 12 的「預告篇」定位)。

主題:**把雲當成生意經營**——Sprint 4 結束時雲「敢動」了;Sprint 5 讓它能**開發票、扛 SLA、過稽核**。這個主題同時回應使用者點名的方向(商業相關/企業內部控制)與四個指定服務(Masakari / Ceilometer / Gnocchi / Aodh)的落地方式。

## 為什麼是這個主題

1. **課程敘事的自然下一步**:計量(誰用了多少)→ 計費(換算成錢)→ SLA(對客戶的承諾)→ 治理稽核(對稽核員的交代)——是雲從「工程品」變「產品」的完整拼圖。
2. **市場時機**:2025 使用者調查生產核數 4,500 萬→5,500 萬,成長主力是中小企業(Broadcom/VMware 授權費效應);企業導入的第一批問題正是分帳、租戶治理與合規。
3. **時效性亮點**:Unified Limits 在 **2025.1 剛從實驗性轉正**(Nova 提供官方遷移工具),是這一版最值得教的新東西之一。

## 專案健康度查證(2026-07-13,Zun 教訓的制度化)

每個候選專案先驗屍再排課。證據為 releases.openstack.org / governance.openstack.org / opendev commits / 下游發行版文件交叉查證。

### 判定:活躍(可排核心天)

| 專案 | 證據摘要 | kolla 2025.1 |
|---|---|---|
| Ceilometer | 26.0.0(2026-04)照週期發版;2026-06 仍有實質 commit | 有 role + 文件 |
| Aodh | 22.0.0(2026-04);**PrometheusEvaluator 已是一等公民**(2023-11 併入);Heat 2024.1 起有 `OS::Aodh::PrometheusAlarm` | 有 role |
| CloudKitty | 22→24.0.0 連三版;collector 官方支援 gnocchi/**prometheus** 二選一;Horizon dashboard 2026-06 仍維護 | 有 role + 專屬 guide |
| Masakari | 19→21.0.0 連三版;kolla 曾 deprecate 後又 **de-deprecate**(CI 恢復);api/engine/hostmonitor/instancemonitor 四容器 | 有 role + 專屬 guide |
| keystonemiddleware(CADF audit) | 13.0.0(2026-05);audit middleware 文件持續修訂 | 內建 |

### 判定:苟延(不部署,講歷史)

| 專案 | 證據摘要 | 課程處置 |
|---|---|---|
| **Gnocchi** | 2017 離開官方治理;最新版 4.7.0(2025-08)距今 11 個月;repo 未封存、仍有零星真 bug 修復;**Red Hat 次世代產品(RHOSO 18)的 Telemetry 堆疊已完全排除它**(只剩 Ceilometer+Prometheus+Aodh);官方 Telemetry 路線圖是 Ceilometer→Prometheus→**Aetos** | **lab 不部署**;Day 28 講歷史脈絡(Aodh 的 gnocchi alarm type 由來);計量儲存走 Prometheus |
| sg-core / STF | 程式碼有微弱心跳,但 Red Hat 官方宣告 STF 不進 RHOSO 18、2027 終止支援 | 一句話帶過 |

### 判定:已死(式微名單,Day 35 版圖課素材)

Monasca(governance 除名、全系 repo 標 RETIRED)、Murano(TC 決議 2024.1 移除)、Sahara、Senlin、Solum(2024-05 封存)、Freezer-DR(官方建議改用 Masakari)。垂死觀察:Vitrage、Venus(2026.1 起列 inactive、PTL 從缺)。

### 澄清(查證後還活著,不得列入式微)

Mistral、Zaqar(仍在官方 41 專案清單、持續發版、2025.2 完成 Eventlet 遷移)、Adjutant(有 PTL 但 **kolla 零支援** → 不排課)。

### 選配與不選

| 專案 | 判定 | 處置 |
|---|---|---|
| Watcher | **復活中但脆弱**:2025.1 轉 Distributed Leadership、新增 Prometheus data source、kolla 有 role 沒文件 | Day 34 選讀 + Day 35「怎麼判斷專案生死」活教材 |
| Blazar | 活著但冷門(HPC 預約情境) | Day 35 一句話帶過 |
| Ironic | 很活躍,但 Azure lab 無裸機可教 | 版圖課註明「值得學、此 lab 教不了」 |

## 課綱(Day 26–35)

> 順序邏輯(採納 spec-flow 分析):**多租戶結構在計費之前**(計費 demo 才有真租戶可分帳)、**TLS 在 OIDC 之前**(redirect URI 直接以 https 定案,免得回頭改)、Masakari 緊接計費後獨立成天(唯一需要多機環境的日子,用完即關)。

| Day | 主題 | 內容與里程碑 | 環境 |
|---|---|---|---|
| **26** | **預告章(本計畫的站上交付物)** | 不動手。Sprint 5 地圖、為什麼是「把雲當生意」、四個指定服務的健康度判決(含 Gnocchi 為什麼繞過)、式微名單預告、成本預告 | — |
| **27** | **多租戶治理:domains、額度與新一代權限** | Keystone domains 與 project 階層、application credentials、**unified limits(2025.1 轉正,`nova-manage limits migrate_to_unified_limits`)**、Secure RBAC 現況(Nova/Neutron/Octavia 已預設 enforce_scope,Cinder 落後——誠實標註「lab 生效」vs「概念討論」) | 單機 |
| **28** | **計量管線:誰用了多少** | Ceilometer(notification + polling)→ `enable_ceilometer_prometheus_pushgateway` → Prometheus;**開場畫清楚兩條平行資料路**(Ceilometer=OpenStack 原生計量 vs Day 21 exporters=基礎設施指標),否則學員會以為 Day 28 是 Day 30 的直接管線;Gnocchi 歷史一節(為什麼課本上都是它、為什麼我們繞過) | 單機 |
| **29** | **告警與自動反應:Aodh** | Aodh `prometheus` alarm type(PromQL 條件)→ alarm 動作;**Heat autoscaling 補課**(Day 5 未教 AutoScalingGroup/ScalingPolicy,先補)→ `OS::Aodh::PrometheusAlarm` 驅動 scaling 全鏈演示 | 單機 |
| **30** | **計費:CloudKitty** | prometheus collector + hashmap rating 規則 → 對 Day 27 的多租戶結構分帳;storage 後端 spike 後定(**優先評估重用 Day 21 的 OpenSearch**,省一顆新資料庫;fallback=influxdb);報表:CLI/API + Horizon cloudkitty-dashboard;**講清楚 Skyline 沒有帳單畫面**(plugin 生態成熟度差異,不是走回頭路) | 單機 |
| **31** | **SLA 與自癒:Masakari** | 多機環境重啟健檢 ritual(6 天沒碰,開頭先環境回顧);instancemonitor 演「VM 程序死亡→自動重啟」(零前提);**evacuate 演練前提要自己補**:多機環境目前沒有 Cinder → 先加 cinder-volume(LVM,集中式 iSCSI)→ boot-from-volume VM 才能安全演 host failure evacuation;hostmonitor(pacemaker)視 spike 結果,降級方案=手動送 host-failure notification(**預先寫在教材裡,不臨場默默降級**);結束即關機 | **多機** |
| **32** | **全站加密:TLS** | `kolla_enable_tls_internal/external`;動手前 **snapshot 必做**;endpoints/openrc 全換的連鎖;教學核心坑:**kolla 內建 CA 不能上生產**;收尾跑全服務回歸 | 單機 |
| **33** | **企業身分接軌:Keystone Federation** | Keystone 當 SP + **Keycloak**(容器,預先備好 realm/client import 檔,課堂時間留給 mapping 而非 IdP 後台)+ OIDC;群組→role mapping;redirect URI 以 Day 32 的 https 端點定案 | 單機 + Keycloak 容器 |
| **34** | **稽核軌跡:CADF** | keystonemiddleware audit middleware → oslo.messaging → **接回 Day 21 集中式 log**(OpenSearch 查稽核事件);「誰在什麼時候對哪個資源做了什麼」的完整證據鏈;選讀:Watcher(prometheus data source 小 demo 或純導讀) | 單機 |
| **35** | **Sprint 5 總結 + 版圖課** | 前後對比;**「怎麼判斷 OpenStack 專案的生死」方法論**(governance 頁、release 節奏、下游發行版動向——用 Zun/Monasca/Gnocchi/Watcher 四個等級當案例);式微名單;所有專案現況陳述**標註「截至 2026-07」**;經 Day 10 程序評估單機 lab 的去留 | — |

## 部署驗證作戰計畫(先驗證後寫教材,沿用 Sprint 4 工法)

通用規則不變:逐日 gate 全過才前進、`~/sprint5-dayN.log`、config tgz、高風險日先 snapshot。

### 開工前 spike(風險等級排序)

1. **CloudKitty × Prometheus 的租戶歸屬**(Day 30 命脈):prometheus collector 的 `scope_key` 需要 metrics 帶 project id label——驗證 Ceilometer pushgateway 出來的 metrics label 結構是否可直接分租戶。**fallback 修正**(spec-flow 抓到的矛盾):不是 gnocchi collector(我們不部署 Gnocchi),而是(a)降級為不分租戶的 flat rating demo 並誠實標註,或(b)臨時部署 Gnocchi 當一次性冷備援並註明其苟延現狀。
2. **多機環境沒有 Cinder**(Day 31 前提):在 ctrl1 加 cinder-volume(LVM VG + iSCSI,集中式)→ 任何 compute 都能接 → evacuate 才成立。需驗 `deploy --limit` 增量上 cinder 的干擾面。
3. **masakari hostmonitor × pacemaker**:kolla 有 hacluster role(`enable_hacluster` 隨 hostmonitor 自動開),但兩節點 quorum(`two_node` / qdevice)是已知邊界情境——spike 過不了就用降級方案(手動 notification 演 evacuate 決策鏈)。
4. **Aodh PrometheusAlarm 全鏈**:2025.1 的 aodh + heat 版本組合下 `OS::Aodh::PrometheusAlarm` 可用性、alarm 評估間隔、與 kolla prometheus 的認證整合(Day 21 有 basic auth)。
5. **unified limits 服務覆蓋度**:2025.1 實際上哪些服務 enforce(Nova 轉正;其他?)——教材要「lab 生效」vs「概念」分欄。
6. **Ceilometer 啟用的漣漪**:各服務 notification 開關 = 全站 reconfigure;pushgateway timestamp 官方自承不精準(**不能當計費的直接資料源**的原因,寫進 Day 28)。
7. **單機資源餘裕**:E16s_v5 再掛 Keycloak + InfluxDB(若選它)+ ceilometer 全家的記憶體預算。
8. **多機重啟檢查單**:公網 IP(Standard SKU=靜態,已確認)、NSG 來源 IP 過期、galera/rabbitmq 開機順序——沿用 Day 31 開頭 ritual。

### 每日 gate 示意(細則執行時定)

- Day 27:unified limits 對 Nova 額度真的擋下超額 create
- Day 28:Prometheus 查得到 ceilometer 來源的 per-project metrics
- Day 29:壓負載 → alarm 觸發 → ASG 長出第二台 VM,全程無人工
- Day 30:兩個租戶各自的帳單金額對得上用量
- Day 31:kill VM 程序 → 自動重啟;模擬 host failure → boot-from-volume VM 在另一台 compute 復活
- Day 32:全 endpoints https、全服務回歸過
- Day 33:Keycloak 使用者 SSO 登入 → 拿到 mapped project/role → 能開 VM
- Day 34:OpenSearch 查到「某使用者刪除某 VM」的 CADF 事件

## Day 26 預告章規格(本計畫最先交付)

- 定位比照 Day 12:預告篇,不動手
- 內容:①「會動→敢動→當生意經營」的敘事橋 ② Day 27–35 課表(一句話版)③ 指定服務健康度判決書(Masakari ✓ / Ceilometer ✓ / Aodh ✓ / **Gnocchi 繞過 + 理由與證據**)④ 官方 Telemetry 路線圖(Ceilometer→Prometheus→Aetos)一張圖 ⑤ 式微名單預告 ⑥ 成本預告 ⑦ 三個設計決定(為何 Prometheus 不是 Gnocchi、為何帳單畫面在 Horizon 不在 Skyline、為何 Masakari 只排一天多機)
- 樣式:沿用站上紀律(標題 `Day 26: `、mermaid 直排、無標語腔、台灣繁體)
- 連動:mkdocs nav、首頁課表加 Day 26 列、Day 25 章末「兩條延伸路」接上第三條(Sprint 5 預告)

## 成本估算

| 項目 | 估算 |
|---|---|
| 單機 lab(7 個動手天 × ~8h × ~US$1.2/hr) | ~US$70 |
| 多機環境(Day 31 一天 × ~8h × ~US$2.6/hr) | ~US$21 |
| Keycloak/InfluxDB | 跑在現有 VM 內,$0 |
| **合計** | **~US$90–100**(不含閒置磁碟月費) |

多機環境在 Day 31 之前保持關機(磁碟費仍在),Day 31 用完即關;是否在 Sprint 5 收尾徹底拆除,列入 kickoff 決策。

## Alternative Approaches Considered

| 方案 | 為什麼不採 |
|---|---|
| 部署 Gnocchi 當計量儲存(傳統教科書路線) | 苟延:11 個月無新版、無官方治理歸屬、RHOSO 18 已排除;官方路線圖已是 Prometheus/Aetos。教它=重蹈 Zun 覆轍 |
| Monasca 做監控計量 | 已死(RETIRED),連 Watcher 都在移除它的 data source |
| Adjutant 做工單自助流程 | 專案活著但 kolla 零支援,部署成本吃掉教學價值 |
| Ironic 裸機(呼聲高的補課題) | Azure lab 沒有裸機;版圖課註明「值得學、此處教不了」 |
| 主題改走「效能調校/大規模」 | 需要更大機隊,成本陡增;且與使用者點名的商業/內控方向不合 |

## Acceptance Criteria

- [ ] Day 26 預告章上線(規格如上),nav/首頁/Day 25 連動完成
- [ ] 課綱 9 章每章有明確里程碑與 gate 方向
- [ ] 四個指定服務(Masakari/Ceilometer/Gnocchi/Aodh)全數有交代(教或繞過,附證據)
- [ ] 8 條 spike 在對應課程日前完成並記錄結論
- [ ] 式微名單只放有 RETIRED 級證據的專案;活專案(Mistral/Zaqar)不得誤傷
- [ ] 所有專案現況陳述標註查證日期(截至 2026-07)

## 開工前決策(kickoff)

1. **CloudKitty storage 後端**:OpenSearch(重用 Day 21,省資源)vs InfluxDB(kolla 預設)——spike 後拍板
2. **多機環境去留**:Day 31 用完即拆(重建有 runbook)vs 保留到 Sprint 5 結束
3. **Day 26 何時上線**:比照 Day 12 經驗,建議規劃定案後先上
4. Sprint 5 開跑時間(影響單機 lab 重啟節奏)

## Sources

### 內部
- Sprint 4 計畫與工法:`docs/plans/2026-07-11-feat-sprint4-advanced-course-plan.md`
- 預告章前例:`docs/runbook/sprint3-day12-sprint4-preview.md`
- kolla-ansible stable/2025.1 原始碼實查(roles 存在性、enable 旗標、cloudkitty collector 白名單、masakari 四容器、hacluster 連動、deprecation notes)

### 外部(2026-07-13 由三個研究 agent 交叉查證,關鍵證據)
- Telemetry 治理與路線:governance.openstack.org/tc/reference/projects/telemetry.html;Aetos:github.com/openstack/aetos;Watcher 的 Prometheus→Aetos 遷移文件(2026.1)
- Gnocchi 現況:github.com/gnocchixyz/gnocchi(pushed 2026-05;4.7.0=2025-08);unmaintained 危機始末 issue #1049;RHOSO 18 Observability 文件(無 Gnocchi)
- CloudKitty:releases.openstack.org/teams/cloudkitty.html;collector/storage 官方文件;cloudkitty-dashboard commits(2026-06)
- Masakari:releases.openstack.org/teams/masakari.html;masakari-monitors hostmonitor 文件;kolla masakari-guide(2025.1)
- Watcher 復活:docs.openstack.org/releasenotes/watcher/2025.1.html(Distributed Leadership 原文)
- 治理主題:Nova 2025.1 release notes(unified limits 轉正);TC secure-RBAC goal 頁;keystone federation 文件 + kolla IdP 設定文件;kolla TLS 文件(2025.1)
- 版圖:TC emerging/inactive 頁(Vitrage/Venus);Murano 移除決議(2024.1);OpenInfra 2025-12 電子報;2025 使用者調查新聞稿(5,500 萬核)
