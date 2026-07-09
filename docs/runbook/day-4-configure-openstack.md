# Day 4: Configure OpenStack — 開第一台 VM

## 目的

官方 [Configure OpenStack](https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/configure-openstack.html) 章節：建 flavor、external network、tenant network、router、keypair、security group，最後 `openstack server create` 開第一台 VM，綁 floating IP，從 Azure VM 用 SSH 進去。

## 2026-04-19 執行結果

| 階段 | 狀態 |
|---|---|
| flavor `cirros.tiny` | ✅ |
| external network `ext_net`（flat / physnet1） | ✅ |
| ext_subnet 172.16.200.0/24 | ✅ |
| br-ex OVS bridge + ovn-bridge-mappings | ✅ 手動建 |
| tenant_net + tenant_subnet 192.168.21.0/24 | ✅ |
| lab_router（ext_gateway + tenant subnet） | ✅ |
| keypair `lab_key` + security group `lab_sg` | ✅ |
| VM `vm1` ACTIVE | ✅（經歷 3 個坑：見 solutions） |
| Floating IP 172.16.200.163 綁定 | ✅ |
| Azure VM → VM 路由 + ping | ✅ |
| **SSH 進 cirros VM (vm1)** | ❌ cirros 0.6 sshd race |
| **SSH 進 Ubuntu VM (vm2)** | ✅ Ubuntu jammy + `--config-drive True` 成功 |

## 踩到的 3 個坑（都有 solution doc）

1. [neutron-security-groups 預設關](../solutions/integration-issues/neutron-security-groups-disabled-default.md) — `/v2.0/security-groups` 回 404
2. [nova.conf 少 `[neutron]` section](../solutions/integration-issues/nova-compute-missing-neutron-auth.md) — `Unknown auth type: None`
3. [OVN version mismatch 導致 chassis 不註冊](../solutions/integration-issues/ovn-version-mismatch-chassis-not-registering.md) — `PortBindingFailed`

## 完整 replay 指令

```bash
source ~/openrc

# 1. Flavor（比官方的 m1.small 小，適合 4vCPU/8GB lab）
openstack flavor create --ram 256 --disk 1 --vcpus 1 --public cirros.tiny

# 2. External network（flat + physnet1）
juju config neutron-api flat-network-providers=physnet1  # 預設關
openstack network create --external \
  --provider-network-type flat \
  --provider-physical-network physnet1 ext_net
openstack subnet create --network ext_net \
  --subnet-range 172.16.200.0/24 \
  --allocation-pool start=172.16.200.100,end=172.16.200.200 \
  --gateway 172.16.200.1 --no-dhcp ext_subnet

# 3. OVN bridge（單機 lab 要手動；charm 不吃 dummy NIC）
juju ssh nova-compute/2 -- 'sudo ip link add dum0 type dummy; sudo ip link set dum0 up'
juju ssh nova-compute/2 -- '
  sudo ovs-vsctl --may-exist add-br br-ex
  sudo ovs-vsctl --may-exist add-port br-ex dum0
  sudo ovs-vsctl set open . external_ids:ovn-bridge-mappings=physnet1:br-ex'

# 4. br-ex 當 Linux gateway + NAT 讓 VM 上網
juju ssh nova-compute/2 -- '
  sudo ip addr add 172.16.200.1/24 dev br-ex
  sudo ip link set br-ex up
  sudo sysctl -w net.ipv4.ip_forward=1
  sudo iptables -t nat -A POSTROUTING -s 172.16.200.0/24 ! -d 172.16.200.0/24 -j MASQUERADE'

# 5. Tenant network + router
openstack network create tenant_net
openstack subnet create --network tenant_net --subnet-range 192.168.21.0/24 --dns-nameserver 8.8.8.8 tenant_subnet
openstack router create lab_router
openstack router set --external-gateway ext_net lab_router
openstack router add subnet lab_router tenant_subnet

# 6. Keypair + security group（先打開 charm option 否則 secgroup 404）
juju config neutron-api neutron-security-groups=True
openstack keypair create lab_key > ~/lab_key.pem && chmod 600 ~/lab_key.pem
openstack security group create lab_sg
openstack security group rule create --proto icmp lab_sg
openstack security group rule create --proto tcp --dst-port 22 lab_sg

# 7. 關掉 OVN 版本匹配 + 重註冊 chassis（必要，除非升級 ovn-central）
juju ssh nova-compute/2 -- '
  sudo ovs-vsctl set open . external_ids:ovn-match-northd-version=false
  sudo systemctl restart ovn-controller
  sleep 5
  sudo ovs-vsctl set open . external_ids:ovn-bridge-mappings=physnet1:br-ex'  # 重設會被清掉

# 8. nova.conf 補 [neutron] section（charm bug）
juju ssh nova-compute/2 -- '... see solution doc ...'
juju ssh nova-compute/2 -- sudo systemctl restart nova-compute

# 9. Azure VM 加路由（floating IP 可達）
sudo ip route add 172.16.200.0/24 via 10.254.154.221

# 10. 開 VM
openstack server create --flavor cirros.tiny --image cirros \
  --network tenant_net --key-name lab_key --security-group lab_sg vm1 --wait

# 11. Floating IP
FIP=$(openstack floating ip create ext_net -f value -c floating_ip_address)
openstack server add floating ip vm1 $FIP
ping -c 3 $FIP    # 應該通
```

## Ubuntu VM（跟官方指南一致）

cirros 有 sshd race，改用官方的 Jammy cloud image 才能 SSH：

```bash
# 下載 + 上傳
curl -sSL -O https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
openstack image create --disk-format qcow2 --container-format bare --public \
  --file ~/jammy-server-cloudimg-amd64.img jammy

# Ubuntu flavor（≥1 GB RAM、≥5 GB disk）
openstack flavor create --ram 1024 --disk 5 --vcpus 1 --public ubuntu.small

# 建 VM 一定要加 --config-drive True（否則 cloud-init 可能 race 169.254.169.254，拿不到 key）
openstack server create --flavor ubuntu.small --image jammy \
  --network tenant_net --key-name lab_key --security-group lab_sg \
  --config-drive True vm2 --wait

FIP=$(openstack floating ip create ext_net -f value -c floating_ip_address)
openstack server add floating ip vm2 $FIP

# ~60 秒後 cloud-init 完成，SSH 進去
ssh -i ~/lab_key.pem ubuntu@$FIP
```

驗證（2026-04-19 實測）：
```
vm2 192.168.21.78 / floating 172.16.200.190
default via 192.168.21.1 dev ens2        ← OVN router
curl https://ubuntu.com/ → HTTP/2 200    ← MASQUERADE 出網成功
```

## 下次進度

1. 換 Ubuntu 22.04 cloud image（官方指南用的）→ cloud-init 正確處理 DHCP race → SSH 應該成功
2. 把 nova.conf `[neutron]` patch 打入一個 charm-idempotent 的 hook（例如 systemd drop-in，或找對應 charm option）
3. 思考後續 Magnum 是否要 Cinder（persistent volume）

## Environment 速查

### VM
| 項目 | 值 |
|---|---|
| VM 名 | vm1 |
| Instance ID | `<openstack server show vm1 -c id>` |
| 內網 IP | 192.168.21.152 |
| Floating IP | 172.16.200.163 |
| Keypair private key | `~/lab_key.pem`（Azure VM 上） |
| SSH target | `cirros@172.16.200.163`（目前 sshd race 失敗） |

### Horizon Dashboard

從 Mac 本機起 SSH tunnel：
```bash
ssh -i ~/.ssh/juju_id_rsa -L 8443:10.254.154.136:443 azureuser@20.210.233.133
```
瀏覽器打 `https://localhost:8443/horizon`（接受自簽證書警告）。

| 欄位 | 值 |
|---|---|
| URL | `https://localhost:8443/horizon`（tunnel 走 443；走 80 會因 `SESSION_COOKIE_SECURE=True` 登入後被踢） |
| Domain | `admin_domain` |
| User Name | `admin` |
| Password | `eiSeeV1oich0yie0`（= `~/openrc` 的 `OS_PASSWORD`，keystone charm 自產） |

### OpenStack CLI（在 Azure VM 上）

```bash
ssh -i ~/.ssh/juju_id_rsa azureuser@20.210.233.133
source ~/openrc   # 自動填 OS_AUTH_URL / OS_USERNAME / OS_PASSWORD / OS_PROJECT_NAME 等
openstack token issue    # 試 auth
```

### 其他機密（vault，僅此次 lab 有效）

| 項目 | 值 |
|---|---|
| VAULT_ADDR | `http://10.254.154.224:8200` |
| ROOT_TOKEN | `s.JXyceOQrGIyVAoxJjJ4gRCoZ` |
| UNSEAL_KEY | `52fede2979714f940c9c920ce8060fb859c7d56f0dbefa2b13d3c318f7ea93bb` |
| 備份位置 | Azure VM `~/vault-secrets` |
