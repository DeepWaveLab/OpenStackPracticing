# Kolla-Ansible AIO:proxysql 與 MariaDB 撞 3306,bootstrap 必死

**Sprint 3 / Day 1(2026-07-08)· Kolla-Ansible 20.x(2025.1 Epoxy)· Ubuntu 24.04 AIO**

## 症狀

`kolla-ansible deploy` 在 mariadb 階段失敗:

```
TASK [mariadb : Wait for first MariaDB service port liveness]
fatal: ... "Timeout when waiting for search string MariaDB in 10.0.0.4:3306"
```

- `docker ps -a`:`mariadb Exited (1)`,但 mariadb log 顯示 daemon 有啟動(`Starting mariadbd daemon`)後隨即退出
- `ss -tlnp | grep 3306`:**proxysql** 在聽 `10.0.0.4:3306`

## 根因

AIO 常見設定 `enable_haproxy: "no"` + `kolla_internal_vip_address` = host IP。但 **Epoxy 的 `enable_proxysql` 預設 `yes`**,而且它跟 `enable_haproxy` 是獨立開關 —— 結果 loadbalancer 角色照樣部了 proxysql(連 haproxy container 都會被帶起來),proxysql 綁 VIP:3306,mariadb 綁 host:3306,VIP == host IP → 同一個 socket,mariadb bind 失敗退場。

單看文件不會發現:quickstart 沒提 proxysql,舊教學(Caracal 以前)只教關 haproxy。

## 解法

```bash
echo 'enable_proxysql: "no"' >> /etc/kolla/globals.yml
sudo docker rm -f proxysql haproxy mariadb
sudo docker volume rm mariadb          # bootstrap 中途死掉的空 DB,砍掉重來最乾淨
kolla-ansible deploy -i ~/all-in-one   # 重跑,冪等
```

重跑結果:`ok=367 changed=214 failed=0`。

## 通則教訓

1. AIO 關 LB 時,`enable_haproxy` 和 `enable_proxysql` **要成對關**。
2. kolla 升版要重讀 release notes 的「預設值變更」——這類 default 翻轉是升版最大地雷來源(同型:Sprint 1 的 22 條有一半是 charm default 變更)。
3. debug 套路:`docker ps -a` 看誰死了 → `docker logs` 看它自己怎麼說 → `ss -tlnp` 看資源被誰佔走。三步內定位,不用翻 ansible log。
