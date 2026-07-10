# 親手蓋一朵雲 — OpenStack 實作課程

> 從 Azure 一台空白 VM 開始,把一朵雲從零蓋起來——直到它能像 AWS 的 EKS 一樣,一行指令長出一整座 Kubernetes。

**📖 課程網站:https://deepwavelab.github.io/OpenStackPracticing/**

## 這是什麼

一份「做出來的」OpenStack 教材:12 個章節的實作課程,所有指令都在真實環境跑過、所有輸出都是實測。虛擬機、虛擬網路、雲端硬碟、負載平衡——公有雲的能力不是魔法,是一套叫 OpenStack 的開源軟體,這門課帶你親手把它蓋起來。

| 內容 | 說明 |
|---|---|
| **課程主線(Day 0–11)** | Kolla-Ansible 部署 → 資源全流程 → Cinder / Octavia / Barbican / Heat → Magnum + Cluster API → 端到端開出 workload K8s → Day-2 維運 → Terraform,固定格式:原理 → 可照抄的步驟 → 驗收 checkpoint → 踩雷記錄 |
| **排錯手冊** | 15 條官方文件查不到、動手才會撞到的整合地雷與解法(症狀 → 根因 → 解法) |
| **前兩次嘗試** | 這條路試了三次才走通:Juju + Charms(9 天)與 OpenStack-Helm(2 天)的完整失敗紀錄與教訓 |
| **部署工具圖鑑** | Charms / Kolla-Ansible / OSH / RHOSO / Sunbeam 各家陣營比較,含向上游實測的服務支援矩陣與晶片架構支援表 |

沒碰過 OpenStack 也能讀:每章開頭有「第一次接觸?」白話導讀,每個服務都對標 AWS / GCP / Azure(Cinder = EBS、Magnum = EKS)。

## 本機預覽

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements-docs.txt
mkdocs serve          # http://127.0.0.1:8000
```

發佈到 GitHub Pages:`mkdocs gh-deploy --force`

## 目錄結構

```
├── mkdocs.yml            # 網站設定(nav / theme / extensions)
├── requirements-docs.txt # mkdocs-material 釘版
└── docs/
    ├── index.md          # 首頁
    ├── runbook/          # 課程章節(sprint3-day*)與歷史紀錄(day-*、sprint2-*)
    ├── solutions/        # 排錯手冊
    ├── assets/           # 官方吉祥物、logo、OpenStack Map
    ├── previous-attempts.md / deployment-tools.md
    └── *-reflection.md   # 各次嘗試的回顧
```

## 圖像出處

站內吉祥物與 OpenStack Map 為 [OpenInfra Foundation 官方資產](https://www.openstack.org/project-mascots/);Kubernetes / kind / Cluster API 標誌屬 CNCF(Linux Foundation);Terraform 標誌屬 HashiCorp。均作社群教學用途,版權歸原基金會/公司所有。
