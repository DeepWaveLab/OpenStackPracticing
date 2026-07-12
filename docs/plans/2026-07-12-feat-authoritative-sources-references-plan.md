---
title: "feat: 為課程各章加上權威來源(延伸閱讀)"
type: feat
status: active
date: 2026-07-12
deepened: 2026-07-12(併入視覺補強工作流)
---

# 為課程各章加上權威來源(延伸閱讀)+ 視覺補強

## 目標

Day 1–10、Day 13–21 每章章末加一節**延伸閱讀**:精選、已驗證的權威來源(OpenStack 官方文件優先),讓讀者能往下深挖,也讓「本課程內容都查證過」這個主張有可點擊的證據。

**先做四章試點給使用者審**:Day 2、Day 7、Day 13、Day 21——涵蓋四種章節型態(基礎操作 / 深度整合 / 輕量部署 / 維運),審過選源品味與呈現風格後,才展開到其餘各章。

## 為什麼值得做

1. **可信度**:課程一路強調「向上游查證」,但讀者看不到證據鏈。掛上來源 = 把查證變成可驗證的。
2. **學習路標**:每章 500–1500 字講不完一個服務;好的延伸閱讀讓認真的讀者知道下一步讀什麼、去哪讀。
3. **對抗內容農場**:中文圈 OpenStack 資料充斥過時轉貼;給讀者一份乾淨的官方入口清單本身就是價值。

## 呈現風格規範(試點要審的核心)

### 位置與結構

- 每章新增 `## 延伸閱讀`,固定放在 `## 下一步` **之前**(有地雷記錄的章節,放地雷之後)。
- **3–6 條,硬上限 6**。寧缺勿濫——這是精選,不是連結堆。
- 固定一句開場:「想往下深挖,從這幾份開始:」

### 每條的格式

```
- **[中文描述性標題](URL)** —— 一句話說明這份講什麼、跟本章哪裡對得上。
```

- 連結文字用**中文描述**(必要時附原名,如 CAPI Book);說明句是**給讀者的導讀**,不是書目註記。
- 說明句禁止:「必讀」「乾貨」等浮誇或支語詞;禁止日記腔(「我當時參考了…」)。
- 全部台灣繁體,技術名詞保留英文。

### Mock(Day 13 · Skyline 的完整範例——風格以此為準)

> ## 延伸閱讀
>
> 想往下深挖,從這幾份開始:
>
> - **[Skyline 官方文件](https://docs.openstack.org/skyline-apiserver/2025.1/)** —— apiserver 的架構與設定參考;本章「兩容器分工」的完整定義在這裡。
> - **[Kolla-Ansible 的 Skyline 部署指南](https://docs.openstack.org/kolla-ansible/2025.1/reference/shared-services/skyline-guide.html)** —— 本章部署步驟的官方對照,SSO 與自訂外觀等進階選項也在這份。
> - **[skyline-console 原始碼](https://opendev.org/openstack/skyline-console)** —— 想看 Vue 前端怎麼組出資源拓樸圖,直接讀源頭。

### 行內引用(節制使用)

章內既有的「官方文件明載/警告」這類句子,在**該句**補上連結(例:Day 15 的 `update_keystone_service_user_passwords` 警告)。每章行內新增連結以 1–2 處為限,不重寫內文。

## 來源品質準則

| 層級 | 來源 | 用法 |
|---|---|---|
| 一級 | docs.openstack.org(**釘 2025.1 版路徑**)、docs.ceph.com(釘 tentacle)、kubernetes.io、cluster-api.sigs.k8s.io | 預設首選 |
| 二級 | opendev/GitHub 原始碼與 release notes、官方 wiki | 佐證具體事實(如 Kolla 移除 Swift 的 commit) |
| 三級 | OpenInfra 基金會出版物(Superuser)、專案作者團隊的技術文(如 StackHPC 談 CAPI driver)、Google SRE Book | 每章至多 1–2 條,須有官方來源沒有的觀點 |
| 禁用 | 內容農場、轉貼站、無日期無署名的教學、CSDN/簡中轉載 | — |

版本釘選:OpenStack 文件一律用 `/2025.1/` 路徑(對齊課程版本);該頁在 2025.1 不存在才退回 `/latest/` 並註記。

## 驗證紀律(執行時逐條做)

1. 每條 URL 以 WebFetch 實際打開:HTTP 200 + **內容確實講我們宣稱的事**(不是只看標題)。
2. 死鏈或內容不符 → 換源或刪條,不硬湊。
3. 完成後跑一輪全站外部連結批次檢查(curl 狀態碼)+ `mkdocs build --strict` + 支語/簡體掃描。

## 試點四章的候選來源(草稿,執行時逐一驗證)

### Day 2 · OpenStack 資源全流程
- Nova「Launch instances」官方流程(nova/2025.1/user/launch-instances)
- Neutron 網路概念總覽(neutron/2025.1/admin/intro-os-networking)
- 官方安裝指南的「Launch an instance」章(install-guide)
- 安全群組官方說明(nova user/security-groups)

### Day 7 · Magnum CAPI driver
- Magnum 官方 user guide(magnum/2025.1/user)
- magnum-capi-helm 原始碼與 README(opendev)
- The Cluster API Book(cluster-api.sigs.k8s.io)
- StackHPC(driver 作者團隊)的 CAPI driver 介紹文(三級來源,驗證後擇優)

### Day 13 · Skyline
- 如上方 mock(skyline-apiserver 文件、kolla skyline 指南、skyline-console 原始碼)

### Day 21 · 可觀測性
- Kolla-Ansible logging-and-monitoring 三份指南(prometheus / grafana / central-logging,2025.1)
- Prometheus 官方概念導覽(prometheus.io/docs)
- **Google SRE Book「Monitoring Distributed Systems」**(四大黃金訊號——本章「該監控什麼」表格的理論出處,示範跨圈權威來源的用法)


## 視覺補強(併入本計畫)

使用者發現的缺口:Skyline、Prometheus、Grafana 都有官方 logo,但章節裡沒放。根因:先前的吉祥物掃描只涵蓋 OpenStack 官方吉祥物集(Skyline 當時不在其中),非 OpenStack 專案(Prometheus/Grafana)則從未進掃描範圍。趁試點一起補。

### 各章 logo 盤點

| 章 | 缺的視覺 | 官方來源候選(執行時驗證) | 優先度 |
|---|---|---|---|
| Day 13 | Skyline | OpenStack 官方吉祥物/品牌頁;備援:skyline-console 原始碼內的 logo 資產 | **試點必做** |
| Day 21 | Prometheus | CNCF 官方 artwork repo(github.com/cncf/artwork,授權明確) | **試點必做** |
| Day 21 | Grafana | Grafana Labs 官方品牌資產(grafana 原始碼 repo 或品牌頁) | **試點必做** |
| Day 21 | OpenSearch | opensearch.org 品牌資產(Apache 2.0) | 選配 |
| Day 14 | podman(海豹吉祥物) | containers 官方 repo logo 資產 | Phase 2 選配 |
| Day 20 | OVN/OVS | ovn.org / openvswitch.org | Phase 2 選配(有才放) |
| Day 19 | RabbitMQ | rabbitmq 官方(Broadcom 品牌頁) | Phase 2 選配 |

### 上架紀律(既有教訓的固化)

1. **來源必須是官方**(專案 repo / 基金會 artwork / 官方品牌頁),抓下來**必先目檢**再入庫——先前抓錯 GitHub 組織 ID 差點把路人自拍當 Ceph logo 上架,目檢是硬規則。
2. **透明底優先**;只有灰/白底版本就去背(Ceph 的處理流程可複用)。
3. 版位沿用既有慣例:`{ align=right width="100" }` 放在導讀段;一章多個 logo 時,主視覺一枚 right-float,其餘放進相關段落(如 Grafana logo 放 Grafana 驗證段),**不硬塞**。
4. 頁尾出處聲明(同 Ceph/吉祥物慣例)。
5. 連動更新:Day 11 服務表 Skyline 列的吉祥物欄(目前是「—」)補上。

## 執行順序

- [x] **Phase 1(試點)**:Day 2、7、13、21 加上延伸閱讀(含驗證)+ **Day 13/21 補官方 logo**(Skyline、Prometheus、Grafana)→ 本機預覽 → **使用者審風格與選源** → 依回饋修正規範
- [x] **Phase 2(展開)**:依修正後規範處理其餘 15 章,分三批:
    - 基礎批:Day 1、3、4、5(Kolla 各服務)
    - K8s 批:Day 6、8、9、10(CAPI/Magnum/維運/Terraform)
    - Sprint 4 批:Day 14–20(Ceph×3、DNS、Trove、原理×2)+ 選配 logo(podman/OVN/RabbitMQ,有官方資產才放)
- [x] 全站連結批次檢查 + build + 用詞掃描 → commit 上線

範圍註記:使用者指定 Day 1–10、13–21。Day 0、11、12 與圖鑑頁不在此次範圍(圖鑑頁已自帶來源);若試點後覺得值得,可加選。

## 驗收標準

- [ ] 試點四章各有 3–6 條已驗證來源,格式完全符合 mock
- [ ] OpenStack 文件連結全部釘 2025.1(例外有註記)
- [ ] 無死鏈、無內容不符的連結、無支語/簡體
- [ ] Day 13/21 的 logo:官方來源、目檢通過、透明底、附出處聲明
- [ ] Day 11 服務表 Skyline 列的吉祥物欄同步更新
- [ ] 使用者確認風格後才展開 Phase 2

## Sources

- 站內既有引用慣例:`docs/deployment-tools.md`(自我驗證一節)、各章行內「官方文件」語句
- Kolla 2025.1 文件路徑已在部署驗證階段大量實測(見 `docs/plans/2026-07-11-feat-sprint4-advanced-course-plan.md`)
