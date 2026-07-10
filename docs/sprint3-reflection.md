# Sprint 3 回顧 —— Kolla-Ansible + Magnum CAPI

> Sprint 3(2026-07-07 起,Azure lab)完整回顧。對照 `docs/reflection.md`(Sprint 1)與 `docs/sprint2-course-plan.md`(Sprint 2 OSH),本 sprint **推翻了兩個舊決定,並把 Sprint 1 的三個未完成項全部完成**。

## 一句話總結

**Sprint 1 做不成的三件事(Octavia / Cinder LVM / Magnum),換成 Kolla-Ansible + Magnum CAPI driver 路線後全部完成,並一路做到 workload cluster 的升版、node group、autoscaler,最後用 Terraform 接管。**

## Sprint 時程實際執行

| Day | 主題 | 成果 |
|---|---|---|
| 0 | Azure lab | E16s_v5 nested-virt VM、成本 guardrails |
| 1 | Kolla AIO core | Keystone/Glance/Nova/Neutron(OVN)/Horizon deploy 成功 |
| 2 | OpenStack 資源全流程 | 純 CLI 開出可 SSH 的 VM |
| 3 | Cinder LVM(**未完成項 #1**) | data disk → VG → volume attach、boot from volume |
| 4 | Octavia(**未完成項 #2**) | amphora LB round-robin + OVN provider 對照 |
| 5 | Barbican + Heat | secret 存取往返、HOT stack |
| 6 | CAPI mgmt cluster | kind + clusterctl init + CAPO(v1.13.2/v0.14.4/ORC v2.2.0)|
| 7 | Magnum CAPI driver 整合(**未完成項 #3 前置**) | Kolla 原生 driver 啟用、ClusterTemplate |
| 8 | E2E workload cluster(**未完成項 #3**) | cluster CREATE_COMPLETE、nodes Ready、LoadBalancer→Octavia、PVC→Cinder |
| 9 | Day-2 ops | 升版 v1.34.8→v1.35.5、node group、cluster-autoscaler 觸發 |
| 10 | Terraform + teardown | HCL 管 network/VM/LB、一鍵 destroy;teardown 程序記錄 |

## Sprint 1 三個未完成項:敗因 vs 本次結果

| Sprint 1 敗因(見 reflection.md) | 根因性質 | 本 Sprint 結果 |
|---|---|---|
| **Octavia** Canonical charm amd64 斷代 | 非技術(publish gap) | ✅ Kolla 的 Octavia 是一等公民;Day 4 手動 LB 全流程 + Day 8 K8s `type=LoadBalancer` 拿到 Octavia LB |
| **Cinder LVM** LXD device-mapper 限制 | 環境(容器跑 IaaS) | ✅ 直接跑 Azure VM host;Day 3 LVM + Day 8 PVC 動態供裝 |
| **Magnum** heat driver 內嵌 image URL 失效 | 生態老舊 | ✅ CAPI driver + capo-image-elements 維護的 node image;cluster CREATE_COMPLETE 並可升版/擴縮 |

**reflection.md 當時的結論「Magnum driver 老舊、2026+ 走 CAPO」在本 sprint 被精緻化**:magnum-cluster-api driver 底層就是 CAPO,學它 = 同時學 Magnum API 面與 CAPO 實作面。兩個結論其實相容,不是非此即彼。

## 推翻的兩個舊決定

1. **「Sprint 2 走 OSH 不走 Kolla」** → 本 sprint 證明 Kolla-Ansible 的 Ansible paradigm 對單機/中小規模又快又直觀,除錯負擔遠低於 OSH 的 K8s 兩層。Kolla 社群市佔(reflection.md 自估 30-35%)名副其實。
2. **「Magnum 不用再碰」** → CAPI driver 讓 Magnum 重新可用且與 K8s 生態(CAPI/CAPO/cluster-autoscaler)接軌。

## 確實學到的(與工具無關,通用)

- **external cloud provider 模型**:kubelet 以 uninitialized taint 註冊、CCM 補 providerID —— CCM 是 CAPO cluster 的單點命脈(Day 8 卡死的核心)。
- **CAPI 宣告式 cluster 生命週期**:Cluster/MachineDeployment/Machine、rolling upgrade 的 surge→drain→replace、managed topology(ClusterClass)對 replicas 的所有權。
- **三層互動**:Magnum(API/driver)→ CAPI/CAPO(kind reconcile)→ OpenStack(Nova/Octavia/Cinder 出實體)。
- **Octavia amphora 模型、Cinder CSI 動態供裝、Barbican 憑證倉庫**在真實 K8s 工作負載下的串接。

## 本 sprint 的真實地雷(institutional knowledge)

寫進 `docs/solutions/integration-issues/` 的 6 條,都是文件層級查不到、只有動手才會撞的整合地雷:

| 地雷 | 一句話 |
|---|---|
| `kolla-proxysql-mariadb-port-conflict` | Day 1 proxysql 與 mariadb port 衝突 |
| `kolla-octavia-dhclient-noble` | Noble 移除 isc-dhcp-client,octavia-interface 起不來 |
| `kolla-octavia-redis-jobboard` | amphora provider 需 Redis jobboard,kolla 不自動開 |
| `kind-kolla-docker-iptables-masquerade` | Kolla 設 docker `iptables:false`,斷 kind egress |
| `magnum-capi-fixed-subnet-overlaps-api` | node 子網撞 API IP(10.0.0.4),CCM 連不到 Keystone → cluster 卡死 |
| `magnum-capi-nodegroup-failuredomain-null` | nodegroup 沒 AZ label,failureDomain=null 被 CAPI 拒 |

**共同教訓**:單機 + 公有雲(Azure 無 L2)+ 多套系統共存(Kolla/OVN/docker/kind)的組合,地雷幾乎都在**元件邊界的網路/資源帳/版本相容**,不在單一元件內部。除錯方法論:分層(host→容器→pod)、分類(firewall/NAT vs MTU、config vs 連通性)、以裝好的原始碼/實際錯誤為準而非部落格。

## 成本

- 規格 E16s_v5($1.216/hr)+ P10/P15 磁碟 + Public IP,推薦情境約 **US$101/週**。
- Sprint 3 實作跨數個工作日(Day 5-10 於 2026-07-09 單日長 session 完成),**確切帳單待 Azure Cost Analysis 核對**(Acceptance Criteria 之一,`purpose=lab` tag 可歸戶)。
- 省錢提醒:nested workload cluster + amphora 是額外 VM;不用時 `openstack coe cluster delete` 或直接 `az vm deallocate`(但 host 關機 nested VM 不存活,隔天需重開 cluster)。

## 給未來回來看的你

1. **本次真正的收穫不是「成功」,是「每個地雷都能定位、能修、能寫下來」** —— Sprint 1 死於無法診斷的 charm/image 斷代(黑箱);Sprint 3 的每個失敗都追到根因並留下 solutions。這才是可累積的能力。
2. **Kolla-Ansible 是這類 lab 的甜蜜點**:比 charm 透明、比 OSH 輕。但它只管到 OpenStack 邊界 —— kind/image/kubeconfig/host 網路設定要另外 IaC 化,否則「一鍵重建」是假的(Day 10 teardown 的體悟)。
3. **magnum-cluster-api 版本相容是主要風險面**:driver × CAPI × CAPO × node-image 四者要對齊(用 driver repo 的 `hack/setup-capo.sh` 當 ground truth,別抓 latest)。
4. **下一步分支**(承 reflection.md 的三選項):本 sprint 已把「A. OpenStack 深化」與「B. K8s 生態」接起來了。真要再往前,是 **C. IaC 全棧化**——把本 lab 的 host 級手動設定(kind、MASQUERADE、o-hm0、disk ratio、image、kubeconfig)全部 Terraform/Ansible 化,做到真正的一鍵重建,那才是生產級的 disaster recovery。

## Acceptance Criteria 對照(計畫書)

- [x] RG + guardrails;既有 RG 零接觸(Day 0)
- [x] Kolla AIO(core + Octavia + Barbican + Magnum)deploy 成功(Day 1-8,51 容器)
- [x] 純 CLI 完成 VM/network/volume/LB 全流程(Day 2-4)
- [x] Magnum CAPI cluster:nodes Ready + LoadBalancer(Octavia)+ PVC(Cinder)**三項全過**(Day 8)
- [x] 每個 Day 有 runbook,新地雷寫入 `docs/solutions/`(11 份 runbook + 6 條 solutions)
- [ ] 週成本落在情境 ±20%(待 Azure Cost Analysis 核對)
- [x] `sprint3-reflection.md` 完成(本文件)
