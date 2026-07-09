# Magnum CAPI:`nodegroup create` 直接 CREATE_FAILED —— failureDomain 為 null 被 CAPI topology 拒

**Sprint 3 / Day 9(2026-07-09)· magnum-cluster-api v0.37.0 / Cluster API v1.13.2(ClusterClass managed topology)**

## 症狀

- `openstack coe nodegroup create <cluster> <ng> --node-count 1 --role app` → nodegroup 立刻 `CREATE_FAILED`
- `openstack coe nodegroup show` 的 status_reason / magnum-conductor.log:

```
Cluster.cluster.x-k8s.io "<cluster>" is invalid:
  spec.topology.workers.machineDeployments[1].failureDomain:
  Invalid value: "null": ... in body must be of type string: "null", <nil>
```

- 第一個(建立 cluster 時就有的)`default-worker` 沒事;**只有事後 `nodegroup create` 新增的那個** MD 踩到。

## 根因

magnum-cluster-api driver 建新 MachineDeployment topology entry 時(`resources.py`):

```python
"failureDomain": node_group.labels.get("availability_zone"),
```

nodegroup 沒有 `availability_zone` label → `.get()` 回 `None` → 序列化成 JSON `null`。而 **Cluster API v1.10+ 的 Cluster topology schema 要求 `failureDomain` 是 string(可省略,但不可為 null)**。建立 cluster 時的 default-worker 走的是另一條路徑、失敗域是 `""`(空字串,合法),所以只有後加的 nodegroup 爆。

這是 driver 對「沒指定 AZ 的 nodegroup」的 null 處理與新版 CAPI topology 驗證不相容。

## 解法

建 nodegroup 時明確給 `availability_zone` label,讓 `failureDomain` 變成合法字串:

```bash
# 先確認 compute AZ 名稱
openstack availability zone list --compute -f value -c "Zone Name"   # 通常 nova

openstack coe nodegroup create <cluster> ng-app --node-count 1 --role app \
  --labels availability_zone=nova
```

驗證 topology 的 failureDomain 變合法字串:

```bash
KUBECONFIG=~/.kube/config kubectl -n magnum-system get cluster <cluster> \
  -o jsonpath='{range .spec.topology.workers.machineDeployments[*]}{.name}{" fd="}{.failureDomain}{"\n"}{end}'
# default-worker fd=      (空字串,合法)
# ng-app         fd=nova  (合法字串)
```

## 教訓

- **在 magnum-cluster-api 上加 nodegroup,永遠帶 `--labels availability_zone=<az>`**,即使單 AZ,也避免 failureDomain=null。
- `failureDomain` 值必須是 CAPO 有登記的 failure domain(= Nova compute AZ);單機 lab 用預設 `nova`。
- 症狀是「CREATE_FAILED + CAPI topology schema 錯誤」時,看 magnum-conductor.log 的 pykube HTTPError 會直接指出是哪個欄位 invalid —— 比猜快。
