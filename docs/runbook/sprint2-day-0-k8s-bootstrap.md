# Sprint 2 / Pre-req: K8s bootstrap（Day 0 環境準備）

![Kubernetes 官方標誌](../assets/logos/kubernetes.png){ align=right width="90" }

## 目的

Sprint 2 走 OpenStack-Helm，需要一個可用的 K8s cluster。本步驟在 Azure VM 上建 single-node K8s + 必要周邊（CNI / Helm / MetalLB / local-path-provisioner）。

**這是 pre-req，不是課程 (1)**。課程 (1) = openstack-helm-infra（下一個 runbook）。

## 清掉前次 Kolla 路線殘留

```bash
ssh -i ~/.ssh/juju_id_rsa azureuser@203.0.113.20
rm -rf ~/kolla-venv ~/demo-inventory.ini ~/demo-playbook.yml
sudo rm -rf /etc/kolla ~/.ansible
sudo apt-get -y purge ansible
sudo apt-get -y autoremove
```

## 1. Kernel pre-req

K8s 要求：
- swap off（kubelet 不跟 swap 共存）
- `br_netfilter` + `overlay` kernel module
- bridge sysctl + ip_forward

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

## 2. 裝 containerd（K8s 的 container runtime）

```bash
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```

**`SystemdCgroup = true`** 必設，不然 kubelet 跟 containerd 的 cgroup driver 不一致，kubeadm 會抱怨。

## 3. 裝 kubeadm / kubelet / kubectl 1.30

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

本次裝到 `v1.30.14`。

## 4. kubeadm init

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=10.0.0.4
```

成功訊息有 `Your Kubernetes control-plane has initialized successfully!`

## 5. Kubeconfig + 去除 control-plane taint

```bash
mkdir -p ~/.kube
sudo cp -i /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# 單節點要拔 control-plane taint，否則沒人能排 pod 到 master
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

## 6. 裝 Cilium CNI

```bash
# CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -sL https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz \
  | sudo tar xzf - -C /usr/local/bin

# Install Cilium 1.19
cilium install --version v1.19.3 --set ipam.mode=kubernetes
cilium status --wait --wait-duration 5m
```

## 7. 裝 Helm v3

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version --short   # v3.20.2
```

## 8. 裝 MetalLB

OpenStack-Helm 的 service 需要 LoadBalancer type IP，Azure VM 本身不給（它是 cloud VM，不是 bare metal），要 MetalLB 自己分 IP。

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml

kubectl wait --for=condition=Available deploy/controller -n metallb-system --timeout=180s
kubectl rollout status ds/speaker -n metallb-system --timeout=180s

# IP Pool：用 172.24.4.0/24（OpenStack-Helm 社群慣用的 VIP range）
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: openstack-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.24.4.200-172.24.4.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: openstack-l2
  namespace: metallb-system
EOF
```

**VIP 只在 VM 內部可達**（Azure 不給 L2 ARP），要從外面打 VIP 需 SSH tunnel。

## 9. 裝 local-path-provisioner + 設 default

OpenStack-Helm 的 MariaDB / RabbitMQ / etcd 都要 PV。單節點 lab 用 local-path 寫到 host filesystem。

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.28/deploy/local-path-storage.yaml

kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

## 10. 驗證測試

```bash
# test pod + PVC + LoadBalancer service
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests: { storage: 1Gi }
---
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  labels:
    run: test-pod
spec:
  containers:
  - name: test
    image: nginx:alpine
    volumeMounts:
    - { name: data, mountPath: /data }
  volumes:
  - name: data
    persistentVolumeClaim: { claimName: test-pvc }
---
apiVersion: v1
kind: Service
metadata:
  name: test-lb
spec:
  type: LoadBalancer
  selector:
    run: test-pod
  ports:
    - port: 80
EOF

kubectl wait --for=condition=Ready pod/test-pod
kubectl get pvc  # Bound
kubectl get svc test-lb   # EXTERNAL-IP 172.24.4.200
IP=$(kubectl get svc test-lb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -sSI http://$IP/   # HTTP/1.1 200 OK

# cleanup
kubectl delete pod test-pod svc test-lb pvc test-pvc
```

## 狀態（Pre-req 完成時）

| 項目 | 版本 / 狀態 |
|---|---|
| Ubuntu | 22.04 (kernel 6.8) |
| containerd | 活躍 (SystemdCgroup=true) |
| K8s cluster | v1.30.14 single-node |
| CNI | Cilium v1.19.3 (kubernetes IPAM) |
| Helm | v3.20.2 |
| CNI | Cilium 1.19.3 |
| MetalLB | v0.14.8 with L2 pool 172.24.4.200-250 |
| StorageClass | local-path (default) |
| 測試 Pod + PVC + LoadBalancer | ✅ curl VIP 回 200 |

## ⚠️ 本階段 Bare Metal 層的模擬

Sprint 2 single-node K8s on Azure VM。Sprint 3 會變：

| Sprint 2（本階段） | Sprint 3（未來） |
|---|---|
| `kubeadm init` 單節點 | 多節點 K8s cluster，control-plane 跟 worker 分開 |
| Azure VM 當 K8s host | Foreman 管的 bare metal 當 K8s host |
| MetalLB L2 只在 VM 內部 | MetalLB BGP 對 ToR switch 宣告 VIP |
| local-path-provisioner 本機寫檔 | Rook-Ceph 或外部 storage 給 PV |
| 單節點無 HA | control-plane 3 個 node HA |

## 下一步

課程 (1) — `openstack-helm-infra` deploy（下一個 runbook `sprint2-course-1-osh-infra.md`）
