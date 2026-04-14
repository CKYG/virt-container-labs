# W04｜Linux 系統基礎：檔案系統、權限、程序與服務管理

## FHS 路徑表

| FHS 路徑 | FHS 定義 | Docker 用途 |
|---|---|---|
| /etc/docker/ | 存放系統設定檔的目錄 | 放 Docker daemon 的設定檔 |
| /var/lib/docker/ | 存放系統資料的目錄 | 儲存映像檔、容器資料、volume 等 |
| /usr/bin/docker | 存放可執行檔的目錄 | Docker 指令程式 |
| /run/docker.sock | 存放執行時 socket 的位置 |  Docker CLI 與 daemon 溝通用的 socket |

## Docker 系統資訊

- Storage Driver：overlayfs
- Docker Root Dir：/var/lib/docker
- 拉取映像前 /var/lib/docker/ 大小：244K
- 拉取映像後 /var/lib/docker/ 大小：244K（系統已存在）

## 權限結構

### Docker Socket 權限解讀
<img width="730" height="103" alt="image" src="https://github.com/user-attachments/assets/f29498d9-06c8-4ab1-9238-2d792ea411da" />
owner（root）：有 rw 權限，可以讀寫 socket
group（docker）：有 rw 權限，docker 群組的使用者也可以使用 docker
others：沒有任何權限（---），一般使用者不能使用 docker

### 使用者群組
<img width="1275" height="65" alt="image" src="https://github.com/user-attachments/assets/18569105-f319-47be-a123-077227f88e4e" />
從輸出可以看到目前使用者有在docker群組中

### 安全意涵
docker群組權限很大因為可以直接操作 docker
而docker可以影響整個系統，所以等於root

像這次測試不用sudo也能用docker
代表權限其實已經很高，會有風險

## 程序與服務管理

### systemctl status docker
<img width="1284" height="538" alt="image" src="https://github.com/user-attachments/assets/45060b8f-efca-4995-8f16-13f5f9b4ef33" />


### journalctl 日誌分析
<img width="1306" height="230" alt="image" src="https://github.com/user-attachments/assets/94dfbe2a-62a6-4b93-9c8c-64a794a4b3b1" />
從日誌中可以看到docker daemon的啟動過程
包含初始化
API listen
image pulled等訊息
代表docker有正常運作，且有成功拉映像。

### CLI vs Daemon 差異
docker是拿來下指令的工具，
dockerd是在背景服務+執行
就算docker --version 正常，
如果daemon沒開docker就還是不能用。

## 環境變數

- $PATH：<img width="806" height="73" alt="image" src="https://github.com/user-attachments/assets/13f6d386-57cc-4d0c-bef9-cecd3e0e52de" />

- which docker：/usr/bin/docker
- 容器內外環境變數差異觀察：容器裡的環境變數跟主機不太一樣，PATH會比較簡單，因為容器是獨立的環境

## 故障場景一：停止 Docker Daemon

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| systemctl status docker | active | inactive | active |
| docker --version | 正常 | 正常 | 正常 |
| docker ps | 正常 | Cannot connect | 正常 |
| ps aux grep dockerd | 有 process | 無 process | 有 process |

## 故障場景二：破壞 Socket 權限

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ls -la docker.sock 權限 | srw-rw---- | srw------- | srw-rw---- |
| docker ps（不加 sudo） | 正常 | permission denied | 正常 |
| sudo docker ps | 正常 | 正常 | 正常 |
| systemctl status docker | active | active | active |

## 錯誤訊息比較

| 錯誤訊息 | 根因 | 診斷方向 |
|---|---|---|
| Cannot connect to the Docker daemon | docker daemon 沒有啟動 | 檢查 systemctl status docker |
| permission denied…docker.sock | 沒有權限存取 docker.sock | 檢查使用者群組或 socket 權限 |

timeout是連不到服務，像daemon沒開或被關掉。

permission denied是有連到，但沒有權限使用。

## 排錯紀錄
- 症狀：docker出現Cannot connect or permission denied
- 診斷：先用systemctl status docker看服務有沒有啟動再用ls -la /var/run/docker.sock檢查權限
- 修正：如果是daemon沒開就啟動docker，如果是權限問題就把使用者加入docker群組或修正socket權限
- 驗證：重新執行docker ps，沒有錯誤訊息就可以正常使用

## 設計決策
這次選擇把使用者加入 docker群組是因為這樣可以直接docker而不用每次都打sudo
缺點是docker群組權限很高 若亂用可能會影響整個系統，有安全風險
