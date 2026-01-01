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
               |

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

### Example Response
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
		"quantity": "450GB",
		"change": 0.8
	},
	"pageView": {
		"quantity": "124K",
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

