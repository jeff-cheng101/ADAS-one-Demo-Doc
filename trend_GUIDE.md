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
|timeRange|String|Y|值：1h ,6h ,1d ,3d ,7d ,14d ,30d|

### Example Parameter
```json
{
  "timeRange": "7d"
}
```

## 共用定義（計算與格式化）

### 時間區間規則（由 timeRange 推導當期/上期）
- 時間基準：UTC
- 當期區間（currentRange）：[now - timeRange, now)
- 上期區間（previousRange）：[now - 2*timeRange, now - timeRange)

> 本文件後續 ES|QL 範例中的時間字串為「示範用固定日期」。實作時必須依上述規則由 timeRange 動態計算。

### 數值處理與函式
- round2(x)：四捨五入到小數第 2 位
- round1(x)：四捨五入到小數第 1 位

- ratioPct(numerator, denominator)：
  - 若 denominator = 0：回傳 0
  - 否則：numerator / denominator * 100

- pctChange(current, previous)（百分比變化，非倍率）：
  - 若 previous = 0 且 current = 0：回傳 0
  - 若 previous = 0 且 current > 0：回傳 100（表示由 0 增長；避免無限大）
  - 否則：((current - previous) / previous) * 100

- formatCount(n)（用於 httpVolume/pageView/visits 等「次數類」輸出）：
  - 基準：1K=1000, 1M=1000^2, 1B=1000^3
  - 轉換後輸出字串（例：129985 → "129.99K"）

- formatBytes(bytes)（用於 dataVolume 輸出）：
  - 基準：1KB=1024, 1MB=1024^2, 1GB=1024^3
  - 轉換後輸出字串（例：3322943002 → "3.09GB"）

## Response item define

- 攻擊活動量: totalAttack
- 活動量佔比: httpPct
- 封鎖率: lockdownRate
- 當期 攻擊趨勢: currentAttackTrend
- 上期 攻擊趨勢: previousAttackTrend
- HTTP 活動量: httpVolume
- 資料傳送: dataVolume
- 頁面瀏覽次數(PV): pageView
- 造訪次數: visits
- 來源 IP 位址 (Top 5): sourceIP
- 觸發規則 (Top 5): triggerRule
- 主機 (Top 5): hosts
- 路徑 (Top 5): path
- 國家 (Top 5): country

---

### 當期 攻擊活動量: currentTotalAttack

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" AND SecurityAction.keyword IN (\"jschallenge\", \"block\", \"managedChallenge\") | STATS count = COUNT(*)"
    }
  }
}
```

### 當期 currentTotalAttack Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"count\":72697}]"
            }
        ],
        "isError": false
    }
}
```

### 上期 攻擊活動量: previousTotalAttack

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" AND SecurityAction.keyword IN (\"jschallenge\", \"block\", \"managedChallenge\") | STATS count = COUNT(*)"
    }
  }
}
```

### 上期 previousTotalAttack Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"count\":73505}]"
            }
        ],
        "isError": false
    }
}
```

totalAttack.quantity = 當期 currentTotalAttack Data 的 count

totalAttack.change = pctChange(currentTotalAttack.count, previousTotalAttack.count)（百分比變化，取到小數第二位）

---

### 當期 HTTP 活動量: currentHttpVolume

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" | STATS count = COUNT(*)"
    }
  }
}
```

### 當期 currentHttpVolume Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"count\":1566556}]"
            }
        ],
        "isError": false
    }
}
```

### 上期 HTTP 活動量: previousHttpVolume

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" | STATS count = COUNT(*)"
    }
  }
}
```

### 上期 previousHttpVolume Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"count\":1596252}]"
            }
        ],
        "isError": false
    }
}
```

httpVolume.quantity = 當期 currentHttpVolume Data 的 count 流量總數，轉換成字串。

httpVolume.change = pctChange(currentHttpVolume.count, previousHttpVolume.count)（百分比變化，取到小數第二位）

---

### 當期 活動量佔比: currentHttpPct

- 攻擊流量佔比計算方式： 當期 攻擊活動量 (currentTotalAttack.count) / 當期 HTTP 流量 (currentHttpVolume.count) * 100（0~100 百分比）


### 上期 活動量佔比: previousHttpPct

- 攻擊流量佔比計算方式： 上期 攻擊活動量 (previousTotalAttack.count) / 上期 HTTP 流量 (previousHttpVolume.count) * 100（0~100 百分比）

定義：
- currentHttpPctPct = currentTotalAttack.count / currentHttpVolume.count * 100
- previousHttpPctPct = previousTotalAttack.count / previousHttpVolume.count * 100


httpPct.quantity = currentHttpPctPct（0~100 的百分比）

httpPct.change = pctChange(currentHttpPctPct, previousHttpPctPct)（百分比變化，取到小數第二位）

---

### 當期 封鎖率: currentLockdownRate

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" AND SecurityAction.keyword IN ( \"block\") | STATS count = COUNT(*)"
    }
  }
}
```

### 當期 currentLockdownRate Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"count\":72697}]"
            }
        ],
        "isError": false
    }
}
```

### 上期 封鎖率: previousLockdownRate

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" AND SecurityAction.keyword IN ( \"block\") | STATS count = COUNT(*)"
    }
  }
}
```

### 上期 previousLockdownRate Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"count\":73505}]"
            }
        ],
        "isError": false
    }
}
```

lockdownRate.quantity = ratioPct(currentLockdownRate.count, currentTotalAttack.count)（0~100 的百分比）

lockdownRate.change = pctChange(lockdownRate.quantity, ratioPct(previousLockdownRate.count, previousTotalAttack.count))（百分比變化，取到小數第二位）

---

### 當期 攻擊趨勢: currentAttackTrend

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" | WHERE SecurityAction.keyword IN (\"jschallenge\", \"block\", \"managedChallenge\") | EVAL hour = DATE_TRUNC(1 hour, @timestamp) | STATS count = COUNT(*) BY hour | SORT hour ASC | KEEP hour, count "
    }
  }
}
```

### 當期 currentAttackTrend Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"count\":3359,\"hour\":\"2025-12-30T00:00:00.000Z\"},{\"count\":3195,\"hour\":\"2025-12-30T01:00:00.000Z\"},{\"count\":2938,\"hour\":\"2025-12-30T02:00:00.000Z\"},{\"count\":3251,\"hour\":\"2025-12-30T03:00:00.000Z\"},{\"count\":3205,\"hour\":\"2025-12-30T04:00:00.000Z\"},{\"count\":2603,\"hour\":\"2025-12-30T05:00:00.000Z\"},{\"count\":3010,\"hour\":\"2025-12-30T06:00:00.000Z\"},{\"count\":2890,\"hour\":\"2025-12-30T07:00:00.000Z\"},{\"count\":3043,\"hour\":\"2025-12-30T08:00:00.000Z\"},{\"count\":3167,\"hour\":\"2025-12-30T09:00:00.000Z\"},{\"count\":3430,\"hour\":\"2025-12-30T10:00:00.000Z\"},{\"count\":2643,\"hour\":\"2025-12-30T11:00:00.000Z\"},{\"count\":2140,\"hour\":\"2025-12-30T12:00:00.000Z\"},{\"count\":2839,\"hour\":\"2025-12-30T13:00:00.000Z\"},{\"count\":2789,\"hour\":\"2025-12-30T14:00:00.000Z\"},{\"count\":3550,\"hour\":\"2025-12-30T15:00:00.000Z\"},{\"count\":3565,\"hour\":\"2025-12-30T16:00:00.000Z\"},{\"count\":3186,\"hour\":\"2025-12-30T17:00:00.000Z\"},{\"count\":3369,\"hour\":\"2025-12-30T18:00:00.000Z\"},{\"count\":3427,\"hour\":\"2025-12-30T19:00:00.000Z\"},{\"count\":3462,\"hour\":\"2025-12-30T20:00:00.000Z\"},{\"count\":3527,\"hour\":\"2025-12-30T21:00:00.000Z\"},{\"count\":3411,\"hour\":\"2025-12-30T22:00:00.000Z\"},{\"count\":3423,\"hour\":\"2025-12-30T23:00:00.000Z\"}]"
            }
        ],
        "isError": false
    }
}
```

### 上期 攻擊趨勢: previousAttackTrend

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" | WHERE SecurityAction.keyword IN (\"jschallenge\", \"block\", \"managedChallenge\") | EVAL hour = DATE_TRUNC(1 hour, @timestamp) | STATS count = COUNT(*) BY hour | SORT hour ASC | KEEP hour, count "
    }
  }
}
```

### 上期 previousAttackTrend Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"count\":3421,\"hour\":\"2025-12-29T00:00:00.000Z\"},{\"count\":3334,\"hour\":\"2025-12-29T01:00:00.000Z\"},{\"count\":3208,\"hour\":\"2025-12-29T02:00:00.000Z\"},{\"count\":3202,\"hour\":\"2025-12-29T03:00:00.000Z\"},{\"count\":1876,\"hour\":\"2025-12-29T04:00:00.000Z\"},{\"count\":4337,\"hour\":\"2025-12-29T05:00:00.000Z\"},{\"count\":2710,\"hour\":\"2025-12-29T06:00:00.000Z\"},{\"count\":3031,\"hour\":\"2025-12-29T07:00:00.000Z\"},{\"count\":3052,\"hour\":\"2025-12-29T08:00:00.000Z\"},{\"count\":3288,\"hour\":\"2025-12-29T09:00:00.000Z\"},{\"count\":3327,\"hour\":\"2025-12-29T10:00:00.000Z\"},{\"count\":2711,\"hour\":\"2025-12-29T11:00:00.000Z\"},{\"count\":2087,\"hour\":\"2025-12-29T12:00:00.000Z\"},{\"count\":2851,\"hour\":\"2025-12-29T13:00:00.000Z\"},{\"count\":2814,\"hour\":\"2025-12-29T14:00:00.000Z\"},{\"count\":2930,\"hour\":\"2025-12-29T15:00:00.000Z\"},{\"count\":4215,\"hour\":\"2025-12-29T16:00:00.000Z\"},{\"count\":3254,\"hour\":\"2025-12-29T17:00:00.000Z\"},{\"count\":3411,\"hour\":\"2025-12-29T18:00:00.000Z\"},{\"count\":3430,\"hour\":\"2025-12-29T19:00:00.000Z\"},{\"count\":3461,\"hour\":\"2025-12-29T20:00:00.000Z\"},{\"count\":3465,\"hour\":\"2025-12-29T21:00:00.000Z\"},{\"count\":3445,\"hour\":\"2025-12-29T22:00:00.000Z\"},{\"count\":3407,\"hour\":\"2025-12-29T23:00:00.000Z\"}]"
            }
        ],
        "isError": false
    }
}
```

---

### 當期 資料傳送: currentData

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" | STATS currentData = SUM(EdgeResponseBytes)"
    }
  }
}
```

### 當期 currentData Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"currentData\":3322943002}]"
            }
        ],
        "isError": false
    }
}
```

### 上期 資料傳送: previousData

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" | STATS previousData = SUM(EdgeResponseBytes)"
    }
  }
}
```

### 上期 previousData Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"previousData\":3379439819}]"
            }
        ],
        "isError": false
    }
}
```

dataVolume.quantity = 當期 currentData Data 的 currentData 總數，初始單位byte，輸出可以 為KB、MB、GB 並轉換成字串。

dataVolume.change = pctChange(currentData, previousData)（百分比變化，取到小數第二位）

---

### 當期 頁面瀏覽次數(PV): currentPageView

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" AND EdgeResponseContentType IN (\"text/html\") | STATS count = COUNT(*)"
    }
  }
}
```

### 當期 currentPageView Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"count\":129985}]"
            }
        ],
        "isError": false
    }
}
```

### 上期 頁面瀏覽次數(PV): previousPageView

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" AND EdgeResponseContentType IN (\"text/html\") | STATS count = COUNT(*)"
    }
  }
}
```

### 上期 previousPageView Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"count\":132264}]"
            }
        ],
        "isError": false
    }
}
```

pageView.quantity = currentPageView.count（頁面瀏覽次數 PV），初始單位為個數，輸出可用 K/M/B 等縮寫並轉換成字串。

pageView.change = pctChange(currentPageView.count, previousPageView.count)（百分比變化，取到小數第二位）

---

### 當期 造訪次數: currentVisits

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" AND ClientRequestReferer.keyword IN (\"None\") | STATS currentVisits = COUNT(*)"
    }
  }
}
```

### 當期 currentVisits Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"currentVisits\":1460733}]"
            }
        ],
        "isError": false
    }
}
```

### 上期 造訪次數: previousVisits

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" AND ClientRequestReferer.keyword IN (\"None\") | STATS previousVisits = COUNT(*)"
    }
  }
}
```

### 上期 previousVisits Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"previousVisits\":1488363}]"
            }
        ],
        "isError": false
    }
}
```

visits.quantity = currentVisits.currentVisits，初始單位為個數，輸出可以用K、M、G等單位來表示並轉換成字串。

visits.change = pctChange(currentVisits.currentVisits, previousVisits.previousVisits)（百分比變化，取到小數第二位）

---

### 當期 來源 IP 位址 (Top 5): currentSourceIP

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" | STATS cnt = count(*) BY ClientIP | SORT cnt DESC | EVAL currentSourceIP = ClientIP| DROP ClientIP |LIMIT 5"
    }
  }
}
```

### 當期 currentSourceIP Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"cnt\":43008,\"currentSourceIP\":\"193.41.206.36\"},{\"cnt\":16747,\"currentSourceIP\":\"2a06:98c0:360d:481f:84ac:44b:f51c:3f9d\"},{\"cnt\":15695,\"currentSourceIP\":\"2a06:98c0:360c:fed5:b9e:45e4:6630:fb6\"},{\"cnt\":15413,\"currentSourceIP\":\"2a06:98c0:360d:6e8c:3ed:43a3:1223:87d0\"},{\"cnt\":15127,\"currentSourceIP\":\"2a06:98c0:360c:19c0:a485:1915:f162:c962\"}]"
            }
        ],
        "isError": false
    }
}
```

### 上期 來源 IP 位址 (Top 5): previousSourceIP

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" | WHERE ClientIP IN (\"193.41.206.36\",\"2a06:98c0:360d:481f:84ac:44b:f51c:3f9d\",\"2a06:98c0:360c:fed5:b9e:45e4:6630:fb6\",\"2a06:98c0:360d:6e8c:3ed:43a3:1223:87d0\",\"2a06:98c0:360c:19c0:a485:1915:f162:c962\") | STATS cnt = COUNT(*) BY ClientIP | SORT cnt DESC | EVAL previousSourceIP = ClientIP | DROP ClientIP"
    }
  }
}
```

### 上期 previousSourceIP Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"cnt\":43008,\"previousSourceIP\":\"193.41.206.36\"},{\"cnt\":16965,\"previousSourceIP\":\"2a06:98c0:360d:481f:84ac:44b:f51c:3f9d\"},{\"cnt\":15958,\"previousSourceIP\":\"2a06:98c0:360c:fed5:b9e:45e4:6630:fb6\"},{\"cnt\":15691,\"previousSourceIP\":\"2a06:98c0:360d:6e8c:3ed:43a3:1223:87d0\"},{\"cnt\":15399,\"previousSourceIP\":\"2a06:98c0:360c:19c0:a485:1915:f162:c962\"}]"
            }
        ],
        "isError": false
    }
}
```
- ES|QL 不能在單一 query 內把「第二段 query 的結果」當成 IN (subquery) 直接引用

- 先取得 currentSourceIP 後 再回填到 previousSourceIP 查詢的 IN 中

- sourceIP.ClientIP = currentSourceIP.currentSourceIP

- 依 sourceIP.ClientIP 名稱 各別計算： ClientIP.change = pctChange(currentSourceIP.cnt, previousSourceIP.cnt)

- ClientIP.cnt = currentSourceIP.cnt

---

### 當期 觸發規則 (Top 5): currentTriggerRule

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" | STATS cnt = count(*) BY SecurityRuleDescription | SORT cnt DESC | EVAL currentTriggerRule = SecurityRuleDescription| DROP SecurityRuleDescription |LIMIT 5"
    }
  }
}
```

### 當期 currentTriggerRule Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"cnt\":1378583,\"currentTriggerRule\":\"log\"},{\"cnt\":79612,\"currentTriggerRule\":\"\"},{\"cnt\":50450,\"currentTriggerRule\":\"Version Control - Information Disclosure\"},{\"cnt\":17966,\"currentTriggerRule\":\"erp\"},{\"cnt\":9848,\"currentTriggerRule\":\"no response when not in 80_443 port\"}]"
            }
        ],
        "isError": false
    }
}
```

### 上期 觸發規則 (Top 5): previousTriggerRule

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" | WHERE SecurityRuleDescription IN (\"log\",\"\",\"Version Control - Information Disclosure\",\"erp\",\"no response when not in 80_443 port\") | STATS cnt = COUNT(*) BY SecurityRuleDescription | SORT cnt DESC | LIMIT 5 | EVAL previousTriggerRule = SecurityRuleDescription | DROP SecurityRuleDescription"
    }
  }
}
```

### 上期 previousTriggerRule Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"cnt\":1405638,\"previousTriggerRule\":\"log\"},{\"cnt\":80975,\"previousTriggerRule\":\"\"},{\"cnt\":50791,\"previousTriggerRule\":\"Version Control - Information Disclosure\"},{\"cnt\":18226,\"previousTriggerRule\":\"erp\"},{\"cnt\":10183,\"previousTriggerRule\":\"no response when not in 80_443 port\"}]"
            }
        ],
        "isError": false
    }
}
```
- ES|QL 不能在單一 query 內把「第二段 query 的結果」當成 IN (subquery) 直接引用

- 先取得 currentTriggerRule 後 再回填到 previousTriggerRule 查詢的 IN 中

- triggerRule.SecurityRuleDescription = currentTriggerRule.currentTriggerRule

- 依 triggerRule.SecurityRuleDescription 名稱 各別計算： triggerRule.change = pctChange(currentTriggerRule.cnt, previousTriggerRule.cnt)

- triggerRule.cnt = currentTriggerRule.cnt

---

### 當期 主機 (Top 5): currentHosts

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" | STATS cnt = count(*) BY ClientRequestHost | SORT cnt DESC | EVAL currentHosts = ClientRequestHost| DROP ClientRequestHost |LIMIT 5"
    }
  }
}
```

### 當期 currentHosts Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"cnt\":848776,\"currentHosts\":\"logstash-test.twister5.cf\"},{\"cnt\":370229,\"currentHosts\":\"bde-70.twister5.cf\"},{\"cnt\":43749,\"currentHosts\":\"nginx.twister5.cf\"},{\"cnt\":34254,\"currentHosts\":\"aaaa.twister5.cf\"},{\"cnt\":27069,\"currentHosts\":\"ipfs.twister5.cf\"}]"
            }
        ],
        "isError": false
    }
}
```

### 上期 主機 (Top 5): previousHosts

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" | WHERE ClientRequestHost IN (\"logstash-test.twister5.cf\",\"bde-70.twister5.cf\",\"nginx.twister5.cf\",\"aaaa.twister5.cf\",\"ipfs.twister5.cf\") | STATS cnt = COUNT(*) BY ClientRequestHost | SORT cnt DESC | LIMIT 5 | EVAL previousHosts = ClientRequestHost | DROP ClientRequestHost"
    }
  }
}
```

### 上期 previousHosts Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"cnt\":865538,\"previousHosts\":\"logstash-test.twister5.cf\"},{\"cnt\":377812,\"previousHosts\":\"bde-70.twister5.cf\"},{\"cnt\":44664,\"previousHosts\":\"nginx.twister5.cf\"},{\"cnt\":34366,\"previousHosts\":\"aaaa.twister5.cf\"},{\"cnt\":27169,\"previousHosts\":\"ipfs.twister5.cf\"}]"
            }
        ],
        "isError": false
    }
}
```
- ES|QL 不能在單一 query 內把「第二段 query 的結果」當成 IN (subquery) 直接引用

- 先取得 currentHosts 後 再回填到 previousHosts 查詢的 IN 中

- hosts.ClientRequestHost = currentHosts.currentHosts

- 依 hosts.ClientRequestHost 名稱 各別計算： hosts.change = pctChange(currentHosts.cnt, previousHosts.cnt)

- hosts.cnt = currentHosts.cnt

---

### 當期 路徑 (Top 5): currentPath

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" | STATS cnt = count(*) BY ClientRequestPath | SORT cnt DESC | EVAL currentPath = ClientRequestPath| DROP ClientRequestPath |LIMIT 5"
    }
  }
}
```

### 當期 currentPath Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"cnt\":1360151,\"currentPath\":\"/\"},{\"cnt\":11011,\"currentPath\":\"/favicon.ico\"},{\"cnt\":7737,\"currentPath\":\"/cdn-cgi/challenge-platform/scripts/jsd/main.js\"},{\"cnt\":7648,\"currentPath\":\"/.env\"},{\"cnt\":5586,\"currentPath\":\"/.git/config\"}]"
            }
        ],
        "isError": false
    }
}
```

### 上期 路徑 (Top 5): previousPath

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" | WHERE ClientRequestPath IN (\"/\",\"/favicon.ico\",\"/cdn-cgi/challenge-platform/scripts/jsd/main.js\",\"/.env\",\"/.git/config\") | STATS cnt = COUNT(*) BY ClientRequestPath | SORT cnt DESC | LIMIT 5 | EVAL previousPath = ClientRequestPath | DROP ClientRequestPath"
    }
  }
}
```

### 上期 previousPath Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"cnt\":1387889,\"previousPath\":\"/\"},{\"cnt\":11284,\"previousPath\":\"/favicon.ico\"},{\"cnt\":7797,\"previousPath\":\"/cdn-cgi/challenge-platform/scripts/jsd/main.js\"},{\"cnt\":7728,\"previousPath\":\"/.env\"},{\"cnt\":5625,\"previousPath\":\"/.git/config\"}]"
            }
        ],
        "isError": false
    }
}
```
- ES|QL 不能在單一 query 內把「第二段 query 的結果」當成 IN (subquery) 直接引用

- 先取得 currentPath 後 再回填到 previousPath 查詢的 IN 中

- path.ClientRequestPath = currentPath.currentPath

- 依 path.ClientRequestPath 名稱 各別計算： path.change = pctChange(currentPath.cnt, previousPath.cnt)

- path.cnt = currentPath.cnt

---

### 當期 國家 (Top 5): currentCountry

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" | STATS cnt = count(*) BY geoip_client.country_name | SORT cnt DESC | EVAL currentCountry = geoip_client.country_name| DROP geoip_client.country_name |LIMIT 5"
    }
  }
}
```

### 當期 currentCountry Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"cnt\":784860,\"currentCountry\":\"Taiwan\"},{\"cnt\":335437,\"currentCountry\":\"United States\"},{\"cnt\":166231,\"currentCountry\":\"Hong Kong\"},{\"cnt\":73526,\"currentCountry\":\"The Netherlands\"},{\"cnt\":50562,\"currentCountry\":\"France\"}]"
            }
        ],
        "isError": false
    }
}
```

### 上期 國家 (Top 5): previousCountry

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" | WHERE geoip_client.country_name IN (\"Taiwan\",\"United States\",\"Hong Kong\",\"The Netherlands\",\"France\") | STATS cnt = COUNT(*) BY geoip_client.country_name | SORT cnt DESC | LIMIT 5 | EVAL previousCountry = geoip_client.country_name | DROP geoip_client.country_name"
    }
  }
}
```

### 上期 previousCountry Data

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "Results"
            },
            {
                "type": "text",
                "text": "[{\"cnt\":800257,\"previousCountry\":\"Taiwan\"},{\"cnt\":342313,\"previousCountry\":\"United States\"},{\"cnt\":169784,\"previousCountry\":\"Hong Kong\"},{\"cnt\":74848,\"previousCountry\":\"The Netherlands\"},{\"cnt\":50406,\"previousCountry\":\"France\"}]"
            }
        ],
        "isError": false
    }
}
```
- ES|QL 不能在單一 query 內把「第二段 query 的結果」當成 IN (subquery) 直接引用

- 先取得 currentCountry 後 再回填到 previousCountry 查詢的 IN 中

- country.geoip_client.country_name = currentCountry.currentCountry

- 依 country.geoip_client.country_name 名稱 各別計算： country.change = pctChange(currentCountry.cnt, previousCountry.cnt)

- country.cnt = currentCountry.cnt

---

## Response data sample

### Response fields

| Attribute         | Type           | Required | Description |
|------------------|----------------|----------|-------------|
| success          | Boolean        | Y        | 是否成功 |
| totalAttack      | Object         | Y        | 攻擊活動量（總攻擊次數） |
| httpPct          | Object         | Y        | HTTP 攻擊佔比（活動量佔比，0~100%） |
| lockdownRate     | Object         | Y        | 封鎖成功率（0~100%） |
| currentAttackTrend  | Array(Object) | Y     | 當期攻擊趨勢（依小時彙總） |
| previousAttackTrend | Array(Object) | Y     | 上期攻擊趨勢（依小時彙總） |
| httpVolume       | Object         | Y        | HTTP 流量（請求數） |
| dataVolume       | Object         | Y        | 資料量（bytes 格式化輸出） |
| pageView         | Object         | Y        | 頁面瀏覽次數（PV） |
| visits           | Object         | Y        | 造訪次數 |
| sourceIP         | Array(Object)  | Y        | 來源 IP 位址 Top 5 |
| triggerRule      | Array(Object)  | Y        | 觸發規則 Top 5 |
| hosts            | Array(Object)  | Y        | 主機 Top 5 |
| path             | Array(Object)  | Y        | 路徑 Top 5 |
| country          | Array(Object)  | Y        | 國家 Top 5 |
| other            | Object         | N        | 預留擴展欄位（建議預設回傳 `{}`） |

### totalAttack fields
| Attribute | Type   | Description |
|----------|--------|-------------|
| quantity | Number | 攻擊活動量（總攻擊次數） |
| change  | Number | 變化（百分比變化，%） |

### httpPct fields
| Attribute | Type   | Description |
|----------|--------|-------------|
| quantity | Number | HTTP 攻擊佔比（0~100 的百分比） |
| change  | Number | 變化（百分比變化，%） |

### lockdownRate fields
| Attribute | Type   | Description |
|----------|--------|-------------|
| quantity | Number | 封鎖成功率（0~100 的百分比） |
| change  | Number | 變化（百分比變化，%） |

### httpVolume fields
| Attribute | Type   | Description |
|----------|--------|-------------|
| quantity | String | HTTP 流量（請求數，formatCount 後字串） |
| change  | Number | 變化（百分比變化，%） |

### dataVolume fields
| Attribute | Type   | Description |
|----------|--------|-------------|
| quantity | String | 資料量（bytes，formatBytes 後字串） |
| change  | Number | 變化（百分比變化，%） |

### pageView fields
| Attribute | Type   | Description |
|----------|--------|-------------|
| quantity | String | 頁面瀏覽次數（PV，formatCount 後字串） |
| change  | Number | 變化（百分比變化，%） |

### visits fields
| Attribute | Type   | Description |
|----------|--------|-------------|
| quantity | String | 造訪次數（formatCount 後字串） |
| change  | Number | 變化（百分比變化，%） |

### currentAttackTrend item fields
| Attribute | Type   | Description |
|----------|--------|-------------|
| hour     | String | 時間（ISO 8601） |
| count    | Number | 攻擊次數 |

### previousAttackTrend item fields
| Attribute | Type   | Description |
|----------|--------|-------------|
| hour     | String | 時間（ISO 8601） |
| count    | Number | 攻擊次數 |

### sourceIP item fields
| Attribute | Type   | Description |
|----------|--------|-------------|
| ClientIP | String | 來源 IP 位址 |
| cnt      | Number | 當期請求次數 |
| change   | Number | 變化（百分比變化，%） |

### triggerRule item fields
| Attribute               | Type   | Description |
|------------------------|--------|-------------|
| SecurityRuleDescription | String | 規則名稱 |
| cnt                    | Number | 當期請求次數 |
| change                 | Number | 變化（百分比變化，%） |

### hosts item fields
| Attribute         | Type   | Description |
|------------------|--------|-------------|
| ClientRequestHost | String | Host 名稱 |
| cnt               | Number | 當期請求次數 |
| change            | Number | 變化（百分比變化，%） |

### path item fields
| Attribute         | Type   | Description |
|------------------|--------|-------------|
| ClientRequestPath | String | Path |
| cnt               | Number | 當期請求次數 |
| change            | Number | 變化（百分比變化，%） |

### country item fields
| Attribute              | Type   | Description |
|-----------------------|--------|-------------|
| geoip_client.country_name | String | 國家名稱 |
| cnt                   | Number | 當期請求次數 |
| change                | Number | 變化（百分比變化，%） |

### Response example（依本文件「範例查詢結果」計算）

```json
{
  "success": true,
  "totalAttack": {
    "quantity": 72697,
    "change": -1.1
  },
  "httpPct": {
    "quantity": 4.6,
    "change": 0.78
  },
  "lockdownRate": {
    "quantity": 100.0,
    "change": 0.0
  },
  "currentAttackTrend": [
    {
      "count": 3359,
      "hour": "2025-12-30T00:00:00.000Z"
    },
    {
      "count": 3195,
      "hour": "2025-12-30T01:00:00.000Z"
    },
    {
      "count": 2938,
      "hour": "2025-12-30T02:00:00.000Z"
    },
    {
      "count": 3251,
      "hour": "2025-12-30T03:00:00.000Z"
    },
    {
      "count": 3205,
      "hour": "2025-12-30T04:00:00.000Z"
    },
    {
      "count": 2603,
      "hour": "2025-12-30T05:00:00.000Z"
    },
    {
      "count": 3010,
      "hour": "2025-12-30T06:00:00.000Z"
    },
    {
      "count": 2890,
      "hour": "2025-12-30T07:00:00.000Z"
    },
    {
      "count": 3043,
      "hour": "2025-12-30T08:00:00.000Z"
    },
    {
      "count": 3167,
      "hour": "2025-12-30T09:00:00.000Z"
    },
    {
      "count": 3430,
      "hour": "2025-12-30T10:00:00.000Z"
    },
    {
      "count": 2643,
      "hour": "2025-12-30T11:00:00.000Z"
    },
    {
      "count": 2140,
      "hour": "2025-12-30T12:00:00.000Z"
    },
    {
      "count": 2839,
      "hour": "2025-12-30T13:00:00.000Z"
    },
    {
      "count": 2789,
      "hour": "2025-12-30T14:00:00.000Z"
    },
    {
      "count": 3550,
      "hour": "2025-12-30T15:00:00.000Z"
    },
    {
      "count": 3565,
      "hour": "2025-12-30T16:00:00.000Z"
    },
    {
      "count": 3186,
      "hour": "2025-12-30T17:00:00.000Z"
    },
    {
      "count": 3369,
      "hour": "2025-12-30T18:00:00.000Z"
    },
    {
      "count": 3427,
      "hour": "2025-12-30T19:00:00.000Z"
    },
    {
      "count": 3462,
      "hour": "2025-12-30T20:00:00.000Z"
    },
    {
      "count": 3527,
      "hour": "2025-12-30T21:00:00.000Z"
    },
    {
      "count": 3411,
      "hour": "2025-12-30T22:00:00.000Z"
    },
    {
      "count": 3423,
      "hour": "2025-12-30T23:00:00.000Z"
    }
  ],
  "previousAttackTrend": [
    {
      "count": 3421,
      "hour": "2025-12-29T00:00:00.000Z"
    },
    {
      "count": 3334,
      "hour": "2025-12-29T01:00:00.000Z"
    },
    {
      "count": 3208,
      "hour": "2025-12-29T02:00:00.000Z"
    },
    {
      "count": 3202,
      "hour": "2025-12-29T03:00:00.000Z"
    },
    {
      "count": 1876,
      "hour": "2025-12-29T04:00:00.000Z"
    },
    {
      "count": 4337,
      "hour": "2025-12-29T05:00:00.000Z"
    },
    {
      "count": 2710,
      "hour": "2025-12-29T06:00:00.000Z"
    },
    {
      "count": 3031,
      "hour": "2025-12-29T07:00:00.000Z"
    },
    {
      "count": 3052,
      "hour": "2025-12-29T08:00:00.000Z"
    },
    {
      "count": 3288,
      "hour": "2025-12-29T09:00:00.000Z"
    },
    {
      "count": 3327,
      "hour": "2025-12-29T10:00:00.000Z"
    },
    {
      "count": 2711,
      "hour": "2025-12-29T11:00:00.000Z"
    },
    {
      "count": 2087,
      "hour": "2025-12-29T12:00:00.000Z"
    },
    {
      "count": 2851,
      "hour": "2025-12-29T13:00:00.000Z"
    },
    {
      "count": 2814,
      "hour": "2025-12-29T14:00:00.000Z"
    },
    {
      "count": 2930,
      "hour": "2025-12-29T15:00:00.000Z"
    },
    {
      "count": 4215,
      "hour": "2025-12-29T16:00:00.000Z"
    },
    {
      "count": 3254,
      "hour": "2025-12-29T17:00:00.000Z"
    },
    {
      "count": 3411,
      "hour": "2025-12-29T18:00:00.000Z"
    },
    {
      "count": 3430,
      "hour": "2025-12-29T19:00:00.000Z"
    },
    {
      "count": 3461,
      "hour": "2025-12-29T20:00:00.000Z"
    },
    {
      "count": 3465,
      "hour": "2025-12-29T21:00:00.000Z"
    },
    {
      "count": 3445,
      "hour": "2025-12-29T22:00:00.000Z"
    },
    {
      "count": 3407,
      "hour": "2025-12-29T23:00:00.000Z"
    }
  ],
  "httpVolume": {
    "quantity": "1.57M",
    "change": -1.86
  },
  "dataVolume": {
    "quantity": "3.09GB",
    "change": -1.67
  },
  "pageView": {
    "quantity": "129.99K",
    "change": -1.72
  },
  "visits": {
    "quantity": "1.46M",
    "change": -1.86
  },
  "sourceIP": [
    {
      "ClientIP": "193.41.206.36",
      "cnt": 43008,
      "change": 0.0
    },
    {
      "ClientIP": "2a06:98c0:360d:481f:84ac:44b:f51c:3f9d",
      "cnt": 16747,
      "change": -1.28
    },
    {
      "ClientIP": "2a06:98c0:360c:fed5:b9e:45e4:6630:fb6",
      "cnt": 15695,
      "change": -1.65
    },
    {
      "ClientIP": "2a06:98c0:360d:6e8c:3ed:43a3:1223:87d0",
      "cnt": 15413,
      "change": -1.77
    },
    {
      "ClientIP": "2a06:98c0:360c:19c0:a485:1915:f162:c962",
      "cnt": 15127,
      "change": -1.77
    }
  ],
  "triggerRule": [
    {
      "SecurityRuleDescription": "log",
      "cnt": 1378583,
      "change": -1.92
    },
    {
      "SecurityRuleDescription": "",
      "cnt": 79612,
      "change": -1.68
    },
    {
      "SecurityRuleDescription": "Version Control - Information Disclosure",
      "cnt": 50450,
      "change": -0.67
    },
    {
      "SecurityRuleDescription": "erp",
      "cnt": 17966,
      "change": -1.43
    },
    {
      "SecurityRuleDescription": "no response when not in 80_443 port",
      "cnt": 9848,
      "change": -3.29
    }
  ],
  "hosts": [
    {
      "ClientRequestHost": "logstash-test.twister5.cf",
      "cnt": 848776,
      "change": -1.94
    },
    {
      "ClientRequestHost": "bde-70.twister5.cf",
      "cnt": 370229,
      "change": -2.01
    },
    {
      "ClientRequestHost": "nginx.twister5.cf",
      "cnt": 43749,
      "change": -2.05
    },
    {
      "ClientRequestHost": "aaaa.twister5.cf",
      "cnt": 34254,
      "change": -0.33
    },
    {
      "ClientRequestHost": "ipfs.twister5.cf",
      "cnt": 27069,
      "change": -0.37
    }
  ],
  "path": [
    {
      "ClientRequestPath": "/",
      "cnt": 1360151,
      "change": -2.0
    },
    {
      "ClientRequestPath": "/favicon.ico",
      "cnt": 11011,
      "change": -2.42
    },
    {
      "ClientRequestPath": "/cdn-cgi/challenge-platform/scripts/jsd/main.js",
      "cnt": 7737,
      "change": -0.77
    },
    {
      "ClientRequestPath": "/.env",
      "cnt": 7648,
      "change": -1.04
    },
    {
      "ClientRequestPath": "/.git/config",
      "cnt": 5586,
      "change": -0.69
    }
  ],
  "country": [
    {
      "geoip_client.country_name": "Taiwan",
      "cnt": 784860,
      "change": -1.92
    },
    {
      "geoip_client.country_name": "United States",
      "cnt": 335437,
      "change": -2.01
    },
    {
      "geoip_client.country_name": "Hong Kong",
      "cnt": 166231,
      "change": -2.09
    },
    {
      "geoip_client.country_name": "The Netherlands",
      "cnt": 73526,
      "change": -1.77
    },
    {
      "geoip_client.country_name": "France",
      "cnt": 50562,
      "change": 0.31
    }
  ],
  "other": {}
}
```

