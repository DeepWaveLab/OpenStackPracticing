# Sprint 3 / Day 1: Kolla-Ansible 原理 + AIO 部署 core 服務

> 課程定位:理解 Kolla-Ansible 的部署模型,並在 Azure lab VM 上部出 OpenStack core(Keystone / Glance / Nova / Neutron-OVN / Horizon)。
> **刻意不含 Cinder** —— Day 3 用 `reconfigure` 增量加入,順便學「已上線環境加服務」的工作流。

## 原理與架構

### 1. Kolla vs Kolla-Ansible

- **Kolla**:把每個 OpenStack 服務打包成 Docker image(quay.io/openstack.kolla/*)
- **Kolla-Ansible**:用 Ansible 把這些 image 部到主機、產生設定、兜成一朵雲

部署模型:**每服務一個 container、host network mode、沒有 K8s**。設定檔由「內建 template + 你在 `/etc/kolla/` 的覆寫」merge 產生。與 Sprint 1 的對照:charm 的 relation 在這裡不存在,服務間的連接資訊(DB 帳密、rabbit URL、keystone endpoint)全部由 kolla-ansible 在部署期算好寫進每個服務的 conf —— 這是「編排期綁定」vs charm 的「執行期協商」。

### 2. 版本選擇:2025.1 Epoxy(SLURP)

kolla-ansible 版號 20.x = OpenStack 2025.1。從 git stable branch 裝(`stable/2025.1`)比猜 pip 版號區間可靠。Ubuntu 24.04 host 在支援矩陣內。

### 3. 本 lab 的 AIO 網路設計(Azure 限制下)

| globals.yml 設定 | 值 | 為什麼 |
|---|---|---|
| `network_interface` | `eth0` | 管理/API/tunnel 全走這張(AIO 單網卡);另一張 `enP51424s1` 是 accelerated networking 的 SR-IOV VF,不碰 |
| `neutron_external_interface` | `dummy0` | Kolla 要求一張「沒有 IP」的網卡給 br-ex;Azure VM 只有一張真網卡,標準解法是造一張 dummy。後果:external network 只在 host 內部有效(Azure 無 L2,見 Day 0 原理章) |
| `enable_haproxy` | `"no"` + VIP=`10.0.0.4`(host IP) | 單節點不需要 HA;關掉 haproxy/keepalived 後 VIP 必須等於 host IP。生產 multinode 這裡會是真 VIP |
| `neutron_plugin_agent` | `ovn` | 沿用 Sprint 1 學過的 OVN 知識(logical router/switch、localnet) |
| `nova_compute_virt_type` | `kvm` | Day 0 已驗證 `/dev/kvm`(預設值,寫明是課程用意) |
| MTU | 不動(預設 1500) | Neutron 會自動幫 Geneve tenant network 扣 overhead;Azure eth0 MTU=1500,physnet 預設即正確 |

## 步驟

### 1. 裝依賴 + kolla-ansible(venv)

```bash
sudo apt-get install -y git python3-venv python3-dev libffi-dev gcc libssl-dev
python3 -m venv ~/kolla-venv
source ~/kolla-venv/bin/activate        # 之後所有 kolla 指令都要在 venv 內跑(見踩坑 #1)
pip install -U pip
pip install "ansible-core>=2.16,<2.18"
pip install git+https://opendev.org/openstack/kolla-ansible@stable/2025.1
kolla-ansible --version                 # 20.4.x = Epoxy
kolla-ansible install-deps              # 裝 ansible galaxy collections(openstack.kolla 等)
pip install docker dbus-python          # prechecks 需要(見踩坑 #2/#3;dbus-python 要先 apt 裝
sudo apt-get install -y pkg-config libdbus-1-dev libdbus-glib-1-dev   # ← 這個)
```

### 2. 組態

```bash
sudo mkdir -p /etc/kolla && sudo chown $USER:$USER /etc/kolla
cp -r ~/kolla-venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla/
cp ~/kolla-venv/share/kolla-ansible/ansible/inventory/all-in-one ~/all-in-one
```

dummy0(立即生效 + 開機持久化):

```bash
sudo ip link add dummy0 type dummy && sudo ip link set dummy0 up
sudo tee /etc/systemd/network/10-dummy0.netdev <<EOF
[NetDev]
Name=dummy0
Kind=dummy
EOF
sudo tee /etc/systemd/network/10-dummy0.network <<EOF
[Match]
Name=dummy0

[Network]
LinkLocalAddressing=no
ConfigureWithoutCarrier=yes
EOF
```

globals.yml 附加區塊(附在檔尾,原檔全註解不動,diff 一眼可見):

```yaml
kolla_base_distro: "ubuntu"
kolla_internal_vip_address: "10.0.0.4"
enable_haproxy: "no"
network_interface: "eth0"
neutron_external_interface: "dummy0"
neutron_plugin_agent: "ovn"
nova_compute_virt_type: "kvm"
```

```bash
kolla-genpwd    # 生成 /etc/kolla/passwords.yml 全部密碼
```

### 3. bootstrap → prechecks → deploy

```bash
kolla-ansible bootstrap-servers -i ~/all-in-one   # 裝 Docker engine 等(~3 分)
kolla-ansible prechecks -i ~/all-in-one           # 要全綠才繼續
# deploy 跑很久,用 nohup 讓 SSH 斷線也不中斷:
nohup bash -c "source ~/kolla-venv/bin/activate && kolla-ansible deploy -i ~/all-in-one" \
  > ~/kolla-deploy.log 2>&1 &
tail -f ~/kolla-deploy.log
```

### 4. post-deploy + CLI

```bash
kolla-ansible post-deploy -i ~/all-in-one   # 產生 /etc/kolla/clouds.yaml + admin-openrc.sh
pip install python-openstackclient
source /etc/kolla/admin-openrc.sh
openstack service list
```

從 Mac 看 Horizon(external network 出不了 host,一律 SSH tunnel):

```bash
ssh -L 8080:10.0.0.4:80 -i ~/.ssh/juju_id_rsa azureuser@203.0.113.10
# 瀏覽器開 http://localhost:8080,帳號 admin,密碼:
grep keystone_admin_password /etc/kolla/passwords.yml
```

## Checkpoint(全數通過 2026-07-08)

| 驗證 | 判準 | 實測 |
|---|---|---|
| deploy PLAY RECAP | failed=0 | ✅ ok=367 changed=214 failed=0(第二輪;第一輪死於踩坑 #4) |
| `openstack service list` | keystone/glance/placement/nova/neutron 註冊 | ✅ 7 個服務(Heat 是 Epoxy 預設啟用,白撿,Day 5 直接用) |
| `openstack compute service list` | nova-compute State=up | ✅ scheduler/conductor/compute 全 up |
| `openstack network agent list` | OVN agents Alive | ✅ OVN Controller Gateway + OVN Metadata 皆 `:-)` |
| `docker ps` | 全部 Up | ✅ 33 個 container,無 Exited/Restarting |
| Horizon | HTTP 有回應 | ✅ `curl http://10.0.0.4/` → 302(login redirect) |

部署耗時參考:第一輪(含拉 image)約 12 分鐘死於 mariadb;修正後第二輪(image 已在本地)約 25 分鐘全綠。

## 踩坑

### 1. `install-deps` 靜默失敗:`[Errno 2] No such file or directory: 'ansible-galaxy'`

直接呼叫 `~/kolla-venv/bin/kolla-ansible install-deps`(不 activate venv)會找不到 `ansible-galaxy` —— 它是用 PATH 找的。**教訓:kolla 指令一律在 `source ~/kolla-venv/bin/activate` 之後跑。** 症狀:bootstrap-servers 報 `roles: - { role: openstack.kolla.baremetal` 解析錯誤(collection 根本沒裝)。

### 2. prechecks:`No module named 'docker'`

bootstrap-servers 只裝 Docker **engine**,venv 裡的 Docker **Python SDK** 要自己 `pip install docker`。

### 3. prechecks:`No module named 'dbus'`

`dbus-python` 是原始碼編譯套件,要先 `apt install pkg-config libdbus-1-dev libdbus-glib-1-dev` 再 `pip install dbus-python`。

### 4. MariaDB bootstrap 失敗:proxysql 佔走 3306(Epoxy 預設值陷阱)

症狀:`Wait for first MariaDB service port liveness` timeout,`docker ps -a` 看到 `mariadb Exited (1)`,mariadb log 顯示 daemon 有啟動但隨即退出;`ss -tlnp | grep 3306` 發現是 **proxysql** 在聽。

原因:**Epoxy 的 `enable_proxysql` 預設 `yes`**(MariaDB 前端 LB),與 `enable_haproxy` 是獨立開關。AIO 設 `enable_haproxy: "no"` + VIP=host IP 後,proxysql 綁 VIP:3306、mariadb 綁 host:3306 —— 同一個 IP:port 相撞,mariadb bind 失敗。

修法:

```bash
echo 'enable_proxysql: "no"' >> /etc/kolla/globals.yml
sudo docker rm -f proxysql haproxy mariadb
sudo docker volume rm mariadb     # DB 還沒資料,砍掉讓 bootstrap 重來最乾淨
kolla-ansible deploy -i ~/all-in-one   # 重跑
```

教訓:關 haproxy 做 AIO 時,**haproxy 和 proxysql 要一起關**。

## 下一步(Day 2)

OpenStack 資源全流程:project/user/quota → image → OVN network/router → VM 開機 + SSH。
