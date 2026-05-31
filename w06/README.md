# W06｜Docker Image 與 Dockerfile

## 映像組成
- Layers 是什麼：是Docker image一層一層的內容像安裝套件、複製檔案這些動作都可能變成一個layer。Docker會把很多layer疊起來變成最後的image，而且一樣的layer可以共用，所以比較不會浪費空間。
- Config 是什麼：是image的一些設定像container啟動時要跑什麼指令、工作路徑在哪、環境變數有哪些，然摟Docker啟動container時會去讀這些設定。
- Manifest 是什麼：像image的說明清單，因為裡面會記錄這image有哪些layers還有config的資訊。Docker pull image的時候會先看manifest，然後再把對應的layer抓下來。

## python:3.12-slim inspect 摘錄
- Config.Cmd：["python3"]
- Config.Env：["PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin","LANG=C.UTF-8","GPG_KEY=7169605F62C751356D54A6A821F680F5FA6305","PYTHON_VERSION=3.12.13","PYTHON_SHA256=c08bc65a81971c1dd5783182826503369466c7e67374d1646519adf05207b684"]
- Config.WorkingDir：""
- RootFS.Layers 數量：4

## Layer 快取實驗
| 情境 | build 時間 |
|---|---|
| v1 首次 build | 0m5.132s |
| v1 改 app.py 後 rebuild | 0m4.969s |
| v2 首次 build | 0m4.994s |
| v2 改 app.py 後 rebuild | 0m0.817s |

觀察（用自己的話寫）：為什麼 v2 的 rebuild 這麼快？ A:因為她先把requirements.txt複製進去安裝套件，之後才複製app.py。
這樣只改app.py時pip install那層沒有變就可以直接用cache，不用重新安裝套件。
所以v2只需要重跑後面比較少的layer時間短很多。

## CMD vs ENTRYPOINT 實驗
| 寫法 | `docker run <img>` 輸出 | `docker run <img> extra1 extra2` 輸出 |
|---|---|---|
| CMD shell form | `argv = ['show_args.py', 'default1', 'default2']` | 出現 `exec: "extra1": executable file not found` 錯誤 |
| CMD exec form | `argv = ['show_args.py', 'default1', 'default2']` | 出現 `exec: "extra1": executable file not found` 錯誤 |
| ENTRYPOINT + CMD | `argv = ['show_args.py', 'default1', 'default2']` | `argv = ['show_args.py', 'extra1', 'extra2']` |

結論：  
CMD比較像預設指令，如果docker run後面加參數的話，原本CMD的內容可能會被覆蓋掉。
ENTRYPOINT則是固定執行的主程式，後面的參數會接在後面，所以通常會搭配CMD一起用。

## Multi-stage 大小對照
| Image | SIZE |
|---|---|
| python:3.12（builder base） | 428MB |
| python:3.12-slim（runtime base） | 45.4MB |
| myapp:v2（單階段） | 48.1MB |
| myapp:multi（多階段） | 44.8MB |

解釋（用自己的話寫）：builder stage 的 layer 去哪了？
A:layer沒有真的消失，只是最後的image不會把builder包進去。multi-stage build只會保留最後runtime stag用到的layer所以image會比較小。不過builder的layer其實還會留在本機cache，之後build的時候還能繼續用。

## .dockerignore 故障注入
| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| du -sh . | 44K | 151M | 151M |
| build context 傳輸大小 | 10.75kB | 157.3MB | 6.656kB |
| build 時間 | 0m0.130s | 0m0.377s | 0m0.111s |

## 排錯紀錄
- 症狀：改了app.py一行之後docker build還是重新跑pip install。
- 診斷：Dockerfile.v1把COPY app/ .放在pip install前面導致app.py一改後面的layer cache全部失效。
- 修正：改成先COPY requirements.txt再RUN pip install最後才COPY app.py。
- 驗證：改成Dockerfile.v2後只改app.py時pip install那層會顯示Using cache，build時間也從大約5秒降到1秒內。

## 設計決策
這次我選python:3.12-slim而不是alpine因為slim比較穩定，相容性也比較好。雖然alpine體積更小，但很多Python套件在alpine 上容易有編譯問題，可能要另外裝gcc or musl-dev之類的東西。這次只是Flask app，用slim就已經夠小，而且比較不容易出問題。
