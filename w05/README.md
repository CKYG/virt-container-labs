# W05｜把容器拆開來看：Namespace / Cgroups / Union FS / OCI

## Docker 環境

- Storage Driver：overlay2
- Cgroup Version：2
- Cgroup Driver：systemd
- Default Runtime：runc

## Namespace 觀察

### 六種 namespace 用途（用自己的話）
- PID：隔離process讓容器裡只能看到自己的process
- NET：隔離網路讓每個容器有自己的IP、port、網卡
- MNT：隔離檔案系統掛載點讓每個容器看到自己的root filesystem
- UTS：隔離hostname讓容器可以有自己的hostname
- IPC：隔離共享記憶體、message queue、semaphore
- USER：隔離使用者權限讓容器內root不一定等於host root

### Host vs 容器 inode 對照
[namespace-table.md](./namespace-table.md)

### 容器內 `ps aux` 輸出
容器內只看到少量 process
因為PID namespace會把process隔離開來，所以容器只能看到自己的process
看不到host上其他的process

## Cgroups 實驗

### 容器內讀到的限制
- memory.max：134217728
- cpu.max：50000 100000

### Host 端對照
- memory.max：134217728
- cpu.max：50000 100000
- memory.current（執行時某一刻）：430080

### OOM 故障三階段
| 項目 | 故障前 | 故障中（memory=32m + dd 200m）| 回復後（memory=256m）|
|---|---|---|---|
| 容器 exit code | - | 137 | 0 |
| OOMKilled | - | true | false |
| dmesg 關鍵字 | 無 OOM | Memory cgroup out of memory  / OOMKilled=true | 無 OOM |

## Image 分層

### `docker image inspect nginx:1.27-alpine` layer 數量
（8）

### 兩個同源 image 共享 layer 的證據
（前幾個 sha256 是否相同？）
nginx:1.27-alpine跟nginx:1.26-alpine
前幾層的sha256有相同代表兩個image會共享layer
所以Docker不會重複下載或重複佔用空間

### `docker diff` 輸出範例與解讀
（貼上 A/C/D 實例並說明）
<img width="799" height="323" alt="image" src="https://github.com/user-attachments/assets/98a6646d-1c2c-464e-aa99-d280c63ea5f9" />
A=Added 代表新增檔案或資料夾
D=Deleted 代表檔案被刪除
C=Changed 代表檔案或資料夾內容被修改

## OCI 呼叫鏈

（用自己的話說明 dockerd → containerd → containerd-shim → runc 各自負責什麼，以及 OCI Runtime Spec `config.json` 裡哪些欄位對應到 namespace / cgroup 設定）
A:dockerd是Docker的主程式負責接收docker指令
containerd負責管理container的建立跟執行
containerd-shim會幫忙讓container在背景持續運作
就算dockerd重啟container也不一定會停
runc是建立container的工具
會根據config.json去設定namespace和cgroup
config.json裡:
namespaces:設定PID、NET、MNT這些隔離
linux.resources:設定memory跟cpu限制
mounts:設定掛載資料夾

## 排錯紀錄
- 症狀：讀cgroup檔案時一直出現No such file or directory
- 診斷：cgroup路徑打錯 container id也有少打一部分
- 修正：改用find指令自動找完整路徑
- 驗證：最後成功讀到memory.max、cpu.max、memory.current

## 想一想（回答 3 題）
1. 容器裡的 PID 1 跟 host PID 1 是同一支 process 嗎？`kill -9 1`（在容器內）會發生什麼？
不是同一支 container有自己的PID namespace。如果在container裡面kill PID 1後container會直接停止

2. 兩個容器都基於 `ubuntu:24.04`，磁碟空間是吃兩份還是共用？怎麼驗證？
大部分是共用，因為Docker會共用一樣的image layer
所以不會每個container都完整存一份ubuntu
container自己改動的部分才會另外新增

可以用docker image inspect看layer的sha256是否相同
或docker system df查看空間使用情況

3. 如果 host 的 kernel 爆漏洞，容器還能稱為「隔離」嗎？這個限制跟 VM 差在哪？
container是跟host共用同一個kernel
所以kernel出問題
container也可能受到影響

VM是每台都有自己的系統跟kernel
像真正分開的電腦
所以通常比container安全
