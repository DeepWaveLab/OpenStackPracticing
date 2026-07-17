# Day 32: 全站 TLS——把每個 API 換上 https

> 前 31 天,這朵雲的每一個 API 都走明文 http:token、密碼、資料全裸奔在網路上。生產化的第一件事就是把它們全部換成 https。今天要做的表面上是「加密」,實際上你會發現這件事牽動的是**整個入口架構**——加密只是結果,真正動的是「大門在哪、誰來守」。本章有一半的篇幅在講四次被部署前檢查擋下的失敗,而**那四次擋下全都是對的**。

!!! abstract "你在課程的哪裡"
    - **Day 21**:Prometheus/Grafana 上線,但全部走 http。
    - **今天**:回單機環境,把 haproxy 裝回來當 TLS 終結點,全站 API 換 https,跑完整回歸。
    - **今天之後**:[Day 33](sprint5-day33-keycloak-federation.md) 的企業 SSO 需要 https 的 redirect URI——今天是它的硬前提。

## TLS 要有人終結,而這朵雲把大門拆了

「開 TLS」在 kolla 裡是兩個開關:

```yaml
kolla_enable_tls_internal: "yes"
kolla_enable_tls_external: "yes"
```

但開關背後有個假設:**TLS 由 haproxy 終結**。外部請求打 haproxy 的 https 前端,haproxy 解密後轉給後端服務。看 kolla 原始碼,只有三個 role 認得 TLS 開關——`loadbalancer`(haproxy)、`horizon`、`cloudkitty`,其餘服務自己完全不管加密,全靠 haproxy 在前面擋。

問題來了:本課的單機 lab 從 [Sprint 3](sprint3-day1-kolla-aio-core.md) 起就**關掉了 haproxy**(單機不需要負載平衡,省資源):

```yaml
enable_haproxy: "no"
enable_proxysql: "no"
kolla_internal_vip_address: "10.0.0.4"    ← 直接就是主機 IP
```

**沒有大門,就沒有地方終結 TLS。** 所以今天的第一步不是開 TLS,是**把大門裝回來**——haproxy、proxysql、以及它們需要的 VIP。這是今天最反直覺的地方:一個關於「加密」的任務,第一個動作是重建入口架構。

## 步驟

### 步驟 1:給這朵雲一個 VIP

haproxy 需要一個獨立於主機主 IP 的虛擬 IP(VIP)當前端位址。在 Azure 上這要兩層設定——雲層加 secondary IP,OS 層把它接起來:

```bash
# Azure 層:給網卡加一個 secondary private IP
az network nic ip-config create ... --private-ip-address 10.0.0.250
```

```bash
# OS 層:把 10.0.0.250 接上 eth0
sudo ip addr add 10.0.0.250/32 dev eth0
```

!!! danger "OS 層這一步藏著一顆會擋下整個部署的雷"
    直覺會用 netplan 做持久化,但那會讓 **10.0.0.250 篡位成主機的主 IP**,ansible 於是把 VIP 當成主機位址,部署前檢查直接擋下。正確做法是用 systemd 把它掛成**次要** IP——細節見[地雷 2](#mine-2)。

### 步驟 2:改 globals,把大門與 TLS 一起打開

```yaml
kolla_internal_vip_address: "10.0.0.250"     # 從主機 IP 改成 VIP
enable_haproxy: "yes"                         # 大門裝回來
enable_proxysql: "yes"
kolla_enable_tls_internal: "yes"
kolla_enable_tls_external: "yes"
kolla_copy_ca_into_containers: "yes"          # 讓容器信任自簽 CA
```

### 步驟 3:產生 CA 與憑證

kolla 有一鍵產憑證的指令,會建一個**自簽的內建 CA**(`KollaTestCA`)並用它簽所有服務的憑證:

```bash
kolla-ansible certificates -i ~/all-in-one
```

```text
subject=C = US, ST = NC, L = RTP, OU = kolla
issuer=CN = KollaTestCA
notBefore=Jul 16 08:05:09 2026 GMT
notAfter =Jul 16 08:05:09 2027 GMT
```

!!! warning "`KollaTestCA` 的名字沒在開玩笑——它不能上生產"
    這個 CA 是自簽的,沒有任何公開信任鏈:客戶端要嘛手動信任它、要嘛關掉驗證。它適合 lab 和內部測試,**不能拿去對外**。生產環境要用真正的 CA(公司內部 PKI,或 Let's Encrypt 這類公開 CA)簽的憑證。本課用它學會「TLS 全鏈怎麼接」,但那條鏈的信任錨點,上線前必須換掉。

### 步驟 4:部署前檢查——四次擋下,四次都對

這是今天最有教學價值的一段。`prechecks` 連續擋下四次,而**每一次都是在阻止一場真的會發生的災難**:

| 次 | precheck 說什麼 | 真正的問題 | 對應地雷 |
|---|---|---|---|
| 1 | `Timeout waiting for 10.0.0.250:7480 to stop` | RGW 綁在 `0.0.0.0`,佔住了 VIP 的埠 | [地雷 3](#mine-3) |
| 2 | `Hostname has to resolve uniquely to the IP address of api_interface` | VIP 篡位成主機主 IP | [地雷 2](#mine-2) |
| 3 | `enable_ceph_rgw_loadbalancer ... no HAProxy members` | 開了 RGW 負載平衡卻沒指定後端 | [地雷 4](#mine-4) |
| 4 | (通過) | | |

四關全過之後,`ok=142 failed=0`,才進全量部署:

```text
=== prechecks ===
localhost  : ok=142  failed=0
=== 全量 deploy ===
localhost  : ok=872  changed=342  failed=0
=== post-deploy(重產 openrc)===
localhost  : ok=10   changed=6    failed=0
```

**這四次擋下值得停下來想**：如果 precheck 不擋,直接 deploy 下去——RGW 和 haproxy 搶同一個埠、ansible 對著 VIP 當主機部署、RGW 負載平衡指向空氣。每一個都會炸,而且炸得比 precheck 的錯誤訊息難懂得多。部署前檢查不是官僚流程,是**把「跑到一半才爆」提前成「動手前就擋」**。

### 步驟 5:openrc 換 https——第一次連線會失敗

`post-deploy` 重產的 openrc 已經指向 https:

```text
export OS_AUTH_URL='https://10.0.0.250:5000'
```

但直接用會撞 SSL 錯誤:

```text
SSLError: certificate verify failed: unable to get local issuer certificate
```

因為 `KollaTestCA` 是自簽的,客戶端不認得。openrc 要另外帶 CA 檔——而這需要一個獨立的設定([地雷 6](#mine-6)):

```yaml
kolla_admin_openrc_cacert: "{{ kolla_certificates_dir }}/ca/root.crt"
```

重跑 post-deploy 後,openrc 補上 `OS_CACERT`,API 就通了:

```text
export OS_CACERT='/etc/kolla/certificates/ca/root.crt'
openstack endpoint list --service identity
  public   https://10.0.0.250:5000
  internal https://10.0.0.250:5000
```

### 步驟 6:全服務回歸——TLS 會咬到你手寫的設定

換了大門和協定,得確認**每個服務都還活著**。多數服務無痛(它們的加密由 haproxy 代勞),但有一個會爆:

```text
HttpException: 500 ... Unexpected API Error
nova-api log: The identity service for 10.0.0.4:None exists but does not
              have any supported versions.
```

Nova 掛了,而根因在 **Day 27 手寫的 `[oslo_limit]` 設定**——它寫死了 `auth_url = http://10.0.0.4:5000`,TLS 上線後那個 http 位址不再有效([地雷 5](#mine-5))。改成 https + 補 CA:

```ini
[oslo_limit]
auth_url = https://10.0.0.250:5000/v3
cafile = /etc/ssl/certs/ca-certificates.crt
```

修完全站回歸:

```text
--- 容器 ---  總數 91 | 不健康:(無)
--- endpoints 還有非 https 的嗎 ---  (只剩 prometheus 後端,見下)
--- 核心讀操作 ---  server/volume/network/image/container list 全部正常
--- UI ---  Skyline https://…:9999 → 200 | Horizon http → 301 轉 https
--- Swift 往返 ---  上傳下載,hello-rgw.txt 原樣取回
```

91 個容器零不健康、所有面向使用者的 endpoint 都是 https、UI 把 http 自動轉向 https、連 Day 15 的物件儲存往返都還通。

!!! note "為什麼 Prometheus 還是 http"
    endpoint 清單裡 prometheus 仍顯示 `http://10.0.0.4:9091`——那是**後端內部位址**,不是使用者入口。使用者透過 haproxy 的 `https://…:9091` 進來(TLS 在那裡終結),haproxy 到後端那一小段走內網 http。這是 TLS-終結架構的正常樣貌:**加密在邊界,內網那一跳是另一個決定**(要不要連內網都加密,是 `kolla_enable_tls_backend` 的課題,本課未開)。

## 驗收 checkpoint

| 驗證 | 判準 | 本課環境的結果 |
|---|---|---|
| CA 與憑證 | `KollaTestCA` 簽出各服務憑證 | 一年效期,已產生 |
| 部署前檢查 | 四關全過(`failed=0`) | ok=142 |
| 全量部署 | `failed=0` | ok=872 changed=342 |
| openrc | 指向 https 且帶 CA,API 可認證 | endpoint 全 https |
| **全服務回歸** | 91 容器健康、核心操作正常 | 零不健康 |
| endpoint 協定 | 使用者入口全 https | 符合(prometheus 後端除外) |
| UI | http 自動轉 https | Skyline 200、Horizon 301→https |
| 既有功能 | Day 15 的 Swift 往返仍通 | `hello-rgw.txt` 原樣取回 |

## 地雷記錄

### 地雷 1:沒有 haproxy 就沒有地方終結 TLS {#mine-1}

**症狀**:單機 lab(關掉 haproxy)直接開 `kolla_enable_tls_internal`,precheck 就擋——它要求內外 VIP 不同,而單機根本沒有 VIP。

**根因**:kolla 的 TLS 由 haproxy 終結,只有三個 role 認得 TLS 開關。沒有 haproxy,TLS 開關無處著力。

**解法**:先把 haproxy 裝回來,連帶 proxysql 與 VIP。

**教訓**:**一個功能的開關,常常預設了一整套你以為早就有、其實被你關掉的架構。** 「開 TLS」讀起來像一行設定,做起來是重建入口。動手前先問:這個開關假設了什麼前提?

### 地雷 2:VIP 用 netplan 掛會篡位成主機主 IP {#mine-2}

**症狀**:VIP 設好、部署跑起來,precheck 卻擋:`Hostname has to resolve uniquely to the IP address of api_interface`。

**判讀關鍵**:錯誤訊息完全不提 VIP,只說「主機名沒有唯一解析到 api_interface 的 IP」。查 ansible 的 fact,它認為 eth0 的主 IP 是 `10.0.0.250`(VIP),不是 `10.0.0.4`(真主機 IP)。

**根因**:用 netplan 加位址時,新位址會被當成介面的**主** IP。於是 ansible 把 VIP 當成了主機位址,一切錯亂。

**解法**:改用 systemd oneshot 在開機後 `ip addr add`,讓 VIP 掛成**次要** IP,主 IP 仍是 `10.0.0.4`:

```text
inet 10.0.0.4/24   metric 100 ... eth0     ← 主
inet 10.0.0.250/32 ...          eth0       ← 次要
ansible fact address: 10.0.0.4             ← 對了
```

**教訓**:**同一個「加一個 IP」的動作,不同工具給的語意不同。** netplan 把它當「這個介面的位址」,`ip addr add` 當「多掛一個」。VIP 這種「必須是次要」的位址,對掛法很敏感——而症狀(主機名解析錯誤)離根因(掛法)有十萬八千里。

### 地雷 3:cephadm 的 RGW 綁 `0.0.0.0`,佔住 VIP 的埠 {#mine-3}

**症狀**:precheck 擋在 `Timeout when waiting for 10.0.0.250:7480 to stop`——它要那個埠淨空好讓 haproxy 用,但埠上有人。

**根因**:[Day 15](sprint4-day15-object-storage-rgw.md) 的 RGW 綁在 `0.0.0.0:7480`(所有位址),自然包含了 VIP 的 `10.0.0.250:7480`。haproxy 要在那個埠開 RGW 的前端,撞車。

**解法**:把 RGW 改綁到主機 IP,騰出 VIP 的埠。而這裡有第二層坑:RGW 的綁定設定藏在**兩層** ceph config(`client.rgw` 全域級 vs daemon 級),`ceph orch apply` 的 `networks` 欄位在這個版本不收斂,最後是直接設 daemon 級的 `rgw_frontends` 才生效:

```bash
ceph config set client.rgw.<daemon> rgw_frontends 'beast endpoint=10.0.0.4:7480'
ceph orch restart rgw.kolla
```

**教訓**:**兩套系統同機時,「綁所有位址」是一種對未來的透支。** [Day 15](sprint4-day15-object-storage-rgw.md) 綁 `0.0.0.0` 當時毫無問題,直到今天要在同一個埠上放 haproxy 才爆。同機共存的服務,綁 IP 要綁具體位址,別綁 `0.0.0.0`。

### 地雷 4:開了 RGW 負載平衡卻沒告訴它後端在哪 {#mine-4}

**症狀**:precheck 擋 `enable_ceph_rgw_loadbalancer is enabled but no HAProxy members are configured. Have you set ceph_rgw_hosts?`。

**根因**:把 RGW 也放進 haproxy(讓它也走 VIP + TLS)需要兩件事:開 `enable_ceph_rgw_loadbalancer`,**以及**用 `ceph_rgw_hosts` 告訴 haproxy 後端 RGW 在哪台哪個埠。只開前者,haproxy 不知道要往哪轉。

**解法**:補 `ceph_rgw_hosts`:

```yaml
ceph_rgw_hosts:
  - host: openstack-lab
    ip: 10.0.0.4
    port: 7480
```

**教訓**:這顆 precheck 是**正面教材**——它不只說「錯了」,還直接問「你是不是忘了設 `ceph_rgw_hosts`?」。好的部署前檢查會把根因和解法一起遞給你。

### 地雷 5:TLS 引爆了你三天前手寫的設定 {#mine-5}

**症狀**:全站換 https 後,只有 Nova 掛掉,`500 ... identity service for 10.0.0.4:None ... does not have any supported versions`。

**根因**:[Day 27](sprint5-day27-multi-tenant-governance.md#mine-1) 為了 unified limits 手寫了一段 `[oslo_limit]`,裡面**寫死** `auth_url = http://10.0.0.4:5000`。TLS 上線後,那個 http 位址不再提供服務,Nova 每次查額度都失敗。

**解法**:把手寫設定裡的 auth_url 改成 https,並補 cafile。

**教訓**:這是本章最重要的一顆,因為它揭露了**手寫覆寫設定的長期成本**——kolla 永遠不會更新你手寫的檔。模板產生的設定會隨著 TLS 上線自動改成 https,但你三天前手寫的那一段,只有你自己記得要改。**每一個手寫覆寫,都是一筆未來某次架構變動時會引爆的債。** 手寫之前先想:有沒有 kolla 變數能達到同樣效果?

### 地雷 6:openrc 的 CA 要另外指定 {#mine-6}

**症狀**:post-deploy 產的 openrc 指向 https,但用它認證撞 `certificate verify failed: unable to get local issuer certificate`。

**根因**:openrc 有了 `OS_AUTH_URL=https://...` 卻沒有 `OS_CACERT`。客戶端不認得自簽的 `KollaTestCA`,無法驗證伺服器憑證。

**解法**:設 `kolla_admin_openrc_cacert` 指向 CA 檔,重跑 post-deploy,openrc 就會補上 `OS_CACERT`。

**教訓**:自簽 CA 的世界裡,**「伺服器出示憑證」和「客戶端信任那張憑證」是兩件要分別安排的事**。伺服器端的 TLS 開了不代表客戶端就能連——每個客戶端都得被告知「這個 CA 可信」。這也正是 `KollaTestCA` 不能上生產的實務原因:你不可能叫全世界的客戶端手動信任你的自簽 CA。

## 帶得走的東西

- **一行「開 TLS」的設定,底下是一整套入口架構。** TLS 由 haproxy 終結——沒有 haproxy,加密無處著力。功能開關常預設了你以為早就有的前提。
- **部署前檢查是把「跑到一半才爆」提前成「動手前就擋」。** 今天四次擋下,每一次都攔住一場真災難;好的 precheck 還會把解法一起遞給你。
- **每一個手寫覆寫設定,都是一筆未來會引爆的債。** 模板產的設定會隨架構變動自動更新,手寫的那段只有你記得。手寫前先找有沒有官方變數。
- **加密在邊界終結,內網那一跳是另一個決定。** 使用者入口全 https,不代表內網每一跳都加密——那是獨立的、要另外開的課題。
- **自簽 CA 教你接通全鏈,但它的信任錨點上線前必須換掉。** 伺服器出示憑證與客戶端信任憑證是兩件事,而你沒法叫全世界信任你的 `KollaTestCA`。

## 延伸閱讀

想往下深挖,從這幾份開始:

- **[Kolla-Ansible 的 TLS 指南](https://docs.openstack.org/kolla-ansible/2025.1/admin/tls.html)** —— `kolla_enable_tls_internal/external`、`kolla_copy_ca_into_containers`、backend TLS 的官方對照。
- **[Kolla-Ansible 的憑證管理](https://docs.openstack.org/kolla-ansible/2025.1/admin/tls.html#generating-a-private-certificate-authority)** —— `kolla-ansible certificates` 產生 CA 的流程,以及換成外部 CA 的做法。
- **[OpenStack 的 API 保護最佳實務](https://docs.openstack.org/security-guide/secure-communication.html)** —— TLS 終結位置、內網加密、憑證管理的安全指南;本章「加密在邊界」那一節的理論背景。

## 下一步

這朵雲的每一次對話現在都加密了。[Day 33](sprint5-day33-keycloak-federation.md) 用上今天的 https 端點,把企業的身分系統接進來:員工用公司的 Keycloak 帳號直接登入這朵雲,不必在 Keystone 裡重建一份使用者——而你會發現,連 Keycloak 自己都得靠今天這張 `KollaTestCA` 才進得來。
