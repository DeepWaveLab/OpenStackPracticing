# Day 20:封包怎麼流過這朵雲的網路

> Day 2 的網路對你來說可能還帶著一點魔法:建個網路、接個 router、掛個 floating IP,VM 就能上網了。今天把魔法拆光——用 OVN 的原生工具讀出邏輯拓樸、看安全群組怎麼變成規則、用 `ovn-trace` 讓一個封包「在紙上」走完全程,親眼看到它在哪一刻被 NAT 改寫。全程不發一個真封包、不用 tcpdump。

!!! abstract "你在課程的哪裡"
    - **Day 2**:你從使用者視角建了網路、router、FIP,VM 通了。
    - **今天**:潛到 Neutron 底下的 **OVN**,看那些資源實際上變成了什麼、封包怎麼流。**不部署任何東西**。
    - **今天之後**:階段二結束——你已經能不靠 `openstack` CLI 讀懂這朵雲的運算面與網路面。

## 先建立地圖:Neutron 與 OVN 的對應

你在 Day 2 用 Neutron 建的東西,底下都變成了 OVN 的邏輯物件。對應關係很乾淨:

| 你在 Neutron 建的 | 變成 OVN 的 | 工具 |
|---|---|---|
| network | logical switch | `ovn-nbctl show` |
| router | logical router | `ovn-nbctl lr-list` |
| security group rule | logical flow(ACL) | `ovn-sbctl lflow-list` |
| floating IP | NAT 規則(dnat_and_snat) | `ovn-nbctl lr-nat-list` |

OVN 有兩個資料庫:**NB(northbound)** 存「你想要的樣子」(邏輯拓樸),**SB(southbound)** 存「怎麼實現」(編譯後的 flow)。工具落點(都在既有容器內,不用安裝):

```bash
NB="sudo docker exec ovn_nb_db ovn-nbctl"
SB="sudo docker exec ovn_sb_db ovn-sbctl"
# ovn-trace 也在 ovn_sb_db 容器;ovs-ofctl 在 openvswitch_vswitchd
```

## 拆解點一:邏輯拓樸

```bash
$NB show
```

```text
switch (aka day17-net)
    port trove-a802f2…  addresses: ["fa:16:3e:f1:17:de 10.17.0.159"]
    port (type: router) router-port: lrp-25feab…       ← 接到 router 的埠
switch (aka ext-net)
    ...
router (aka day17-r)
    port lrp-25feab…  networks: ["10.17.0.1/24"]        ← 內網那一側
    port lrp-43d94f…  networks: ["172.24.4.125/24"]     ← 外網那一側
```

一眼就看懂結構:**每個 Neutron 網路 = 一個 logical switch**;router 靠 `router-port` 接進 switch。你的 VM 是 switch 上的一個 port(帶著它的 MAC 與 IP)。這張圖就是 Day 2 那些指令的成果,只是換成 OVN 的語言。

## 拆解點二:NAT 規則在哪

Day 18 的 Trove guest 能連回主機的 RabbitMQ,靠的是 router 上的 SNAT。把它挖出來:

```bash
$NB lr-nat-list neutron-<router-uuid>
```

```text
TYPE   EXTERNAL_IP     LOGICAL_IP
snat   172.24.4.125    10.17.0.0/24
```

這一條就是答案:凡是從 `10.17.0.0/24`(day17-net)出去的封包,源 IP 會被改寫成 `172.24.4.125`(router 的外網位址)。**這正是 Day 18 沒建管理網卻能通的原因**——三天前的實測,在這裡找到了規則層的解釋。(floating IP 則會是 `dnat_and_snat` 型別,做雙向改寫。)

## 拆解點三:安全群組 = logical flow

安全群組規則不是存在某張表裡等人查,而是被**編譯成封包要通過的邏輯關卡**(flow)。看 day17-net 的 pipeline:

```bash
$SB lflow-list day17-net | grep ls_in_acl | head -4
```

```text
table=7 (ls_in_acl_hint), priority=7, match=(ct.new && !ct.est), action=(...)
table=7 (ls_in_acl_hint), priority=5, match=(!ct.trk), action=(...)
```

`ct.*` 是連線追蹤(conntrack)——OVN 用它實現「有狀態」的安全群組:放行一個出站連線,回程封包自動放行。你在 Day 2/Day 4 加的每一條 security group rule,最後都落在這種 flow 裡。

## 拆解點四:ovn-trace —— 讓封包在紙上走一遍

這是今天的主角。`ovn-trace` 不發真封包,而是**模擬**一個封包餵進 OVN pipeline,把它經過的每一級 table、每一個動作全部印出來。模擬 Trove VM 對外送封包:

```bash
$SB ovn-trace day17-net '
  inport=="trove-a802f2…" &&
  eth.src==fa:16:3e:f1:17:de && eth.dst==fa:16:3e:27:b2:65 &&
  ip4.src==10.17.0.159 && ip4.dst==8.8.8.8 && ip.ttl==64
'
```

輸出很長,但精華在封包進入 router 之後這幾行:

```text
ingress(dp="day17-r", inport="lrp-25feab")
  ...
3. lr_out_snat: ip4.src == 10.17.0.0/24 ... priority 153
     ct_snat(172.24.4.125);          ← 源 IP 就在這一刻被改寫!
6. lr_out_delivery: outport == "lrp-43d94f"
     output;                          ← 從外網埠送出
```

你**看到了 NAT 發生的確切那一級**(`lr_out_snat`)、規則的優先權、以及改寫後的位址。這比 tcpdump 強大的地方在於:它是靜態分析,封包還沒真的送出,你就知道它會怎麼被處理。

## 拆解點五:邏輯 flow → 實體 OpenFlow

上面那些「邏輯 flow」是給人看的抽象;OVN 的 `northd` 會把它們編譯成 **OpenFlow**——OVS 真正執行的規則。看實體流表:

```bash
sudo docker exec openvswitch_vswitchd ovs-ofctl dump-flows br-int | wc -l
```

```text
2021
```

兩千多條。這就是完整的翻譯鏈:**你的一條 security group rule → OVN logical flow(northd 編譯)→ br-int 上的 OpenFlow → 網卡上的封包處理**。四層抽象,一路對得起來。

## 帶得走的除錯法:trace 找 drop

`ovn-trace` 最實用的場景是回答「**為什麼這個封包不通?**」。模擬一個你認為該通(或不該通)的封包,看輸出——

- 如果封包在某一級出現 **`drop;`**,那一級的 table 就是它死掉的地方(通常是 ACL,也就是安全群組沒放行)。
- 如果一路走到 `output;`,那 OVN 這層是通的,問題在更下面(實體網路、MTU 之類)。

**不用開 VM、不用抓封包,靜態就能判斷一個封包通不通、卡在哪一層。** 這是排除 OpenStack 網路問題最快的一張牌。

## 驗收 checkpoint

今天沒有部署,驗收標準是「能不能徒手讀懂網路」:

| 驗證 | 判準 | 本課環境的結果 |
|---|---|---|
| 拓樸 | `ovn-nbctl show` 讀出 switch/router,對應得上 Neutron 資源 | 4 switch / 3 router |
| NAT | `lr-nat-list` 找到 SNAT 規則,對上 Day 18 的路徑 | `10.17.0.0/24 → 172.24.4.125` |
| ACL | `lflow-list` 看到安全群組的 ct flow | 符合 |
| **ovn-trace** | 模擬封包,看到 `ct_snat` 改寫源 IP 的確切一級 | 符合 |
| 編譯 | `ovs-ofctl` 證實邏輯 flow → 實體 OpenFlow | 2021 條 |

## 階段二完成 🎉

兩天,兩個面向:

- **Day 19（運算面）**:一個 API 請求 → token → RPC → qemu 進程。
- **Day 20（網路面）**:一個封包 → logical switch → router NAT → OpenFlow。

合起來就是這個階段的畢業標準:**不靠 `openstack` CLI,你也能讀懂這朵雲在做什麼、以及它不動時卡在哪。** 而且最漂亮的是——今天 `ovn-trace` 印出的那條 SNAT,正好解釋了三天前 Day 18 為什麼一個管理網都不用建就通了。前後串成一條線,這就是把東西「用過」再「拆開」的價值。

## 延伸閱讀

想往下深挖,從這幾份開始:

- **[ovn-architecture 手冊](https://www.ovn.org/support/dist-docs/ovn-architecture.7.html)** —— OVN 的正典:NB/SB 資料庫、northd、logical pipeline 的完整定義。
- **[ovn-trace 手冊](https://www.ovn.org/support/dist-docs/ovn-trace.8.html)** —— 本章主角工具的完整選項;模擬封包欄位的語法都在這。
- **[Neutron 的 OVN 文件](https://docs.openstack.org/neutron/2025.1/ovn/index.html)** —— Neutron 資源怎麼對應到 OVN 物件的官方版;本章那張對應表的出處。

## 下一步

拆解告一段落,回到動手。[Day 21](sprint4-day21-observability.md) 進入階段三——給這朵雲裝上眼睛:Prometheus、Grafana、集中式 log,以及最實際的一課「有觀測和沒觀測,除錯速度差幾個數量級」。
