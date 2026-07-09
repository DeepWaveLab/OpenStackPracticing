# Kolla Octavia:LB 永遠 PENDING_CREATE —— amphora driver 需要 Redis,kolla 不自動開

**Sprint 3 / Day 4(2026-07-08)· Kolla-Ansible 20.x(Epoxy)**

## 症狀

- `openstack loadbalancer create` 後 LB 卡 `PENDING_CREATE`,amphora VM 從未被建立
- octavia 五個容器全部 healthy(這最會騙人)
- `octavia-worker.log`:

```
taskflow.exceptions.JobFailure: Failed to connect to redis
redis.sentinel.MasterNotFoundError: No master found for 'kolla'
ConnectionError('Error 111 connecting to 127.0.0.1:6379. Connection refused.')
```

## 根因

Epoxy 的 amphora provider 用 **taskflow jobboard(Redis sentinel)** 做任務佇列與斷點續跑。kolla 的 octavia.conf 已寫好 sentinel 設定,**但 `enable_redis` 預設 "no",octavia 也不會宣告依賴把它拉起來** —— worker 啟動正常、健檢通過,直到第一個 LB job 才爆。

## 解法

```bash
echo 'enable_redis: "yes"' >> /etc/kolla/globals.yml
kolla-ansible deploy -i ~/all-in-one --tags redis,octavia
```

## 後遺症:Redis 壞掉期間建立的 LB 卡死

該期間送出的 LB 永遠停 `PENDING_CREATE`(job 沒進 queue,沒人會來改狀態),delete 回 409(PENDING 為 immutable)。Octavia 沒有官方重置工具,社群公認處置(lab 限定):

```bash
# 在 mariadb 容器內把狀態改成 ERROR,再正常 cascade delete
UPDATE load_balancer SET provisioning_status='ERROR' WHERE name='lb1';
openstack loadbalancer delete lb1 --cascade
```

## 教訓

1. **「容器 healthy ≠ 服務可用」**:健檢只測 process/port,測不到「第一個真實 job 才會走到的依賴」。
2. kolla 的服務開關不會自動解依賴(對照 Day 3 的 cinder-backup:開了沒用;這裡是:沒開會死)。開新服務前查它的 runtime 依賴鏈。
3. 中斷期間送入的 async 任務,修好後**不會自動重放**,要清殘骸。
