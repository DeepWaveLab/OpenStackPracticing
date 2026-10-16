# OpenStack 實作課程

從 Azure 上的實驗 VM 開始，用 Kolla-Ansible 部署 OpenStack，依序練習虛擬機、網路、儲存與 Kubernetes 叢集服務，再延伸到多節點維運、租戶治理、計量、計價與安全。

[閱讀課程網站](https://deepwavelab.github.io/OpenStackPracticing/)

課程主線包含 Day 0–35 與結語。讀者需具備 Linux 命令列與基本網路知識；Day 6 起使用 Kubernetes，Day 28 起會讀 Prometheus 查詢。章節以問題、操作與輸出解讀串接，保留實測版本與尚未驗證的限制。

- `docs/index.md`：課程順序、先備知識與全部章節連結。
- `docs/runbook/`：教學主線及早期部署紀錄。
- `docs/solutions/`：依症狀查找的疑難排解。
- `docs/deployment-tools.md`：部署工具與適用情境。

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
    ├── runbook/          # 課程章節(sprint3/4/5-day*、epilogue)與歷史紀錄
    ├── solutions/        # 排錯手冊
    ├── assets/           # 官方吉祥物、logo、OpenStack Map
    ├── previous-attempts.md / deployment-tools.md
    └── *-reflection.md   # 各次嘗試的回顧
```

## 圖像出處

站內吉祥物與 OpenStack Map 為 [OpenInfra Foundation 官方資產](https://www.openstack.org/project-mascots/);Kubernetes / kind / Cluster API 標誌屬 CNCF(Linux Foundation);Terraform 標誌屬 HashiCorp。均作社群教學用途,版權歸原基金會/公司所有。
