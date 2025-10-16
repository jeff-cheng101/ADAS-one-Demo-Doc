# API 文件規格說明

## **簡介（Introduction）**

此文件說明後端服務的 API 設計與使用規範。

---

## **Resource 資源**

| Project | Method | URL |
|----------|---------|-----|
| WEB | GET | `/apis/user/:id` |
| APP | GET | `/bde/user/:id` |

### **Project 說明**
- **WEB**：Web 或 WAP 平台使用的基礎服務端點。
- **APP**：行動端 API。
- **APIS**：Backend 系統內部 API。

### **Method**
- HTTP Method（如 GET、POST、PUT、DELETE）

### **URL**
- API 路徑

---

## **需求（Requirements）**

| Requirements | Description |
|---------------|-------------|
| Auth | 需附帶 Token 認證 |

- 所有 API 請求需攜帶合法授權 Token。
- 無授權或過期 Token 將返回錯誤。

---

## **請求參數（Request Parameters）**

**Type**：`json`（請求參數採 JSON 格式）

| Attribute | Type | Required | Description |
|------------|------|-----------|--------------|
| :id | string | Y | 成員 ID |
| type | int | N | 類型代碼 |

### **欄位說明**
- **Attribute**：參數名稱（若有 `:` 表示為路徑參數）
- **Type**：
  - string：字串
  - int：整數
  - float：浮點數
  - bool：布林值
  - list：列表
  - object：物件
- **Required**：
  - Y：必填
  - N：非必填
- **Description**：參數用途與說明

---

## **回應欄位（Response Fields）**

| Attribute | Type | Required | Description |
|------------|------|-----------|--------------|
| name | string | Y | 姓名 |
| age | int | Y | 年齡 |
| job | object | Y | 職業資訊 |
| skill | list(string) | Y | 技能清單 |

### **job 物件內容**

| Attribute | Type | Required | Description |
|------------|------|-----------|--------------|
| name | string | Y | 職稱 |
| years | float | Y | 年資（年） |

---

## **回應範例（Example Response）**

```json
{
  "name": "jeff",
  "age": 17,
  "job": {
    "name": "RD",
    "years": 1
  },
  "skill": [
    "go",
    "java"
  ]
}
```

---

## **錯誤碼（Errors）**

| Code | Message |
|------|----------|
| 100-001 | Token 無效 |
| 100-002 | 權限不足 |

---

## **附註（Notes）**

- 若 API 回傳 `HTTP 401`，表示 Token 無效或過期。
- 若回傳 `HTTP 403`，代表使用者沒有存取權限。
- 所有時間欄位採用 UTC 格式。
- 請求範例與回應 JSON 僅供參考，實際內容依業務邏輯為準。

---

## **版本紀錄（Changelog）**

- 由 **Jeff_Cheng** 編寫，更新於 **2025 年 10 月**
- 文件版本：v1.0
- 平台： github
