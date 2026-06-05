# W08｜容器生產實踐

## Healthcheck 故障測試
- 停 db 後幾秒被標 unhealthy：約30秒
- 對應的 log 訊息：
curl: (7) Failed to connect to localhost port 8080
Could not connect to server

## Log 失控估算
- noisy 容器 30s log 大小：57MB
- 預估 24h 大小：約164GB
- 套 rotation 後穩定上限：約 30 MB（max-size=10m、max-file=3）

## 資源限制實驗

| 實驗 | 命令 | 觀察結果 | 對應 cgroup 檔 | 值 |
|---|---|---|---|---|
| OOM | stress-ng --vm 1 --vm-bytes 200m | exit 137，OOMKilled=true | memory.max | 134217728 |
| CPU throttle | stress-ng --cpu 4 | docker stats CPU% ≈ 51.56% | cpu.max | 50000 100000 |

## 權限四階對照

| 階梯 | id | CapEff | NoNewPrivs | curl /healthz |
|---|---|---|---|---|
| 0 | uid=0(root) gid=0(root) | 00000000a80425fb | 0 | ok |
| 1 | uid=1000 gid=1000 | 0000000000000000 | 0 | ok |
| 2 | uid=1000 gid=1000 | 0000000000000000 | 0 | ok |
| 3 | uid=1000 gid=1000 | 0000000000000000 | 1 | ok |
| 4 | uid=1000 gid=1000 | 0000000000000000 | 1 | ok |

## 排錯紀錄

- 症狀：執行docker compose up時app無法啟動
- 診斷：8080port被W07的app container佔用
- 修正：先到w07執行docker compose down再重新啟動w08
- 驗證：docker compose ps顯示app、db正常執行，curl /healthz回傳ok

## 設計決策
（你選的 mem_limit / cpus 數值理由是什麼？read_only 之後你補了哪些 tmpfs，為什麼？）

我將app的mem_limit設為128m
因Flask範例程式需求不高，所以128m足夠執行，也可以避免異常程式佔用過多記憶體

cpus設為0.5，為了限制單一container最多使用半顆CPU
避免影響主機或其他container的效能

啟用read_only後容器根檔案系統會變成唯讀
因此額外加入tmpfs掛載到/tmp讓程式仍然可以使用暫存檔案功能

tmpfs的資料存放在記憶體中
container停止後會自動清除
適合放暫存資料而不適合永久保存資料
