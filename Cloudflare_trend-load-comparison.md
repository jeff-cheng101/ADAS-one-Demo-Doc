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
### Response fields

| Attribute       | Type    | Required | Description      |
| --------------- | ------- | -------- | ---------------- |
| success         | boolean | Y        | 請求是否成功           |
| periods         | object  | Y        | 時期區間資訊（當前/上一）    |
| currentPeriod   | object  | Y        | 當前時期統計與明細        |
| previousPeriod  | object  | Y        | 上一時期統計與明細        |
| comparisonChart | object  | Y        | 當前/上一時期對照圖表資料    |
| statistics      | object  | Y        | 當前 vs 上一期的統計變化摘要 |
| queryInfo       | object  | Y        | 查詢批次執行資訊（含進度紀錄）  |

### periods fields

| Attribute | Type   | Required | Description |
| --------- | ------ | -------- | ----------- |
| current   | object | Y        | 當前時期資訊      |
| previous  | object | Y        | 上一時期資訊      |

### periods.current / periods.previous fields

| Attribute | Type   | Required | Description             |
| --------- | ------ | -------- | ----------------------- |
| start     | string | Y        | ISO 8601 起始時間（UTC, `Z`） |
| end       | string | Y        | ISO 8601 結束時間（UTC, `Z`） |
| label     | string | Y        | 顯示用標籤                   |

### currentPeriod fields

| Attribute            | Type          | Required | Description                    |
| -------------------- | ------------- | -------- | ------------------------------ |
| period               | object        | Y        | 當前時期區間資訊（同 periods.current 結構） |
| timeSeries           | array<object> | Y        | 時間序列資料（依 groupInterval 分組）     |
| totalRequestTraffic  | number        | Y        | 總流量（以 response 內 traffic 加總口徑） |
| totalRequests        | number        | Y        | 總請求數                           |
| avgTrafficPerRequest | number        | Y        | 平均每請求流量                        |
| topTrafficIPs        | array<object> | Y        | Top 流量來源 IP 清單                 |
| peakTrafficHour      | number        | Y        | 峰值流量（欄位名稱為 Hour，但值以分組口徑提供）     |
| uniqueIPs            | number        | Y        | 不重複 IP 數                       |
| attackIPs            | number        | Y        | 被判定為攻擊 IP 數                    |
| groupInterval        | number        | Y        | 分組間隔（毫秒），例：86400000 = 1 天      |

### currentPeriod.period fields

| Attribute | Type   | Required | Description             |
| --------- | ------ | -------- | ----------------------- |
| start     | string | Y        | ISO 8601 起始時間（UTC, `Z`） |
| end       | string | Y        | ISO 8601 結束時間（UTC, `Z`） |
| label     | string | Y        | 顯示用標籤                   |

### currentPeriod.timeSeries item fields

| Attribute | Type   | Required | Description |
| --------- | ------ | -------- | ----------- |
| timestamp | number | Y        | 時間戳（毫秒）     |
| traffic   | number | Y        | 該分組流量       |
| requests  | number | Y        | 該分組請求數      |
| uniqueIPs | number | Y        | 該分組不重複 IP 數 |
| attackIPs | number | Y        | 該分組攻擊 IP 數  |

### currentPeriod.topTrafficIPs item fields

| Attribute  | Type    | Required | Description            |
| ---------- | ------- | -------- | ---------------------- |
| ip         | string  | Y        | 來源 IP（可能為 IPv4 或 IPv6） |
| traffic    | number  | Y        | 該 IP 流量                |
| requests   | number  | Y        | 該 IP 請求數               |
| country    | string  | Y        | 國別代碼（小寫，如 `tw`, `cn`）  |
| asn        | number  | Y        | ASN（自治系統號）             |
| isAttackIP | boolean | Y        | 是否為攻擊 IP               |

### previousPeriod fields

| Attribute            | Type          | Required | Description                     |
| -------------------- | ------------- | -------- | ------------------------------- |
| period               | object        | Y        | 上一時期區間資訊（同 periods.previous 結構） |
| timeSeries           | array<object> | Y        | 時間序列資料（依 groupInterval 分組）      |
| totalRequestTraffic  | number        | Y        | 總流量                             |
| totalRequests        | number        | Y        | 總請求數                            |
| avgTrafficPerRequest | number        | Y        | 平均每請求流量                         |
| topTrafficIPs        | array<object> | Y        | Top 流量來源 IP 清單                  |
| peakTrafficHour      | number        | Y        | 峰值流量                            |
| uniqueIPs            | number        | Y        | 不重複 IP 數                        |
| attackIPs            | number        | Y        | 被判定為攻擊 IP 數                     |
| groupInterval        | number        | Y        | 分組間隔（毫秒）                        |

### previousPeriod.period fields

| Attribute | Type   | Required | Description             |
| --------- | ------ | -------- | ----------------------- |
| start     | string | Y        | ISO 8601 起始時間（UTC, `Z`） |
| end       | string | Y        | ISO 8601 結束時間（UTC, `Z`） |
| label     | string | Y        | 顯示用標籤                   |

### previousPeriod.timeSeries item fields

| Attribute | Type   | Required | Description |
| --------- | ------ | -------- | ----------- |
| timestamp | number | Y        | 時間戳（毫秒）     |
| traffic   | number | Y        | 該分組流量       |
| requests  | number | Y        | 該分組請求數      |
| uniqueIPs | number | Y        | 該分組不重複 IP 數 |
| attackIPs | number | Y        | 該分組攻擊 IP 數  |

### previousPeriod.topTrafficIPs item fields

| Attribute  | Type    | Required | Description            |
| ---------- | ------- | -------- | ---------------------- |
| ip         | string  | Y        | 來源 IP（可能為 IPv4 或 IPv6） |
| traffic    | number  | Y        | 該 IP 流量                |
| requests   | number  | Y        | 該 IP 請求數               |
| country    | string  | Y        | 國別代碼（小寫）               |
| asn        | number  | Y        | ASN（自治系統號）             |
| isAttackIP | boolean | Y        | 是否為攻擊 IP               |

### comparisonChart fields

| Attribute     | Type          | Required | Description |
| ------------- | ------------- | -------- | ----------- |
| data          | array<object> | Y        | 對照序列資料      |
| currentLabel  | string        | Y        | 當前時期標籤      |
| previousLabel | string        | Y        | 上一時期標籤      |


###　comparisonChart.data item fields

| Attribute         | Type        | Required | Description          |
| ----------------- | ----------- | -------- | -------------------- |
| timeLabel         | string      | Y        | 顯示用時間序列標籤（如 第1天）     |
| currentPeriod     | number      | Y        | 當前時期該點流量             |
| previousPeriod    | number      | Y        | 上一時期該點流量（可能為 0）      |
| currentTimestamp  | number      | Y        | 當前時期時間戳（毫秒）          |
| previousTimestamp | number|null | Y        | 上一時期時間戳（毫秒；可能為 null） |
| currentRequests   | number      | Y        | 當前時期該點請求數            |
| previousRequests  | number      | Y        | 上一時期該點請求數（可能為 0）     |

### statistics fields

| Attribute        | Type   | Required | Description |
| ---------------- | ------ | -------- | ----------- |
| trafficChange    | object | Y        | 流量變化        |
| requestsChange   | object | Y        | 請求數變化       |
| ipsChange        | object | Y        | 不重複 IP 變化   |
| attackIPsChange  | object | Y        | 攻擊 IP 變化    |
| avgTrafficChange | object | Y        | 平均每請求流量變化   |

### statistics.*Change fields（通用結構）

| Attribute  | Type   | Required | Description           |
| ---------- | ------ | -------- | --------------------- |
| current    | number | Y        | 當前時期值                 |
| previous   | number | Y        | 上一時期值                 |
| changeRate | string | Y        | 變化率（字串型態，例：`"10.08"`） |

### queryInfo fields

| Attribute         | Type          | Required | Description    |
| ----------------- | ------------- | -------- | -------------- |
| totalBatches      | number        | Y        | 總批次數           |
| successfulBatches | number        | Y        | 成功批次數          |
| failedBatches     | number        | Y        | 失敗批次數          |
| totalRecords      | number        | Y        | 總紀錄數           |
| queryMethod       | string        | Y        | 查詢方法（例：`esql`） |
| progressLog       | array<object> | Y        | 批次執行進度紀錄       |

### queryInfo.progressLog item fields

| Attribute    | Type    | Required | Description                             |
| ------------ | ------- | -------- | --------------------------------------- |
| type         | string  | Y        | 紀錄類型（如 `batch_start`, `batch_complete`） |
| batchIndex   | number  | Y        | 批次序號（1-based）                           |
| totalBatches | number  | Y        | 總批次數                                    |
| description  | string  | Y        | 顯示用描述                                   |
| timeRange    | string  | Y        | 本批次涵蓋時間區間（字串）                           |
| timestamp    | string  | Y        | 事件時間（ISO 8601 字串）                       |
| recordCount  | number  | N        | 僅 `batch_complete` 出現：該批次回傳筆數           |
| success      | boolean | N        | 僅 `batch_complete` 出現：該批次是否成功           |



### Example Response
```json
{
    "success": true,
    "periods": {
        "current": {
            "start": "2025-12-29T14:52:03.000Z",
            "end": "2025-12-31T05:45:43.000Z",
            "label": "當前時期 (12/29 下午10:52 - 12/31 下午01:45)"
        },
        "previous": {
            "start": "2025-12-28T00:00:30.000Z",
            "end": "2025-12-29T14:52:01.000Z",
            "label": "上一時期 (12/28 上午08:00 - 12/29 下午10:52)"
        }
    },
    "currentPeriod": {
        "period": {
            "start": "2025-12-29T14:52:03.000Z",
            "end": "2025-12-31T05:45:43.000Z",
            "label": "當前時期 (12/29 下午10:52 - 12/31 下午01:45)"
        },
        "timeSeries": [
            {
                "timestamp": 1766937600000,
                "traffic": 345410,
                "requests": 121,
                "uniqueIPs": 106,
                "attackIPs": 106
            },
            {
                "timestamp": 1767024000000,
                "traffic": 7353960,
                "requests": 2626,
                "uniqueIPs": 1789,
                "attackIPs": 1789
            },
            {
                "timestamp": 1767110400000,
                "traffic": 4310282,
                "requests": 1530,
                "uniqueIPs": 1072,
                "attackIPs": 1072
            }
        ],
        "totalRequestTraffic": 12009652,
        "totalRequests": 4277,
        "avgTrafficPerRequest": 2807.9616553659107,
        "topTrafficIPs": [
            {
                "ip": "14.127.201.112",
                "traffic": 366996,
                "requests": 142,
                "country": "cn",
                "asn": 4134,
                "isAttackIP": true
            },
            {
                "ip": "14.154.127.114",
                "traffic": 349887,
                "requests": 136,
                "country": "cn",
                "asn": 4134,
                "isAttackIP": true
            },
            {
                "ip": "220.228.194.155",
                "traffic": 333779,
                "requests": 81,
                "country": "tw",
                "asn": 9919,
                "isAttackIP": true
            },
            {
                "ip": "2601:646:8f01:9f60:9d9d:3b1f:c3d:47ad",
                "traffic": 293042,
                "requests": 45,
                "country": "us",
                "asn": 7922,
                "isAttackIP": true
            },
            {
                "ip": "201.219.109.118",
                "traffic": 250965,
                "requests": 50,
                "country": "ar",
                "asn": 14232,
                "isAttackIP": true
            },
            {
                "ip": "50.232.53.50",
                "traffic": 186456,
                "requests": 42,
                "country": "us",
                "asn": 7922,
                "isAttackIP": true
            },
            {
                "ip": "75.70.34.6",
                "traffic": 184062,
                "requests": 36,
                "country": "us",
                "asn": 7922,
                "isAttackIP": true
            },
            {
                "ip": "2601:646:8f01:9f60:e5fa:fe6a:c94b:63d5",
                "traffic": 166077,
                "requests": 22,
                "country": "us",
                "asn": 7922,
                "isAttackIP": true
            },
            {
                "ip": "211.72.13.223",
                "traffic": 159757,
                "requests": 60,
                "country": "tw",
                "asn": 3462,
                "isAttackIP": true
            },
            {
                "ip": "172.6.184.125",
                "traffic": 125987,
                "requests": 26,
                "country": "us",
                "asn": 7018,
                "isAttackIP": true
            }
        ],
        "peakTrafficHour": 7353960,
        "uniqueIPs": 2773,
        "attackIPs": 2773,
        "groupInterval": 86400000
    },
    "previousPeriod": {
        "period": {
            "start": "2025-12-28T00:00:30.000Z",
            "end": "2025-12-29T14:52:01.000Z",
            "label": "上一時期 (12/28 上午08:00 - 12/29 下午10:52)"
        },
        "timeSeries": [
            {
                "timestamp": 1766851200000,
                "traffic": 4422470,
                "requests": 1763,
                "uniqueIPs": 1326,
                "attackIPs": 1326
            },
            {
                "timestamp": 1766937600000,
                "traffic": 6487874,
                "requests": 2513,
                "uniqueIPs": 1817,
                "attackIPs": 1817
            }
        ],
        "totalRequestTraffic": 10910344,
        "totalRequests": 4276,
        "avgTrafficPerRequest": 2551.5304022450887,
        "topTrafficIPs": [
            {
                "ip": "220.228.194.155",
                "traffic": 177600,
                "requests": 49,
                "country": "tw",
                "asn": 9919,
                "isAttackIP": true
            },
            {
                "ip": "202.39.33.231",
                "traffic": 154080,
                "requests": 72,
                "country": "tw",
                "asn": 3462,
                "isAttackIP": true
            },
            {
                "ip": "1.163.246.197",
                "traffic": 135861,
                "requests": 46,
                "country": "tw",
                "asn": 3462,
                "isAttackIP": true
            },
            {
                "ip": "144.76.32.241",
                "traffic": 134690,
                "requests": 62,
                "country": "de",
                "asn": 24940,
                "isAttackIP": true
            },
            {
                "ip": "211.72.13.223",
                "traffic": 133494,
                "requests": 49,
                "country": "tw",
                "asn": 3462,
                "isAttackIP": true
            },
            {
                "ip": "2407:4d00:8c03:265:d809:198d:b315:36c1",
                "traffic": 98910,
                "requests": 21,
                "country": "tw",
                "asn": 38841,
                "isAttackIP": true
            },
            {
                "ip": "2402:7500:942:cb10:cc7d:16eb:7ffe:d59d",
                "traffic": 96108,
                "requests": 19,
                "country": "tw",
                "asn": 24158,
                "isAttackIP": true
            },
            {
                "ip": "2402:7500:942:cb10:74ea:f409:674f:985d",
                "traffic": 63467,
                "requests": 12,
                "country": "tw",
                "asn": 24158,
                "isAttackIP": true
            },
            {
                "ip": "14.154.127.114",
                "traffic": 61741,
                "requests": 24,
                "country": "cn",
                "asn": 4134,
                "isAttackIP": true
            },
            {
                "ip": "2a06:98c0:3616:b0b9:33a6:9c7a:d0c6:82ff",
                "traffic": 56955,
                "requests": 1,
                "country": "nl",
                "asn": 202623,
                "isAttackIP": true
            }
        ],
        "peakTrafficHour": 6487874,
        "uniqueIPs": 2964,
        "attackIPs": 2964,
        "groupInterval": 86400000
    },
    "comparisonChart": {
        "data": [
            {
                "timeLabel": "第1天",
                "currentPeriod": 345410,
                "previousPeriod": 4422470,
                "currentTimestamp": 1766937600000,
                "previousTimestamp": 1766851200000,
                "currentRequests": 121,
                "previousRequests": 1763
            },
            {
                "timeLabel": "第2天",
                "currentPeriod": 7353960,
                "previousPeriod": 6487874,
                "currentTimestamp": 1767024000000,
                "previousTimestamp": 1766937600000,
                "currentRequests": 2626,
                "previousRequests": 2513
            },
            {
                "timeLabel": "第3天",
                "currentPeriod": 4310282,
                "previousPeriod": 0,
                "currentTimestamp": 1767110400000,
                "previousTimestamp": null,
                "currentRequests": 1530,
                "previousRequests": 0
            }
        ],
        "currentLabel": "當前時期 (12/29 下午10:52 - 12/31 下午01:45)",
        "previousLabel": "上一時期 (12/28 上午08:00 - 12/29 下午10:52)"
    },
    "statistics": {
        "trafficChange": {
            "current": 12009652,
            "previous": 10910344,
            "changeRate": "10.08"
        },
        "requestsChange": {
            "current": 4277,
            "previous": 4276,
            "changeRate": "0.02"
        },
        "ipsChange": {
            "current": 2773,
            "previous": 2964,
            "changeRate": "-6.44"
        },
        "attackIPsChange": {
            "current": 2773,
            "previous": 2964,
            "changeRate": "-6.44"
        },
        "avgTrafficChange": {
            "current": 2807.9616553659107,
            "previous": 2551.5304022450887,
            "changeRate": "10.05"
        }
    },
    "queryInfo": {
        "totalBatches": 7,
        "successfulBatches": 7,
        "failedBatches": 0,
        "totalRecords": 8553,
        "queryMethod": "esql",
        "progressLog": [
            {
                "type": "batch_start",
                "batchIndex": 7,
                "totalBatches": 7,
                "description": "批次 7/7",
                "timeRange": "2025-12-24T05:48:19.512Z - 2025-12-25T05:48:19.512Z",
                "timestamp": "2025-12-31T05:48:19.512Z"
            },
            {
                "type": "batch_complete",
                "batchIndex": 7,
                "totalBatches": 7,
                "recordCount": 0,
                "success": true,
                "timestamp": "2025-12-31T05:48:19.568Z"
            },
            {
                "type": "batch_start",
                "batchIndex": 6,
                "totalBatches": 7,
                "description": "批次 6/7",
                "timeRange": "2025-12-25T05:48:19.512Z - 2025-12-26T05:48:19.512Z",
                "timestamp": "2025-12-31T05:48:19.568Z"
            },
            {
                "type": "batch_complete",
                "batchIndex": 6,
                "totalBatches": 7,
                "recordCount": 0,
                "success": true,
                "timestamp": "2025-12-31T05:48:19.601Z"
            },
            {
                "type": "batch_start",
                "batchIndex": 5,
                "totalBatches": 7,
                "description": "批次 5/7",
                "timeRange": "2025-12-26T05:48:19.512Z - 2025-12-27T05:48:19.512Z",
                "timestamp": "2025-12-31T05:48:20.112Z"
            },
            {
                "type": "batch_complete",
                "batchIndex": 5,
                "totalBatches": 7,
                "recordCount": 0,
                "success": true,
                "timestamp": "2025-12-31T05:48:20.150Z"
            },
            {
                "type": "batch_start",
                "batchIndex": 4,
                "totalBatches": 7,
                "description": "批次 4/7",
                "timeRange": "2025-12-27T05:48:19.512Z - 2025-12-28T05:48:19.512Z",
                "timestamp": "2025-12-31T05:48:20.653Z"
            },
            {
                "type": "batch_complete",
                "batchIndex": 4,
                "totalBatches": 7,
                "recordCount": 633,
                "success": true,
                "timestamp": "2025-12-31T05:48:21.135Z"
            },
            {
                "type": "batch_start",
                "batchIndex": 3,
                "totalBatches": 7,
                "description": "批次 3/7",
                "timeRange": "2025-12-28T05:48:19.512Z - 2025-12-29T05:48:19.512Z",
                "timestamp": "2025-12-31T05:48:21.638Z"
            },
            {
                "type": "batch_complete",
                "batchIndex": 3,
                "totalBatches": 7,
                "recordCount": 2655,
                "success": true,
                "timestamp": "2025-12-31T05:48:23.207Z"
            },
            {
                "type": "batch_start",
                "batchIndex": 2,
                "totalBatches": 7,
                "description": "批次 2/7",
                "timeRange": "2025-12-29T05:48:19.512Z - 2025-12-30T05:48:19.512Z",
                "timestamp": "2025-12-31T05:48:23.709Z"
            },
            {
                "type": "batch_complete",
                "batchIndex": 2,
                "totalBatches": 7,
                "recordCount": 2605,
                "success": true,
                "timestamp": "2025-12-31T05:48:26.633Z"
            },
            {
                "type": "batch_start",
                "batchIndex": 1,
                "totalBatches": 7,
                "description": "批次 1/7",
                "timeRange": "2025-12-30T05:48:19.512Z - 2025-12-31T05:48:19.512Z",
                "timestamp": "2025-12-31T05:48:27.146Z"
            },
            {
                "type": "batch_complete",
                "batchIndex": 1,
                "totalBatches": 7,
                "recordCount": 2660,
                "success": true,
                "timestamp": "2025-12-31T05:48:28.753Z"
            }
        ]
    }
}
```

