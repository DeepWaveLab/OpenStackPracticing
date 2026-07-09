# Day 3: Vault TLS 供應（OVN 專用路徑）

## 目的

用 **Vault 1.8/stable**（舊版 OpenStack-charmers Vault）給 **OVN stack** 提供 TLS 憑證，繞過 `self-signed-certificates` 與 OVN charm 的 **tls-certificates interface 版本不相容** 問題。

## 為什麼不能用 self-signed-certificates？

- `self-signed-certificates` (1/stable) 用 **新版 tls-certificates v3 interface**
- OVN 22.03/stable charm 家族用 **舊版 tls-certificates legacy interface**
- interface 名字相同但資料格式完全不同 — SSC 收到 CSR 但不會回應

**症狀**：ovn-central 卡在 "certificates awaiting server certificate data"，SSC log 完全沒在處理

## 為什麼用 Vault 1.8/stable 而不是 1.16/stable

```
vault charm 版本分界：
- 1.8 及以下：OpenStack-charmers 維護，支援 legacy tls-certificates
- 1.15 及以上：HashiCorp 維護 K8s operator 風格，新 interface

我們 OVN 需要 legacy → 一定用 1.8/stable
```

## Prerequisites

- mysql-innodb-cluster active (Vault 的 shared-db backend)
- self-signed-certificates 拿掉對 ovn-central、ovn-chassis、neutron-api-plugin-ovn 的 relation

## Step 1: 拔掉 SSC 對 OVN 的 relation

```bash
juju remove-relation ovn-central:certificates self-signed-certificates:certificates
juju remove-relation ovn-chassis:certificates self-signed-certificates:certificates
juju remove-relation neutron-api-plugin-ovn:certificates self-signed-certificates:certificates
```

## Step 2: Deploy Vault 1.8/stable

```bash
juju deploy vault --channel=1.8/stable
juju integrate vault:shared-db mysql-innodb-cluster:shared-db
```

## Step 3: Integrate Vault 給 OVN charm

```bash
juju integrate vault:certificates ovn-central:certificates
juju integrate vault:certificates ovn-chassis:certificates
juju integrate vault:certificates neutron-api-plugin-ovn:certificates
```

## Step 4: 等 Vault 到 "needs to be initialized"

```bash
juju wait-for application vault --query='status=="blocked"' --timeout=15m
```

預期 message: `Vault needs to be initialized`

## Step 5: Init + Unseal + Authorize Charm

Vault 跑在 **HTTP**（不是 HTTPS！）port 8200。

```bash
# 取 Vault unit IP
VAULT_IP=$(juju status vault --format=json | python3 -c "
import sys,json
d=json.load(sys.stdin)
print(list(d['applications']['vault']['units'].values())[0]['public-address'])
")
VAULT_ADDR=http://${VAULT_IP}:8200

# 1) Initialize
INIT=$(curl -s --request POST \
  --data '{"secret_shares":1,"secret_threshold":1}' \
  ${VAULT_ADDR}/v1/sys/init)

ROOT_TOKEN=$(echo "$INIT" | python3 -c "import sys,json; print(json.load(sys.stdin)['root_token'])")
UNSEAL_KEY=$(echo "$INIT" | python3 -c "import sys,json; print(json.load(sys.stdin)['keys'][0])")

# ⚠️ 立刻存檔，弄丟就重來
cat > ~/vault-secrets <<EOF
VAULT_ADDR=${VAULT_ADDR}
ROOT_TOKEN=${ROOT_TOKEN}
UNSEAL_KEY=${UNSEAL_KEY}
EOF
chmod 600 ~/vault-secrets

# 2) Unseal
curl -s --request POST \
  --data "{\"key\":\"${UNSEAL_KEY}\"}" \
  ${VAULT_ADDR}/v1/sys/unseal

# 3) Authorize charm (讓 charm 能用 Vault API)
juju run vault/leader authorize-charm token=${ROOT_TOKEN}

# 4) Generate self-signed root CA (給 OpenStack 服務簽憑證)
juju run vault/leader generate-root-ca
```

## Step 6: 驗證

```bash
juju status vault
# Message: "Unit is ready (active: true, mlock: disabled)"

juju status ovn-central ovn-chassis neutron-api-plugin-ovn
# 全部 active "Unit is ready"
```

---

## ⚠️ 實測環境 Vault secrets（2026-04-19 初始化）

```
VAULT_ADDR=http://10.254.154.224:8200
ROOT_TOKEN=s.JXyceOQrGIyVAoxJjJ4gRCoZ
UNSEAL_KEY=52fede2979714f940c9c920ce8060fb859c7d56f0dbefa2b13d3c318f7ea93bb
```

**Vault deallocate 重開後要重新 unseal**：
```bash
curl -s --request POST --data '{"key":"52fede2979714f940c9c920ce8060fb859c7d56f0dbefa2b13d3c318f7ea93bb"}' \
  http://10.254.154.224:8200/v1/sys/unseal
```

（也可以設 `juju config vault totally-unsecure-auto-unseal=true` 讓重啟自動 unseal — 學習環境可，生產千萬別。）

---

## Gotchas

### 🔴 Vault 走 HTTP 不走 HTTPS

會下意識 curl https → 得到 SSL WRONG_VERSION_NUMBER。先用 http://。

### 🔴 `self-signed-certificates` 無法給 OVN 用

參見本文開頭。用 Vault 1.8/stable。

### ⚠️ Vault Root Token 存丟後

`~/vault-secrets` 是唯一副本。弄丟要**整個 Vault 重建**（unsealkey 也在同一份）。建議：
```bash
# 備份到 local Mac
scp -i ~/.ssh/juju_id_rsa azureuser@20.210.233.133:~/vault-secrets ~/openstack-lab-vault-secrets
```

---

## Replay

```bash
# Assuming Vault already deployed + integrated
curl -s --request POST --data '{"secret_shares":1,"secret_threshold":1}' http://VAULT_IP:8200/v1/sys/init
# Save ROOT_TOKEN + UNSEAL_KEY
curl -s --request POST --data "{\"key\":\"${UNSEAL_KEY}\"}" http://VAULT_IP:8200/v1/sys/unseal
juju run vault/leader authorize-charm token=${ROOT_TOKEN}
juju run vault/leader generate-root-ca
```
