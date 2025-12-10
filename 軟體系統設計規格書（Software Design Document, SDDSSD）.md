# 二、軟體系統設計規格書（Software Design Document, SDD/SSD）

## 1. 文件資訊

### 1.1 文件基本資料
- 文件名稱：線上購物平台－軟體系統設計規格書（SDD）
- 版本號：v1.0.0
- 專案名稱：NextShop 線上購物平台
- 編寫單位：XXX 系統架構組
- 文件狀態：草稿 / 審核中 / 已核准

### 1.2 文件版本控制

| 版本  | 日期       | 作者   | 變更說明                     | 審核人 |
|-------|------------|--------|------------------------------|--------|
| 0.1.0 | 2025-02-01 | 李工程師 | 初版架構設計草稿           | 林主管 |
| 0.9.0 | 2025-02-10 | 李工程師 | 完成模組設計、資料庫設計   | 林主管 |
| 1.0.0 | 2025-02-20 | 李工程師 | 審查修訂後定版             | 張經理 |

### 1.3 參考文件
- 《NextShop 線上購物平台－軟體需求規格書（SRS）》 v1.0.0
- 公司內部「系統架構設計準則」
- 相關框架與資料庫官方文件（依實際選型）

---

## 2. 系統架構設計

### 2.1 整體架構概述

採用三層式＋服務化架構：
- 表現層（Presentation Layer）：Web 前端（SPA）、未來可擴充至 Mobile App
- 應用層（Application / Service Layer）：RESTful API 服務
- 資料存取層（Data Access Layer）：資料庫、快取、訊息佇列（如有）

### 2.2 高階架構（文字描述）

架構流程示意：

1. Client（Browser / Mobile App）
2. API Gateway / 反向代理（Nginx / API Gateway）
3. Application Service 群組：
   - User Service
   - Product Service
   - Order Service
   - Admin Service
   - Notification Service
   - Integration Service（金流、物流）
4. 基礎設施：
   - Database（RDBMS：PostgreSQL / MySQL）
   - Cache（Redis）
   - Message Queue（如必要）
5. 外部服務：
   - 金流 API
   - 物流 API
   - Email/SMS Service

### 2.3 技術堆疊（示例）

- 前端：
  - Vue 3 + TypeScript + Vue Router + Pinia
- 後端：
  - Golang + Gin/Fiber Framework
  - RESTful API + OpenAPI (Swagger) 文件
- 資料庫：
  - PostgreSQL 14
- 快取：
  - Redis 6
- 佈署與基礎建設：
  - Docker / Kubernetes
  - Nginx（反向代理）

---

## 3. 模組設計

### 3.1 模組分解概述

後端服務模組：

1. User Service（使用者與會員服務）
2. Product Service（商品服務）
3. Order Service（訂單／付款服務）
4. Admin Service（後台管理服務）
5. Notification Service（通知服務）
6. Integration Service（金流與物流整合）

### 3.2 模組職責與介面

#### 3.2.1 User Service

- 職責：
  - 處理註冊、登入、登出
  - 會員資料維護（基本資料、聯絡方式）
  - 密碼變更與重設
- 主要 API 介面：
  - `POST /api/v1/users/register`：註冊帳號
  - `POST /api/v1/users/login`：登入
  - `POST /api/v1/users/logout`：登出
  - `GET /api/v1/users/me`：取得當前登入使用者資料
  - `PUT /api/v1/users/me`：更新個人資料
- 依賴：
  - 資料庫：User Table
  - Notification Service：註冊驗證信 / 重設密碼通知

#### 3.2.2 Product Service

- 職責：
  - 商品列表、分類、搜尋
  - 商品詳細資訊查詢
- 主要 API：
  - `GET /api/v1/products`：查詢商品列表（支援篩選、排序、分頁）
  - `GET /api/v1/products/{id}`：查詢單一商品詳細
  - `GET /api/v1/categories`：查詢商品分類

#### 3.2.3 Order Service

- 職責：
  - 購物車管理（可視需求部分由前端處理）
  - 建立訂單、更新訂單狀態
  - 呼叫金流平台進行付款
- 主要 API：
  - `POST /api/v1/orders`：建立訂單
  - `GET /api/v1/orders`：查詢會員訂單列表
  - `GET /api/v1/orders/{id}`：查詢訂單詳細
  - `POST /api/v1/orders/{id}/pay`：初始化付款流程
- 依賴：
  - 資料庫：Order, OrderItem, Payment
  - Integration Service（金流）

#### 3.2.4 Admin Service

- 職責：
  - 商品 CRUD
  - 訂單查詢與狀態調整
  - 報表匯出
- 主要 API：
  - `POST /api/v1/admin/products`
  - `PUT /api/v1/admin/products/{id}`
  - `DELETE /api/v1/admin/products/{id}`
  - `GET /api/v1/admin/orders`
  - `GET /api/v1/admin/reports/sales`

#### 3.2.5 Notification Service

- 職責：
  - 統一管理 Email / SMS 發送
  - 提供同步或非同步通知能力
- 介面：
  - 內部呼叫或事件觸發（Message Queue）
  - 對外連線至 SMTP Server / SMS Gateway

#### 3.2.6 Integration Service

- 職責：
  - 包裝第三方金流與物流 API
  - 提供統一錯誤處理與重試機制
- 介面：
  - 提供 Order Service 呼叫之內部 API 或函式
  - 對外金流 API、物流 API

---

## 4. 資料庫設計

### 4.1 ER 模型概述

主要實體：

- `User`：使用者／會員基本資料
- `Product`：商品資料
- `Category`：商品分類
- `Order`：訂單主檔
- `OrderItem`：訂單明細
- `Payment`：付款紀錄
- `Shipment`：出貨資料
- `Address`：會員收件地址

主要關聯：

- User 1 ── N Order
- Order 1 ── N OrderItem
- Product 1 ── N OrderItem
- Category 1 ── N Product（或依需求改為多對多）
- User 1 ── N Address
- Order 1 ── 1 Payment
- Order 1 ── 1 Shipment

### 4.2 資料表結構範例

#### 4.2.1 User Table

| 欄位名稱      | 資料型別     | 說明                        | 限制                              |
|---------------|--------------|-----------------------------|-----------------------------------|
| id            | BIGSERIAL    | 使用者唯一識別碼           | PK                                |
| email         | VARCHAR(255) | Email                       | NOT NULL, UNIQUE                  |
| password_hash | VARCHAR(255) | 密碼雜湊值                 | NOT NULL                          |
| name          | VARCHAR(100) | 姓名                        | NOT NULL                          |
| phone         | VARCHAR(20)  | 手機號碼                   | UNIQUE                            |
| status        | SMALLINT     | 狀態（0:停用,1:啟用）      | 預設 1                            |
| created_at    | TIMESTAMP    | 建立時間                    | NOT NULL, default now()           |
| updated_at    | TIMESTAMP    | 更新時間                    | NOT NULL, default now()           |

#### 4.2.2 Product Table

| 欄位名稱   | 型別          | 說明                 | 限制         |
|------------|---------------|----------------------|--------------|
| id         | BIGSERIAL     | 商品 ID（PK）       | PK           |
| name       | VARCHAR(255)  | 商品名稱             | NOT NULL     |
| description| TEXT          | 商品描述             |              |
| price      | NUMERIC(10,2) | 售價                 | NOT NULL     |
| stock_qty  | INT           | 庫存數量             | NOT NULL     |
| category_id| BIGINT        | 類別 ID（FK）       | FK -> Category.id |
| status     | SMALLINT      | 狀態（0:下架,1:上架）| 預設 1       |
| created_at | TIMESTAMP     | 建立時間             | NOT NULL     |
| updated_at | TIMESTAMP     | 更新時間             | NOT NULL     |

#### 4.2.3 Order Table

| 欄位名稱       | 型別          | 說明                                          |
|----------------|---------------|-----------------------------------------------|
| id             | BIGSERIAL     | 訂單 ID（PK）                                |
| user_id        | BIGINT        | 會員 ID（FK -> User.id）                     |
| order_no       | VARCHAR(50)   | 訂單編號（需 UNIQUE）                        |
| status         | SMALLINT      | 狀態（0:草稿,1:待付款,2:已付款,3:已出貨,4:完成,5:取消） |
| total_amount   | NUMERIC(10,2) | 訂單總金額                                    |
| payment_method | VARCHAR(50)   | 付款方式（CREDIT_CARD/ATM/...）             |
| shipping_method| VARCHAR(50)   | 配送方式（HOME_DELIVERY/STORE_PICKUP/...）  |
| created_at     | TIMESTAMP     | 建立時間                                      |
| updated_at     | TIMESTAMP     | 更新時間                                      |

#### 4.2.4 OrderItem Table

| 欄位名稱   | 型別          | 說明                      |
|------------|---------------|---------------------------|
| id         | BIGSERIAL     | 訂單明細 ID（PK）        |
| order_id   | BIGINT        | 訂單 ID（FK -> Order.id）|
| product_id | BIGINT        | 商品 ID（FK -> Product.id）|
| quantity   | INT           | 數量                      |
| unit_price | NUMERIC(10,2) | 單價                      |
| subtotal   | NUMERIC(10,2) | 小計（quantity * unit_price） |

---

## 5. 介面設計

### 5.1 API 介面設計（示例）

採用 RESTful API + JSON。

#### 5.1.1 建立訂單 API

- Method：`POST`
- URL：`/api/v1/orders`
- Header：
  - `Authorization: Bearer <token>`
- Request Body：
  ```json
  {
    "items": [
      { "product_id": 1001, "quantity": 2 },
      { "product_id": 2003, "quantity": 1 }
    ],
    "shipping_address_id": 10,
    "payment_method": "CREDIT_CARD",
    "shipping_method": "HOME_DELIVERY"
  }
  ```
- Response 200：
  ```json
  {
    "order_id": 12345,
    "order_no": "NS202501010001",
    "status": "PENDING_PAYMENT",
    "total_amount": 3500
  }
  ```
- 可能錯誤：
  - 400：庫存不足或參數錯誤
  - 401：未登入或 Token 過期
  - 422：資料格式驗證失敗

#### 5.1.2 商品查詢 API

- Method：`GET`
- URL：`/api/v1/products?category_id=1&keyword=手機&page=1&page_size=20`
- 功能：
  - 回傳商品列表與分頁資訊（total_count、page、page_size）。

### 5.2 使用者介面設計（高階文字）

- 首頁：
  - Banner 區、熱門分類區、熱門商品列表
- 商品列表頁：
  - 左側：分類與篩選條件
  - 右側：商品卡片列表（圖片、名稱、價格、加入購物車）
- 商品詳細頁：
  - 商品主圖、輪播副圖
  - 價格、庫存狀態
  - 規格、描述
  - 評價與評論區
- 購物車頁：
  - 商品列表（名稱、數量、單價、小計）
  - 總金額區
  - 前往結帳按鈕

---

## 6. 安全性設計

### 6.1 身分驗證與授權

- 採用 JWT Token 機制：
  - 登入成功後簽發 Access Token 與 Refresh Token。
  - Access Token 於 Header `Authorization: Bearer <token>` 傳遞。
- API 保護：
  - 前台會員功能需登入（會員角色）。
  - 後台管理介面需具備管理者角色或更高權限。
- Token 失效與登出流程：
  - Access Token 過期後可使用 Refresh Token 重新取得。
  - 登出時可使伺服端維護的 Refresh Token 失效（如使用白名單／黑名單機制）。

### 6.2 資料保護

- 密碼：
  - 不得以明文儲存。
  - 使用加鹽雜湊演算法（如 bcrypt/Argon2）。
- 個資欄位：
  - 可選擇性加密（AES）儲存在資料庫。
- 備份：
  - 資料庫備份檔需加密存放並限制存取權限。

### 6.3 Web 安全控制

- CSRF：
  - 對敏感操作採用 CSRF Token 或 SameSite Cookie 設定。
- XSS：
  - 前端輸入過濾、後端輸出編碼。
- SQL Injection：
  - 使用參數化查詢或 ORM。
- 資安日誌：
  - 登入失敗、權限拒絕等事件需詳細記錄。

---

## 7. 部署架構

### 7.1 環境區分

- DEV（開發環境）
- QA/UAT（測試／驗收環境）
- PROD（正式環境）

各環境需分離：
- 資料庫實例
- 金流測試金鑰與正式金鑰
- Domain / URL
- Log 與監控配置

### 7.2 部署拓樸（Kubernetes 示例）

- Namespace：
  - `nextshop-prod`
- Deployments：
  - `nextshop-api-deployment`
  - `nextshop-web-deployment`
- Services：
  - `nextshop-api-service`（ClusterIP）
  - `nextshop-web-service`（ClusterIP）
- Ingress：
  - `nextshop-ingress` 綁定網域 `www.nextshop.com`
- 資料庫：
  - Managed PostgreSQL 或自建主從複寫架構
- 監控：
  - Prometheus + Grafana
  - ELK / OpenSearch Stack

### 7.3 設定與機密管理

- Config：
  - 以 ConfigMap / 環境變數存放非機密設定（如 log level）。
- Secret：
  - 儲存資料庫密碼、金流 API Key、SMTP 帳密等。
- 若使用外部服務（如 Vault / KMS），在啟動時載入必要機密。

---

## 8. 日誌與監控設計

### 8.1 日誌分類

- Access Log：
  - 記錄 API 請求 URL、HTTP Method、Status Code、延遲時間等。
- Application Log：
  - 記錄業務流程事件（建立訂單、付款成功／失敗等）。
- Error Log：
  - 記錄例外錯誤堆疊、第三方 API 呼叫失敗。

### 8.2 監控指標

- 系統層級：
  - CPU、Memory、Disk、Network I/O。
- 應用層級：
  - QPS
  - 延遲分佈（p50, p95, p99）
  - 錯誤率（4xx、5xx）
- 業務指標（可選）：
  - 每日訂單數
  - 每日成功付款金額

---

## 9. 測試與品質保證設計

### 9.1 測試分類

- 單元測試（Unit Test）：
  - 目標覆蓋率（例如：≥ 70%）
- 整合測試（Integration Test）：
  - 針對跨模組流程（登入、下單、付款、物流更新）
- API 自動化測試：
  - 對關鍵 API 編寫自動化測試腳本
- 端對端測試（E2E）：
  - 模擬實際使用者操作情境

### 9.2 測試環境需求

- 與 PROD 架構一致或接近（縮小版）
- 使用金流／物流測試環境與測試帳號
- 匹配 SRS 中定義的性能與安全性驗收指標
