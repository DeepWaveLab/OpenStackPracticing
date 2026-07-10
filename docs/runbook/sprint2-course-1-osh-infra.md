# Sprint 2 / Course (1): openstack-helm-infra 部署

## 目的

部 OpenStack 的「依賴基礎設施」到 K8s：DB / MQ / cache / SDN data plane。
OpenStack 本身的服務（keystone / nova / neutron）留 course (2)+。

## 2026 年 repo 變更重要說明

**`openstack-helm-infra` 已 retire 併回 `openstack-helm`**。
- 看 `git log` 發現 `openstack-helm-infra` 最新 commit = `Retire openstack-helm-infra repository`
- 所有 infra chart（helm-toolkit / mariadb / rabbitmq / memcached / openvswitch / libvirt / ovn / ceph-*）現在都在 `openstack-helm` repo 裡

新的 repo 結構：
```
openstack-helm/
├── helm-toolkit/       ← 共用模組（所有 chart 都 dep 這個）
├── mariadb/            ← infra
├── rabbitmq/           ← infra
├── memcached/          ← infra
├── openvswitch/        ← infra
├── libvirt/            ← infra（但課程 3 才部，依賴 neutron agent）
├── ovn/                ← infra（OVN data plane 若用）
├── ceph-*/             ← storage
├── keystone/           ← service
├── nova/               ← service
└── ... 共 88 個 chart
```

## 本課部的 4 個 infra chart

| Chart | 角色 |
|---|---|
| `mariadb` | OpenStack 服務共用的 DB |
| `rabbitmq` | AMQP message bus |
| `memcached` | Keystone token cache |
| `openvswitch` | Neutron data plane 底層 OVS |

**libvirt 從課程 (1) 拿掉**：它 init container 依賴 `neutron-ovs-agent` pod，要 neutron 部好才能起來，移到課程 (3)。

## 步驟

### 1. Clone repo + helm-toolkit dep

```bash
mkdir -p ~/osh
cd ~/osh
git clone https://opendev.org/openstack/openstack-helm.git
cd openstack-helm

helm dep update helm-toolkit
helm package helm-toolkit
# 產出 helm-toolkit-2026.1.0.tgz
```

### 2. Helm plugin（osh wrapper）+ jq

```bash
helm plugin install https://opendev.org/openstack/openstack-helm-plugin
sudo apt-get install -y jq
```

### 3. Namespace + node labels

OpenStack-Helm 用 K8s node label 做排程選擇。單節點 lab 全部打：

```bash
kubectl create namespace openstack
kubectl label namespace openstack name=openstack --overwrite

kubectl label node openstack-lab \
  openstack-control-plane=enabled \
  openstack-compute-node=enabled \
  openstack-network-node=enabled \
  openstack-storage-node=enabled \
  openvswitch=enabled \
  linuxbridge=enabled \
  ovn-controller=enabled \
  l3-agent=enabled \
  dhcp-agent=enabled \
  metadata-agent=enabled \
  --overwrite
```

**不打這些 label，pod 會 FailedScheduling** — chart 預設都有 `nodeSelector: openstack-control-plane=enabled` 之類。

### 4. Build chart dep（每個 chart 要先做）

```bash
cd ~/osh/openstack-helm
for c in mariadb rabbitmq memcached openvswitch; do
  helm dep build $c
done
```

`helm dep build` 讀 chart 的 `Chart.yaml` `dependencies` 把 helm-toolkit 拉進對應 `charts/` 目錄。

### 5. Deploy MariaDB（單節點）

```bash
helm upgrade --install mariadb ./mariadb \
  --namespace=openstack \
  --set pod.replicas.server=1 \
  --set volume.enabled=false \
  --set volume.use_local_path_for_single_pod_cluster.enabled=true \
  --set monitoring.prometheus.enabled=false \
  --timeout=600s
```

- `pod.replicas.server=1`：單節點 lab 不做 Galera 3 replica
- `volume.enabled=false` + `volume.use_local_path_for_single_pod_cluster.enabled=true`：用 host path 當 PV
- `monitoring.prometheus.enabled=false`：lab 不裝 monitoring

等 `mariadb-server-0` 變 `1/1 Running`（~1-2 min）。

### 6. Deploy RabbitMQ

```bash
helm upgrade --install rabbitmq ./rabbitmq \
  --namespace=openstack \
  --set pod.replicas.server=1 \
  --set volume.enabled=false \
  --set volume.use_local_path.enabled=true \
  --timeout=600s
```

### 7. Deploy Memcached

```bash
helm upgrade --install memcached ./memcached \
  --namespace=openstack \
  --timeout=300s
```

Memcached 是 stateless，沒 volume 議題。

### 8. Deploy OpenvSwitch

```bash
helm upgrade --install openvswitch ./openvswitch \
  --namespace=openstack \
  --timeout=300s
```

`openvswitch` chart 是 DaemonSet（每個 node 一份 OVS）。pod 有兩個 container：`openvswitch-db`（ovsdb-server）+ `openvswitch-vswitchd`。

### 9. 驗證

```bash
$ kubectl get pods -n openstack
NAME                                 READY   STATUS      RESTARTS   AGE
mariadb-controller-8874fc6cb-jvhnh   1/1     Running     0          12m
mariadb-server-0                     1/1     Running     0          12m
memcached-memcached-0                1/1     Running     0          3m47s
openvswitch-xmk4g                    2/2     Running     0          3m47s
rabbitmq-cluster-wait-gfgbh          0/1     Completed   0          5m30s
rabbitmq-rabbitmq-0                  1/1     Running     0          5m30s
```

MariaDB 登入測試：
```bash
kubectl exec -n openstack mariadb-server-0 -c mariadb -- \
  mariadb --defaults-file=/etc/mysql/admin_user.cnf -e "SHOW DATABASES;"
# Database
# information_schema
# mysql
# performance_schema
# sys
```

## 踩到的 5 個雷

### #1 openstack-helm-infra 已 retire

本以為要 clone 兩個 repo，實際只剩 `openstack-helm` 一個。文件跟舊 tutorial（2023 以前）會誤導，**以 repo 實際狀況為準**。

### #2 MariaDB Pending 沒人排程

原因：chart 預設 `nodeSelector: openstack-control-plane=enabled`。沒打 label 的 node 完全沒 pod 匹配。
解：為 node 打 OpenStack-Helm 慣用 label 組（10+ 個）。

### #3 helm dep build 必跑

`Chart.yaml` 寫 `dependencies: file://../helm-toolkit`。沒跑 `helm dep build` 的話 `helm install` 會噴 `missing in charts/ directory: helm-toolkit`。

### #4 libvirt init 等 neutron-ovs-agent 無限卡

libvirt chart 的 kubernetes-entrypoint init container 會等 `application=neutron,component=neutron-ovs-agent` 的 pod 出現在同一個 node 才通。Neutron 還沒部，等永遠。**libvirt 移到課程 (3) 跟 Nova/Neutron 一起**。

### #5 mariadb client 不叫 mysql 叫 mariadb

MariaDB 11.4 image 裡 binary 是 `/usr/bin/mariadb` 不是 `mysql`（mysql wrapper 還有但命名變）。`mysql` command 會找不到。用：
```bash
kubectl exec ... -- mariadb --defaults-file=/etc/mysql/admin_user.cnf -e "..."
```

## ⚠️ Sprint 3 Foreman 對應

| Sprint 2 （本）| Sprint 3（未來 Foreman） |
|---|---|
| 單節點 K8s，所有 infra pod 同台 | 3+ node K8s，mariadb / rabbit 排到 control-plane node |
| local-path-provisioner | Rook-Ceph 或外部 storage 的 CSI |
| `pod.replicas.server=1` | 3 replica（mariadb Galera、rabbitmq HA） |
| 單一 node label | Foreman 部不同角色 node 打不同 label（storage-node / compute-node / network-node） |

## 下一步

**Course (2)**：`openstack-helm` 核心服務
- keystone（identity）
- glance（image）
- placement（resource tracking）
- horizon（UI）
