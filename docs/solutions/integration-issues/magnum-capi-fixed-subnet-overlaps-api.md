# Magnum CAPI:cluster 卡 CREATE_IN_PROGRESS —— node 子網與 OpenStack API IP 撞號,CCM 連不到 Keystone

**Sprint 3 / Day 8(2026-07-09)· Kolla Epoxy 單機 AIO + magnum-cluster-api v0.37.0 / CAPO v0.14.4**

## 症狀

- `openstack coe cluster create` 後,cluster 長時間 `CREATE_IN_PROGRESS` 不前進(>20 分)
- Nova VM(master/worker)都 `ACTIVE`,但 CAPI Machine 停在 `Provisioned`(非 `Running`)
- KubeadmControlPlane 條件:`Initialized=True`(kubeadm init 明明成功)但 `Waiting for a Node with spec.providerID openstack:///<uuid> to exist`、`NodeStartupTimeout`
- 進 workload cluster 看:兩節點其實都 **Ready**,但 **providerID 是空的**、帶 `node.cloudprovider.kubernetes.io/uninitialized:NoSchedule` taint;`openstack-cloud-controller-manager` **CrashLoopBackOff**;coredns / CSI-controller / calico-kube-controllers 全 `Pending`(排不進被 taint 的節點)

## 根因

現代 CAPO 走 **external cloud provider**:kubelet 以 uninitialized taint 註冊,**必須由 OpenStack CCM 補上 `providerID` 並移除 taint**。CAPI 用 providerID 配對 Machine↔Node,所以 **CCM 掛掉 = 整座 cluster 卡死**。

CCM crash 的真因:它的 `cloud.conf` `auth-url=http://10.0.0.4:5000`(Keystone 在 host 的 Azure IP 10.0.0.4),而 magnum-cluster-api driver 的 **`fixed_subnet_cidr` 預設 `10.0.0.0/24`**(`resources.py`)。節點拿到 `10.0.0.x`,把 **`10.0.0.4` 當成自己同網段的 on-link 位址** → 直接 ARP(而非送去 gateway 路由)→ 撈不到真正的 Keystone(它在 host,不在 tenant overlay)→ CCM 連線失敗、crash。

這是**單機 Kolla + Azure(或任何 API endpoint 落在某 /24)**的結構性撞號:tenant 網路預設 CIDR 剛好蓋住 API endpoint IP。

## 診斷關鍵路徑

1. cluster 卡住 → 進 workload 看 `kubectl get nodes`:節點 Ready 但 `-o jsonpath='{..providerID}'` 空 → 問題在 CCM
2. `kubectl -n kube-system get pods | grep cloud-controller` → CrashLoopBackOff
3. 看 cloud.conf:`auth-url=http://10.0.0.4:5000`;看 tenant subnet:`openstack subnet list --network <cluster-net>` → `10.0.0.0/24`
4. 10.0.0.4 ∈ 10.0.0.0/24 → 撞號確認

## 解法

建 cluster 時用 label **`fixed_subnet_cidr`** 指定不與任何關鍵網段重疊的節點子網:

```bash
openstack coe cluster template create <name> ... \
  --label fixed_subnet_cidr=10.6.0.0/24 \
  ...
# 或 cluster create 時 --label 覆蓋
```

**選 CIDR 要避開的清單**(本 lab):
| 網段 | 用途 |
|---|---|
| `10.0.0.0/24` | ❌ host / OpenStack API endpoint(10.0.0.4)|
| `10.1.0.0/24` | Octavia lb-mgmt-net |
| `10.100.0.0/16` | CAPI calico pod CIDR(driver 預設)|
| `10.254.0.0/16` | K8s service CIDR(driver 預設)|
| `172.24.4.0/24` | ext-net |
| `172.17/172.18` | kind docker network |
→ 本 lab 採 **`10.6.0.0/24`**,全數避開。

換 CIDR 需重建 cluster(子網在建立時就定),`fixed_subnet_cidr` 改了要 `cluster delete` + 重新 `create`。

## 驗證修好

```bash
# 新 cluster 建置中,盯 CCM 與 providerID:
KUBECONFIG=/tmp/wl.kubeconfig kubectl -n kube-system get pod | grep cloud-controller   # Running(0 重啟)
KUBECONFIG=/tmp/wl.kubeconfig kubectl get nodes -o jsonpath='{..providerID}'           # openstack:///... 有值
# → taint 移除 → coredns/CSI 排得進 → cluster CREATE_COMPLETE
```

## 教訓

- **CCM 是 CAPO cluster 的單點命脈**:卡在「node 沒 providerID」時,先看 CCM pod 狀態,再看它 cloud.conf 的 auth-url 能不能從節點網段路由到。
- **單機/lab 上,tenant 網段預設值常撞 API endpoint IP** —— 任何把 OpenStack API 放在 `10.0.0.x` 的環境,都要把 workload 節點子網移開該 /24。
- 症狀「VM 都 ACTIVE、cluster 卻不 COMPLETE」時,問題幾乎都在 **VM 內部 / K8s 層**(kubeadm、CNI、CCM),而非 OpenStack 基礎設施層——要進 workload cluster 用 CAPI 存的 kubeconfig(`kubectl -n magnum-system get secret <cluster>-kubeconfig`)診斷。
