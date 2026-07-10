# 排錯手冊

實作中真實撞到、**官方文件查不到**的問題與解法。每篇固定格式:**症狀 → 根因 → 解法**(多數附診斷方法與預防措施)。

!!! tip "怎麼用這本手冊"
    1. **拿錯誤訊息搜**:複製你看到的錯誤關鍵字(如 `MissingAuthPlugin`、`No valid host`),用站內搜尋或 `grep -rn "關鍵字" docs/solutions/`。
    2. **按症狀瀏覽**:下表的「症狀」欄就是你會看到的現象,掃一眼找相似的。
    3. 每篇都標注了它發生在課程的哪一天,可以對照 runbook 的脈絡讀。

## 本課程(Kolla-Ansible + Magnum CAPI)的地雷

| 症狀 | 根因一句話 | 出處 |
|---|---|---|
| [mariadb 容器起不來,port 3306 被佔](integration-issues/kolla-proxysql-mariadb-port-conflict.md) | ProxySQL 與 MariaDB 的 port 衝突 | Day 1 |
| [octavia-interface.service 起不來,`dhclient` 執行檔不存在](integration-issues/kolla-octavia-dhclient-noble.md) | Ubuntu 24.04 移除了 isc-dhcp-client,Kolla 的 unit 還寫死 dhclient | Day 4 |
| [LB 永遠卡 `PENDING_CREATE`,amphora 沒開機](integration-issues/kolla-octavia-redis-jobboard.md) | amphora driver 需要 Redis jobboard,Kolla 預設不開 Redis | Day 4 |
| [kind 叢集所有 image pull `i/o timeout`](integration-issues/kind-kolla-docker-iptables-masquerade.md) | Kolla 把 docker 設 `iptables:false`,kind 的對外 NAT 因此不存在 | Day 6 |
| [K8s cluster 卡 `CREATE_IN_PROGRESS`,CCM CrashLoop](integration-issues/magnum-capi-fixed-subnet-overlaps-api.md) | 節點子網預設 `10.0.0.0/24` 撞到 OpenStack API 的 IP,節點連不到 Keystone | Day 8 |
| [`nodegroup create` 直接 CREATE_FAILED,`failureDomain: null`](integration-issues/magnum-capi-nodegroup-failuredomain-null.md) | 沒帶 `availability_zone` label,driver 塞 null 被 CAPI 拒收 | Day 9 |

## 歷史路線(Juju + Charms,Sprint 1)的地雷

> 這批發生在[前一次嘗試](../previous-attempts.md)的 Juju/Charm 部署。**本課程的 Kolla 路線不會撞到**,保留原因:它們記錄了「charm 生態為什麼撐不起這個 lab」的第一手證據,也是任何還在評估 Juju 路線的人最需要的情報。

| 症狀 | 根因一句話 | 嚴重度 |
|---|---|---|
| [OVN charm 不認 caracal、TLS 介面不相容](integration-issues/ovn-charms-caracal-release-mismatch.md) | charm channel 斷代,需 Vault 1.8 + 手動 patch charmhelpers | 🔴 |
| [`shared-db` 接了沒反應,keystone 等不到 DB](integration-issues/mysql-charm-not-compatible-with-openstack.md) | Canonical `mysql` charm 宣稱支援 shared-db 實際不行,要用 `mysql-innodb-cluster` | 🔴 |
| [nova-compute 跟 placement 不通](integration-issues/nova-compute-caracal-cloud-credentials.md) | Caracal 起 nova-compute 必須接 `cloud-credentials` relation,文件沒寫 | 🔴 |
| [VM 開機 ERROR:`Unknown auth type: None`](integration-issues/nova-compute-missing-neutron-auth.md) | charm 忘了把 `[neutron]` 寫進 nova.conf(charm bug),手動補 | 🔴 |
| [VM 開機 `PortBindingFailed`,chassis 沒註冊](integration-issues/ovn-version-mismatch-chassis-not-registering.md) | ovn-controller 24.03 對上 northd 22.03,版本不相容 | 🔴 |
| [`security group` API 回 404](integration-issues/neutron-security-groups-disabled-default.md) | neutron-api charm 預設 `neutron-security-groups=false` | 🟡 |
| [Heat stack 建立失敗,domain 錯誤](integration-issues/heat-missing-domain-setup-action.md) | heat charm 不自動建 keystone domain,必須手跑 `domain-setup` action | 🟡 |
| [magnum API `Connection reset`](integration-issues/magnum-api-bind-mismatch-with-haproxy.md) | magnum-api 綁 127.0.0.1 但 haproxy backend 指 container IP | 🟡 |
| [cirros SSH 連線被拒 / Ubuntu 拿不到 SSH key](runtime-errors/cirros-sshd-race-vs-ubuntu-config-drive.md) | cirros sshd 啟動 race;Ubuntu 要用 `--config-drive True` | 🟡 |

## 維護這個網站本身的地雷

> 這批不是 OpenStack 的問題,是把這個 repo 做成公開教學網站的過程中撞到的。留給任何想把內部 repo 轉公開、或架同款 docs 站的人。

| 症狀 | 根因一句話 | 出處 |
|---|---|---|
| [repo 要轉公開,但歷史 commit 裡有真實 IP / 訂閱 ID / 密碼](security-issues/git-history-scrub-sensitive-data-filter-repo.md) | 改 HEAD 不夠,敏感資料活在歷史;需 filter-repo 全歷史抹除,且有四個非顯而易見的地雷 | 轉公開前 |

## 跨越三次嘗試的共同教訓

1. **地雷幾乎都在元件邊界**(網路 NAT、版本相容、資源計帳),不在單一元件內部——除錯先想「這兩個東西的交界」。
2. **以裝好的原始碼與實際錯誤為準**,不要信部落格文章(常是舊版做法)。
3. **除錯先分層**:host 通不通 → 容器通不通 → 服務通不通;每層用最小指令驗證,一路排除比猜快。
