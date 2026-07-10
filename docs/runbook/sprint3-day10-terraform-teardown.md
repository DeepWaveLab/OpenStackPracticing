# Sprint 3 / Day 10: Terraform 接管 + Teardown 演練(Sprint 收尾)

> 課程定位:Sprint 3 最後一天。用 **terraform-provider-openstack**(Sprint 1 未竟項)以 HCL 管 network/VM/LB,並整理 **teardown/重建**程序。回顧另見 `docs/sprint3-reflection.md`。

## 原理與架構

### 1. Terraform vs Heat:兩種 IaC 的定位

![Terraform 官方標誌](../assets/logos/terraform.png){ align=right width="90" }

| | Heat(Day 5)| **Terraform(本日)** |
|---|---|---|
| 狀態 | 存在 heat DB(OpenStack 內) | 外部 tfstate(檔案/後端)|
| 範圍 | 只管 OpenStack | 跨雲/跨 provider(同一份 workflow 管 OpenStack + AWS + DNS…)|
| 介面 | HOT(YAML)| HCL + `plan`/`apply`/`destroy` |
| 適用 | OpenStack 原生編排、Magnum 舊 driver | 企業 IaC 標準、多雲、CI 整合 |

Terraform 的價值:**`plan` 先看 diff 再 `apply`**、狀態可版控、一份 HCL 管全棧。provider `terraform-provider-openstack/openstack`(v3.x)認證直接吃 `OS_*` 環境變數(`source admin-openrc.sh`)。

### 2. Teardown/重建 = disaster recovery 驗證

IaC 的終極測試:**能不能一鍵拆、一鍵重建**。本 lab 分兩層:
- **TF 層**(本日實作):`terraform destroy` 把 HCL 管的資源一次清乾淨。
- **Kolla 層**(本日僅記錄程序,見 §Teardown):`kolla-ansible destroy` 拆整套 OpenStack,再 `deploy` 重建 —— 極破壞性,實務上是換機/災難重建的演練。

## 步驟(Terraform,已實作)

### 1. 裝 terraform

```bash
VER=$(curl -s https://checkpoint-api.hashicorp.com/v1/check/terraform | python3 -c "import sys,json;print(json.load(sys.stdin)['current_version'])")
curl -fsSLo /tmp/tf.zip "https://releases.hashicorp.com/terraform/${VER}/terraform_${VER}_linux_amd64.zip"
python3 -c "import zipfile; zipfile.ZipFile('/tmp/tf.zip').extractall('/tmp')"
sudo install -m0755 /tmp/terraform /usr/local/bin/terraform    # v1.15.8
```

### 2. HCL(`~/tf-demo/main.tf`)

```hcl
terraform {
  required_providers {
    openstack = { source = "terraform-provider-openstack/openstack", version = "~> 3.0" }
  }
}
provider "openstack" {}                       # 讀 OS_* env

data "openstack_networking_network_v2" "extnet" { name = "ext-net" }

resource "openstack_networking_network_v2" "tf_net"    { name = "tf-net" }
resource "openstack_networking_subnet_v2"  "tf_subnet" {
  name = "tf-subnet"; network_id = openstack_networking_network_v2.tf_net.id
  cidr = "10.20.0.0/24"; ip_version = 4; dns_nameservers = ["8.8.8.8"]
}
resource "openstack_networking_router_v2" "tf_router" {
  name = "tf-router"; admin_state_up = true
  external_network_id = data.openstack_networking_network_v2.extnet.id
}
resource "openstack_networking_router_interface_v2" "tf_ri" {
  router_id = openstack_networking_router_v2.tf_router.id
  subnet_id = openstack_networking_subnet_v2.tf_subnet.id
}
resource "openstack_compute_instance_v2" "tf_vm" {
  name        = "tf-vm"
  depends_on  = [openstack_networking_router_interface_v2.tf_ri]   # ← 見踩雷#1
  image_name  = "cirros"; flavor_name = "m1.tiny"; key_pair = "k8s-admin"
  network { uuid = openstack_networking_network_v2.tf_net.id }
}
resource "openstack_lb_loadbalancer_v2" "tf_lb" {
  name = "tf-lb"; vip_subnet_id = openstack_networking_subnet_v2.tf_subnet.id
  loadbalancer_provider = "amphora"
}
resource "openstack_lb_listener_v2" "tf_listener" {
  name = "tf-listener"; protocol = "HTTP"; protocol_port = 80
  loadbalancer_id = openstack_lb_loadbalancer_v2.tf_lb.id
}
```

### 3. apply / verify / destroy

```bash
source /etc/kolla/admin-openrc.sh
terraform init
terraform plan          # 7 to add
terraform apply -auto-approve      # LB amphora ~56s、VM ~20s
terraform output        # tf_vm_ip / tf_lb_vip
terraform destroy -auto-approve    # 一鍵拆 7 資源(含 amphora)
```

## Checkpoint(全數通過 2026-07-09)

| 驗證 | 判準 | 實測 |
|---|---|---|
| provider auth | OS_* env 認證成功 | ✅ |
| apply | network/subnet/router/VM/LB/listener 全建 | ✅ 7 資源;LB amphora ACTIVE |
| outputs | VM/LB IP 取得 | ✅ tf_vm_ip=10.20.0.46 / tf_lb_vip=10.20.0.234 |
| **teardown** | `terraform destroy` 清乾淨、OpenStack 端 0 殘留 | ✅ 7 destroyed |

## 踩雷

### 1. instance vs subnet 的 ordering(terraform-provider-openstack 經典地雷)

`openstack_compute_instance_v2` 只用 `network { uuid = ... }` 引用 network、**沒引用 subnet**,TF 會並行建、VM 送到 Nova 時 subnet 還沒掛上 → `Network ... requires a subnet in order to boot instances on` (HTTP 400)。修:給 instance 加 `depends_on = [openstack_networking_router_interface_v2.tf_ri]`(或 subnet),強制順序。這是 provider 的 dependency graph 抓不到隱性依賴的典型情況。

### 2. provider 認證

不必在 HCL 寫死帳密 —— `provider "openstack" {}` 空 block 直接吃 `OS_AUTH_URL`/`OS_USERNAME`/`OS_PASSWORD`/`OS_PROJECT_NAME`/`OS_*_DOMAIN_NAME` env(`source admin-openrc.sh`)。生產環境用 `clouds.yaml` + `cloud = "..."` 或 application credential 更佳。

## Teardown 程序(僅記錄,本次未執行)

> 決策:TF 層 teardown 已完整示範;full `kolla-ansible destroy` 會清掉整套 OpenStack + workload cluster,重建 ~30-60 分且多花 VM 費用,故本次只記錄程序供未來災難重建演練參考。

**完整拆除 + 重建步驟:**

```bash
# 0. 先備份會遺失的狀態(node image 已在 Glance repo 外、magnum kubeconfig、globals.yml)
cp /etc/kolla/globals.yml ~/globals.backup.yml

# 1. 拆整套 OpenStack(清所有容器 + volumes + 資料;不可逆)
kolla-ansible destroy -i ~/all-in-one --yes-i-really-really-mean-it

# 2. 重建(~30-60 分,依服務數)
kolla-ansible bootstrap-servers -i ~/all-in-one   # 若 host 也被清
kolla-ansible deploy -i ~/all-in-one              # core + octavia + barbican + magnum

# 3. 重建後必做的 post-steps(這些不在 kolla deploy 範圍內):
#    - Octavia 憑證: kolla-ansible octavia-certificates -i ~/all-in-one(Day 4)
#    - 重傳 amphora / node image 到 Glance(Day 4/7)
#    - 重建 CAPI mgmt cluster(kind + clusterctl init + ORC,Day 6)——kind 資料不隨 kolla 走
#    - magnum kubeconfig 換新 kind API port,重放 /etc/kolla/config/magnum/kubeconfig(Day 7)
#    - kind MASQUERADE / octavia o-hm0 / disk_allocation_ratio 等 host 級設定(有 systemd unit / config 的會自動回)
```

**重建速度的教訓**:Kolla 的 `deploy` 本身是冪等、可重跑的,但 lab 有一堆**手動 host 級設定與外部狀態**(kind cluster、MASQUERADE、image、kubeconfig)不在 Kolla 管轄內 —— 真正的「一鍵重建」要把這些也 IaC 化(這正是 Terraform/Ansible 該補的一層)。

## Sprint 3 完結

十天走完:Azure lab → Kolla AIO → OpenStack 全資源流 → Cinder/Octavia/Barbican/Heat → Magnum CAPI driver → E2E workload cluster(Sprint 1 三個未完成項全數補完)→ Day-2 ops → Terraform。

還有一篇不用動手的[後日談:OpenStack 服務全景圖](sprint3-day11-openstack-service-map.md),把這次用過與沒用過的服務一次盤點;回顧與產業對照見 [Sprint 3 回顧](../sprint3-reflection.md)。

---

*Terraform 標誌為 HashiCorp 之商標,此處作社群教學用途。*
