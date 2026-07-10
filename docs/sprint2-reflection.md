# Sprint 2 回顧 —— OpenStack-Helm:兩天就停,但停得值得

> Sprint 2(2026-04-21 起)是三次嘗試中最短的一次:規劃了 8 堂課,實際走了 2 天就主動喊停。這篇回顧誠實記錄「為什麼開始、做到哪、為什麼停、學到什麼」。

## 一句話總結

**用兩天的成本驗證了「在 K8s 上跑 OpenStack」對單人單機 lab 是錯的架構選擇,並因此找到了對的(Kolla-Ansible)——以試錯而言,這是划算的一筆。**

## 當時的決策脈絡

Sprint 1(Juju + Charms)因 charm 生態斷代收場後,[Sprint 2 課程計畫](sprint2-course-plan.md)選了 OpenStack-Helm 路線,理由寫得很清楚:

- 已經會 K8s / Helm / kubectl,可以省掉學 Ansible
- OpenStack-Helm 的編排本質與 K8s 原生能力對齊
- 當時甚至明確寫下「**不走 Kolla-Ansible**」

事後看,這個決策的盲點是:**用「我會什麼工具」選路線,而不是用「這個場景需要什麼」選路線**。

## 實際做到哪

| 進度 | 內容 | 紀錄 |
|---|---|---|
| Day 0 | 沿用 Azure VM,清掉 Sprint 1 殘留 | [sprint2-day-0-azure-vm](runbook/sprint2-day-0-azure-vm.md) |
| Day 0 | kubeadm 單節點 K8s + CNI + MetalLB + local-path | [sprint2-day-0-k8s-bootstrap](runbook/sprint2-day-0-k8s-bootstrap.md) |
| Course 1 | OpenStack 基礎依賴上 K8s:MariaDB / RabbitMQ / Memcached / OVS(Helm chart) | [sprint2-course-1-osh-infra](runbook/sprint2-course-1-osh-infra.md) |
| Course 2–8 | Keystone 以降的 OpenStack 本體 | **未執行,在此喊停** |

兩天內踩的 5 個雷(詳見 course-1 runbook):`openstack-helm-infra` repo 已 retire 併回主 repo、node 不打 label 全部 FailedScheduling、`helm dep build` 必跑、libvirt init container 依賴 neutron 無限等待、MariaDB 11.4 的 client 改名。

## 為什麼喊停

還沒碰到 OpenStack 本體,光部「資料庫和訊息佇列」就已經體感到這條路線的結構性代價:

1. **除錯永遠跨兩層**:一個 pod 起不來,要先分辨是 K8s 的問題(排程、label、init container、PV)還是 OpenStack 的問題(設定、依賴順序)。兩層的知識都要在腦中同時運轉。
2. **chart 生態的漂移**:repo 重組、chart 預設值落後、image tag 要逐一覆蓋——這些維護負擔在生產環境有專職團隊吸收,單人 lab 全部自己扛。
3. **學習目標被稀釋**:目標是學 OpenStack,結果一半時間在跟 Helm chart 和 K8s 排程搏鬥。

當下的判斷:繼續走完 8 堂課「做得到」,但每一步的摩擦都會是 course-1 的重演。**及早停損,把力氣留給對的架構。**

## 學到什麼(讓 Sprint 3 成功的養分)

1. **部署工具的複雜度要匹配場景**:OSH 適合「已有 K8s 維運團隊的生產環境」;單人 lab 要的是透明、單層、可直接看原始碼的工具——這直接推導出 Sprint 3 的 Kolla-Ansible。
2. **「會什麼工具」不該決定「走什麼路線」**:Sprint 2 因為會 K8s 而選 OSH、Sprint 3 反而因為場景需要而回頭學 Ansible——後者才是對的順序。
3. **及早停損是有效策略**:2 天的沉沒成本換掉可能 2 週的錯路。判斷依據不是「做不做得到」,是「每一步的摩擦會不會複利」。
4. K8s bootstrap 的手感(kubeadm / CNI / MetalLB)沒有白練——Sprint 3 Day 6 用 kind 立 CAPI management cluster 時全部用得上。

## 給未來的你

如果哪天真的要在生產環境評估 OpenStack-Helm:它不是爛工具,它是**別人的好工具**——前提是你有一個本來就在維運 K8s 的團隊,而且 OpenStack 只是他們平台上的其中一個工作負載。單人、學習、單機,三個條件踩中任何兩個,都請直接用 Kolla-Ansible。
