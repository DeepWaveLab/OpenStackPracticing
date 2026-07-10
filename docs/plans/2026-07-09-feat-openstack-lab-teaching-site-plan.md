---
title: "feat: OpenStack Lab 教學網站(MkDocs Material)"
type: feat
status: active
date: 2026-07-09
---

# feat: OpenStack Lab 教學網站(MkDocs Material)

## Overview

把既有的 OpenStack 實作文件(56 檔:36 篇 runbook、17 篇 solutions、3 篇 reflection/plan,橫跨 3 個 sprint)包裝成**人類好讀的教學網站**。技術選型 **MkDocs Material**,**先本機 build 看效果**;內容以 **Sprint 3(Kolla + Magnum CAPI 的完整「雪恥」故事,Day 0→10)為教學主線**,Sprint 1(Juju/charm)/ Sprint 2(OSH)收成「歷史/為什麼換路線」的背景章節。

**核心洞察**:repo 的 `docs/` 佈局**已經完全符合 MkDocs 預設**(`mkdocs.yml` 放 repo root、`docs/` 當 `docs_dir`),而且交叉連結已是標準 `[text](path.md)` 相對連結(研究時 grep 全樹,**零 `[[wikilink]]`**)。所以這不是「重寫內容」的專案,而是「**加一層導覽/呈現**」的專案 —— 主要工時在 `mkdocs.yml`(nav/theme/extensions)、新增 landing pages、以及輕度潤飾,**現有 .md 幾乎不動**。

## Problem Statement / Motivation

- **現況**:文件是一堆 markdown,對作者友善、對讀者不友善:沒有導覽樹、沒有搜尋、沒有明確學習路徑、跨檔連結只能在編輯器點、mermaid/表格在 GitHub 原生呈現普通、沒有深色模式。
- **痛點場景**:(1) 新人 onboarding 想「從 Day 0 跟著做到 workload cluster」找不到入口與順序;(2) 遇到某個坑想查排錯手冊,得自己翻 `solutions/`;(3) 自己三個月後回來看,得重新爬檔案結構。
- **目標**:一個有導覽/搜尋/學習路徑/圖表渲染/深色模式的教學站,把「十天雪恥」講成**可照著走的課程**,把 6+ 條踩坑做成**可搜尋可查的排錯手冊**。

## Proposed Solution

1. **工具:MkDocs Material 9.7.x(pin 版)** —— markdown-native、GitHub Pages 一鍵、技術文件業界標配。註:Material 2026 起進 maintenance mode(維護到 ~2026-11+,團隊轉做 Zensical),但仍 production-ready,且內容是純 markdown 可無痛搬到任何後繼者 —— **pin `9.7.6`,不押未釋出的 Zensical**。
2. **內容架構:Diátaxis-informed IA**(Canonical/Python 都採用的框架):
   - **runbook → 學習路徑(tutorial)**:Sprint 3 Day 0-10 為主線,依序引導。
   - **solutions → 排錯手冊(how-to / reference)**:獨立 tab,可查不必照讀。
   - **reflection → 背景解釋(explanation)**:為什麼 Kolla 勝過 OSH/Juju、雪恥回顧。
3. **佈局零搬動**:`mkdocs.yml` 放 repo root,`docs/` 不動;新增 `docs/index.md`(首頁,目前不存在)+ 各 sprint section landing。
4. **先本機**:`mkdocs serve` 看滿意;GitHub Pages(Actions)列為 Phase 4 後續選項。

## Technical Approach

### 專案佈局(已符合預設,只新增標註 ← 的檔)

```
OpenStackPracticing/
├─ mkdocs.yml               ← 新增(repo root, docs/ 的 sibling)
├─ requirements-docs.txt    ← 新增(pin mkdocs-material==9.7.6)
└─ docs/                    ← 已存在,內容不搬
   ├─ index.md              ← 新增(首頁 landing,目前無)
   ├─ runbook/              (day-*.md / sprint2-* / sprint3-* + README.md)
   ├─ solutions/            (README.md + integration-issues/ + runtime-errors/)
   ├─ plans/                ← exclude_docs 排除,不進站
   ├─ reflection.md, sprint2-course-plan.md, sprint3-reflection.md
```

### `mkdocs.yml`(核心設定,來自官方 docs 研究)

```yaml
site_name: OpenStack Lab Course
site_url: https://<user>.github.io/OpenStackPracticing/   # GH Pages 用;本機可留佔位
site_description: OpenStack 實作課程 —— runbook、踩坑解法與反思

theme:
  name: material
  language: zh-TW                       # 雙語內容,顧到搜尋 stemming + UI
  palette:
    - scheme: default
      primary: indigo
      toggle: { icon: material/brightness-7, name: 切換深色 }
    - scheme: slate
      primary: indigo
      toggle: { icon: material/brightness-4, name: 切換淺色 }
  features:
    - navigation.instant
    - navigation.instant.progress
    - navigation.tabs            # 頂層 tab:主線 / 背景 / 排錯 / 回顧
    - navigation.tabs.sticky
    - navigation.sections
    - navigation.indexes         # 用既有 README.md 當 section landing
    - navigation.top
    - navigation.footer          # 每頁底 prev/next —— 對「Day 序列」關鍵
    - toc.follow
    - content.code.copy
    - content.code.annotate
    - search.suggest
    - search.highlight
    - search.share

plugins:
  - search                       # 加了 plugins list 就必須重新宣告 search

markdown_extensions:
  - admonition                   # 踩坑 callout
  - pymdownx.details             # 可折疊 ??? callout
  - pymdownx.highlight: { anchor_linenums: true, line_spans: __span, pygments_lang_class: true }
  - pymdownx.inlinehilite
  - pymdownx.snippets
  - pymdownx.superfences:
      custom_fences:
        - name: mermaid          # Material 原生 mermaid,勿加 mermaid2-plugin
          class: mermaid
          format: !!python/name:pymdownx.superfences.fence_code_format
  - pymdownx.tabbed: { alternate_style: true }
  - pymdownx.emoji:
      emoji_index: !!python/name:material.extensions.emoji.twemoji
      emoji_generator: !!python/name:material.extensions.emoji.to_svg
  - pymdownx.tasklist: { custom_checkbox: true }
  - tables
  - attr_list
  - md_in_html
  - toc: { permalink: true }

validation:                      # MkDocs 1.6+:抓斷連結/漂移錨點
  links: { absolute_links: warn, unrecognized_links: warn, anchors: warn }

exclude_docs: |
  plans/                         # 內部規劃文件不進站

nav:                             # 見下方「內容架構」——必須明確排序
  - ...
```

> ⚠️ `!!python/name:` 是 load-bearing(mermaid/emoji 需要),但一般 `yaml.safe_load`/yamllint/pre-commit 會爆 —— CI 若跑 YAML lint,**排除 `mkdocs.yml`**。

### 內容架構 / `nav`(Diátaxis 對應,明確排序)

> 為什麼必須手寫 nav:沒有明確 nav 時 MkDocs 依檔名字母排序 → `day-10` 會排在 `day-2` 前、`sprint3-day10` 排在 `sprint3-day2` 前,把課程順序打亂。**順序就是教學法,不能交給 auto-sort。**

```yaml
nav:
  - 首頁: index.md
  - 課程主線 · Sprint 3(Kolla + Magnum CAPI):        # ← 教學主軸(tutorial)
      - runbook/README.md                              # section landing(navigation.indexes)
      - "Day 0 · Azure Lab 環境": runbook/sprint3-day0-azure-vm.md
      - "Day 1 · Kolla AIO core": runbook/sprint3-day1-kolla-aio-core.md
      - "Day 2 · OpenStack 資源全流程": runbook/sprint3-day2-openstack-resource-flow.md
      - "Day 3 · Cinder LVM(雪恥#1)": runbook/sprint3-day3-cinder-lvm.md
      - "Day 4 · Octavia LB(雪恥#2)": runbook/sprint3-day4-octavia.md
      - "Day 5 · Barbican + Heat": runbook/sprint3-day5-barbican-heat.md
      - "Day 6 · CAPI 管理叢集": runbook/sprint3-day6-capi-management-cluster.md
      - "Day 7 · Magnum CAPI driver": runbook/sprint3-day7-magnum-capi-driver.md
      - "Day 8 · E2E workload cluster(雪恥#3)": runbook/sprint3-day8-e2e-workload-cluster.md
      - "Day 9 · Day-2 operations": runbook/sprint3-day9-day2-operations.md
      - "Day 10 · Terraform + teardown": runbook/sprint3-day10-terraform-teardown.md
  - 排錯手冊 · Solutions:                              # ← how-to / reference
      - solutions/README.md
      - "整合坑 · Kolla/Magnum(Sprint 3)":
          - runbook 對應的 6 條:kolla-proxysql / octavia-dhclient / octavia-redis /
            kind-docker-iptables / capi-fixed-subnet / capi-nodegroup-failuredomain
      - "整合坑 · Juju/charm 時代(Sprint 1)":
          - mysql-charm / ovn-* / nova-compute-* / neutron-sg / magnum-api-bind / heat-domain
      - "Runtime errors":
          - solutions/runtime-errors/cirros-sshd-race-vs-ubuntu-config-drive.md
  - 背景與歷史 · 為什麼換路線:                        # ← explanation
      - "Sprint 1 · Juju + Charms(失敗)": runbook/day-0-azure-vm.md   # + 其餘 day-* 依學習序
      - "Sprint 2 · OSH 嘗試": runbook/sprint2-day-0-azure-vm.md        # + 其餘 sprint2-*
      - "Sprint 2 課程計畫(被推翻)": sprint2-course-plan.md
  - 回顧 · Reflections:                               # ← explanation
      - "Sprint 3 回顧(雪恥完成)": sprint3-reflection.md
      - "Sprint 1 回顧": reflection.md
```

> Sprint 1 的 `day-3-*` 子系統建議依**依賴序**排(keystone → mysql → rabbitmq → vault → glance → placement → nova-cc → nova-compute → network → horizon),而非字母序;屬背景章節,Phase 2 再細排。

### Mermaid / 圖表

- ```mermaid``` fenced block 用 Material **原生 superfences**(上方 config),`day-2-openstack-charm-map.md` 等現有圖零改渲染。**不要**裝 `mkdocs-mermaid2-plugin`(重複載入 Mermaid)。
- runbook 內大量 ASCII 架構圖留在 ```` ```text ```` fence,原樣 monospace 呈現即可,不用轉。

### 現有內容重用策略(關鍵:低改動)

- **連結**:已是相對 `.md` 連結,MkDocs 自動 rewrite 成正確 URL —— 不動。
- **README.md**:`runbook/README.md`、`solutions/README.md` 已存在且內容剛好適合當 section landing(`navigation.indexes` 直接吃)—— 不動、不改名。
- **踩坑段落**:現在是 `## 踩坑` 純標題,render 正常;可**選擇性**把重點坑升級成 `!!! warning "踩坑"` callout(純作者潤飾,非必須)。

## Implementation Phases

### Phase 1 — Scaffold(骨架能跑)
- `requirements-docs.txt`(`mkdocs-material==9.7.6`)+ venv `pip install`。
- 寫 `mkdocs.yml`(theme + features + extensions + mermaid + zh-TW + palette + validation + exclude plans/)。
- `docs/index.md` 首頁草稿(先能連到 Sprint 3)。
- 先掛 **Sprint 3 nav**。`mkdocs serve` 本機驗:mermaid 渲染、表格、程式碼 copy、深色切換、中文搜尋。
- **Deliverable**:本機站起得來,Sprint 3 Day 0-10 可依序瀏覽。

### Phase 2 — 完整 IA / nav
- 補齊全 nav(排錯手冊 + 背景 Sprint 1/2 + 回顧),明確排序。
- `exclude_docs: plans/`;`mkdocs build --strict` 必須零 error(斷連結 / not-in-nav 清乾淨)。
- 修 --strict 抓到的問題(相對連結漂移、heading 錨點)。
- **Deliverable**:全站可瀏覽、`build --strict` 綠。

### Phase 3 — 教學潤飾
- `index.md` 完稿:雪恥 arc(三項雪恥表)、學習路徑圖、成本、「怎麼用這站」。
- 各 sprint section landing:learning path + 先修 + checkpoint 總表(可從 reflection 摘)。
- 選擇性把關鍵踩坑升級成 admonition callout(示範幾個)。
- 確認 `navigation.footer` 的 prev/next 讓 Day 序流暢。
- (可選)consolidated **Reference** 頁:總架構圖 + 版本矩陣(CAPI/CAPO/driver/k8s)+ 指令 cheat-sheet。

### Phase 4 —(後續/選項)部署
- 決定 public vs private(目前:先本機)。
- **若對外公開:先脫敏 secrets**(見 Risks)。
- GitHub Actions `ci.yml`(`mkdocs gh-deploy --force`)或手動 `mkdocs gh-deploy`;`site_url` 設對 project subpath。

## System-Wide Impact / 邊界考量

- **設定交互**:加 `plugins:` list 後 `search` 必須重新宣告(否則搜尋消失);`!!python/name` tag 會讓外部 YAML linter 爆(排除 `mkdocs.yml`)。
- **連結傳播**:相對 `.md` 連結由 MkDocs rewrite;`validation.links.anchors: warn` 抓 `#heading` 錨點漂移;`--strict` 把 warning 變 fail(CI 用)。
- **狀態/內容生命週期**:零搬動,repo 既有結構不變,只**新增** `mkdocs.yml` + `index.md` + landing;可安全 revert(刪這幾個新檔即回到現況)。
- **API/介面 parity**:無程式碼介面;唯一「介面」是 nav —— 未來新增 sprint 要手動加 nav 一行(可接受的維護成本)。
- **整合測試情境**:(1) mermaid 在深/淺色都要正確;(2) 中文搜尋要命中(zh-TW stemming);(3) 跨檔連結(runbook→solutions)點得到;(4) `build --strict` 在 CI 綠。

## Acceptance Criteria

- [x] `mkdocs serve` 本機起站,**Sprint 3 Day 0-10 依序可瀏覽**(navigation.footer prev/next 串起)
- [x] mermaid 圖 render(首頁三層互動圖已驗;各 day 架構圖同機制)
- [x] 踩坑 callout / 表格 / 程式碼 copy / 深色模式 / **中文搜尋**皆設定(admonition/tables/content.code.copy/palette/zh-TW search)
- [x] nav **明確排序**(`day-10` 不排在 `day-2` 前)
- [x] `solutions/` 成獨立「排錯手冊」tab,可搜尋可查
- [x] `mkdocs build --strict` **零 error**(57 頁;修掉 1 條斷錨點 nova-compute)
- [x] `docs/plans/` **不進站**(exclude_docs)
- [x] `index.md` 首頁講清楚雪恥故事 + 學習路徑 + 怎麼用
- [ ] (部署前才需)若對外,secrets 已脫敏 —— **尚未做,對外前必做**

## Success Metrics

- 新人能靠這站**從 Day 0 跟到 workload cluster**,不用翻 raw markdown。
- 遇到坑時,關鍵字搜尋能命中對應 solutions 頁。

## Dependencies & Risks

| 風險 | 影響 | 對策 |
|---|---|---|
| **Secrets 外洩**(僅對外公開時)| runbook 內有真實 public IP(`203.0.113.10`)、`admin-openrc` 路徑、`~/.ssh/juju_id_rsa`/`k8s-admin.pem`、subscription id | **對外前** grep 掃 `ifconfig`/`openrc`/`\.pem`/IP/subscription,脫敏或占位符化。**本機/內部則無此問題** |
| Material maintenance mode | 未來大版本可能停更 | pin `9.7.6`;內容純 markdown 可搬 |
| 內容量大(56 檔)| nav 手排 + landing 撰寫是主要工時 | 分 phase;Sprint 3 先上,Sprint 1/2 背景後補 |
| 雙語搜尋 | 中文搜尋 stemming | `theme.language: zh-TW` |
| 維護 | 新 sprint 要手動加 nav | 文件化「加一頁 = nav 加一行」 |

## Mock Files

### `requirements-docs.txt`
```
mkdocs-material==9.7.6
```

### `mkdocs.yml`
(見上「核心設定」+「內容架構 nav」;drop-in。)

### `docs/index.md`(首頁骨架)
```markdown
# OpenStack 實作課程 —— 十天雪恥

> 從 Azure 一台裸 VM,到用 Kolla-Ansible 部署 OpenStack、再用 Magnum + Cluster API
> 開出能跑 workload 的 Kubernetes cluster。Sprint 1 判死刑的三件事,這次全數翻案。

## 這站是什麼

- **課程主線**:Sprint 3 Day 0→10 —— 照著走就能重現整套 lab。
- **排錯手冊**:實作中撞到、文件查不到的整合坑與解法。
- **背景**:為什麼從 Juju charm → OSH → 落腳 Kolla + CAPI driver。

## 三項雪恥(Sprint 1 敗因 → 本次結果)

| Sprint 1 死因 | 本次 |
|---|---|
| Octavia charm amd64 斷代 | ✅ Kolla Octavia + K8s type=LoadBalancer |
| Cinder LVM 卡 LXD | ✅ VM host LVM + PVC 動態供裝 |
| Magnum heat driver image 失效 | ✅ CAPI driver + 維護中的 node image |

## 從哪開始

→ [課程主線 Day 0 · Azure Lab 環境](runbook/sprint3-day0-azure-vm.md)
```

## Sources & References

### 內容架構(IA)
- **Diátaxis framework**(tutorial / how-to / reference / explanation,Canonical/Python 採用):https://diataxis.fr/ · [start-here](https://diataxis.fr/start-here/)

### 工具(MkDocs Material,官方 docs)
- [Getting started](https://squidfunk.github.io/mkdocs-material/getting-started/) · [Creating your site](https://squidfunk.github.io/mkdocs-material/creating-your-site/)
- [Setting up navigation](https://squidfunk.github.io/mkdocs-material/setup/setting-up-navigation/)(navigation.indexes / tabs / footer)
- [Python Markdown Extensions](https://squidfunk.github.io/mkdocs-material/setup/extensions/python-markdown-extensions/) · [Admonitions](https://squidfunk.github.io/mkdocs-material/reference/admonitions/) · [Code blocks](https://squidfunk.github.io/mkdocs-material/reference/code-blocks/)
- [Diagrams / Mermaid](https://squidfunk.github.io/mkdocs-material/reference/diagrams/)(原生 superfences)
- [Publishing your site](https://squidfunk.github.io/mkdocs-material/publishing-your-site/)(gh-deploy / Actions)
- [mkdocs-material on PyPI](https://pypi.org/project/mkdocs-material/)(9.7.x,requires-python ≥3.8,mkdocs ≥1.6,<2)

### 內部(要被網站化的來源)
- `docs/runbook/sprint3-day0…day10-*.md`(11 篇,教學主線)
- `docs/solutions/**`(17 篇,排錯手冊;Sprint 3 佔 6 篇)
- `docs/sprint3-reflection.md`、`docs/reflection.md`、`docs/sprint2-course-plan.md`(背景/回顧)
- `docs/runbook/README.md`、`docs/solutions/README.md`(現成 section landing)
