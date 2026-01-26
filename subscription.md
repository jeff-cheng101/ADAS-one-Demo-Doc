# Gateway Subscription API

## 概述

Gateway Subscription API 提供訂閱清單資料的讀取功能，資料來源為 `backend/data/subscription.json`。

---

## 端點

### GET /api/gateway/subscription

取得所有訂閱清單資料。

#### 請求

- **方法**：`GET`
- **路徑**：`/api/gateway/subscription`
- **認證**：無需認證
- **Content-Type**：無需請求主體

#### 成功回應

- **狀態碼**：`200 OK`
- **Content-Type**：`application/json`

```json
[
  {
    "ai_name": "GPT Plus",
    "price": 20,
    "duration": 1,
    "subscribe_time": "2025-01-01 10:00:00",
    "currency_code": "USD",
    "create_time": "2025-01-01 10:00:00",
    "update_time": ""
  },
  {
    "ai_name": "Cursor Pro",
    "price": 20,
    "duration": 1,
    "subscribe_time": "2025-01-05 11:00:00",
    "currency_code": "USD",
    "create_time": "2025-01-05 11:00:00",
    "update_time": ""
  },
  {
    "ai_name": "Gemini Advanced",
    "price": 19.99,
    "duration": 1,
    "subscribe_time": "2025-01-12 08:45:00",
    "currency_code": "USD",
    "create_time": "2025-01-12 08:45:00",
    "update_time": ""
  }
]
```

#### 回應欄位說明

| 欄位 | 類型 | 說明 |
|------|------|------|
| `ai_name` | string | AI 服務名稱 |
| `price` | number | 訂閱價格 |
| `duration` | number | 訂閱期間（月） |
| `subscribe_time` | string | 訂閱時間（格式：`YYYY-MM-DD HH:mm:ss`） |
| `currency_code` | string | 貨幣代碼（如 `USD`） |
| `create_time` | string | 建立時間（格式：`YYYY-MM-DD HH:mm:ss`） |
| `update_time` | string | 更新時間（格式：`YYYY-MM-DD HH:mm:ss`） |

#### 錯誤回應

- **狀態碼**：`500 Internal Server Error`
- **Content-Type**：`application/json`

##### 檔案不存在

```json
{
  "error": "subscription.json 不存在"
}
```

##### 格式錯誤（非陣列）

```json
{
  "error": "subscription.json 格式錯誤"
}
```

##### 讀取或解析失敗

```json
{
  "error": "讀取 subscription.json 失敗",
  "details": "錯誤詳細訊息"
}
```

---

## 使用範例

### cURL

```bash
curl -X GET http://localhost:8081/api/gateway/subscription
```

### JavaScript (fetch)

```javascript
const response = await fetch('/api/gateway/subscription');
const subscriptions = await response.json();
console.log(subscriptions);
```

### Axios

```javascript
import axios from 'axios';

const { data } = await axios.get('/api/gateway/subscription');
console.log(data);
```

---

## 資料來源

訂閱清單資料儲存於 `backend/data/subscription.json`，如需修改訂閱資料，請直接編輯該檔案。

---

## 相關檔案

| 檔案 | 說明 |
|------|------|
| `backend/routes/gateway.routes.js` | 路由實作 |
| `backend/data/subscription.json` | 訂閱資料檔案 |
| `backend/index.js` | 路由註冊（`/api/gateway`） |
