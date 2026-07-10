# Day 12 · 下一步的地圖:Sprint 4 預告

> 正課(Day 0–10)教你把雲蓋出來,後日談(Day 11)帶你看全景。這一篇望向下一段旅程:**Sprint 4 已經排入規劃**——讓這朵雲從「會用」變成「像生產環境」。本頁是課綱預告,實際內容會隨開課時的上游現況微調。

!!! abstract "你在課程的哪裡"
    - **Day 0–10**:動手蓋雲,11 個服務上線,K8s-as-a-Service 走通。
    - **Day 11**:不動手,盤點整個 OpenStack 版圖——哪些用過、哪些還沒。
    - **Day 12(本篇)**:下一段的地圖。Day 11 那些「還沒拼上的積木」,加上單機 lab 教不了的東西,組成 Sprint 4。

## 從「能用」到「能維運、能擴展」

Sprint 3 結束時,你擁有一朵能開 VM、能長 K8s 的雲。但把時間軸拉長,真實世界的 OpenStack 工程師每天面對的是另外三類問題:

1. **這朵雲還缺什麼能力?**——物件儲存、共享檔案系統、DNS、DBaaS……
2. **怎麼長期維運?**——監控、升版、備份、資料庫維護。
3. **怎麼隨需求擴容?**——加節點、控制面高可用、容量規劃。

Sprint 4 用 **12 天、四個階段**回答這三類問題:

<div style="border:1px solid var(--md-default-fg-color--lightest); border-radius:10px; padding:14px 18px; background:var(--md-code-bg-color);">
  <strong>階段一 · 服務擴充</strong>&ensp;<span style="color:var(--md-default-fg-color--light); font-size:.85em;">Day 13–18 · 六個新服務上線</span>
  <div style="display:flex; flex-wrap:wrap; gap:10px; margin-top:12px;">
    <div style="display:flex; align-items:center; gap:7px; border:1px solid var(--md-default-fg-color--lightest); border-radius:20px; padding:5px 14px 5px 8px; background:var(--md-default-bg-color);"><span style="font-size:22px; line-height:1;">🐙</span><span><b>13</b>&nbsp;Ceph 基礎</span></div>
    <div style="display:flex; align-items:center; gap:7px; border:1px solid var(--md-default-fg-color--lightest); border-radius:20px; padding:5px 14px 5px 8px; background:var(--md-default-bg-color);"><img src="../../assets/mascots/swift.png" width="28" style="display:block;"><span><b>14</b>&nbsp;物件儲存 RGW</span></div>
    <div style="display:flex; align-items:center; gap:7px; border:1px solid var(--md-default-fg-color--lightest); border-radius:20px; padding:5px 14px 5px 8px; background:var(--md-default-bg-color);"><img src="../../assets/mascots/manila.png" width="28" style="display:block;"><span><b>15</b>&nbsp;Manila 共享檔案</span></div>
    <div style="display:flex; align-items:center; gap:7px; border:1px solid var(--md-default-fg-color--lightest); border-radius:20px; padding:5px 14px 5px 8px; background:var(--md-default-bg-color);"><img src="../../assets/mascots/designate.png" width="28" style="display:block;"><span><b>16</b>&nbsp;Designate DNS</span></div>
    <div style="display:flex; align-items:center; gap:7px; border:1px solid var(--md-default-fg-color--lightest); border-radius:20px; padding:5px 14px 5px 8px; background:var(--md-default-bg-color);"><img src="../../assets/mascots/zun.png" width="28" style="display:block;"><span><b>17</b>&nbsp;Zun 容器</span></div>
    <div style="display:flex; align-items:center; gap:7px; border:1px solid var(--md-default-fg-color--lightest); border-radius:20px; padding:5px 14px 5px 8px; background:var(--md-default-bg-color);"><img src="../../assets/mascots/trove.png" width="28" style="display:block;"><span><b>18</b>&nbsp;Trove 資料庫</span></div>
  </div>
</div>
<div style="text-align:center; color:var(--md-default-fg-color--light); line-height:1.6; font-size:1.1em;">▼</div>
<div style="border:1px solid var(--md-default-fg-color--lightest); border-radius:10px; padding:14px 18px; background:var(--md-code-bg-color);">
  <strong>階段二 · 內部原理</strong>&ensp;<span style="color:var(--md-default-fg-color--light); font-size:.85em;">Day 19–20 · 拆開引擎看內部</span>
  <div style="display:flex; flex-wrap:wrap; gap:10px; margin-top:12px;">
    <div style="display:flex; align-items:center; gap:7px; border:1px solid var(--md-default-fg-color--lightest); border-radius:20px; padding:5px 14px 5px 8px; background:var(--md-default-bg-color);"><span style="font-size:22px; line-height:1;">🔬</span><span><b>19</b>&nbsp;API 請求生命週期</span></div>
    <div style="display:flex; align-items:center; gap:7px; border:1px solid var(--md-default-fg-color--lightest); border-radius:20px; padding:5px 14px 5px 8px; background:var(--md-default-bg-color);"><img src="../../assets/mascots/neutron.png" width="28" style="display:block;"><span><b>20</b>&nbsp;OVN 封包剖析</span></div>
  </div>
</div>
<div style="text-align:center; color:var(--md-default-fg-color--light); line-height:1.6; font-size:1.1em;">▼</div>
<div style="border:1px solid var(--md-default-fg-color--lightest); border-radius:10px; padding:14px 18px; background:var(--md-code-bg-color);">
  <strong>階段三 · 維運實務</strong>&ensp;<span style="color:var(--md-default-fg-color--light); font-size:.85em;">Day 21–22 · 監控、升版、備份</span>
  <div style="display:flex; flex-wrap:wrap; gap:10px; margin-top:12px;">
    <div style="display:flex; align-items:center; gap:7px; border:1px solid var(--md-default-fg-color--lightest); border-radius:20px; padding:5px 14px 5px 8px; background:var(--md-default-bg-color);"><img src="../../assets/mascots/telemetry.png" width="28" style="display:block;"><span><b>21</b>&nbsp;可觀測性</span></div>
    <div style="display:flex; align-items:center; gap:7px; border:1px solid var(--md-default-fg-color--lightest); border-radius:20px; padding:5px 14px 5px 8px; background:var(--md-default-bg-color);"><img src="../../assets/mascots/kolla-ansible.png" width="28" style="display:block;"><span><b>22</b>&nbsp;升版、備份與 DB 維運</span></div>
  </div>
</div>
<div style="text-align:center; color:var(--md-default-fg-color--light); line-height:1.6; font-size:1.1em;">▼</div>
<div style="border:1px solid var(--md-default-fg-color--lightest); border-radius:10px; padding:14px 18px; background:var(--md-code-bg-color);">
  <strong>階段四 · 水平擴展</strong>&ensp;<span style="color:var(--md-default-fg-color--light); font-size:.85em;">Day 23–24 · 從單機到多機</span>
  <div style="display:flex; flex-wrap:wrap; gap:10px; margin-top:12px;">
    <div style="display:flex; align-items:center; gap:7px; border:1px solid var(--md-default-fg-color--lightest); border-radius:20px; padding:5px 14px 5px 8px; background:var(--md-default-bg-color);"><span style="font-size:22px; line-height:1;">🖥️</span><span><b>23</b>&nbsp;多節點部署</span></div>
    <div style="display:flex; align-items:center; gap:7px; border:1px solid var(--md-default-fg-color--lightest); border-radius:20px; padding:5px 14px 5px 8px; background:var(--md-default-bg-color);"><img src="../../assets/mascots/nova.png" width="28" style="display:block;"><img src="../../assets/mascots/magnum.png" width="28" style="display:block;"><span><b>24</b>&nbsp;橫向擴展實戰</span></div>
  </div>
</div>

<p style="text-align:center;"><em>圖上的臉都是各服務的官方吉祥物;逐日細節見下方課表。</em></p>

## 預定課表(Day 13–24)

| Day | 主題 | 一句話 |
|---|---|---|
| 13 | **Ceph 基礎** | 生產 OpenStack 的儲存標配,單機 bootstrap 一套當骨幹 |
| 14 | **物件儲存(RGW)** | S3 與 Swift 兩種 API 一次學會;為什麼不是原生 Swift,見下方設計決定 |
| 15 | **Manila 共享檔案系統** | 多台 VM 同掛一顆碟;壓軸把 **K8s RWX PVC** 接回 Day 8 的 cluster |
| 16 | **Designate DNS** | zone 與 recordset、Neutron 整合,幫 Day 4 的 LB 掛上域名 |
| 17 | **Zun 容器服務** | 容器直接跑在雲上(不經 K8s),與 Magnum 的定位比較 |
| 18 | **Trove 資料庫服務** | 一鍵開 MySQL 的 RDS 體驗;開工前先驗證 image 生態(Sprint 1 的教訓) |
| 19 | **API 請求的完整生命週期** | fernet token 解剖、request-id 跨服務追蹤、RPC 實況、qemu 進程對讀 |
| 20 | **OVN 封包轉送剖析** | OVN 深潛:logical flow、`ovn-trace` 追封包、FIP 的 NAT 在哪一條規則 |
| 21 | **可觀測性** | Prometheus + Grafana + 集中式 log;該盯什麼、怎麼用它把除錯加速十倍 |
| 22 | **升版、備份與 DB 維運** | SLURP 升級政策、`kolla-ansible upgrade` 實戰、備份還原演練 |
| 23 | **多節點部署** | 多台真機、haproxy/keepalived VIP 復活、關掉一台 controller 給你看 |
| 24 | **橫向擴展實戰** | **雲怎麼「長」**:新 compute 節點怎麼加入、容量怎麼規劃、Magnum 要長的前提 |

## 三個先講清楚的設計決定

### 為什麼物件儲存教 RGW,而不是 Swift

Swift 是 OpenStack 的元老級物件儲存服務,專案至今仍在維護;但 **Kolla-Ansible 已在 2025 年移除了 Swift 的部署支援**——整合長期乏人維護,社群選擇只保留 Ceph 路線(各部署工具的支援現況,見[部署工具圖鑑](../deployment-tools.md))。

這反映的是整個業界的走向:物件儲存的主流實作已經是 **Ceph**。一套 Ceph 同時提供三種儲存——區塊(Cinder 後端)、檔案(Manila 後端)、物件(RGW),不需要為了物件儲存另外維護一套獨立系統。而 RGW 同時支援 **S3 與 Swift 兩種 API**,原本寫給 Swift 的應用程式指向 RGW 可以直接沿用。

因此 Sprint 4 用「一天 Ceph、一天 RGW」來教物件儲存:Swift 的 API 照樣學到,還多學到 S3 與 Ceph 本身;Day 15 的 Manila 也建立在同一套 Ceph 上。

### 為什麼深層原理排在服務之後

Day 19–20 的驗收標準只有一條:**不靠 OpenStack CLI,你能不能讀懂這朵雲在做什麼?** 解剖 token、追 request-id、用 `ovn-trace` 模擬封包——這些需要你先累積夠多「用過的東西」當解剖對象。前 18 天攢的每個服務,都是這兩天的教材。

### 為什麼最後兩天要租新機器

橫向擴展在單機上教不了——「多機」的意義就在於真的有網路隔在中間。Day 23–24 會另租 3–4 台小型 VM(兩天約 US$36–49),部一套真正的多節點 OpenStack:你會看到單機時被關掉的 haproxy/VIP 為什麼存在、新的 compute 節點怎麼在使用者無感的情況下讓雲變大——**這正是「Nova/Magnum 怎麼準備好機器讓使用者去長」的答案**。教完即拆,Day 10 記錄的重建程序終於派上用場。

## 誠實的風險預告

照本課程的慣例,先把可能翻車的地方講在前面:

- **Trove(Day 18)**:guest image 生態有斷代風險——跟 Sprint 1 的 Magnum 死法同型。開工前會先做 30 分鐘查證,斷代就把該日轉為「DBaaS 現況考證與替代方案」。
- **Zun(Day 17)**:kuryr 要接管 Docker 網路,而 Kolla 把 Docker 的 iptables 關了——[Day 6 那顆地雷](sprint3-day6-capi-management-cluster.md#mine-1)的續集幾乎注定上演。這是預期中的教材,不是意外。
- 全部版本照舊釘 stable 分支;所有支援狀態以開課當日向上游查證為準(方法見[部署工具圖鑑](../deployment-tools.md)的「自己驗貨」一節)。

## 下一步

- Sprint 4 開課後,課表每一列會變成一篇完整 runbook,格式與 Day 0–10 相同。
- 想先暖身 → [Day 11 · 服務全景圖](sprint3-day11-openstack-service-map.md)把 Sprint 4 要拼的積木都介紹過了。
- 想回顧這一段 → [Sprint 3 回顧](../sprint3-reflection.md)。
