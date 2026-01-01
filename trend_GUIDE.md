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

## Response item define

- 攻擊活動量: totalAttack
- 活動量佔比: httpPct
- 封鎖率: lockdownRate
- 當期 攻擊趨勢: currentAttackTrend
- 上期 攻擊趨勢: previousAttackTrend
- HTTP 活動量: httpVolume
- 資料傳送: dataVolume
- 點閱率: pageView
- 造訪次數: visits
- 來源 IP 位址 (Top 5): sourceIP
- 觸發規則 (Top 5): triggerRule
- 主機 (Top 5): hosts
- 路徑 (Top 5): path
- 國家 (Top 5): country
- 預留欄位: other
## 計算與格式化規則（最終版）

### 時間窗推導（UTC）
- 令 `duration` 為 `timeRange` 對應的秒數：`1h=3600`、`6h=21600`、`1d=86400`、`3d=259200`、`7d=604800`、`14d=1209600`、`30d=2592000`。
- 令 `endTimeUtc` 為查詢當下時間（UTC）向下取整到小時（minute/second/millis = 0）的時間點。
- 當期區間：`[endTimeUtc - duration, endTimeUtc)`。
- 上期區間：`[endTimeUtc - 2*duration, endTimeUtc - duration)`。

### 安全除法
- `safe_div(a, b)`：若 `b = 0`，回傳 `0`；否則回傳 `a / b`。

### 變化百分比 change（Number）
- `pct_change(current, previous) = ((current - previous) / previous) * 100`。
- 若 `previous = 0`：
  - `current = 0`：回傳 `0`。
  - `current ≠ 0`：回傳 `null`。
- change 統一四捨五入到小數點後 2 位。

### 單位格式化
- `formatCount(n)`：輸出 `String`，採 1000 進位縮寫 `K/M/B`，小數最多 2 位並移除尾端 0（例：`982K`、`1.57M`）。
- `formatBytes(nBytes)`：輸出 `String`，採 1024 進位 `B/KB/MB/GB/TB`，小數最多 2 位，值與單位間保留 1 個半形空白（例：`450 GB`）。
- `formatPct(p)`：輸出 `String`，格式 `${round(p, 1)}%`（例：`8.4%`）。



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
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" AND SecurityAction.keyword IN (\"jschallenge\", \"block\", \"managedChallenge\") | STATS count = COUNT(*)"
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
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" AND SecurityAction.keyword IN (\"jschallenge\", \"block\", \"managedChallenge\") | STATS count = COUNT(*)"
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

totalAttack.change = pct_change(currentTotalAttack.count, previousTotalAttack.count)  // 四捨五入到小數點後 2 位

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
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" | STATS count = COUNT(*)"
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
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" | STATS count = COUNT(*)"
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

httpVolume.quantity = formatCount(currentHttpVolume.count)

httpVolume.change = pct_change(currentHttpVolume.count, previousHttpVolume.count)  // 四捨五入到小數點後 2 位

---

### 當期 活動量佔比: currentHttpPct

- 攻擊流量佔比計算方式： `currentHttpPct = safe_div(currentTotalAttack.count, currentHttpVolume.count) * 100`（0~100 的百分比）


### 上期 活動量佔比: previousHttpPct

- 攻擊流量佔比計算方式： `previousHttpPct = safe_div(previousTotalAttack.count, previousHttpVolume.count) * 100`（0~100 的百分比）


httpPct.quantity = round(safe_div(currentTotalAttack.count, currentHttpVolume.count) * 100, 1)  // 0~100 的百分比

httpPct.change = pct_change(currentHttpPct, previousHttpPct)  // 四捨五入到小數點後 2 位；currentHttpPct/previousHttpPct 皆為百分比數值

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
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" AND SecurityAction.keyword IN ( \"block\") | STATS count = COUNT(*)"
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
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" AND SecurityAction.keyword IN ( \"block\") | STATS count = COUNT(*)"
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

lockdownRate.quantity = round(safe_div(currentLockdownRate.count, currentTotalAttack.count) * 100, 1)  // 0~100 的百分比

previousLockdownRatePct = round(safe_div(previousLockdownRate.count, previousTotalAttack.count) * 100, 1)

lockdownRate.change = pct_change(lockdownRate.quantity, previousLockdownRatePct)  // 四捨五入到小數點後 2 位

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
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" | WHERE SecurityAction.keyword IN (\"jschallenge\", \"block\", \"managedChallenge\") | EVAL hour = DATE_TRUNC(1 hour, @timestamp) | STATS count = COUNT(*) BY hour | SORT hour ASC | KEEP hour, count "
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
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" | WHERE SecurityAction.keyword IN (\"jschallenge\", \"block\", \"managedChallenge\") | EVAL hour = DATE_TRUNC(1 hour, @timestamp) | STATS count = COUNT(*) BY hour | SORT hour ASC | KEEP hour, count "
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
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" | STATS currentData = SUM(EdgeResponseBytes)"
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
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" | STATS previousData = SUM(EdgeResponseBytes)"
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

dataVolume.quantity = formatBytes(currentData)

dataVolume.change = pct_change(currentData, previousData)  // 四捨五入到小數點後 2 位

---

### 當期 點閱率: currentPageView

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" AND EdgeResponseContentType IN (\"text/html\") | STATS count = COUNT(*)"
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

### 上期 點閱率: previousPageView

- esql 查詢條件

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "esql",
    "arguments": {
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" AND EdgeResponseContentType IN (\"text/html\") | STATS count = COUNT(*)"
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

currentPageViewPct = safe_div(currentPageView.count, currentHttpVolume.count) * 100
previousPageViewPct = safe_div(previousPageView.count, previousHttpVolume.count) * 100

pageView.quantity = formatPct(safe_div(currentPageView.count, currentHttpVolume.count) * 100)  // 0~100 的百分比字串

pageView.change = pct_change(currentPageViewPct, previousPageViewPct)  // 四捨五入到小數點後 2 位；Pct 為百分比數值

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
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" AND ClientRequestReferer.keyword IN (\"None\") | STATS currentVisits = COUNT(*)"
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
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" AND ClientRequestReferer.keyword IN (\"None\") | STATS previousVisits = COUNT(*)"
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

visits.quantity = formatCount(currentVisits)

visits.change = pct_change(currentVisits, previousVisits)  // 四捨五入到小數點後 2 位

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
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" | STATS cnt = count(*) BY ClientIP | SORT cnt DESC | EVAL currentSourceIP = ClientIP| DROP ClientIP |LIMIT 5"
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
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" | WHERE ClientIP IN (\"193.41.206.36\",\"2a06:98c0:360d:481f:84ac:44b:f51c:3f9d\",\"2a06:98c0:360c:fed5:b9e:45e4:6630:fb6\",\"2a06:98c0:360d:6e8c:3ed:43a3:1223:87d0\",\"2a06:98c0:360c:19c0:a485:1915:f162:c962\") | STATS cnt = COUNT(*) BY ClientIP | SORT cnt DESC | EVAL previousSourceIP = ClientIP | DROP ClientIP"
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

- sourceIP.ClientIP = currentSourceIP

- 依 sourceIP.ClientIP 名稱 各別計算： ClientIP.change = pct_change(currentSourceIP.cnt, previousSourceIP.cnt)

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
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" | STATS cnt = count(*) BY SecurityRuleDescription | SORT cnt DESC | EVAL currentTriggerRule = SecurityRuleDescription| DROP SecurityRuleDescription |LIMIT 5"
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
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" | WHERE SecurityRuleDescription IN (\"log\",\"\",\"Version Control - Information Disclosure\",\"erp\",\"no response when not in 80_443 port\") | STATS cnt = COUNT(*) BY SecurityRuleDescription | SORT cnt DESC | LIMIT 5 | EVAL previousTriggerRule = SecurityRuleDescription | DROP SecurityRuleDescription"
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

- triggerRule.SecurityRuleDescription = previousTriggerRule

- 依 triggerRule.SecurityRuleDescription 名稱 各別計算： triggerRule.change = pct_change(currentTriggerRule.cnt, previousTriggerRule.cnt)

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
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" | STATS cnt = count(*) BY ClientRequestHost | SORT cnt DESC | EVAL currentHosts = ClientRequestHost| DROP ClientRequestHost |LIMIT 5"
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
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" | WHERE ClientRequestHost IN (\"logstash-test.twister5.cf\",\"bde-70.twister5.cf\",\"nginx.twister5.cf\",\"aaaa.twister5.cf\",\"ipfs.twister5.cf\") | STATS cnt = COUNT(*) BY ClientRequestHost | SORT cnt DESC | LIMIT 5 | EVAL previousHosts = ClientRequestHost | DROP ClientRequestHost"
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

- Hosts.ClientRequestHost = previousHosts

- 依 Hosts.ClientRequestHost 名稱 各別計算： Hosts.change = pct_change(currentHosts.cnt, previousHosts.cnt)

- Hosts.cnt = currentHosts.cnt

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
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" | STATS cnt = count(*) BY ClientRequestPath | SORT cnt DESC | EVAL currentPath = ClientRequestPath| DROP ClientRequestPath |LIMIT 5"
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
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" | WHERE ClientRequestPath IN (\"/\",\"/favicon.ico\",\"/cdn-cgi/challenge-platform/scripts/jsd/main.js\",\"/.env\",\"/.git/config\") | STATS cnt = COUNT(*) BY ClientRequestPath | SORT cnt DESC | LIMIT 5 | EVAL previousPath = ClientRequestPath | DROP ClientRequestPath"
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

- path.ClientRequestPath = previousPath

- 依 path.ClientRequestPath 名稱 各別計算： path.change = pct_change(currentPath.cnt, previousPath.cnt)

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
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-30T00:00:00.000Z\" AND @timestamp <= \"2025-12-31T00:00:00.000Z\" | STATS cnt = count(*) BY geoip.geo.country_name | SORT cnt DESC | EVAL currentCountry = geoip.geo.country_name| DROP geoip.geo.country_name |LIMIT 5"
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
      "query": "FROM adasone-cf-*,across-cf-logpush-* | WHERE @timestamp >= \"2025-12-29T00:00:00.000Z\" AND @timestamp <= \"2025-12-30T00:00:00.000Z\" | WHERE geoip.geo.country_name IN (\"Taiwan\",\"United States\",\"Hong Kong\",\"The Netherlands\",\"France\") | STATS cnt = COUNT(*) BY geoip.geo.country_name | SORT cnt DESC | LIMIT 5 | EVAL previousCountry = geoip.geo.country_name | DROP geoip.geo.country_name"
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

- country.geoip.geo.country_name = previousCountry

- 依 country.geoip.geo.country_name 名稱 各別計算： country.change = pct_change(currentCountry.cnt, previousCountry.cnt)

- country.cnt = currentCountry.cnt

---

## Response data sample

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
| other               | Object        | N        | 預留欄位（可為空物件 `{}`） |

### totalAttack fields

| Attribute | Type   | Required | Description   |
| --------- | ------ | -------- | ------------- |
| quantity  | Number | Y        | 總攻擊次數         |
| change    | Number (nullable) | Y        | 與前一期間相比的變化(百分比) |

### httpPct fields

| Attribute | Type   | Required | Description    |
| --------- | ------ | -------- | -------------- |
| quantity  | Number | Y        | 活動量佔比（0~100 的百分比） |
| change    | Number (nullable) | Y        | 活動量佔比變化 (百分比)      |


### lockdownRate fields

| Attribute | Type   | Required | Description |
| --------- | ------ | -------- | ----------- |
| quantity  | Number | Y        | 封鎖率（0~100 的百分比） |
| change    | Number (nullable) | Y        | 封鎖率變化(百分比)     |


### currentAttackTrend(當期) / previousAttackTrend(上期) item fields

| Attribute | Type              | Required | Description   |
| --------- | ----------------- | -------- | ------------- |
| count     | Number            | Y        | 該小時的攻擊次數      |
| hour      | String (ISO 8601) | Y        | 統計的小時時間點（UTC） |


### httpVolume fields

| Attribute | Type   | Required | Description    |
| --------- | ------ | -------- | -------------- |
| quantity  | String | Y        | HTTP 流量數量（含單位） |
| change    | Number (nullable) | Y        | HTTP 流量變化百分比   |


### dataVolume fields

| Attribute | Type   | Required | Description |
| --------- | ------ | -------- | ----------- |
| quantity  | String | Y        | 資料傳輸量（含單位）  |
| change    | Number (nullable) | Y        | 資料量變化百分比    |


### pageView fields

| Attribute | Type   | Required | Description |
| --------- | ------ | -------- | ----------- |
| quantity  | String | Y        | 頁面瀏覽比例(點閱率)      |
| change    | Number (nullable) | Y        | 與前一期間相比的變化(百分比) |

### visits fields

| Attribute | Type   | Required | Description |
| --------- | ------ | -------- | ----------- |
| quantity  | String | Y        | 訪問次數（含單位）   |
| change    | Number (nullable) | Y        | 與前一期間相比的變化(百分比) |


### sourceIP item fields

| Attribute | Type   | Required | Description           |
| --------- | ------ | -------- | --------------------- |
| ClientIP  | String | Y        | 來源 IP 位址（IPv4 或 IPv6） |
| cnt       | Number | Y        | 該 IP 的請求次數            |
| change    | Number (nullable) | Y        | 與前一期間相比的變化(百分比)       |


### triggerRule item fields

| Attribute               | Type   | Required | Description |
| ----------------------- | ------ | -------- | ----------- |
| SecurityRuleDescription | String | Y        | 觸發該規則描述      |
| cnt                     | Number | Y        | 觸發該規則的次數    |
| change    | Number (nullable) | Y        | 觸發次數變化      |


### hosts item fields

| Attribute         | Type   | Required | Description  |
| ----------------- | ------ | -------- | ------------ |
| ClientRequestHost | String | Y        | 請求的 Host 名稱  |
| cnt               | Number | Y        | 該 Host 的請求次數 |
| change    | Number (nullable) | Y        | 請求次數變化       |


###　path item fields

| Attribute         | Type   | Required | Description |
| ----------------- | ------ | -------- | ----------- |
| ClientRequestPath | String | Y        | 請求路徑        |
| cnt               | Number | Y        | 該路徑的請求次數    |
| change    | Number (nullable) | Y        | 請求次數變化      |

### scountry item fields

| Attribute              | Type   | Required | Description |
| ---------------------- | ------ | -------- | ----------- |
| geoip.geo.country_name | String | Y        | 來源國家名稱      |
| cnt                    | Number | Y        | 該國家的請求次數    |
| change    | Number (nullable) | Y        | 請求次數變化      |

### Example Response（格式示例；數值不對應上方各段落的查詢範例輸出）
```json
{
	"success": true,
	"totalAttack": {
		"quantity": 380208,
		"change": 17.64
	},
	"httpPct": {
		"quantity": 68.5,
		"change": 10
	},
	"lockdownRate": {
		"quantity": 96.7,
		"change": 3.65
	},
	"currentAttackTrend": [
		{
			"count": 413,
			"hour": "2025-12-30T12:00:00.000Z"
		},
		{
			"count": 2738,
			"hour": "2025-12-30T13:00:00.000Z"
		},
		{
			"count": 2688,
			"hour": "2025-12-30T14:00:00.000Z"
		},
		{
			"count": 3446,
			"hour": "2025-12-30T15:00:00.000Z"
		},
		{
			"count": 3454,
			"hour": "2025-12-30T16:00:00.000Z"
		},
		{
			"count": 3070,
			"hour": "2025-12-30T17:00:00.000Z"
		},
		{
			"count": 3250,
			"hour": "2025-12-30T18:00:00.000Z"
		},
		{
			"count": 3302,
			"hour": "2025-12-30T19:00:00.000Z"
		},
		{
			"count": 3333,
			"hour": "2025-12-30T20:00:00.000Z"
		},
		{
			"count": 3351,
			"hour": "2025-12-30T21:00:00.000Z"
		},
		{
			"count": 3336,
			"hour": "2025-12-30T22:00:00.000Z"
		},
		{
			"count": 3299,
			"hour": "2025-12-30T23:00:00.000Z"
		},
		{
			"count": 3234,
			"hour": "2025-12-31T00:00:00.000Z"
		},
		{
			"count": 3084,
			"hour": "2025-12-31T01:00:00.000Z"
		},
		{
			"count": 2892,
			"hour": "2025-12-31T02:00:00.000Z"
		},
		{
			"count": 3089,
			"hour": "2025-12-31T03:00:00.000Z"
		},
		{
			"count": 3089,
			"hour": "2025-12-31T04:00:00.000Z"
		},
		{
			"count": 2601,
			"hour": "2025-12-31T05:00:00.000Z"
		},
		{
			"count": 2835,
			"hour": "2025-12-31T06:00:00.000Z"
		},
		{
			"count": 1298,
			"hour": "2025-12-31T07:00:00.000Z"
		}
	],
	"previousAttackTrend": [
		{
			"count": 413,
			"hour": "2025-12-30T12:00:00.000Z"
		},
		{
			"count": 2738,
			"hour": "2025-12-30T13:00:00.000Z"
		},
		{
			"count": 2688,
			"hour": "2025-12-30T14:00:00.000Z"
		},
		{
			"count": 3446,
			"hour": "2025-12-30T15:00:00.000Z"
		},
		{
			"count": 3454,
			"hour": "2025-12-30T16:00:00.000Z"
		},
		{
			"count": 3070,
			"hour": "2025-12-30T17:00:00.000Z"
		},
		{
			"count": 3250,
			"hour": "2025-12-30T18:00:00.000Z"
		},
		{
			"count": 3302,
			"hour": "2025-12-30T19:00:00.000Z"
		},
		{
			"count": 3333,
			"hour": "2025-12-30T20:00:00.000Z"
		},
		{
			"count": 3351,
			"hour": "2025-12-30T21:00:00.000Z"
		},
		{
			"count": 3336,
			"hour": "2025-12-30T22:00:00.000Z"
		},
		{
			"count": 3299,
			"hour": "2025-12-30T23:00:00.000Z"
		},
		{
			"count": 3234,
			"hour": "2025-12-31T00:00:00.000Z"
		},
		{
			"count": 3084,
			"hour": "2025-12-31T01:00:00.000Z"
		},
		{
			"count": 2892,
			"hour": "2025-12-31T02:00:00.000Z"
		},
		{
			"count": 3089,
			"hour": "2025-12-31T03:00:00.000Z"
		},
		{
			"count": 3089,
			"hour": "2025-12-31T04:00:00.000Z"
		},
		{
			"count": 2601,
			"hour": "2025-12-31T05:00:00.000Z"
		},
		{
			"count": 2835,
			"hour": "2025-12-31T06:00:00.000Z"
		},
		{
			"count": 1298,
			"hour": "2025-12-31T07:00:00.000Z"
		}
	],
	"httpVolume": {
		"quantity": "1.2M",
		"change": 15.3
	},
	"dataVolume": {
		"quantity": "450 GB",
		"change": -8.20
	},
	"pageView": {
		"quantity": "85.3%",
		"change": 3.5
	},
	"visits": {
		"quantity": "982K",
		"change": 12.8
	},
	"sourceIP": [
		{
			"ClientIP": "193.41.206.36",
			"change": 3.5,
			"cnt": 33999
		},
		{
			"ClientIP": "2a06:98c0:360d:481f:84ac:44b:f51c:3f9d",
			"change": 3.5,
			"cnt": 13073
		},
		{
			"ClientIP": "2a06:98c0:360c:fed5:b9e:45e4:6630:fb6",
			"change": 3.5,
			"cnt": 12320
		},
		{
			"ClientIP": "2a06:98c0:360d:6e8c:3ed:43a3:1223:87d0",
			"change": 3.5,
			"cnt": 12102
		},
		{
			"ClientIP": "2a06:98c0:360c:19c0:a485:1915:f162:c962",
			"change": 3.5,
			"cnt": 11915
		}
	],
	"triggerRule": [
		{
			"SecurityRuleDescription": "log",
			"change": 3.5,
			"cnt": 1030642
		},
		{
			"SecurityRuleDescription": "",
			"change": 3.5,
			"cnt": 58519
		},
		{
			"SecurityRuleDescription": "Version Control - Information Disclosure",
			"change": 3.5,
			"cnt": 39687
		},
		{
			"SecurityRuleDescription": "no response when not in 80_443 port",
			"change": 3.5,
			"cnt": 7943
		},
		{
			"SecurityRuleDescription": "Information Disclosure - Common Files",
			"change": 3.5,
			"cnt": 4854
		}
	],
	"hosts": [
		{
			"ClientRequestHost": "logstash-test.twister5.cf",
			"change": 3.5,
			"cnt": 668110
		},
		{
			"ClientRequestHost": "bde-70.twister5.cf",
			"change": 3.5,
			"cnt": 291753
		},
		{
			"ClientRequestHost": "aaaa.twister5.cf",
			"change": 3.5,
			"cnt": 24871
		},
		{
			"ClientRequestHost": "ipfs.twister5.cf",
			"change": 3.5,
			"cnt": 19561
		},
		{
			"ClientRequestHost": "hikarisearch.twister5.cf",
			"change": 3.5,
			"cnt": 16327
		}
	],
	"path": [
		{
			"ClientRequestPath": "/",
			"change": 3.5,
			"cnt": 1023979
		},
		{
			"ClientRequestPath": "/cdn-cgi/challenge-platform/scripts/jsd/main.js",
			"change": 3.5,
			"cnt": 5942
		},
		{
			"ClientRequestPath": "/.env",
			"change": 3.5,
			"cnt": 5837
		},
		{
			"ClientRequestPath": "/.git/config",
			"change": 3.5,
			"cnt": 4315
		},
		{
			"ClientRequestPath": "/proxy.pac",
			"change": 3.5,
			"cnt": 4122
		}
	],
	"country": [
		{
			"cnt": 599607,
			"change": 3.5,
			"geoip.geo.country_name": "Taiwan"
		},
		{
			"cnt": 235431,
			"change": 3.5,
			"geoip.geo.country_name": "United States"
		},
		{
			"cnt": 129947,
			"change": 3.5,
			"geoip.geo.country_name": "Hong Kong"
		},
		{
			"cnt": 54817,
			"change": 3.5,
			"geoip.geo.country_name": "The Netherlands"
		},
		{
			"cnt": 37913,
			"change": 3.5,
			"geoip.geo.country_name": "France"
		}
	],
	"other": {}
}
```
