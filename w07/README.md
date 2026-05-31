# W07｜Docker Compose 與資料持久化

## 拓樸圖

```text
                ┌─────────────────────┐
                │     w07_default     │
                │       network       │
                └─────────┬───────────┘
                          │
          ┌───────────────┴───────────────┐
          │                               │
 ┌────────▼────────┐             ┌────────▼────────┐
 │       app       │             │        db       │
 │    Flask App    │             │    PostgreSQL   │
 │     port 80     │             │     port 5432   │
 └────────┬────────┘             └────────┬────────┘
          │                               │
          │                               │
          │                     ┌─────────▼─────────┐
          │                     │      db-data      │
          │                     │    named volume   │
          │                     └───────────────────┘
          │
   localhost:8080
```


## 從 docker run 到 compose.yaml
（自己的話：你最有感的一個改善是什麼？）
A:我覺得最有感的改善是不用再打一長串docker run指令

## 三種掛載對照
| 掛載類型 | 路徑（host） | 容器砍重起資料還在嗎 | 重啟容器資料狀態 | 適合情境 |
|---|---|---|---|---|
| named volume | Docker 自己管理（/var/lib/docker/volumes） | 會 | 資料保留 | 資料庫、正式環境 |
| bind mount | 自己指定的資料夾（例如 ./app） | 會 | host 跟 container 同步 | 開發環境、修改程式碼 |
| tmpfs | 記憶體內（RAM） | 不會 | container 停止後資料消失 | 暫存資料、cache |


## healthcheck 前後對照
| 寫法 | curl /healthz t=1s | t=3s | t=5s | t=10s |
|---|---|---|---|---|
| 只 depends_on | 503 | 503 | 503 | 200 |
| service_healthy | refused | refused | 200 | 200 |

觀察（自己的話）：只用depends_on時只代表db container有先啟動
但不代表資料庫已經真的可以連線
所以app可能會先起來然後bealthz就會出現503
加上healthcheck跟service_healthy後，app會等db真的healthy才啟動
所以一開始可能還連不到app但app起來後就比較穩定
healthz會直接變200

## 排錯紀錄
- 症狀：剛開始curl/healthz時出現503，代表app有起來但連不到db
- 診斷：只寫depends_on只能控制啟動順序，不能保證postgres已經初始完成
- 修正：在db加上healthcheck並把app的depends_on改成condition: service_healthy
- 驗證：重新docker compose up後app會等db healthy才啟動，/healthz最後可以正常回傳 ok

## 設計決策
（為什麼 db 用 named volume 而不是 bind mount？為什麼不能在生產用 tmpfs 存資料庫？）
A:因為資料庫的資料比較重要 交給Docker管理比較穩定
也比較不會遇到權限問題
bind mount比較適合開發時掛程式碼，不適合放postgres的資料目錄
tmpfs則是存在記憶體裡，container停掉資料就會消失
所以不能在生產環境拿來存資料庫，只適合放暫存或cache
