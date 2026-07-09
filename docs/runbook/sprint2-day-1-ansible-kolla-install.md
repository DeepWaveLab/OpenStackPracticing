# Sprint 2 / Day 1: Ansible 概念 + Kolla-Ansible 安裝

## 目的

Sprint 2 的 paradigm 是 Ansible-based deployment。本日：
1. 搞懂 Ansible 4 個核心物件（inventory / playbook / module / vars）
2. 跑 3 個實例理解體感
3. 裝 Kolla-Ansible 工具鏈 + collection
4. 看 Kolla 帶了什麼東西進來

## Part 1: Ansible 概念（給新手的鋪陳）

### 什麼是 Ansible

用 YAML 描述「host 該長怎樣」，然後 SSH 進去把它們變成那樣的工具。**Agentless**（目標 host 不用裝東西）、**push-based**（從你的 controller 推出去）、**idempotent**（重跑不壞）。

### 跟其他工具對比

| | Shell | Juju | Terraform | Ansible |
|---|---|---|---|---|
| 描述風格 | 命令式 | 宣告式（持續 reconcile） | 宣告式（plan/apply） | 混合（task 命令式、整體宣告式） |
| Agent | 無 | **需要**（juju-agent） | 無 | **無**（SSH + Python 就行） |
| 重跑 | 不安全 | 安全 | 安全（diff） | 安全（idempotent） |
| 連線 | SSH | Juju RPC | API call | SSH |

### 4 個核心物件

1. **Inventory** — 哪些 host、怎麼分組（`.ini` 或 YAML）
2. **Playbook** — 對哪些 host 跑什麼 task（YAML）
3. **Module** — 每個 task 用的「做事單位」（`apt` / `file` / `copy` / `service` / ...幾百個）
4. **Variables** — `group_vars/` / `host_vars/` / `-e` / playbook 內 `vars:`

### 最常用 3 指令

```bash
# Ad-hoc
ansible -i inventory.ini all -m ping
ansible -i inventory.ini all -m shell -a 'uptime'

# Playbook
ansible-playbook -i inventory.ini site.yml

# Dry-run
ansible-playbook -i inventory.ini site.yml --check --diff
```

## Part 2: 實跑 3 個例子

### Inventory（示範用）

```ini
# ~/demo-inventory.ini
[myhosts]
localhost ansible_connection=local
```

### Playbook（示範用）

```yaml
# ~/demo-playbook.yml
---
- name: Demo playbook
  hosts: myhosts
  become: yes
  tasks:
    - name: 建 /tmp/ansible-demo
      copy:
        dest: /tmp/ansible-demo
        content: 'hello from ansible\n'
        mode: '0644'

    - name: 讀回來
      command: cat /tmp/ansible-demo
      register: result

    - name: 印
      debug:
        msg: 'Got: {{ result.stdout }}'
```

### 執行結果摘要

```
1. ping：SUCCESS, changed=false
2. shell uptime：CHANGED, 印 uptime
3. playbook：changed=2（建檔案 + 讀檔）
4. playbook 第二次跑：changed=0（idempotent，檔案已在）
```

## Part 3: Kolla-Ansible 安裝

### 為什麼用 Dalmatian (2024.2) 不是 Caracal (2024.1)

Kolla 18.x (Caracal) 的 `requirements.yml` 指向 `ansible-collection-kolla` 的 `stable/2024.1` branch，但該專案最早的 stable branch 是 `stable/2024.2`。18.x 的 packaging 層 bug。

解：直接用 kolla-ansible 19.x (Dalmatian)，對應 OpenStack 2024.2 release。

### 指令

```bash
# 1. Python venv 隔離
python3 -m venv ~/kolla-venv
source ~/kolla-venv/bin/activate
pip install -U pip

# 2. 裝 kolla-ansible 19.x（會連帶拉 ansible-core 2.17）
pip install 'kolla-ansible==19.*'

# 3. 拉 Kolla 需要的 Ansible collections
kolla-ansible install-deps

# 4. 驗證
kolla-ansible --version   # 19.7.0
ansible --version         # ansible-core 2.17.14
ansible-galaxy collection list | grep kolla   # openstack.kolla 1.0.0
```

### Kolla 帶來的東西

```
~/kolla-venv/share/kolla-ansible/
├── ansible/
│   ├── roles/              # 71 個 role（每個對應一個 OpenStack service / util）
│   │   ├── aodh / barbican / ceilometer / cinder / 
│   │   ├── designate / etcd / glance / haproxy-config /
│   │   ├── heat / horizon / ironic / iscsi /
│   │   ├── keystone / magnum / manila / mariadb /
│   │   ├── memcached / neutron / nova / octavia /
│   │   ├── placement / rabbitmq / redis / ...
│   │   └── (共 71 個)
│   ├── site.yml            # 主 playbook，按順序 include 每個 role
│   ├── group_vars/         # 變數預設值
│   └── library/            # custom modules
├── etc_examples/kolla/     # sample config
│   ├── globals.yml         # 主要你會改的 — 33KB，幾百個選項
│   └── passwords.yml       # 服務密碼
└── tools/                  # helper scripts
```

### 複製 sample 設定到 /etc/kolla/

```bash
sudo mkdir -p /etc/kolla
sudo cp -rf ~/kolla-venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla/
sudo chown -R $USER:$USER /etc/kolla

ls /etc/kolla/
# globals.yml   (33 KB，要大量客製)
# passwords.yml (4.9 KB，用 kolla-genpwd 自動填)
```

## 你會下的常用 kolla 指令（預習）

```bash
kolla-ansible bootstrap-servers   # 把 host 準備好（裝 Docker / 系統設定）
kolla-ansible prechecks           # dry-run 型檢查
kolla-ansible pull                # 預先拉 Docker image
kolla-ansible deploy              # 正式部署 OpenStack
kolla-ansible post-deploy         # 生成 admin-openrc.sh
kolla-ansible reconfigure         # 改 config 後重新 apply
kolla-ansible upgrade             # 升級到新版
kolla-ansible destroy             # 拆掉 lab
```

**每一條本質是 `ansible-playbook` 跑對應的 playbook**。kolla-ansible 只是 wrapper 讓你少打字。

## 狀態（Day 1 結束時）

| 項目 | |
|---|---|
| Ansible 2.10.8（系統） + 2.17.14（venv） | ✅ |
| kolla-ansible 19.7.0 | ✅ |
| openstack.kolla 1.0.0 | ✅ |
| /etc/kolla/globals.yml 初稿（預設值） | ✅ |
| /etc/kolla/passwords.yml 初稿（預設值） | ✅ |
| 71 個 role 在 venv 裡準備好 | ✅ |

## 下一步

Day 2：**改 globals.yml 跟 passwords.yml**，決定要 enable 哪些服務，設 network interface 這些。33KB 的 globals.yml 會一行一行挑你關心的來改，其他留 default。
