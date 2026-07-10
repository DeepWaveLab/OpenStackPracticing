---
title: "feat: 首頁改版 —— 吸引人的開頭 + 吉祥物視覺 + 服務卡片"
type: feat
status: active
date: 2026-07-10
---

# feat: 首頁改版 —— 吸引人的開頭 + 吉祥物視覺 + 服務卡片

## Overview

現在的首頁(`docs/index.md`)資訊正確但開場平淡:標題 → 定義式介紹 → 表格。本次改版把首頁變成**有鉤子的落地頁**:成果前置的開頭、官方吉祥物當視覺主角、Material grid cards 做服務導覽——全部用站內既有素材與已啟用的功能,零新依賴。

## Problem Statement / Motivation

- 開頭是「這是一門十天的實作課程…」——定義式開場,沒有畫面感,讀者第一秒拿不到「我為什麼要讀」。
- 站內攢了 23 隻官方吉祥物 + OpenStack Map + CAPI 烏龜這些「酷素材」,但首頁一張圖都沒有(全部藏在內頁)。
- 好的技術文開頭的共識原則:**把成果放最前面**(讀者先看到終點)、**具體**(真指令、真數字,不是形容詞)、**誠實的承諾**(這裡是「含失敗史」——反而是差異化賣點)。

## Proposed Solution

### 1. 新開頭(hook)—— 成果前置

用課程最強的 payoff 畫面開場:

```markdown
# 一行指令,長出一整座 Kubernetes

​```bash
openstack coe cluster create k8s-lab --cluster-template k8s-v1.34.8 …
​```

四分鐘後:虛擬機自動開好、負載平衡器自動掛上、持久磁碟自動供裝——
一座能跑正式服務的 Kubernetes 叢集,從一朵**你自己蓋的雲**裡長出來。

這個網站教你從一台空白 VM 開始,親手蓋出做得到這件事的 OpenStack——
包含我們**失敗過兩次**才找到的路。
```

Hook 原則對應:第一行是結果不是定義;指令與「四分鐘」是具體錨點;「失敗過兩次」製造好奇缺口並誠實。

### 2. 吉祥物列隊(hero 視覺)

開頭正下方放一排吉祥物(本課 8 位主角,各 ~52px,置中一行),配一句:「這些是你這趟旅程會認識的夥伴——OpenStack 每個服務都有官方吉祥物,**記臉比記名字快**。」窄螢幕自動換行,天然響應式。

### 3. 服務卡片(Material grid cards)——「這朵雲的居民」

官方 grid cards 語法(`<div class="grid cards" markdown>`,attr_list + md_in_html 已啟用),8 張卡:

| 卡 | 內容 |
|---|---|
| Keystone | 吉祥物 + 「帳號與權限的守門人」 + → Day 2 |
| Nova | 「開虛擬機的引擎」 + → Day 2 |
| Neutron | 「一切網路的織網者」 + → Day 2 |
| Cinder | 「拔得下來的硬碟」 + → Day 3 |
| Octavia | 「流量的帶位員」 + → Day 4 |
| Barbican | 「秘密的保險箱」 + → Day 5 |
| Heat | 「照藍圖蓋房子」 + → Day 5 |
| Magnum | 「一鍵長出 K8s」 + → Day 7 |

卡片 hover 有懸浮效果(Material 內建),行動裝置自動單欄。

### 4. 入口卡片(第二組 grid)——「從這裡出發」

4 張:🚀 課程主線(Day 0 開始)/ 🔧 排錯手冊 / 📜 歷史·前兩次嘗試 / 🗺️ 服務全景圖(Day 11)。

### 5. 鎮館之寶彩蛋

一小段放 CAPI 烏龜(`assets/logos/cluster-api.svg`,右浮動)+「一路往下全是烏龜」一句話,連到 Day 6——它是站上最有記憶點的圖。

### 6. 保留與重排

- 「三次嘗試」背景**壓縮成三行摘要**(詳情連 previous-attempts)——首頁不再長篇講史
- Day 0-11 課程表保留(卡片給感性、表給理性)
- 三層互動 mermaid 保留(移到課程表後)
- 頁尾 CTA 保留

### 版面順序(after)

hook → 吉祥物列隊 → 從這裡出發(4 入口卡)→ 這朵雲的居民(8 服務卡)→ 三行版三次嘗試 → 課程表 → mermaid → 烏龜彩蛋 → CTA

## Technical Considerations

- **零新依賴**:grid cards 只需 `attr_list` + `md_in_html`(mkdocs.yml 已有)。
- 若置中/間距需要微調,加最小 `docs/assets/stylesheets/extra.css` + mkdocs.yml `extra_css`(~10 行,只作用於首頁 hero 類名)。
- 圖片全部站內既有(mascots/、logos/),無新下載。
- 深色模式:吉祥物 PNG 皆透明底,深色下正常;卡片是 Material 原生元件,兩主題自動適配。

## Acceptance Criteria

- [ ] 開頭第一屏 = hook + 吉祥物列隊(不再是定義式開場)
- [ ] 8 張服務卡 + 4 張入口卡正常渲染,hover 有懸浮效果,連結正確
- [ ] 窄視窗(手機寬度)卡片單欄、吉祥物列隊換行不破版
- [ ] 深色模式下全部正常
- [ ] `mkdocs build --strict` 零 warning
- [ ] `mkdocs gh-deploy` 後線上站更新,抽驗圖片與連結 200

## Dependencies & Risks

| 風險 | 對策 |
|---|---|
| 首頁過度設計壓過內容 | 卡片文案一句話為限;課程表保留當「理性層」 |
| grid cards 語法縮排敏感(list item 內含圖) | 本機 serve 逐項目視;--strict 兜底 |
| hook 太浮誇 | 「四分鐘」是 Day 8 實測數據(06:52→06:56),誠實 |

## Sources & References

- [Material for MkDocs — Grids(官方)](https://squidfunk.github.io/mkdocs-material/reference/grids/)
- [Material hero/landing 實作例](https://medium.com/@wishula/implementing-a-left-sidebar-theme-toggle-and-custom-hero-in-mkdocs-material-b2d5d71a1278)
- 開頭寫作原則(成果前置/具體/誠實承諾):[Squarespace Engineering: Technical Writing How To Start](https://engineering.squarespace.com/blog/2023/technical-writing-how-to-start)、[storygrid: beginning hook](https://storygrid.com/beginning-hook/)
- 站內素材:`docs/assets/mascots/*`(23 隻)、`docs/assets/logos/cluster-api.svg`、Day 8 實測時間軸(4 分鐘依據)
