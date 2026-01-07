# Cloudflare 趨勢分析功能 API 接口

## POST: /api/cloudflare/trend/load-comparison
- Cloudflare 趨勢分析功能

### Requirements

|Attribute|Type|Required|Description|
|---|---|---|---|
|NA||||


### Parameter fields

|Attribute|Type|Required|Description|
|---|---|---|---|
|timeRange|String|Y|值：1h ,6h ,1d ,7d ,30d|

### Example Parameter
```json
{
  "timeRange": "7d"
}
```
### Response fields

| Attribute           | Type          | Required | Description   |
| ------------------- | ------------- | -------- | ------------- |
| success             | Boolean       | Y        | 是否成功取得資料      |
| totalAttack         | Object        | Y        | 總攻擊量統計        |
| httpPct             | Object        | Y        | HTTP 攻擊佔比(活動量佔比) |
| lockdownRate        | Object        | Y        | 封鎖成功率         |
| currentAttackTrend  | Array<Object> | Y        | 目前時段攻擊趨勢（依小時） |
| previousAttackTrend | Array<Object> | Y        | 前一期間攻擊趨勢（依小時） |
| httpVolume          | Object        | Y        | HTTP 流量統計     |
| dataVolume          | Object        | Y        | 資料傳輸量統計       |
| pageView            | Object        | Y        | 頁面瀏覽比例        |
| visits              | Object        | Y        | 訪問次數          |
| sourceIP            | Array<Object> | Y        | 主要來源 IP 統計    |
| triggerRule         | Array<Object> | Y        | 觸發的安全規則統計     |
| hosts               | Array<Object> | Y        | 受攻擊主機（Host）統計 |
| path                | Array<Object> | Y        | 請求路徑統計        |
| country             | Array<Object> | Y        | 來源國家統計        |
| other               | Object        | Y        | 預留欄位，未定義內容    |
| aiAnalysis          | Object        | Y        | AI 攻擊趨勢分析報告    |

### totalAttack fields

| Attribute | Type   | Required | Description   |
| --------- | ------ | -------- | ------------- |
| quantity  | Number | Y        | 總攻擊次數         |
| change    | Number | Y        | 與前一期間相比的變化(百分比) |

### httpPct fields

| Attribute | Type   | Required | Description    |
| --------- | ------ | -------- | -------------- |
| quantity  | Number | Y        | 活動量佔比 |
| change    | Number | Y        | 活動量佔比變化 (百分比)      |


### lockdownRate fields

| Attribute | Type   | Required | Description |
| --------- | ------ | -------- | ----------- |
| quantity  | Number | Y        | 封鎖率  |
| change    | Number | Y        | 封鎖率變化(百分比)     |


### currentAttackTrend(當期) / previousAttackTrend(上期) item fields

| Attribute | Type              | Required | Description   |
| --------- | ----------------- | -------- | ------------- |
| count     | Number            | Y        | 該小時的攻擊次數      |
| hour      | String (ISO 8601) | Y        | 統計的小時時間點（UTC） |


### httpVolume fields

| Attribute | Type   | Required | Description    |
| --------- | ------ | -------- | -------------- |
| quantity  | String | Y        | HTTP 流量數量（含單位） |
| change    | Number | Y        | HTTP 流量變化百分比   |


### dataVolume fields

| Attribute | Type   | Required | Description |
| --------- | ------ | -------- | ----------- |
| quantity  | String | Y        | 資料傳輸量（含單位）  |
| change    | Number | Y        | 資料量變化百分比    |


### pageView fields

| Attribute | Type   | Required | Description |
| --------- | ------ | -------- | ----------- |
| quantity  | String | Y        | 頁面瀏覽次數（含單位）    |
| change    | Number | Y        | 瀏覽比例變化(點閱率)      |

### visits fields

| Attribute | Type   | Required | Description |
| --------- | ------ | -------- | ----------- |
| quantity  | String | Y        | 訪問次數（含單位）   |
| change    | Number | Y        | 訪問次數變化      |


### sourceIP item fields

| Attribute | Type   | Required | Description           |
| --------- | ------ | -------- | --------------------- |
| ClientIP  | String | Y        | 來源 IP 位址（IPv4 或 IPv6） |
| cnt       | Number | Y        | 該 IP 的請求次數            |
| change    | Number | Y        | 與前一期間相比的變化            |


### triggerRule item fields

| Attribute               | Type   | Required | Description |
| ----------------------- | ------ | -------- | ----------- |
| SecurityRuleDescription | String | Y        | 觸發該規則描述      |
| cnt                     | Number | Y        | 觸發該規則的次數    |
| change                  | Number | Y        | 觸發次數變化      |


### hosts item fields

| Attribute         | Type   | Required | Description  |
| ----------------- | ------ | -------- | ------------ |
| ClientRequestHost | String | Y        | 請求的 Host 名稱  |
| cnt               | Number | Y        | 該 Host 的請求次數 |
| change            | Number | Y        | 請求次數變化       |


###　path item fields

| Attribute         | Type   | Required | Description |
| ----------------- | ------ | -------- | ----------- |
| ClientRequestPath | String | Y        | 請求路徑        |
| cnt               | Number | Y        | 該路徑的請求次數    |
| change            | Number | Y        | 請求次數變化      |

### scountry item fields

| Attribute              | Type   | Required | Description |
| ---------------------- | ------ | -------- | ----------- |
| geoip.geo.country_name | String | Y        | 來源國家名稱      |
| cnt                    | Number | Y        | 該國家的請求次數    |
| change                 | Number | Y        | 請求次數變化      |

### aiAnalysis item fields

| Attribute              | Type   | Required | Description |
| ---------------------- | ------ | -------- | ----------- |
| trendAnalysis          | String | Y        | 分析報告內容 |
| metadata               | String | Y        | 分析資訊    |

### metadata item fields

| Attribute              | Type   | Required | Description |
| ---------------------- | ------ | -------- | ----------- |
| analysisId             | String | Y        | 分析任務的唯一識別碼 |
| timestamp              | String | Y        | 分析完成或回應產生的時間點 |
| model                  | String | Y        | 實際執行分析的 AI 模型名稱 |
| aiProvider             | String | Y        | 提供模型推論能力的實際執行平台 |
| isAIGenerated          | String | Y        | 是否為 AI 自動生成結果的標記 |
| analysisType           | String | Y        | 本次分析任務的業務語義分類    |
| responseTime           | String | Y        | AI 完整產生回應所耗費的時間（毫秒） |
| promptLength           | String | Y        | 送入模型的 Prompt 長度指標 |

### Example Response
```json
{
    "success": true,
    "totalAttack": {
        "quantity": 0,
        "change": -100
    },
    "httpPct": {
        "quantity": 0,
        "change": -100
    },
    "lockdownRate": {
        "quantity": 0,
        "change": -100
    },
    "currentAttackTrend": [],
    "previousAttackTrend": [
        {
            "hour": "2026-01-06T19:00:00.000Z",
            "count": 1
        }
    ],
    "httpVolume": {
        "quantity": "647",
        "change": -2.85
    },
    "dataVolume": {
        "quantity": "18.37MB",
        "change": -29.35
    },
    "pageView": {
        "quantity": "43",
        "change": 95.45
    },
    "visits": {
        "quantity": "463",
        "change": -15.05
    },
    "sourceIP": [
        {
            "ClientIP": "75.166.148.110",
            "cnt": 20,
            "change": 1900
        },
        {
            "ClientIP": "202.39.33.231",
            "cnt": 16,
            "change": 77.78
        },
        {
            "ClientIP": "50.232.53.50",
            "cnt": 13,
            "change": -27.78
        },
        {
            "ClientIP": "201.219.95.220",
            "cnt": 13,
            "change": 116.67
        },
        {
            "ClientIP": "211.72.13.223",
            "cnt": 10,
            "change": 150
        }
    ],
    "triggerRule": [
        {
            "SecurityRuleDescription": "log",
            "cnt": 618,
            "change": -4.19
        },
        {
            "SecurityRuleDescription": "",
            "cnt": 29,
            "change": 45
        }
    ],
    "hosts": [
        {
            "ClientRequestHost": "www.phison.com",
            "cnt": 448,
            "change": -20
        },
        {
            "ClientRequestHost": "webmail.phison.com",
            "cnt": 105,
            "change": 41.89
        },
        {
            "ClientRequestHost": "webapp.phison.com",
            "cnt": 36,
            "change": 800
        },
        {
            "ClientRequestHost": "grm.phison.com",
            "cnt": 19,
            "change": 137.5
        },
        {
            "ClientRequestHost": "phison.com",
            "cnt": 15,
            "change": 275
        }
    ],
    "path": [
        {
            "ClientRequestPath": "/en/category/press-releases",
            "cnt": 130,
            "change": -57.24
        },
        {
            "ClientRequestPath": "/en/category/phison-in-the-news|press-releases",
            "cnt": 66,
            "change": 53.49
        },
        {
            "ClientRequestPath": "/owa/service.svc",
            "cnt": 50,
            "change": 56.25
        },
        {
            "ClientRequestPath": "/",
            "cnt": 46,
            "change": 76.92
        },
        {
            "ClientRequestPath": "/owa/ev.owa2",
            "cnt": 40,
            "change": 29.03
        }
    ],
    "country": [
        {
            "geoip_client.country_name": "United States",
            "cnt": 250,
            "change": 42.86
        },
        {
            "geoip_client.country_name": "Taiwan",
            "cnt": 107,
            "change": 75.41
        },
        {
            "geoip_client.country_name": "China",
            "cnt": 92,
            "change": -69.23
        },
        {
            "geoip_client.country_name": "United Kingdom",
            "cnt": 61,
            "change": 38.64
        },
        {
            "geoip_client.country_name": "France",
            "cnt": 30,
            "change": 36.36
        }
    ],
    "other": {},
    "aiAnalysis": {
        "trendAnalysis": "## 一、整體攻擊活動趨勢\n\n| 指標 | 本期數值 | 與上期比較 | 變化幅度 | 可能意義 |\n|------|----------|------------|----------|----------|\n| **攻擊活動量** | **0** | -100 % | 完全消失 | 代表偵測或阻斷機制未再捕捉到已知攻擊類型（如 OWASP Top‑10、已設定的自訂規則） |\n| **HTTP 攻擊佔比** | 0 % | -100 % | 無任何 HTTP 攻擊 | 可能是攻擊者切換至非 HTTP 協議（如 DNS、SMTP）或已被防火牆/IPS 完全阻斷 |\n| **封鎖成功率** | 0 % | -100 % | 無封鎖紀錄 | 由於本期沒有偵測到攻擊，封鎖率自然為 0，並不代表防禦失效，而是偵測面向不足 |\n\n**結論**  \n整體「攻擊活動」指標呈現 **0**，顯示在本期的偵測規則與流量樣本中未觀測到明顯的惡意行為。這可能是因為：\n\n1. **防禦機制提升**（上期已阻斷的攻擊被成功攔截，未留下痕跡）。  \n2. **攻擊手法轉變**（攻擊者改用非 HTTP 或加密通道，未被現有規則捕獲）。  \n3. **偵測規則衰退**（某些規則被誤關閉或閾值調高，導致未能偵測）。\n\n---\n\n## 二、流量模式變化\n\n| 指標 | 本期 | 與上期比較 | 變化幅度 | 解讀 |\n|------|------|------------|----------|------|\n| **HTTP 請求次數** | 647 | -2.85 % | 輕微下降 | 大體穩定，說明正常使用者流量仍維持。 |\n| **資料量** | 18.37 MB | -29.35 % | 大幅下降 | 代表單位請求平均傳輸量下降，可能是 **靜態資源請求**（圖片、影片）減少，或壓縮/快取機制生效。 |\n| **頁面瀏覽（Page Views）** | 43 | +95.45 % | 幾乎翻倍 | 雖然請求總量略降，但頁面瀏覽數激增，代表 **每個請求的深度提升**（使用者在同一頁面多次點擊/滾動），或是 **單頁應用（SPA）** 的 AJAX 請求未被計入請求次數但被計入頁面瀏覽。 |\n| **造訪次數（Sessions）** | 463 | -15.05 % | 中度下降 | 訪客數較上期下降，可能是 **季節性流量波動** 或 **行銷活動減少**。 |\n\n**綜合觀察**  \n- **請求量與資料量** 同步下降，表示 **帶寬使用降低**，對資源負載有正面效益。  \n- **頁面瀏覽激增** 但 **訪客減少**，暗示剩餘訪客的互動更深入，可能是 **核心客戶** 或 **內部測試人員** 的行為。  \n- 若此頁面瀏覽的增長集中在少數 URL（如 `/en/category/phison-in-the-news|press-releases`），需留意是否有 **爬蟲或自動化腳本** 重複抓取同一頁面。\n\n---\n\n## 三、來源變化分析\n\n### 1. Top‑5 來源 IP\n\n| IP | 本期次數 | 變化 | 可能來源 |\n|----|----------|------|----------|\n| **75.166.148.110** | 20 | **+1900 %** (前期約 1 次) | 突然增量，值得進一步追蹤，如 VPN、雲端服務或被劫持的主機。 |\n| **202.39.33.231** | 16 | **+77.78 %** | 近期活躍，可能是合法用戶或持續掃描來源。 |\n| **50.232.53.50** | 13 | **-27.78 %** | 活躍度下降，若為已知惡意 IP，可視為風險降低。 |\n| **201.219.95.220** | 13 | **+116.67 %** | 大幅增長，需檢查是否屬於同一組織的子網。 |\n| **211.72.13.223** | 10 | **+150 %** | 同樣顯著上升，建議加入黑名單或執行額外驗證。 |\n\n**觀察**：有 **三個 IP**（75.166.148.110、201.219.95.220、211.72.13.223）在本期呈現 **高速增長**，可能是新興的 **掃描器、爬蟲或嘗試暴力登入** 行為。\n\n### 2. 觸發規則 Top‑5\n\n| 規則 | 本期次數 | 變化 |\n|------|----------|------|\n| **log** | 618 | **-4.19 %** (輕微下降) |\n| **空規則 (\":\")** | 29 | **+45 %** |\n\n> **說明**：  \n> - `log` 為通用記錄規則，仍占大多數觸發，顯示大多數流量仍屬於正常/低危害行為。  \n> - `\":\"` 為可能的自訂或錯誤規則（名稱缺失），其次數提升 45 % 代表 **規則配置異常** 或 **偵測器誤報**，需要立即檢查規則內容。\n\n### 3. 主機 Top‑5\n\n| 主機 | 本期次數 | 變化 |\n|------|----------|------|\n| **www.phison.com** | 448 | -20 % |\n| **webmail.phison.com** | 105 | +41.89 % |\n| **webapp.phison.com** | 36 | **+800 %** (前期約 4 次) |\n| **grm.phison.com** | 19 | +137.5 % |\n| **phison.com** | 15 | +275 % |\n\n**解讀**  \n- **公司主站**流量下降 20 %，可能因為使用者改以子域名或 API 存取。  \n- **Webmail、Webapp、GRM**等子域名流量激增，代表 **內部應用或服務的使用頻率提升**，也可能是 **自動化工具（如 Outlook OWA）** 的大量請求。\n\n### 4. 路徑 Top‑5\n\n| 路徑 | 本期次數 | 變化 |\n|------|----------|------|\n| `/en/category/press-releases` | 130 | -57.24 % |\n| `/en/category/phison-in-the-news|press-releases` | 66 | +53.49 % |\n| `/owa/service.svc` | 50 | +56.25 % |\n| `/` (根目錄) | 46 | +76.92 % |\n| `/owa/ev.owa2` | 40 | +29.03 % |\n\n**觀察**  \n- 主要新聞發佈頁面流量急速下降，可能是 **內容更新頻率降低** 或 **外部連結減少**。  \n- OWA（Outlook Web Access）相關路徑出現顯著上升，暗示 **郵件服務被大量存取**，需要注意是否有 **自動化登入、暴力破解** 或 **資料外洩掃描**。  \n\n### 5. 國家 Top‑5\n\n| 國家 | 本期次數 | 變化 |\n|------|----------|------|\n| **United States** | 250 | +42.86 % |\n| **Taiwan** | 107 | +75.41 % |\n| **China** | 92 | -69.23 % |\n| **United Kingdom** | 61 | +38.64 % |\n| **France** | 30 | +36.36 % |\n\n**解讀**  \n- **美國、台灣、英國、法國**的請求均顯著上升，顯示 **國際使用者活動回暖**。  \n- **中國**流量大幅下降，可能是 **網路封鎖、IP 被封或攻擊者轉移至其他來源**。  \n\n---\n\n## 四、異常模式識別\n\n| 異常類型 | 具體現象 | 潛在原因 | 建議的驗證方式 |\n|----------|----------|----------|----------------|\n| **新增高頻惡意 IP** | 75.166.148.110、201.219.95.220、211.72.13.223 為急增 IP | 可能是惡意掃描、僵屍主機或代理服務 | - 透過 WHOIS、IP Reputation API（如 VirusTotal, AbuseIPDB）檢查<br>- 在 WAF/IPS 中臨時封鎖或設定速率限制 |\n| **規則名稱缺失** (`:`) | 觸發次數 +45 % | 規則配置錯誤或測試規則遺留 | - 檢查規則庫，刪除或補全名稱<br>- 重新部署規則後觀測變化 |\n| **OWA 路徑激增** | `/owa/service.svc`、`/owa/ev.owa2` 分別 +56 %/+29 % | 潛在暴力登入、憑證猜測或自動化抓取 | - 啟用帳號鎖定與多因素驗證（MFA）<br>- 在 WAF 加入 OWASP‑OTL‑Auth‑Brute‑Force 規則 |\n| **Webapp 流量暴增** | `webapp.phison.com` +800 % | 內部新應用上線或測試流量，亦可能是 API 掃描 | - 檢查 API 日誌，確認合法來源<br>- 若為測試環境，應把測試 IP 加入白名單 |\n| **頁面瀏覽激增** | Page Views +95 % 但 Sessions -15 % | 單一或少數使用者大量操作，或自動化腳本重複載入頁面 | - 查看 User‑Agent、Referer<br>- 使用行為分析（行為指紋）偵測機器人 |\n| **中國流量下降** | -69 % | 可能是 IP 被封、或攻擊者遷移來源 | - 若是合法業務需求，檢查是否誤封<br>- 若為攻擊來源，持續觀察是否有新 IP 取代 |\n\n---\n\n## 五、潛在安全威脅評估\n\n| 威脅類別 | 依據 | 風險等級 | 可能影響 |\n|----------|------|----------|----------|\n| **IP 識別的掃描/暴力攻擊** | 多個 IP 短期內請求次數劇增（+1900 %） | 高 | 可能嘗試登入 OWA、Webmail，若成功將取得內部郵件資料。 |\n| **OWA API 濫用** | `/owa/service.svc`、`/owa/ev.owa2` 請求上升 50‑60 % | 中‑高 | 若未加強驗證，可導致帳號暴力破解或資訊洩漏（行事曆、聯絡人）。 |\n| **Webapp API 掃描** | `webapp.phison.com` +800 % | 中 | 若 Webapp 暴露未授權的 API，攻擊者可能抓取或修改資料。 |\n| **規則配置錯誤** | 觸發 `:` 規則 +45 % | 中 | 誤報可能掩蓋真實威脅，或造成資源浪費。 |\n| **高頻頁面瀏覽** | Page Views +95 % | 低‑中（視為自動化） | 若為爬蟲，可能對搜尋引擎優化（SEO）或內容盜用造成影響。 |\n| **國際流量回升** | 美國、台灣、英國、法國 +30‑75 % | 中 | 更多外部使用者意味更大的攻擊面；需確保 CDN、WAF 設定適當。 |\n\n---\n\n## 六、建議的監控與防護措施\n\n以下為 **可直接執行** 的 12 項具體建議，依重要性排序：\n\n### 1. 立即對異常 IP 採取速率限制或暫時封鎖\n- **動作**：在 WAF / IPS 設定 `rate‑limit`（如 10 req/sec）或 `block` 針對 IP `75.166.148.110`, `201.219.95.220`, `211.72.13.223`。  \n- **驗證**：觀察 24 小時內該 IP 的觸發次數是否降為 0。\n\n### 2. 強化 OWA（Outlook Web Access）安全\n- **啟用多因素驗證（MFA）**：所有使用 OWA 的帳號必須完成 MFA。  \n- **帳號鎖定政策**：5 次失敗登入即鎖定 15 分鐘。  \n- **在 WAF 加入 OWASP‑OTL‑Auth‑Brute‑Force 及 `OWA‑Path‑Protection` 規則**。\n\n### 3. 檢查與修正空名稱規則 (`:`)\n- **動作**：在規則庫中搜尋名稱為 `:` 的條目，確認其原始意圖。若為測試規則，直接移除；若遺失名稱，補上正確描述。  \n- **測試**：重新部署規則後，觀察觸發次數是否回歸正常。\n\n### 4. 為 `webapp.phison.com` 加入 API 防護\n- **API 金鑰或 Token**：要求所有外部呼叫必須附帶有效憑證。  \n- **速率限制**：每個憑證每秒不超過 5 次請求。  \n- **日誌細分**：將 `webapp` 相關的日誌寫入單獨檔案，方便後續分析。\n\n### 5. 建立行為分析（Behavioral Analytics）系統\n- **使用**：UEBA（User and Entity Behaviour Analytics）或 Cloudflare Bot Management。  \n- **重點**：辨識「高頁面瀏覽 + 低 Session」的行為是否為機器人/爬蟲。\n\n### 6. 加強 CDN / WAF 設定針對國際流量\n- **地理封鎖**：若確定中國流量非業務需求，可在 WAF 中設定 **China IP Block**。  \n- **動態速率限制**：對於美國、台灣等流量高峰期，啟用自動調整的速率限制，防止突發 DDoS。\n\n### 7. 強化日誌保留與可視化\n- **保留期限**：至少 90 天的完整 HTTP/HTTPS, WAF, IDS/IPS 日誌。  \n- **Dashboard**：使用 Grafana / Kibana 建立「Top IP、Top Path、Top Rule」即時圖表，設定異常閾值（如單 IP 1 小時內 > 1000 次請求即發送 Slack/Email 警報）。\n\n### 8. 完整檢查 OWA 與 Webmail 服務的認證機制\n- **SMTP AUTH**：僅允許內部 IP 或已啟用 MFA 的使用者。  \n- **TLS 1.2+**：確保所有通信都使用強加密，防止中間人攻擊。\n\n### 9. 針對高頻路徑執行漏洞掃描\n- **路徑**：`/owa/service.svc`, `/en/category/phison-in-the-news|press-releases` 等。  \n- **工具**：OWASP ZAP、Nessus、Qualys。  \n- **結果**：若發現未授權的 API，立即加上驗證或封鎖。\n\n### 10. 定期更新威脅情資與 IP Reputation\n- **頻率**：每日自動拉取 AbuseIPDB、AlienVault OTX、Google Safe Browsing。  \n- **自動化**：若新情資將 IP 標記為惡意，自動匯入防火牆黑名單。\n\n### 11. 實施「最小權限」原則於所有子域名服務\n- **Webmail、Webapp、GRM**：僅允許必要的 IP 範圍（如公司 VPN）存取管理介面。  \n- **零信任**：使用 Azure AD Conditional Access 或類似方案，根據使用者、設備、位置動態授權。\n\n### 12. 進行「藍隊演練」或桌上推演\n- **情境**：模擬 OWA 暴力破解、API 掃描、惡意 IP 大量請求。  \n- **目標**：驗證上述防護措施的有效性、偵測時間與回應流程。\n\n---\n\n### 執行順序建議\n\n| 優先度 | 工作項目 |\n|--------|----------|\n| ★★★★★ | 1️⃣ IP 速率限制/封鎖、2️⃣ OWA MFA與帳號鎖定 |\n| ★★★★☆ | 3️⃣ 修正空規則、4️⃣ Webapp API 金鑰、5️⃣ 行為分析平台 |\n| ★★★☆☆ | 6️⃣ CDN/Geo‑Block、7️⃣ 日誌可視化、8️⃣ 認證機制加固 |\n| ★★☆☆☆ | 9️⃣ 漏洞掃描、10️⃣ 威脅情資自動更新 |\n| ★☆☆☆☆ | 11️⃣ 最小權限、12️⃣ 藍隊演練 |\n\n---\n\n## 七、結語\n\n雖然本期 **攻擊量指標降至 0**，但透過流量、來源與路徑的細部分析，我們仍可辨識出 **潛在的異常活動**（IP 暴增、OWA 路徑異常、Webapp 流量激增）。建議盡快落實上述 **即時防護**（IP 限制、MFA、規則修正）以及 **長期監控**（行為分析、威脅情資、藍隊演練），以保持對新興威脅的可視性與快速回應能力。 \n\n如需針對特定規則或日誌檔案進一步的深度分析，歡迎提供原始 log 以便進行更精確的異常偵測與根因追蹤。祝系統安全穩定！",
        "metadata": {
            "analysisId": "yj10wymuc",
            "timestamp": "2026-01-07T01:34:15.661Z",
            "model": "gpt-oss:120b",
            "aiProvider": "ollama",
            "isAIGenerated": true,
            "analysisType": "traffic_trend_comparison",
            "responseTime": 55628,
            "promptLength": 1351
        }
    }
}

```

