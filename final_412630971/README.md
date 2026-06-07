# 期末實作 — 412630971 邱秉智

## 1. 架構總覽
<Mermaid 圖 + 一段話說明>

## 2. Part A：底座與基準點
<ssh 證據 + 版本 + snapshot>
![ssh-and-versions](screenshots/ssh-and-versions.png)

## 3. Part B：Dockerfile 與快取
<Dockerfile + 兩次 build 對照>
![build-cache-diff](screenshots/build-cache-diff.png)
### 為什麼聽 8080 不聽 80？
因為我在Dockerfile建立appuser，然後用USER appuser執行Flask，不是直接用root執行
Linux的80 Port算特權埠，需要root權限才能用。且我以比較安全的方式運行，所以改用一般使用者可使用的8080 Port
可以避免應用程式有過高權限，也符合最小權限原則。所以這次的Flask服務監聽8080而不是80

## 4. Part C：Compose 與資料持久化
<compose.yaml 重點 + 三段對照>

![volume-3-stages](screenshots/volume-3-stages.png)
### down vs down -v
必答：down 跟 down -v 差在哪？named volume 的生命週期跟著誰？
A:down會直接停止然後刪除container和network，但不會刪除named volume，所以資料庫的資料還會保留下來。
down -v 則會把named volume一起刪除，所以重新啟動後會得到全新的資料庫，原本的資料表和資料都會消失。
named volume的生命週期不是跟container，而是獨立的。就算container被刪除重建但只要volume還在，資料就會保留。只有刪除volume時資料才會消失。

## 5. Part D：生產化加固
<權限驗證輸出 + cgroup 讀值對照表>
### yaml 的值怎麼對回 cgroup 檔案？

## 6. Part E：故障演練
### 故障 1：<F1–F4 擇一>
- 注入方式：
- 故障前：
- 故障中：
- 回復後：
- 診斷推論：

### 故障 2：<另一個>
（同上）

### 三症狀分層表（必答）
| 症狀 | 最可能的層 | 第一條驗證命令 |
| ---- | ---------- | -------------- |
| timeout |  |  |
| connection refused |  |  |
| HTTP 503 |  |  |

## 7. 反思（200 字）
這學期從 VM 做到 production-ready 容器，「隔離」這個概念在 VM、namespace、
cgroup、權限階梯四個地方各出現一次——它們在防的東西一樣嗎？

## 8. Bonus（選做）
