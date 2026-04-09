# W02｜VMware 網路模式與雙 VM 排錯

## 網路配置

| VM | 網卡 | 模式 | IP | 用途 |
|---|---|---|---|---|
| dev-a | NIC 1 | NAT | 192.168.168.128 | 上網 |
| dev-a | NIC 2 | Host-only | 192.168.152.128 | 內網互連 |
| server-b | NIC 1 | Host-only | 192.168.152.129 | 內網互連 |

## 連線驗證紀錄

- [x] dev-a NAT 可上網：`ping google.com` 輸出
<img width="1482" height="967" alt="image" src="https://github.com/user-attachments/assets/cbd4f435-5fab-4347-a976-20fdb0ae71fe" />

- [x] 雙向互 ping 成功：貼上雙方 `ping` 輸出
<img width="658" height="197" alt="image" src="https://github.com/user-attachments/assets/127e0a0d-54d9-4705-9195-d901cd4875e2" />
<img width="636" height="174" alt="image" src="https://github.com/user-attachments/assets/5a733d2b-804c-4790-8b39-2e97e2ed17e5" />

- [x] SSH 連線成功：`ssh <user>@<ip> "hostname"` 輸出
<img width="814" height="223" alt="image" src="https://github.com/user-attachments/assets/d74fbe09-5b97-478f-b4de-79472393011b" />

- [x] SCP 傳檔成功：`cat /tmp/test-from-dev.txt` 在 server-b 上的輸出
<img width="813" height="176" alt="image" src="https://github.com/user-attachments/assets/44cdb72a-f229-4606-87b0-9fe44e22b2ca" />

- [x] server-b 不能上網：`ping 8.8.8.8` 失敗輸出
<img width="406" height="44" alt="image" src="https://github.com/user-attachments/assets/62075d27-55ee-4371-914d-60fd2b62958b" />


## 故障演練一：介面停用

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| server-b 介面狀態 | UP | DOWN | UP |
| dev-a ping server-b | 成功 | 失敗 | 成功 |
| dev-a SSH server-b | 成功 | 失敗 | 成功 |
<img width="821" height="326" alt="image" src="https://github.com/user-attachments/assets/9d47cf55-3013-4241-8269-d4391ccd60a9" />
<img width="663" height="177" alt="image" src="https://github.com/user-attachments/assets/ef348d7e-3bbe-47a1-a093-e8454f6eaffd" />
<img width="808" height="393" alt="image" src="https://github.com/user-attachments/assets/6fe19b79-911b-4f3a-9aa8-6417c3974095" />

## 故障演練二：SSH 服務停止

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ss -tlnp grep :22 | 有監聽 | 無監聽 | 有監聽 |
| dev-a ping server-b | 成功 | 成功 | 成功  |
| dev-a SSH server-b | 成功 | Connection refused | 成功 |
<img width="803" height="107" alt="image" src="https://github.com/user-attachments/assets/5c1cd329-b009-4983-9cd8-82c1a745d3e6" />


## 排錯順序
（寫出你的 L2 → L3 → L4 排錯步驟與每層使用的命令）

L2:
用`ip address show`檢查網卡是否為UP和確認是否有IP

L3:
用`ip route show`檢查路由設定
用`ping`測試兩台主機是否可連

L4:
用`ss -tlnp | grep :22`檢查SSH服務是否監聽
用 `ssh` 指令測試是否可連線

## 網路拓樸圖
dev-a (NAT + Host-only)
        |
        | 192.168.152.x
        |
server-b (Host-only)

## 排錯紀錄
- 症狀：SSH無法連線 or ping失敗
- 診斷：我先用`ip address show`檢查網卡是否正常嗎
  用`ping`測試網路是否連通嗎
最後用了`ss -tlnp`檢查SSH是否在監聽嗎
- 修正：當網卡為 DOWN 時，使用 `ip link set <介面> up` 啟用網卡
當 SSH 停止時，使用 `systemctl start ssh` 啟動服務
- 驗證：重新使用 ping 測試網路
再使用 ssh 指令確認可以正常連線

## 設計決策

本實驗採用 dev-a 使用 NAT + Host-only 雙網卡設計，而 server-b 僅使用 Host-only。

dev-a 透過 NAT 可連接外網，用於安裝套件與更新系統；
Host-only 則建立一個內部網路，使 dev-a 與 server-b 可以直接通訊。

server-b 不配置 NAT，是為了模擬內部伺服器環境，避免其直接連外，並確保測試環境的隔離性與穩定性。
