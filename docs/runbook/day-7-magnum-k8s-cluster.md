# Day 7: Magnum K8s Cluster 實建

![Magnum 官方吉祥物](../assets/mascots/magnum.png){ align=right width="110" }

## 目的

用 Magnum 從零 provision 一個 K8s cluster：FCOS VM → kubeadm → 2 nodes Ready → `kubectl run`。驗證 Magnum+Heat+Nova+Neutron+OVN 整條鏈。

## 最終結果（2026-04-20）

- **Heat stack CREATE_COMPLETE** ✅
- **master + worker VM 跑 K8s 1.23.3 on FCOS 38** ✅
- **2 nodes Ready**（透過 Calico CNI 取代壞掉的 flannel） ✅
- **kubectl get nodes 正常** ✅
- pod workload 跑不起來（Magnum 內建 k8s-keystone-auth webhook image 壞，apiserver TokenReview call webhook 401 → SA token 無法簽發）

## 步驟（濃縮版，7 個 fix 都在裡）

### 1. Fix Magnum config（不用 Barbican）

```bash
juju config magnum cert-manager-type=x509keypair cluster-user-trust=true
```

### 2. Fix force_config_drive 讓 FCOS ignition 讀 ISO 不 race

```bash
juju config nova-compute config-flags='force_config_drive=True'
# ⚠️ 不是 'always'（Caracal bool 化了）
# charm 會刷掉 nova.conf 的 [neutron] patch，要 re-patch（見 day-4 solution）
```

### 3. Fix ovn-metadata-agent 找不到 Nova

```bash
juju ssh nova-compute/2 -- "
  sudo sed -i '/^\[DEFAULT\]/a nova_metadata_host = 10.254.154.207\nnova_metadata_port = 8775' /etc/neutron/neutron_ovn_metadata_agent.ini
  sudo sed -i 's|metadata_proxy_shared_secret=.*|metadata_proxy_shared_secret=$(juju ssh nova-cloud-controller/0 -- sudo grep metadata_proxy_shared_secret /etc/nova/nova.conf | awk -F= \"{print \\\$2}\" | tr -d \" \")|' /etc/neutron/neutron_ovn_metadata_agent.ini
  sudo systemctl restart neutron-ovn-metadata-agent
"
```

### 4. Download FCOS 38（不是最新，最新沒 Docker）

```bash
curl -sSL -O https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/38.20231014.3.0/x86_64/fedora-coreos-38.20231014.3.0-openstack.x86_64.qcow2.xz
unxz fedora-coreos-38.20231014.3.0-openstack.x86_64.qcow2.xz

source ~/openrc
openstack image create --disk-format qcow2 --container-format bare --public \
  --file ~/fedora-coreos-38.20231014.3.0-openstack.x86_64.qcow2 \
  --property os_distro=fedora-coreos fedora-coreos-38
```

### 5. 建 flavor & cluster template

```bash
openstack flavor create --ram 2048 --disk 15 --vcpus 1 --public k8s.small

openstack coe cluster template create k8s-small \
  --image fedora-coreos-38 \
  --external-network ext_net \
  --keypair lab_key \
  --coe kubernetes \
  --flavor k8s.small --master-flavor k8s.small \
  --network-driver flannel \
  --dns-nameserver 8.8.8.8 \
  --docker-storage-driver overlay2
# ⚠️ 不加 --volume-driver cinder（我們沒部署 cinder）
# ⚠️ 不加 --docker-volume-size（一樣原因）
```

### 6. 建 cluster

```bash
openstack coe cluster create my-k8s \
  --cluster-template k8s-small \
  --master-count 1 --node-count 1
# 等 15-18 分鐘（Heat 建 VM + ignition + bootstrap + kubeadm）
```

### 7. 進 master 換 CNI（flannel image 死了）

```bash
FIP=$(openstack coe cluster show my-k8s -f value -c master_addresses | tr -d "[]'")
ssh -i ~/lab_key.pem core@$FIP

sudo /srv/magnum/bin/kubectl --kubeconfig /etc/kubernetes/admin.conf \
  -n kube-system delete ds kube-flannel-ds

sudo /srv/magnum/bin/kubectl --kubeconfig /etc/kubernetes/admin.conf \
  apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/calico.yaml

# 30-60 秒後 master + worker 變 Ready
sudo /srv/magnum/bin/kubectl --kubeconfig /etc/kubernetes/admin.conf get nodes
```

### 8. 清 cloud-provider uninitialized taint（CCM image 壞，沒人清）

```bash
sudo /srv/magnum/bin/kubectl --kubeconfig /etc/kubernetes/admin.conf \
  taint nodes my-k8s-xxx-node-0 \
  node.cloudprovider.kubernetes.io/uninitialized:NoSchedule-
```

## Known Limitations（本 lab 跑不過去的 Magnum upstream 問題）

Magnum Caracal 的 `k8s_fedora_coreos_v1` driver 內嵌一堆 5 年前的 image URL：

| Image | 現況 | 影響 |
|---|---|---|
| `quay.io/coreos/flannel-cni:v0.3.0` | 401 Unauthorized（namespace 2022 遷走） | flannel 裝不起來（已改 Calico） |
| `openstackmagnum/openstack-cloud-controller-manager:XXX` | ImagePullBackOff | node 一直有 uninitialized taint（手動清） |
| `openstackmagnum/k8s-keystone-auth:XXX` | ImagePullBackOff | kube-apiserver 的 TokenReview webhook call 失敗 → SA token 簽不出 → pod 的 `kube-api-access-*` volume mount 卡死 → workload 跑不起來 |

**結論**：cluster provisioning 層全通、K8s 本體跑，但 Magnum driver 的 default manifest 太舊，實際部署 workload 要額外手工修。

## 下一步建議

- 若只需要學 OpenStack + K8s 整合：到這就夠了
- 若要實際用 K8s on OpenStack：換 [cluster-api-provider-openstack](https://github.com/kubernetes-sigs/cluster-api-provider-openstack)
- 或 fork magnum 把 driver image URL 改成現行版（upstream 已有 PR 討論）

## 環境速查

```
cluster uuid:    de1820ba-b8b3-4268-904b-35e142d4d8ab
master floating: 172.16.200.180
worker floating: 172.16.200.196
SSH:             ssh -i ~/lab_key.pem core@<fip>
kubectl path:    /srv/magnum/bin/kubectl
kubeconfig:      /etc/kubernetes/admin.conf (on master)
```

拉 kubeconfig 回本機：
```bash
openstack coe cluster config my-k8s --dir ~/my-k8s-config
export KUBECONFIG=~/my-k8s-config/config
kubectl get nodes
```
