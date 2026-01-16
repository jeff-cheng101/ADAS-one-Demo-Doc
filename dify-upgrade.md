## 注意 dify 顯示設定的問題，確保要先備份
- [/root/dify/dify/docker/.env]
- [/root/dify/dify/docker/nginx/proxy.conf.template]
- [/etc/nginx/conf.d/adasone.conf]


## 關閉服務

```bash
cd ~/dify/docker
docker compose down
```

## 執行全目錄複製

回到上一層目錄（`dify` 的父目錄），執行複製命令。


```bash
cd ../..
# 假設您的專案資料夾叫做 dify
# 加上 -p 參數可以保留文件的原始權限和時間戳（推薦）
cp -rp dify dify_backup_1.9.2_full
```


## 強制重置代碼

這行指令會強制把所有檔案內容覆蓋為遠端的最新版。

```bash
# 請先確認您在 dify 根目錄
cd ~/dify

# 強制重置 (這會解決檔案卡在 1.9.2 的問題)
git reset --hard origin/main
```

## 再次檢查

執行完上面的指令後，您再檢查一次檔案內容：

```bash
cat docker/docker-compose.yaml | grep "image: langgenius/dify-api"
```

這時候您應該會看到它 **自動** 變成了正確的版本（可能是變數 `${TAG}` 或新的版號），而且結構也是最新的。

## 清乾淨 → 修 tag 衝突 → checkout 最新 tag

``` bash
cd ~/dify

# 1) 還原 tracked 變更（等價於你問的 git restore .）
git restore .

# 2) 若有 untracked 檔案也要清乾淨（避免後續 checkout 被擋）
git clean -fd

# 3) 修正 tag 衝突（你已遇到 would clobber existing tag）
git tag -d 1.9.2
git fetch --tags --force

# 4) 切到最新版本 tag（你 repo tag 沒有 v 前綴）
git checkout 1.11.3

# 5) 確認
git describe --tags --exact-match
git status --porcelain

# 6) 確認 `docker/docker-compose.yaml` 內 image tag 是否正確：

cd ~/dify
grep -nE 'langgenius/dify-(api|web|worker|plugin-daemon|sandbox)' docker/docker-compose.yaml `

```

預期結果：

- `git describe --tags --exact-match` 輸出 `1.11.3`
- `git status --porcelain` 無輸出
- 若仍顯示 `:1.9.2`，就必須把這些 tag 調到 `:1.11.3`（或依該 compose 的變數機制更新 `.env` 內的 tag 變數）


## 調整 proxy.conf.template

```bash
cd ~/dify/docker/nginx
vim proxy.conf.template
```

- proxy.conf.template 內容

```bash
# Please do not directly edit this file. Instead, modify the .env variables related to NGINX configuration.

proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;                # 補上
proxy_set_header X-Forwarded-Host $host;                # 補上
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Port $server_port;

# 把來自外層 (Ubuntu Nginx) 的前綴繼續往後送
proxy_set_header X-Forwarded-Prefix $http_x_forwarded_prefix;  # 關鍵

proxy_http_version 1.1;
proxy_set_header Connection "";
proxy_buffering off;
proxy_read_timeout ${NGINX_PROXY_READ_TIMEOUT};
proxy_send_timeout ${NGINX_PROXY_SEND_TIMEOUT};
```

## 確認 .env 內容

```bash
cd ~/dify/docker
vim .env
```

- env 內容

```bash
# 以下要新增上至.env內容

# 對外 URL（跟瀏覽器實際看到的一致）
CONSOLE_API_URL=https://twister5.phison.com/dify
CONSOLE_WEB_URL=https://twister5.phison.com/dify
SERVICE_API_URL=https://twister5.phison.com/dify
APP_API_URL=https://twister5.phison.com/dify
APP_WEB_URL=https://twister5.phison.com/dify
FILES_URL=https://twister5.phison.com/dify

# 容器內互通
INTERNAL_FILES_URL=http://nginx:80

# 反向代理感知
RESPECT_XFORWARD_HEADERS_ENABLED=true

# 只把容器的 80 映射到宿主 8888；關閉 443 映射
EXPOSE_NGINX_PORT=8888
# EXPOSE_NGINX_SSL_PORT=0

# 不在容器內啟用 HTTPS（避免混淆）
NGINX_HTTPS_ENABLED=false

# 讓 Next.js 知道自己在 /dify 子路徑下
NEXT_PUBLIC_BASE_PATH=/dify
PUBLIC_URL=https://twister5.phison.com/dify

# 1) 允許 iframe 嵌入（參考 GitHub 官方討論 #27270）:contentReference[oaicite:15]{index=15}
NEXT_PUBLIC_ALLOW_EMBED=true

# 2) CORS 控制（若未在文件中出現，請先確認版本支援）
# 只允許你的入口網域；以下為範例，請依實際 Port 調整。
CONSOLE_CORS_ALLOW_ORIGINS=https://twister5.phison.com:3000
WEB_API_CORS_ALLOW_ORIGINS=https://twister5.phison.com:3000
```

## 啟動 DIFY

```bash
cd ~/dify/docker
docker compose pull
docker compose up -d
```

## 檢查 docker compose process

```bash
docker compose ps
```

- 顯示範例

```bash
NAME                     IMAGE                                       COMMAND                  SERVICE         CREATED         STATUS                   PORTS
docker-api-1             langgenius/dify-api:1.9.2                   "/bin/bash /entrypoi…"   api             3 minutes ago   Up 3 minutes             5001/tcp
docker-db-1              postgres:15-alpine                          "docker-entrypoint.s…"   db              3 minutes ago   Up 3 minutes (healthy)   5432/tcp
docker-nginx-1           nginx:latest                                "sh -c 'cp /docker-e…"   nginx           3 minutes ago   Up 3 minutes             0.0.0.0:443->443/tcp, [::]:443->443/tcp, 0.0.0.0:8888->80/tcp, [::]:8888->80/tcp
docker-plugin_daemon-1   langgenius/dify-plugin-daemon:0.3.3-local   "/bin/bash -c /app/e…"   plugin_daemon   3 minutes ago   Up 3 minutes             0.0.0.0:5003->5003/tcp, [::]:5003->5003/tcp
docker-redis-1           redis:6-alpine                              "docker-entrypoint.s…"   redis           3 minutes ago   Up 3 minutes (healthy)   6379/tcp
docker-sandbox-1         langgenius/dify-sandbox:0.2.12              "/main"                  sandbox         3 minutes ago   Up 3 minutes (healthy)   
docker-ssrf_proxy-1      ubuntu/squid:latest                         "sh -c 'cp /docker-e…"   ssrf_proxy      3 minutes ago   Up 3 minutes             3128/tcp
docker-weaviate-1        semitechnologies/weaviate:1.27.0            "/bin/weaviate --hos…"   weaviate        3 minutes ago   Up 3 minutes             
docker-web-1             langgenius/dify-web:1.9.2                   "/bin/sh ./entrypoin…"   web             3 minutes ago   Up 3 minutes             3000/tcp
docker-worker-1          langgenius/dify-api:1.9.2                   "/bin/bash /entrypoi…"   worker          3 minutes ago   Up 3 minutes             5001/tcp
docker-worker_beat-1     langgenius/dify-api:1.9.2                   "/bin/bash /entrypoi…"   worker_beat     3 minutes ago   Up 3 minutes             5001/tcp
```

- 發現仍還是 `:1.9.2` 必須做以下動作檢查

```bash
# 關閉dify
docker compose down

# 重拉docker compose 
docker compose pull

# 啟動
docker compose up -d

# 查看 dokcer process
docker compose ps

# 通用排查順序
docker compose logs --tail=300 worker
docker compose logs --tail=300 worker_beat
docker compose logs --tail=200 db
docker compose logs --tail=200 nginx
```

## 抓 500 的根因（只看關鍵服務 logs）

### 2.1 API logs（最關鍵）

`cd ~/dify/dify/docker
docker compose logs --tail=300 api `

### 2.2 Worker logs（常見是 migration/任務初始化失敗）

`docker compose logs --tail=300 worker
docker compose logs --tail=300 worker_beat `

### 2.3 DB logs（確認 schema/migration/連線）

`docker compose logs --tail=200 db `

### 2.4 Nginx access/error（確認 500 來自哪個 upstream）

`docker compose logs --tail=200 nginx `

 **到這裡你一定會看到明確錯誤字串** （例如 migration error、missing env、DB schema mismatch、plugin daemon 連不上、files url 錯、secret key 問題等）。後面是「依錯誤類型的固定處置」。

**現在所在目錄的 compose 專案，service 名稱沒有 `db`** ，所以 `docker compose logs db` 會回 `no such service: db`。

在你先前的 `docker compose ps` 螢幕中，DB 服務名稱是  **`db_1` 容器，但 service 名稱是 `db`** ；而你現在顯示找不到 `db`，代表你「目前執行 compose 的工作目錄」或「compose 檔」已經不是同一套（或 compose project name/檔案不同）。

以下用命令把 service 名稱一次查清並抓到正確 DB logs，然後用最短路徑定位 Internal Server Error 的根因。

## 查看docekr compose 服務名稱

```
docker compose config --services
```

- `docker compose config --services`  **沒有 db**
- 但你的 `.env` 仍有 `DB_HOST=db`、且 `MIGRATION_ENABLED=true`\
    ⇒ API 啟動時會嘗試連 `db` 進行 migration，但  **compose 專案內根本沒有 db service** ，因此 `/install` 直接 500 是必然結果。

這種狀態在 Dify 新版 compose 很常見： **PostgreSQL 被放到 profile（例如 `postgresql`）** ，沒開 profile 就不會出現 db。

## 1) 確認 db 在 profile 裡（用命令直接驗證）

```bash
cd ~/dify/dify/docker

docker compose --profile postgresql config --services
docker compose --profile postgres config --services
docker compose --profile pg config --services `
```
 
 **正確結果** ：其中一個命令會在輸出中出現 `db`（或類似 `db_postgres`）服務。

## 2) 用正確 profile 重新啟動整套（包含 db）

先停止目前這套（沒有 db 的那套）：

```bash
cd ~/dify/dify/docker
docker compose down --remove-orphans

# 假設你在第 1 步確認 profile 名稱是 `postgresql`（以下以 `postgresql` 示範；若你實際是 `postgres` 就把字改掉）：

cd ~/dify/docker
docker compose --profile postgresql pull
docker compose --profile postgresql up -d --force-recreate --remove-orphans

```

## 3) 立刻驗證：db service 是否回來

```bash
`cd ~/dify/dify/docker
docker compose --profile postgresql ps
docker compose --profile postgresql config --services
docker compose --profile postgresql logs -f --tail=200 api
```
 **你必須看到**

- services 清單包含 `db`
- `ps` 中 db 容器為 `Up`（最好 `healthy`）


## 重新RAG索引

```bash
cd ~/dify/docker

# 前景執行
docker compose --profile postgresql logs -f worker

# 背景執行
nohup docker compose --profile postgresql logs -f worker > /var/log/dify-worker.log 2>&1 &

```

--- 

# 檢查方式

## 1) 先確認 worker 是否真的在消化 queue（最常見卡點）

```bash
cd ~/dify/dify/docker 
docker compose --profile postgresql ps
```

確認 `worker`、`worker_beat` 都是 `Up`。

立刻抓 worker error（只看錯誤，降低雜訊）：

```bash
docker compose --profile postgresql logs --tail=500 worker | egrep -i 'error|exception|traceback|failed|embedding|weaviate|redis|celery'
docker compose --profile postgresql logs --tail=300 worker_beat | egrep -i 'error|exception|traceback|failed'

```

 **判讀（用字面即可）**

- 若出現 `embedding` / `No embedding model` / `provider not configured` / `rate limit`：是  **Embedding 設定或額度**  問題
- 若出現 `weaviate` / `connection refused` / `timeout`：是  **向量庫連線**  問題
- 若出現 `redis` / `connection`：是  **queue/Redis**  問題
- 若完全沒任何相關錯誤，且長時間沒有 indexing 任務輸出：是  **worker 卡住或 queue 堵塞**

## 2) 確認 Redis / Weaviate 容器健康（索引必經）

```bash
docker compose --profile postgresql ps redis weaviate
docker compose --profile postgresql logs --tail=200 redis | egrep -i 'error|oom|killed|fatal' docker compose --profile postgresql logs --tail=200 weaviate | egrep -i 'error|panic|fatal|schema|migrate|readonly|disk'
```

 **判讀**

- `weaviate` 若出現 `readonly`、`disk`、`panic`：通常是磁碟/權限或 schema 問題，索引會直接失敗或永遠排隊
- `redis` 若 OOM / 被 kill：queue 會卡住

* * * * *

## 3) 用「最小重啟」解除卡住的索引 pipeline（不動資料）

只重啟 worker 相關服務，避免影響使用者：

```bash
cd ~/dify/dify/docker
docker compose --profile postgresql restart worker worker_beat
```

重啟後立即觀察 worker 是否開始處理任務：

```bash
docker compose --profile postgresql logs -f --tail=200 worker
```

 **預期結果**

- 會出現處理文件/切分/embedding/upsert 類訊息
- 若重啟後立刻再次報相同 error（例如 embedding provider），就不是卡住，是設定問題

* * * * *

## 4) 若 worker log 明確是 Embedding 設定問題：直接定位到「工作區缺 embedding」

升級後最常見的「無法索引」實因： **原工作區/知識庫沒有可用的 Embedding model** （或 API key 失效、quota 0）。

此時 worker log 通常會包含以下特徵字串之一（只要命中就屬此類）：

- `embedding model` / `No available embedding model`
- `provider not configured`
- `Invalid API key` / `Unauthorized` / `quota exceeded` / `rate limit`

 **處置動作（不靠猜測）**

- 進 Dify Console →  **設定/模型供應商**  → 確認至少一個 Embedding 模型是  **可用狀態** （不是只設定 chat model）
- 對應供應商的 key/endpoint 正確、且有額度
- 完成後回到知識庫對該文件再按「重試/重新索引」

（你現在 UI 顯示可用/排隊中混雜，正符合 embedding 任務部分成功、部分失敗的狀態）

* * * * *

## 5) 若 worker log 明確是 Weaviate 問題：優先處理向量庫可寫入性

若 worker log 有 `weaviate` 連線/寫入錯誤，先做這個最直接的可寫入性確認（只檢查容器層，不改資料）：

```bash
cd ~/dify/dify/docker
docker compose --profile postgresql exec -T weaviate sh -lc 'wget -qO- http://127.0.0.1:8080/v1/.well-known/ready || true'
```

 **預期**

- 有回應（ready/ok 類 JSON）
- 若完全無回應或 timeout：先處理 weaviate 容器本身（log 會有原因）

* * * * *

## 6) 絕對不要先做的事（會造成不可逆）

在沒有看到明確錯誤前，不要做：

- 清 Redis queue / purge celery
- 刪 weaviate volumes
- 重建資料庫

這些會把「已排隊/已完成一半」的任務直接打掉，造成更大範圍重建。

---

# 調查結果

worker log 已經把根因釘死，不需要猜：

- `httpx.ConnectError: [Errno -3] Temporary failure in name resolution`
- `WeaviateStartUpError: Could not connect to Weaviate`
- 失敗發生在 `retry_document_indexing_task`（索引重試任務）

而你後面執行 `docker compose --profile postgresql ps redis weaviate` 的輸出  **只看到 redis** ，沒有 weaviate。\
⇒  **此 compose profile 目前根本沒有啟動 weaviate service** （或你用錯了 compose 檔/目錄導致 weaviate 不在同一個 project）。

所以「2 文件無法被索引」的直接原因是：

> **worker 需要連到 weaviate，但容器內 DNS 解析不到（或根本沒有 weaviate 這個 service），所以索引必定失敗。**

接下來只做「把 weaviate 起來 + 對齊 env」即可。

## 1) 先確認：你的 compose 內是否定義了 weaviate（以及 service 名稱）

```bash
cd ~/dify/dify/docker
docker compose --profile postgresql config --services | grep -i weaviate || true docker compose --profile postgresql config | grep -nE '^\s*weaviate:|WEAVIATE|VECTOR|QDRANT|MILVUS|OPENSEARCH' | head -n 80 
```

 **判定**

- 若第一行沒有輸出：代表此 profile 根本沒定義 weaviate（你現在的情況高度符合）
- 若有輸出但 `ps` 看不到：代表你尚未啟動它（或被移除）

* * * * *

## 2) 直接把 weaviate 起來（若 compose 有該 service）

如果第 1 步查到 service 名稱確實是 `weaviate`：

```bash
cd ~/dify/dify/docker
docker compose --profile postgresql up -d weaviate
docker compose --profile postgresql ps weaviate
```

若 service 名稱不是 `weaviate`（例如 `weaviate_vector`），用第 1 步輸出的正確名稱替換。

* * * * *

## 3) 立即做「容器內 DNS + HTTP」連通性驗證（這一步會直接驗證你修好沒）

### 3.1 先看 worker 解析得到 weaviate 嗎

```bash
cd ~/dify/dify/docker
docker compose --profile postgresql exec -T worker sh -lc 'getent hosts weaviate || getent hosts weaviate-1 || true'
```

### 3.2 直接從 worker 打 weaviate health

```bash
docker compose --profile postgresql exec -T worker sh -lc 'apk add --no-cache curl >/dev/null 2>&1 || true; curl -sS http://weaviate:8080/v1/.well-known/ready || true'
```

 **預期結果**

- 有回應（ready/ok 類 JSON 或文字）
- 若仍是 name resolution failure：表示 service 名稱不叫 `weaviate` 或根本不在同一個 docker network

* * * * *

## 4) 對齊 Dify 使用的 Weaviate endpoint（避免它指到不存在的 host）

在 `~/dify/dify/docker/.env` 內通常會有 Weaviate 相關變數。直接把它們印出來：

```bash
cd ~/dify/dify/docker
grep -nE 'WEAVIATE|VECTOR_STORE|VECTOR_DB|VDB|WEAVIATE_HOST|WEAVIATE_PORT|WEAVIATE_URL' .env || true
```

你要的最小一致性是：

- host =  **weaviate（service 名稱）**
- port =  **8080** （compose 預設）
- protocol = http（容器內）

若你看到 host 不是 `weaviate`，就改成 `weaviate`，例如：

`sed -i 's/^WEAVIATE_HOST=.*/WEAVIATE_HOST=weaviate/' .env  `

（只在 `.env` 內確實存在 `WEAVIATE_HOST=` 這行時才執行。）

* * * * *

## 5) 重啟 worker 讓它立刻重試（不需要重啟整套）

```bash
cd ~/dify/dify/docker
docker compose --profile postgresql restart worker
docker compose --profile postgresql logs -f --tail=200 worker
```

預期結果：

- 不再出現 `Temporary failure in name resolution`
- indexing 會開始成功寫入向量庫
- UI 的「2 文件無法被索引」會逐步消失（或可按重試後成功）

* * * * *

## 6) 若第 1 步顯示「profile postgresql 根本沒有 weaviate」：你目前的部署改成了「非 Weaviate」向量庫模式

這種情況下，worker 還在嘗試連 Weaviate，表示 `.env` / dataset 設定仍指向 Weaviate，但實際 compose 沒有。此時的唯一正確處置是二選一（不提供選項式建議，直接給動作）：

### 6.1 要用 Weaviate：把 weaviate service 加回 compose 並啟動

- 把 1.9.2 時期的 weaviate service 區塊合併回 `docker-compose.yaml`
- `docker compose --profile postgresql up -d weaviate`

### 6.2 不用 Weaviate：把向量庫切到你 compose 裡實際存在的 VDB

- 以 `.env` 內 `VECTOR_STORE` / `VECTOR_DB` 等變數為準，把它改到你實際啟用的 VDB（例如 pgvector / qdrant 等）
- 並確保相應 service 存在且 `Up`

但你目前貼的 `ps` 清單先前曾包含 `weaviate`（1.9.2 時期 mount 也有），所以最短修復路徑通常是  **把 weaviate 起來** 。

* * * * *

### 你現在立刻要執行的最短三條命令

```bash
cd ~/dify/dify/docker
docker compose --profile postgresql config --services | grep -i weaviate
docker compose --profile postgresql up -d weaviate
docker compose --profile postgresql exec -T worker sh -lc 'apk add --no-cache curl >/dev/null 2>&1 || true; curl -sS http://weaviate:8080/v1/.well-known/ready'
```

這三步做完，索引失敗的根因就會被消除或被精確定位到「service 名稱/網路/變數」層。

---

已確認事實：`docker compose --profile postgresql config --services`  **沒有 `weaviate`** 。但你的 `worker` logs 明確顯示它仍在嘗試連 `weaviate`，且失敗原因是  **name resolution** 。\
因此根因是： **部署組態（compose + env）不一致：worker 設定使用 Weaviate，但 compose 專案內沒有 Weaviate 服務可供解析。**

以下提供不分岔、可執行的修復流程： **把 Weaviate service 補回目前啟用的 compose（postgresql profile）** ，讓 `weaviate` 這個 hostname 在同一個 docker network 內可被解析，索引才會恢復。

---

事實結論：

- 你用 `--profile postgresql` 啟動時， **weaviate 不會被包含** （因為它只屬於 `weaviate` profile）
- worker 仍在嘗試連 weaviate ⇒ 你的 `.env / dataset` 設定仍選用 weaviate 作為向量庫
- 所以索引必然失敗（你 logs 的 DNS failure 正是結果）

接下來的修復只有一條路徑： **啟動時把 `weaviate` profile 一起帶上** （不要改 weaviate 的 profile 設定；改了會影響官方組態邏輯）。

## 1) 用雙 profile 啟動（postgresql + weaviate）

```bash
cd ~/dify/dify/docker

docker compose --profile postgresql --profile weaviate up -d --remove-orphans 
```

預期結果：

- `weaviate` 容器會被建立並 `Up`
- worker 能解析 `weaviate` hostname

* * * * *

## 2) 驗證 weaviate 已加入同一個 compose project

```bash
docker compose --profile postgresql --profile weaviate ps | grep -i weaviate 
```

預期：有 `weaviate` 那一行。

* * * * *

## 3) 從 worker 容器內測 DNS 與 ready

```bash
docker compose --profile postgresql --profile weaviate exec -T worker getent hosts weaviate

docker compose --profile postgresql --profile weaviate exec -T worker sh -lc 'apk add --no-cache curl >/dev/null 2>&1 || true; curl -sS http://weaviate:8080/v1/.well-known/ready'  
```

預期：

- `getent` 會回 IP
- `ready` endpoint 會有回應

* * * * *

## 4) 觸發索引重試（重啟 worker 即可）

```bash
docker compose --profile postgresql --profile weaviate restart worker worker_beat
docker compose --profile postgresql --profile weaviate logs -f --tail=200 worker 
```

預期：

- 不再出現 `Temporary failure in name resolution`
- 不再出現 `Could not connect to Weaviate`
- UI 的「2 文件無法被索引」會逐步消失，或可重試成功

* * * * *

## 5) 之後所有操作的固定寫法（避免再次漏啟 weaviate）

你之後的 `ps / logs / up / down` 都要固定用：

```bash
docker compose --profile postgresql --profile weaviate <command> 
```

例如：

```bash
docker compose --profile postgresql --profile weaviate ps
docker compose --profile postgresql --profile weaviate logs -f worker 
```

* * * * *

這個修復對應你提供的確定事實（weaviate 僅屬於 `weaviate` profile），不涉及任何假設。

---

## 總結

# Dify Docker 升級完整 SOP（實戰整理版）

> 適用情境
>
> - Docker Compose 部署
> - 以  **完整目錄複製（`cp -rp`）**  作為回退備份
> - 實際涵蓋版本從  **1.9.2 → 1.11.x**  的升級經驗4

## 0\. 升級前重要原則（不可略過）

### 0.1 必須先備份的檔案（顯示 / Proxy 相關）

以下檔案若遺失，會直接造成登入、500、路徑錯誤：

- `/root/dify/dify/docker/.env`
- `/root/dify/dify/docker/nginx/proxy.conf.template`
- `/etc/nginx/conf.d/adasone.conf`

    dify-upgrade

* * * * *

## 1\. 停止 Dify 服務

```bash
cd ~/dify/docker
docker compose down
``` :contentReference[oaicite:1]{index=1}

**判定標準**
- `docker compose ps` 無任何 Up container

---

## 2. 進行完整目錄備份（核心回退機制）

回到 `dify` 的父目錄：

```bash
cd ../..
cp -rp dify dify_backup_1.9.2_full
``` :contentReference[oaicite:2]{index=2}

**此備份包含**
- 程式碼
- `.env`
- docker-compose.yaml
- volumes（DB / Redis / Weaviate / Files）

---

## 3. 重置程式碼到官方最新狀態

```bash
cd ~/dify
git reset --hard origin/main
``` :contentReference[oaicite:3]{index=3}

用途：
- 清除卡在舊版（1.9.2）的檔案
- 確保 compose 與程式碼結構為最新版

---

## 4. 確認 compose 內容是否已更新

```bash
cat docker/docker-compose.yaml | grep "image: langgenius/dify-api"
``` :contentReference[oaicite:4]{index=4}

**預期**
- 使用 `${TAG}` 或新版 tag
- 不應殘留 `:1.9.2`

---

## 5. 清理狀態 → 解決 tag 衝突 → 切換正式版本

```bash
cd ~/dify

git restore .
git clean -fd

git tag -d 1.9.2
git fetch --tags --force

git checkout 1.11.3
git describe --tags --exact-match
git status --porcelain
``` :contentReference[oaicite:5]{index=5}

**驗證條件**
- `git describe` = `1.11.3`
- `git status` 無輸出

---

## 6. 調整 Nginx Proxy Header（子路徑部署關鍵）

```bash
cd ~/dify/docker/nginx
vim proxy.conf.template

```

必要設定（範例）：

```bash
CONSOLE_API_URL=https://<domain>/dify
CONSOLE_WEB_URL=https://<domain>/dify
SERVICE_API_URL=https://<domain>/dify
APP_API_URL=https://<domain>/dify
APP_WEB_URL=https://<domain>/dify
FILES_URL=https://<domain>/dify

INTERNAL_FILES_URL=http://nginx:80

RESPECT_XFORWARD_HEADERS_ENABLED=true

EXPOSE_NGINX_PORT=8888
NGINX_HTTPS_ENABLED=false

NEXT_PUBLIC_BASE_PATH=/dify
PUBLIC_URL=https://<domain>/dify
NEXT_PUBLIC_ALLOW_EMBED=true
``` :contentReference[oaicite:7]{index=7}

---

## 8. 啟動 Dify（含 profile）

```bash
cd ~/dify/docker
docker compose pull
docker compose up -d

```

若使用 PostgreSQL + Weaviate（實際經驗）：

```bash
docker compose --profile postgresql --profile weaviate up -d --remove-orphans
```

9\. 基礎狀態檢查

```bash
docker compose ps
```

重點服務必須存在：

- api
- web
- worker / worker_beat
- db
- redis
- weaviate

## 10\. 500 Error / 無法索引的實戰排查 SOP

### 10.1 只看關鍵 logs（降噪）

```bash
docker compose logs --tail=300 api
docker compose logs --tail=300 worker
docker compose logs --tail=300 worker_beat
docker compose logs --tail=200 nginx
docker compose logs --tail=200 db
``` :contentReference[oaicite:9]{index=9}

---

### 10.2 判斷是否缺少 db / weaviate service

```bash
docker compose config --services

```

**已確認的實際根因**

- `worker` 嘗試連線 `weaviate`
- compose profile 內卻未啟動 `weaviate`
- 造成 DNS resolution failure

11\. 正確啟動方式（避免再次踩雷）

```bash
docker compose --profile postgresql --profile weaviate ps
docker compose --profile postgresql --profile weaviate up -d

```

驗證：

```bash
docker compose --profile postgresql --profile weaviate exec -T worker \
  sh -lc 'getent hosts weaviate && curl -s http://weaviate:8080/v1/.well-known/ready'
``` :contentReference[oaicite:11]{index=11}

---

## 12. 重新觸發索引（不破壞資料）

```bash
docker compose --profile postgresql --profile weaviate restart worker worker_beat
docker compose --profile postgresql --profile weaviate logs -f worker

```

13. 回退 SOP（任何異常直接用）

```bash
docker compose down
cd ..
rm -rf dify
cp -rp dify_backup_1.9.2_full dify
cd dify/docker
docker compose up -d

```

## 14\. 絕對禁止事項（已驗證風險）

- 清 Redis queue
- 刪 Weaviate volumes
- 手動改 DB schema
- 在未確認 root cause 前重建資料庫

    dify-upgrade

* * * * *

## 最終結論（事實整理）

- 你此次升級問題的 **實際根因** 為：\
     **compose 啟動 profile 與 `.env / worker 設定不一致（Weaviate 未啟動）**

- `cp -rp` 全目錄備份是正確且安全的回退策略
- 升級後「所有操作必須固定帶上 profile」