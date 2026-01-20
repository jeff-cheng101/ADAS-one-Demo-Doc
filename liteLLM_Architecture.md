# LiteLLM 專案分析報告

## 專案概述

**LiteLLM** 是一個統一的開源介面庫，提供對 100+ 大型語言模型 (LLM) 的 OpenAI 格式調用。該專案旨在簡化跨多個 LLM 提供者的複雜管理與調用翻譯工作。

**官方網站**: https://litellm.ai
**GitHub**: https://github.com/BerriAI/litellm
**當前版本**: 1.81.0

---

## 核心功能

### 1. Python SDK
- 提供簡潔的 Python API 調用各種 LLM
- 支援 OpenAI 格式的請求/響應
- 內建重試和備援邏輯
- 可觀測性回調 (Langfuse, MLflow, Datadog 等)

### 2. AI Gateway (Proxy Server)
- 中央化 API 網關服務
- 認證與授權管理
- 速率限制與預算控制
- 負載均衡與路由
- 虛擬 API 金鑰管理
- 管理後台 UI

### 3. 支援的 LLM 提供者 (100+)
| 類別 | 提供者 |
|------|--------|
| 雲端服務商 | OpenAI, Anthropic, Azure, Google Vertex AI, Google Gemini, AWS Bedrock, AWS Sagemaker |
| 开源模型 | Ollama, HuggingFace, VLLM, LM Studio, Llamafile |
| 新興服務商 | Groq, Together AI, Perplexity, DeepSeek, xAI |
| 企業級 | Databricks, Snowflake, IBM Watsonx, Oracle OCI |
| 專業領域 | Cohere (嵌入/重排序), Jina AI, Voyage AI |

### 4. 支援的 API 端點
- `/chat/completions` - 對話完成
- `/messages` - 訊息 API (Anthropic 相容)
- `/responses` - OpenAI Responses API
- `/embeddings` - 嵌入生成
- `/images/generations` - 圖像生成
- `/audio/transcriptions` - 語音轉文字
- `/audio/speech` - 文字轉語音
- `/batches` - 批次處理
- `/rerank` - 重排序
- `/vector_stores` - 向量儲存
- `/files` - 檔案管理
- `/fine_tuning` - 微調工作
- `/a2a` - Agent-to-Agent 協議

---

## 專案結構

```
litellm/
├── litellm/                    # 核心 SDK
│   ├── main.py                 # 主要入口 (completion, acompletion 等)
│   ├── router.py               # 路由器 (負載均衡、備援)
│   ├── __init__.py             # 全域配置與狀態
│   ├── llms/                   # LLM 提供者實作
│   │   ├── openai/             # OpenAI
│   │   ├── anthropic/          # Anthropic Claude
│   │   ├── azure/              # Azure OpenAI
│   │   ├── bedrock/            # AWS Bedrock
│   │   ├── vertex_ai/          # Google Vertex AI
│   │   ├── gemini/             # Google Gemini
│   │   ├── huggingface/        # HuggingFace
│   │   ├── ollama/             # Ollama
│   │   └── ... (80+ providers)
│   ├── proxy/                  # Proxy 伺服器
│   │   ├── proxy_server.py     # 主伺服器
│   │   ├── auth/               # 認證模組
│   │   ├── hooks/              # 鉤子 (預算限制、速率限制)
│   │   ├── management_endpoints/ # 管理 API
│   │   └── _experimental/      # 實驗功能
│   ├── integrations/           # 第三方整合
│   ├── caching/                # 快取機制
│   ├── types/                  # 類型定義
│   ├── litellm_core_utils/     # 核心工具
│   └── experimental_mcp_client/ # MCP 客戶端
├── tests/                      # 測試套件
├── ui/litellm-dashboard/       # 管理 UI (React)
├── docs/                       # 文件
├── enterprise/                 # 企業版功能
└── cookbook/                   # 範例程式碼
```

---

## 技術架構

### SDK 請求流程

```
用戶端請求
    ↓
main.py (completion/acompletion)
    ↓
utils.py (get_llm_provider) - 解析模型提供者
    ↓
llms/custom_httpx/llm_http_handler.py (BaseLLMHTTPHandler)
    ↓
llms/{provider}/chat/transformation.py (翻譯層)
    ├─ transform_request()  - 轉換 OpenAI 格式 → 提供者格式
    └─ transform_response() - 轉換提供者格式 → OpenAI 格式
    ↓
HTTP 請求 → LLM 提供者 API
    ↓
翻譯響應 → ModelResponse
    ↓
回調處理 (非同步)
```

### Proxy 請求流程

```
HTTP 請求 (POST /v1/chat/completions)
    ↓
proxy/proxy_server.py (端點處理)
    ↓
proxy/auth/user_api_key_auth.py (認證)
    ↓
proxy/hooks/ (預算限制、速率限制)
    ↓
router.py (路由選擇)
    ↓
litellm.acompletion() (SDK 調用)
    ↓
成本計算與日誌記錄
    ↓
響應返回用戶端
```

### 翻譯層設計

每個提供者都有獨立的翻譯檔案：

| 提供者 | 翻譯檔案 |
|--------|----------|
| OpenAI | `llms/openai/chat/gpt_transformation.py` |
| Anthropic | `llms/anthropic/chat/transformation.py` |
| Bedrock Converse | `llms/bedrock/chat/converse_transformation.py` |
| Bedrock Invoke | `llms/bedrock/chat/invoke_transformations/*.py` |
| Gemini | `llms/gemini/chat/transformation.py` |
| Vertex AI | `llms/vertex_ai/gemini/transformation.py` |

所有翻譯類別繼承自 `BaseConfig`，實作：
- `transform_request()` - 請求格式轉換
- `transform_response()` - 響應格式轉換

---

## 基礎設施

### 快取層 (Redis + In-Memory)
- 速率限制計數器 (TPM/RPM)
- API 金鑰快取
- LLM 響應快取
- 部署冷卻時間追蹤
- 花費佇列

### 資料庫 (PostgreSQL)
- API 金鑰 (LiteLLM_VerificationToken)
- 團隊 (LiteLLM_TeamTable)
- 用戶 (LiteLLM_UserTable)
- 花費日誌 (LiteLLM_SpendLogs)

### 背景工作 (APScheduler)
- `update_spend` (60s) - 批次寫入花費日誌
- `reset_budget` (10-12min) - 重置預算
- `add_deployment` (10s) - 同步新模型部署
- 健康檢查 (連續執行)

---

## 主要依賴

### 核心依賴
```
python >=3.9, <4.0
httpx >=0.23.0      # HTTP 客戶端
openai >=2.8.0      # OpenAI API 相容
pydantic ^2.5.0     # 資料驗證
tiktoken >=0.7.0    # Token 計數
aiohttp >=3.10      # 非同步 HTTP
jsonschema >=4.23.0 # JSON 驗證
```

### Proxy 額外依賴
```
fastapi >=0.120.1
uvicorn ^0.31.1
gunicorn ^23.0.0
apscheduler ^3.10.4
PyJWT ^2.10.1
prisma 0.11.0       # ORM
boto3 1.36.0        # AWS SDK
redisvl ^0.4.1      # Redis 向量搜尋
mcp ^1.21.2         # Model Context Protocol
```

---

## 開發環境

### 快速開始
```bash
# 安裝依賴
pip install -e ".[all]"

# 運行 Proxy
litellm --model gpt-4o

# 或運行開發伺服器
python litellm/proxy_cli.py
```

### 程式碼品質
- **格式化**: Black
- **Linting**: Ruff
- **類型檢查**: MyPy
- **測試**: pytest + pytest-asyncio

### 貢獻指南
```bash
make install-dev   # 安裝開發依賴
make format        # 格式化程式碼
make lint          # 執行 linting
make test-unit     # 執行單元測試
```

---

## 專案特色

### 1. 統一的 API 介面
所有 LLM 調用都使用 OpenAI 格式，無需學習各提供者獨特的 API。

### 2. 完整的錯誤處理
- `AuthenticationError` - 認證錯誤
- `RateLimitError` - 速率限制
- `ContextWindowExceededError` - 上下文超限
- `ContentPolicyViolationError` - 內容政策違規
- 以及更多標準錯誤類型

### 3. 可觀測性整合
支援 30+ 觀測平台：
- Langfuse, LangSmith, Langtrace
- Datadog, Prometheus, OpenTelemetry
- MLflow, Arize, Galileo
- 郵件通知 (Resend, SendGrid, SMTP)

### 4. MCP (Model Context Protocol) 支援
- MCP Server 整合
- 支援外部 MCP 服務 (Zapier, Jira, Linear 等)
- Cursor IDE 整合

### 5. 企業功能
- 預算管理與追蹤
- 團隊與用戶管理
- SSO 單一登入
- 自定義 SLA

---

## 效能指標

- **延遲**: 8ms P95 (在 1k RPS 下)
- **穩定版本**: 提供 `-stable` Docker 標籤 (經過 12 小時負載測試)

---

# 企業級功能詳細分析

## 1. 使用量分析 (Usage Analytics)

### 功能概述

LiteLLM 提供了全面的使用量分析功能，支援多維度的花費追蹤與分析。

### API 端點

#### 1.1 鍵值花費分析

**端點**: `GET /spend/keys`

**Request 範例**:
```bash
curl -X GET "http://0.0.0.0:4000/spend/keys" \
  -H "Authorization: Bearer sk-1234" \
  -H "Content-Type: application/json"
```

**Response 範例**:
```json
[
  {
    "token_id": "abc123...",
    "token": "sk-***",
    "spend": 125.50,
    "max_budget": 500.00,
    "user_id": "user-123",
    "team_id": "team-456",
    "created_at": "2024-01-19T10:00:00Z"
  }
]
```

#### 1.2 用戶花費分析

**端點**: `GET /spend/users`

**Request 範例**:
```bash
curl -X GET "http://0.0.0.0:4000/spend/users" \
  -H "Authorization: Bearer sk-1234"

# 查看特定用戶
curl -X GET "http://0.0.0.0:4000/spend/users?user_id=user-uuid-123" \
  -H "Authorization: Bearer sk-1234"
```

**Response 範例**:
```json
[
  {
    "user_id": "user-uuid-123",
    "spend": 150.00,
    "max_budget": 500.00,
    "models": ["gpt-4"],
    "requests": 200
  }
]
```

#### 1.3 花費標籤分析

**端點**: `GET /spend/tags`

**Request 範例**:
```bash
curl -X GET "http://0.0.0.0:4000/spend/tags?start_date=2024-01-01&end_date=2024-01-31" \
  -H "Authorization: Bearer sk-1234"
```

**Response 範例**:
```json
[
  {
    "request_tag": "project-alpha",
    "log_count": 500,
    "total_spend": 250.00
  },
  {
    "request_tag": "project-beta",
    "log_count": 300,
    "total_spend": 150.00
  }
]
```

#### 1.4 全域活動分析

**端點**: `GET /global/activity`

**Request 範例**:
```bash
curl -X GET "http://0.0.0.0:4000/global/activity?start_date=2024-01-01&end_date=2024-01-31" \
  -H "Authorization: Bearer sk-1234"
```

**Response 範例**:
```json
{
  "daily_data": [
    {
      "date": "Jan 19",
      "api_requests": 10,
      "total_tokens": 2000
    }
  ],
  "sum_api_requests": 20,
  "sum_total_tokens": 2012
}
```

#### 1.5 全域活動按模型分類

**端點**: `GET /global/activity/model`

**Request 範例**:
```bash
curl -X GET "http://0.0.0.0:4000/global/activity/model?start_date=2024-01-01&end_date=2024-01-31" \
  -H "Authorization: Bearer sk-1234"
```

**Response 範例**:
```json
[
  {
    "model": "gpt-4",
    "daily_data": [
      {
        "date": "Jan 22",
        "api_requests": 10,
        "total_tokens": 2000
      }
    ],
    "sum_api_requests": 20,
    "sum_total_tokens": 2012
  }
]
```

#### 1.6 花費報告

**端點**: `GET /global/spend/report`

**Request 範例**:
```bash
curl -X GET "http://0.0.0.0:4000/global/spend/report?start_date=2024-01-01&end_date=2024-01-31&group_by=team" \
  -H "Authorization: Bearer sk-1234"
```

**Response 範例**:
```json
[
  {
    "group-by-day": "2024-05-10",
    "teams": [
      {
        "team_name": "team-1",
        "spend": 10,
        "keys": [
          {
            "key": "1213",
            "usage": {
              "model-1": {
                "cost": 12.50,
                "input_tokens": 1000,
                "output_tokens": 5000,
                "requests": 100
              }
            }
          }
        ]
      }
    ]
  }
]
```

#### 1.7 按提供者分類的花費

**端點**: `GET /global/spend/provider`

**Request 範例**:
```bash
curl -X GET "http://0.0.0.0:4000/global/spend/provider?start_date=2024-01-01&end_date=2024-01-31" \
  -H "Authorization: Bearer sk-1234"
```

**Response 範例**:
```json
[
  {
    "provider": "Azure OpenAI",
    "spend": 20
  },
  {
    "provider": "OpenAI",
    "spend": 10
  }
]
```

#### 1.8 計算花費

**端點**: `POST /spend/calculate`

**Request 範例**:
```bash
curl --location 'http://0.0.0.0:4000/spend/calculate' \
  --header 'Authorization: Bearer sk-1234' \
  --header 'Content-Type: application/json' \
  --data '{
    "model": "anthropic.claude-v2",
    "messages": [{"role": "user", "content": "Hey, how'\''s it going?"}]
  }'
```

**Response 範例**:
```json
{
  "cost": 0.0
}
```

### 花費分析流程圖

```mermaid
flowchart TD
    A[API 請求] --> B{認證檢查}
    B -->|通過| C[解析參數]
    B -->|失敗| D[返回 401 錯誤]
    
    C --> E[建立資料庫查詢]
    E --> F[查詢每日花費表]
    F --> G[聚合數據]
    
    G --> H[計算維度分析]
    H --> I[按模型分組]
    H --> J[按 API Key 分組]
    H --> K[按團隊分組]
    
    I --> L[組裝 Response]
    J --> L
    K --> L
    
    L --> M[返回 JSON 響應]
```

### 花費追蹤資料流程

```mermaid
sequenceDiagram
    participant Client as 用戶端
    participant Proxy as LiteLLM Proxy
    participant DB as PostgreSQL
    participant Redis as Redis Cache
    participant Scheduler as 背景工作

    Client->>Proxy: POST /chat/completions
    Proxy->>Proxy: 驗證 API Key
    
    alt 請求成功
        Proxy->>LLM: 轉發請求
        LLM-->>Proxy: 返回響應
        
        Proxy->>CostCalc: 計算花費
        CostCalc-->>Proxy: 金額 + Token 數
        
        Proxy->>Redis: 佇列花費更新
        Redis-->>Proxy: 確認
        
        Proxy->>Client: 返回響應
        
        Note over Scheduler: 每 60 秒
        Scheduler->>Redis: 獲取待處理花費
        Redis-->>Scheduler: 花費佇列
        
        Scheduler->>DB: 批次寫入 SpendLogs
        Scheduler->>DB: 更新每日花費表
        DB-->>Scheduler: 確認
    else 請求失敗
        Proxy->>Client: 返回錯誤
    end
```

---

### 標籤管理 API

LiteLLM 提供了標籤相關的分析端點，用於追蹤用戶代理和自定義標籤的使用情況。

#### 1.10 每日活躍用戶 (DAU)

**端點**: `GET /tag/dau`

**Request 範例**:
```bash
curl -X GET "http://0.0.0.0:4000/tag/dau" \
  -H "Authorization: Bearer sk-1234"
```

**Response 範例**:
```json
{
  "results": [
    {
      "tag": "User-Agent:MyApp/1.0",
      "active_users": 50,
      "date": "2024-01-19"
    }
  ]
}
```

#### 1.11 每週活躍用戶 (WAU)

**端點**: `GET /tag/wau`

**Request 範例**:
```bash
curl -X GET "http://0.0.0.0:4000/tag/wau" \
  -H "Authorization: Bearer sk-1234"
```

**Response 範例**:
```json
{
  "results": [
    {
      "tag": "User-Agent:MyApp/1.0",
      "active_users": 200,
      "date": "Week 7 (Jan 13)",
      "period_start": "2024-01-07",
      "period_end": "2024-01-13"
    }
  ]
}
```

#### 1.12 每月活躍用戶 (MAU)

**端點**: `GET /tag/mau`

**Request 範例**:
```bash
curl -X GET "http://0.0.0.0:4000/tag/mau" \
  -H "Authorization: Bearer sk-1234"
```

**Response 範例**:
```json
{
  "results": [
    {
      "tag": "User-Agent:MyApp/1.0",
      "active_users": 500,
      "date": "Month 7 (Dec)",
      "period_start": "2023-12-05",
      "period_end": "2024-01-03"
    }
  ]
}
```

#### 1.13 標籤摘要

**端點**: `GET /tag/summary`

**Request 範例**:
```bash
curl -X GET "http://0.0.0.0:4000/tag/summary?start_date=2024-01-01&end_date=2024-01-31" \
  -H "Authorization: Bearer sk-1234"
```

**Response 範例**:
```json
{
  "results": [
    {
      "tag": "User-Agent:MyApp/1.0",
      "unique_users": 500,
      "total_requests": 10000,
      "successful_requests": 9800,
      "failed_requests": 200,
      "total_tokens": 5000000,
      "total_spend": 250.00
    }
  ]
}
```

#### 1.14 標籤列表

**端點**: `GET /tag/distinct`

**Request 範例**:
```bash
curl -X GET "http://0.0.0.0:4000/tag/distinct" \
  -H "Authorization: Bearer sk-1234"
```

**Response 範例**:
```json
{
  "results": [
    {
      "tag": "User-Agent:MyApp/1.0"
    },
    {
      "tag": "User-Agent:OtherApp/2.0"
    }
  ]
}
```

#### 1.15 每用戶分析

**端點**: `GET /tag/user-agent/per-user-analytics`

**Request 範例**:
```bash
curl -X GET "http://0.0.0.0:4000/tag/user-agent/per-user-analytics?page=1&page_size=50" \
  -H "Authorization: Bearer sk-1234"
```

**Response 範例**:
```json
{
  "results": [
    {
      "user_id": "user-uuid-123",
      "user_email": "user@example.com",
      "user_agent": "User-Agent:MyApp/1.0",
      "successful_requests": 100,
      "failed_requests": 5,
      "total_requests": 105,
      "total_tokens": 50000,
      "spend": 25.00
    }
  ],
  "total_count": 100,
  "page": 1,
  "page_size": 50,
  "total_pages": 2
}
```

---

## 2. 角色權限控制 (RBAC)

### 功能概述

LiteLLM 實作了基於角色的存取控制 (RBAC)，支援多層級的權限管理。

### 角色定義

```python
class LitellmUserRoles(str, enum.Enum):
    """
    管理員角色:
    PROXY_ADMIN: 平台管理員，拥有所有權限
    PROXY_ADMIN_VIEW_ONLY: 僅能查看的管理員
    
    組織管理員角色:
    ORG_ADMIN: 特定組織的管理員，只能在其組織內創建團隊和用戶
    
    內部用戶角色:
    INTERNAL_USER: 可登入，查看/創建/刪除自己的金鑰，查看自己的花費
    INTERNAL_USER_VIEW_ONLY: 可登入，查看自己的金鑰和花費
    
    團隊角色:
    TEAM: 用於 JWT 認證
    
    客戶角色:
    CUSTOMER: 外部用戶
    """
    
    # 管理員角色
    PROXY_ADMIN = "proxy_admin"
    PROXY_ADMIN_VIEW_ONLY = "proxy_admin_viewer"
    
    # 組織管理員
    ORG_ADMIN = "org_admin"
    
    # 內部用戶
    INTERNAL_USER = "internal_user"
    INTERNAL_USER_VIEW_ONLY = "internal_user_viewer"
    
    # 團隊
    TEAM = "team"
    
    # 客戶
    CUSTOMER = "customer"
```

### 團隊角色

```python
class LiteLLMTeamRoles(enum.Enum):
    TEAM_ADMIN = "admin"   # 團隊管理員
    TEAM_MEMBER = "user"   # 團隊成員
```

### 角色權限對照表

| 角色 | 查看所有金鑰 | 查看所有花費 | 創建用戶 | 創建團隊 | 管理模型 |
|------|-------------|-------------|---------|---------|---------|
| PROXY_ADMIN | ✅ | ✅ | ✅ | ✅ | ✅ |
| PROXY_ADMIN_VIEW_ONLY | ✅ | ✅ | ❌ | ❌ | ❌ |
| ORG_ADMIN | 組織內 | 組織內 | 組織內 | 組織內 | 組織內 |
| INTERNAL_USER | 自己的 | 自己的 | ❌ | ❌ | ❌ |
| INTERNAL_USER_VIEW_ONLY | 自己的 | 自己的 | ❌ | ❌ | ❌ |

### API 端點

#### 2.1 獲取當前用戶資訊

**端點**: `GET /user/info`

**Request 範例**:
```bash
curl -X GET "http://0.0.0.0:8000/user/info" \
  -H "Authorization: Bearer sk-1234"
```

**Response 範例**:
```json
{
  "user_id": "user-uuid-123",
  "user_alias": "John Doe",
  "user_email": "john@example.com",
  "user_role": "internal_user",
  "max_budget": 500.00,
  "spend": 150.00,
  "teams": ["team-uuid-456"],
  "models": ["gpt-4", "claude-3-sonnet"],
  "metadata": {
    "department": "Engineering"
  }
}
```

#### 2.2 更新用戶角色

**端點**: `PUT /user/update`

**Request 範例**:
```bash
curl -X PUT "http://0.0.0.0:8000/user/update" \
  -H "Authorization: Bearer sk-1234" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "user-uuid-123",
    "user_role": "internal_user_viewer"
  }'
```

**Response 範例**:
```json
{
  "user_id": "user-uuid-123",
  "user_role": "internal_user_viewer",
  "updated_at": "2024-01-19T10:30:00Z"
}
```

### RBAC 流程圖

```mermaid
flowchart TD
    A[API 請求] --> B[提取 Authorization Header]
    B --> C[解析 API Key]
    C --> D[查詢資料庫]
    D --> E{Key 是否有效?}
    E -->|否| F[返回 401 Unauthorized]
    E -->|是| G[獲取用戶角色]
    
    G --> H{角色是否允許此操作?}
    H -->|否| I[返回 403 Forbidden]
    H -->|是| J[執行操作]
    
    J --> K[記錄審計日誌]
    K --> L[返回結果]
```

### 權限檢查流程

```mermaid
sequenceDiagram
    participant User as 用戶
    participant API as API 端點
    participant Auth as 認證模組
    participant DB as 資料庫

    User->>API: 請求 /team/new
    API->>Auth: 檢查權限
    Auth->>DB: 查詢用戶角色
    
    alt PROXY_ADMIN
        DB-->>Auth: user_role = "proxy_admin"
        Auth-->>API: 允許訪問
    else ORG_ADMIN
        DB-->>Auth: user_role = "org_admin"
        Auth->>DB: 檢查組織歸屬
        DB-->>Auth: organization_id 匹配
        Auth-->>API: 允許訪問
    else INTERNAL_USER
        DB-->>Auth: user_role = "internal_user"
        Auth-->>API: 拒絕訪問 (403)
    end
    
    API-->>User: 返回結果
```

---

## 3. 帳號權限系統 (多租戶)

### 功能概述

LiteLLM 支援多租戶架構，組織 (Organization) 是最上層的租戶單位。

### 組織結構

```mermaid
graph TD
    subgraph 組織層級
        Org[組織<br/>LiteLLM_OrganizationTable]
    end
    
    subgraph 團隊層級
        Team1[團隊 A<br/>LiteLLM_TeamTable]
        Team2[團隊 B<br/>LiteLLM_TeamTable]
    end
    
    subgraph 用戶層級
        User1[用戶 1<br/>LiteLLM_UserTable]
        User2[用戶 2<br/>LiteLLM_UserTable]
        User3[用戶 3<br/>LiteLLM_UserTable]
    end
    
    subgraph API 金鑰層級
        Key1[API 金鑰<br/>LiteLLM_VerificationToken]
        Key2[API 金鑰<br/>LiteLLM_VerificationToken]
    end
    
    Org --> Team1
    Org --> Team2
    Org --> User1
    
    Team1 --> User2
    Team1 --> User3
    
    Team1 --> Key1
    Team2 --> Key2
```

### 組織管理 API

#### 3.1 創建組織

**端點**: `POST /organization/new`

**Request 範例**:
```bash
curl -X POST "http://0.0.0.0:8000/organization/new" \
  -H "Authorization: Bearer sk-1234" \
  -H "Content-Type: application/json" \
  -d '{
    "organization_alias": "Acme Corp",
    "budget_id": "budget-uuid-123",
    "models": ["gpt-4", "claude-3-opus"],
    "metadata": {
      "industry": "Technology",
      "size": "Enterprise"
    }
  }'
```

**Response 範例**:
```json
{
  "organization_id": "org-uuid-456",
  "organization_alias": "Acme Corp",
  "budget_id": "budget-uuid-123",
  "models": ["gpt-4", "claude-3-opus"],
  "spend": 0.0,
  "created_at": "2024-01-19T10:00:00Z"
}
```

#### 3.2 查看組織資訊

**端點**: `GET /organization/info`

**Request 範例**:
```bash
curl -X GET "http://0.0.0.0:8000/organization/info?organization_id=org-uuid-456" \
  -H "Authorization: Bearer sk-1234"
```

**Response 範例**:
```json
{
  "organization_id": "org-uuid-456",
  "organization_alias": "Acme Corp",
  "budget_id": "budget-uuid-123",
  "max_budget": 10000.00,
  "spend": 2500.00,
  "models": ["gpt-4", "claude-3-opus"],
  "teams": ["team-uuid-1", "team-uuid-2"],
  "members": [
    {
      "user_id": "user-uuid-1",
      "user_role": "org_admin"
    }
  ]
}
```

#### 3.3 組織成員管理

**端點**: `POST /organization/member_add`

**Request 範例**:
```bash
curl -X POST "http://0.0.0.0:4000/organization/member_add" \
  -H "Authorization: Bearer sk-1234" \
  -H "Content-Type: application/json" \
  -d '{
    "organization_id": "org-uuid-456",
    "member": {
      "user_id": "user-uuid-789",
      "role": "internal_user"
    }
  }'
```

#### 3.4 組織列表

**端點**: `GET /organization/list`

**Request 範例**:
```bash
curl -X GET "http://0.0.0.0:4000/organization/list" \
  -H "Authorization: Bearer sk-1234"

# 依名稱過濾
curl -X GET "http://0.0.0.0:4000/organization/list?org_alias=my-org" \
  -H "Authorization: Bearer sk-1234"
```

**Response 範例**:
```json
[
  {
    "organization_id": "org-uuid-456",
    "organization_alias": "Acme Corp",
    "max_budget": 10000.00,
    "spend": 2500.00,
    "models": ["gpt-4", "claude-3-opus"]
  }
]
```

#### 3.5 更新組織

**端點**: `PATCH /organization/update`

**Request 範例**:
```bash
curl -X PATCH "http://0.0.0.0:4000/organization/update" \
  -H "Authorization: Bearer sk-1234" \
  -H "Content-Type: application/json" \
  -d '{
    "organization_id": "org-uuid-456",
    "organization_alias": "Updated Corp Name",
    "max_budget": 20000.00
  }'
```

#### 3.6 刪除組織

**端點**: `DELETE /organization/delete`

**Request 範例**:
```bash
curl -X DELETE "http://0.0.0.0:4000/organization/delete" \
  -H "Authorization: Bearer sk-1234" \
  -H "Content-Type: application/json" \
  -d '{
    "organization_ids": ["org-uuid-456"]
  }'
```

#### 3.7 組織成員更新

**端點**: `PATCH /organization/member_update`

**Request 範例**:
```bash
curl -X PATCH "http://0.0.0.0:4000/organization/member_update" \
  -H "Authorization: Bearer sk-1234" \
  -H "Content-Type: application/json" \
  -d '{
    "organization_id": "org-uuid-456",
    "user_id": "user-uuid-789",
    "role": "org_admin"
  }'
```

#### 3.8 組織成員刪除

**端點**: `DELETE /organization/member_delete`

**Request 範例**:
```bash
curl -X DELETE "http://0.0.0.0:4000/organization/member_delete" \
  -H "Authorization: Bearer sk-1234" \
  -H "Content-Type: application/json" \
  -d '{
    "organization_id": "org-uuid-456",
    "user_id": "user-uuid-789"
  }'
```

#### 3.9 組織每日活動

**端點**: `GET /organization/daily/activity`

**Request 範例**:
```bash
curl -X GET "http://0.0.0.0:4000/organization/daily/activity?organization_ids=org-uuid-456&start_date=2024-01-01&end_date=2024-01-31" \
  -H "Authorization: Bearer sk-1234"
```

### 團隊管理 API

#### 3.4 創建團隊

**端點**: `POST /team/new`

**Request 範例**:
```bash
curl -X POST "http://0.0.0.0:8000/team/new" \
  -H "Authorization: Bearer sk-1234" \
  -H "Content-Type: application/json" \
  -d '{
    "team_alias": "AI Research Team",
    "organization_id": "org-uuid-456",
    "models": ["gpt-4", "claude-3-opus"],
    "max_budget": 5000.00,
    "members_with_roles": [
      {
        "user_id": "user-uuid-123",
        "role": "admin"
      },
      {
        "user_id": "user-uuid-456",
        "role": "user"
      }
    ]
  }'
```

**Response 範例**:
```json
{
  "team_id": "team-uuid-789",
  "team_alias": "AI Research Team",
  "organization_id": "org-uuid-456",
  "admins": ["user-uuid-123"],
  "members": ["user-uuid-123", "user-uuid-456"],
  "max_budget": 5000.00,
  "spend": 0.0,
  "created_at": "2024-01-19T10:00:00Z"
}
```

### 多租戶隔離流程

```mermaid
flowchart TD
    A[API 請求] --> B[認證]
    B --> C[獲取用戶組織 ID]
    C --> D[獲取請求的資源]
    
    D --> E{資源是否屬於用戶組織?}
    E -->|否| F[返回 403]
    E -->|是| G[執行操作]
    
    G --> H[記錄操作日誌]
    H --> I[返回結果]
```

---

## 4. 帳號創建系統

### 功能概述

LiteLLM 提供了完整的帳號創建和管理系統，支援自動建立 API 金鑰。

### 用戶創建 API

#### 4.1 創建新用戶

**端點**: `POST /user/new`

**Request 範例**:
```bash
curl -X POST "http://0.0.0.0:8000/user/new" \
  -H "Authorization: Bearer sk-1234" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "user-uuid-123",
    "user_email": "john@example.com",
    "user_alias": "John Doe",
    "user_role": "internal_user",
    "max_budget": 500.00,
    "models": ["gpt-4"],
    "auto_create_key": true,
    "teams": ["team-uuid-456"]
  }'
```

**Request 參數說明**:
| 參數 | 類型 | 必填 | 說明 |
|------|------|------|------|
| user_id | string | 否 | 用戶 ID，未提供則自動生成 |
| user_email | string | 否 | 用戶郵箱 (唯一) |
| user_alias | string | 否 | 用戶顯示名稱 |
| user_role | string | 否 | 角色 (預設: internal_user) |
| max_budget | float | 否 | 最大預算 |
| models | array | 否 | 允許使用的模型列表 |
| auto_create_key | boolean | 否 | 是否自動創建 API 金鑰 (預設: true) |
| teams | array | 否 | 所屬團隊 ID 列表 |

**Response 範例**:
```json
{
  "user_id": "user-uuid-123",
  "user_email": "john@example.com",
  "user_alias": "John Doe",
  "user_role": "internal_user",
  "max_budget": 500.00,
  "spend": 0.0,
  "models": ["gpt-4"],
  "keys": [
    {
      "key": "sk-1234567890abcdef...",
      "key_name": "Auto-generated key for John Doe",
      "expires": null,
      "models": ["gpt-4"]
    }
  ],
  "created_at": "2024-01-19T10:00:00Z"
}
```

#### 4.2 批量創建用戶

**端點**: `POST /user/new_bulk`

**Request 範例**:
```bash
curl -X POST "http://0.0.0.0:8000/user/new_bulk" \
  -H "Authorization: Bearer sk-1234" \
  -H "Content-Type: application/json" \
  -d '{
    "users": [
      {
        "user_email": "alice@example.com",
        "user_alias": "Alice",
        "user_role": "internal_user"
      },
      {
        "user_email": "bob@example.com",
        "user_alias": "Bob",
        "user_role": "internal_user_viewer"
      }
    ],
    "auto_create_keys": true
  }'
```

### 邀請系統

#### 4.3 創建邀請連結

**端點**: `POST /invitation/create`

**Request 範例**:
```bash
curl -X POST "http://0.0.0.0:8000/invitation/create" \
  -H "Authorization: Bearer sk-1234" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "admin-user-id",
    "expires_at": "2024-02-19T10:00:00Z"
  }'
```

**Response 範例**:
```json
{
  "invitation_id": "inv-uuid-123",
  "invitation_url": "http://0.0.0.0:8000/invitation/claim/inv-uuid-123",
  "expires_at": "2024-02-19T10:00:00Z",
  "created_by": "admin-user-id",
  "is_accepted": false
}
```

#### 4.4 領取邀請

**端點**: `POST /invitation/claim/{invitation_id}`

**Request 範例**:
```bash
curl -X POST "http://0.0.0.0:8000/invitation/claim/inv-uuid-123" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "new-user-id",
    "user_email": "newuser@example.com"
  }'
```

### 用戶創建流程圖

```mermaid
flowchart TD
    A[管理員請求創建用戶] --> B[驗證管理員權限]
    B --> C[檢查郵箱唯一性]
    C --> D{郵箱已存在?}
    D -->|是| E[返回 409 錯誤]
    D -->|否| F[生成用戶 ID]
    
    F --> G{auto_create_key?}
    G -->|是| H[生成 API 金鑰]
    G -->|否| I[跳過金鑰創建]
    
    H --> J[創建用戶記錄]
    I --> J
    
    J --> K[建立團隊關聯]
    K --> L[更新組織成員]
    L --> M[返回創建結果]
```

---

## 5. 授權系統 (業務提供販售方式)

### 功能概述

LiteLLM 提供了靈活的授權管理機制，支援透過 API 金鑰進行業務授權和販售。

### API 金鑰管理

#### 5.1 生成 API 金鑰

**端點**: `POST /key/generate`

**Request 範例**:
```bash
curl -X POST "http://0.0.0.0:8000/key/generate" \
  -H "Authorization: Bearer sk-1234" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "user-uuid-123",
    "team_id": "team-uuid-456",
    "models": ["gpt-4", "claude-3-sonnet"],
    "duration": "30d",
    "max_budget": 1000.00,
    "key_alias": "Production API Key",
    "rpm_limit": 1000,
    "tpm_limit": 1000000
  }'
```

**Request 參數說明**:
| 參數 | 類型 | 必填 | 說明 |
|------|------|------|------|
| user_id | string | 否 | 關聯用戶 ID |
| team_id | string | 否 | 關聯團隊 ID |
| models | array | 否 | 允許的模型列表 |
| duration | string | 否 | 金鑰有效期 (如 "30d", "90d") |
| max_budget | float | 否 | 最大花費限制 |
| key_alias | string | 否 | 金鑰別名 |
| rpm_limit | int | 否 | 每分鐘請求限制 |
| tpm_limit | int | 否 | 每分鐘 Token 限制 |
| allowed_routes | array | 否 | 允許的 API 路由 |

**Response 範例**:
```json
{
  "key": "sk-1234567890abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ",
  "key_name": "sk-1234...ABCD",
  "key_alias": "Production API Key",
  "expires": "2024-02-19T10:00:00Z",
  "max_budget": 1000.00,
  "budget_duration": "30d",
  "models": ["gpt-4", "claude-3-sonnet"],
  "rpm_limit": 1000,
  "tpm_limit": 1000000,
  "spend": 0.0,
  "user_id": "user-uuid-123",
  "team_id": "team-uuid-456"
}
```

#### 5.2 服務帳號金鑰

**端點**: `POST /key/service-account/generate`

**Request 範例**:
```bash
curl -X POST "http://0.0.0.0:4000/key/service-account/generate" \
  -H "Authorization: Bearer sk-1234" \
  -H "Content-Type: application/json" \
  -d '{
    "team_id": "team-uuid-456",
    "key_alias": "Service Account Key",
    "duration": "365d"
  }'
```

**特點**:
- 不屬於任何用戶，屬於團隊
- 防止用戶刪除時金鑰被刪除
- 應用團隊限制，而非團隊成員限制

#### 5.3 更新 API 金鑰

**端點**: `POST /key/update`

**Request 範例**:
```bash
curl -X POST "http://0.0.0.0:4000/key/update" \
  -H "Authorization: Bearer sk-1234" \
  -H "Content-Type: application/json" \
  -d '{
    "keys": ["sk-1234...ABCD"],
    "max_budget": 2000.00,
    "models": ["gpt-4", "claude-3-opus", "gpt-3.5-turbo"]
  }'
```

#### 5.4 刪除 API 金鑰

**端點**: `POST /key/delete`

**Request 範例**:
```bash
curl -X POST "http://0.0.0.0:4000/key/delete" \
  -H "Authorization: Bearer sk-1234" \
  -H "Content-Type: application/json" \
  -d '{
    "keys": ["sk-1234...ABCD"]
  }'
```

### 金鑰自動輪換

**Request 範例**:
```bash
curl -X POST "http://0.0.0.0:8000/key/generate" \
  -H "Authorization: Bearer sk-1234" \
  -H "Content-Type: application/json" \
  -d '{
    "models": ["gpt-4"],
    "auto_rotate": true,
    "rotation_interval": "90d"
  }'
```

### 授權流程圖

```mermaid
flowchart TD
    A[業務系統] --> B[生成訂單]
    B --> C[創建客戶帳號]
    C --> D[生成 API 金鑰]
    
    D --> E[設置預算限制]
    E --> F[設置模型限制]
    F --> G[設置有效期限]
    
    G --> H[返回金鑰給客戶]
    H --> I[客戶使用金鑰調用 API]
    
    I --> J{驗證金鑰}
    J -->|失敗| K[返回 401]
    J -->|成功| L{檢查預算}
    
    L -->|預算超限| M[返回 402 Payment Required]
    L -->|正常| N{檢查速率限制}
    
    N -->|超限| O[返回 429 Too Many Requests]
    N -->|正常| P[執行請求]
    
    P --> Q[記錄花費]
    Q --> R[更新剩餘預算]
```

### 業務販售流程

```mermaid
sequenceDiagram
    participant Customer as 客戶
    participant Sales as 銷售系統
    participant LiteLLM as LiteLLM Proxy
    participant DB as 資料庫

    Customer->>Sales: 購買訂閱
    Sales->>Sales: 處理付款
    
    Sales->>LiteLLM: 創建客戶帳號
    LiteLLM->>DB: 創建用戶記錄
    
    Sales->>LiteLLM: 生成 API 金鑰 (設定預算)
    LiteLLM->>DB: 生成金鑰並設置限制
    
    LiteLLM-->>Sales: 返回 API 金鑰
    Sales-->>Customer: 提供 API 金鑰
    
    loop 使用期間
        Customer->>LiteLLM: API 請求
        LiteLLM->>DB: 檢查預算
        DB-->>LiteLLM: 剩餘預算
        
        LiteLLM->>DB: 記錄花費
        DB-->>LiteLLM: 更新預算
        
        LiteLLM-->>Customer: API 響應
    end
    
    alt 預算不足
        LiteLLM->>Sales: 觸發續費通知
        Sales->>Customer: 續費提醒
    end
```

---

## 6. 帳號與授權資料庫

### 資料庫 Schema

LiteLLM 使用 PostgreSQL 資料庫，透過 Prisma ORM 管理。

#### 6.1 用戶表 (LiteLLM_UserTable)

```prisma
model LiteLLM_UserTable {
  user_id              String   @id
  user_alias           String?
  team_id              String?
  sso_user_id          String?  @unique
  organization_id      String?
  object_permission_id String?
  password             String?
  user_role            String?
  max_budget           Float?
  spend                Float    @default(0.0)
  user_email           String?
  models               String[]
  metadata             Json     @default("{}")
  max_parallel_requests Int?
  tpm_limit            BigInt?
  rpm_limit            BigInt?
  budget_duration      String?
  budget_reset_at      DateTime?
  model_spend          Json     @default("{}")
  model_max_budget     Json     @default("{}")
  created_at           DateTime? @default(now())
  updated_at           DateTime? @updatedAt
  
  organization         LiteLLM_OrganizationTable? @relation(fields: [organization_id], references: [organization_id])
  organization_membershipes LiteLLM_OrganizationMembership[]
}
```

#### 6.2 團隊表 (LiteLLM_TeamTable)

```prisma
model LiteLLM_TeamTable {
  team_id              String   @id @default(uuid())
  team_alias           String?
  organization_id      String?
  object_permission_id String?
  admins               String[]
  members              String[]
  members_with_roles   Json     @default("{}")
  metadata             Json     @default("{}")
  max_budget           Float?
  spend                Float    @default(0.0)
  models               String[]
  max_parallel_requests Int?
  tpm_limit            BigInt?
  rpm_limit            BigInt?
  budget_duration      String?
  budget_reset_at      DateTime?
  blocked              Boolean  @default(false)
  model_spend          Json     @default("{}")
  model_max_budget     Json     @default("{}")
  router_settings      Json?    @default("{}")
  team_member_permissions String[] @default([])
  created_at           DateTime @default(now())
  updated_at           DateTime @updatedAt
  
  organization         LiteLLM_OrganizationTable? @relation(fields: [organization_id], references: [organization_id])
}
```

#### 6.3 組織表 (LiteLLM_OrganizationTable)

```prisma
model LiteLLM_OrganizationTable {
  organization_id      String   @id @default(uuid())
  organization_alias   String
  budget_id            String
  metadata             Json     @default("{}")
  models               String[]
  spend                Float    @default(0.0)
  model_spend          Json     @default("{}")
  object_permission_id String?
  created_at           DateTime @default(now())
  created_by           String
  updated_at           DateTime @updatedAt
  updated_by           String
  
  budget               LiteLLM_BudgetTable? @relation(fields: [budget_id], references: [budget_id])
  teams                LiteLLM_TeamTable[]
  users                LiteLLM_UserTable[]
  keys                 LiteLLM_VerificationToken[]
  members              LiteLLM_OrganizationMembership[]
  object_permission    LiteLLM_ObjectPermissionTable? @relation(fields: [object_permission_id], references: [object_permission_id])
}
```

#### 6.4 API 金鑰表 (LiteLLM_VerificationToken)

```prisma
model LiteLLM_VerificationToken {
  token                String   @id
  key_name             String?
  key_alias            String?
  soft_budget_cooldown Boolean  @default(false)
  spend                Float    @default(0.0)
  expires              DateTime?
  models               String[]
  aliases              Json     @default("{}")
  config               Json     @default("{}}")
  router_settings      Json?    @default("{}")
  user_id              String?
  team_id              String?
  permissions          Json     @default("{}")
  max_parallel_requests Int?
  metadata             Json     @default("{}")
  blocked              Boolean?
  tpm_limit            BigInt?
  rpm_limit            BigInt?
  max_budget           Float?
  budget_duration      String?
  budget_reset_at      DateTime?
  allowed_cache_controls String[] @default([])
  allowed_routes       String[] @default([])
  model_spend          Json     @default("{}")
  model_max_budget     Json     @default("{}")
  budget_id            String?
  organization_id      String?
  object_permission_id String?
  rotation_count       Int?     @default(0)
  auto_rotate          Boolean? @default(false)
  rotation_interval    String?
  last_rotation_at     DateTime?
  key_rotation_at      DateTime?
  created_at           DateTime? @default(now())
  created_by           String?
  updated_at           DateTime? @updatedAt
  
  budget               LiteLLM_BudgetTable? @relation(fields: [budget_id], references: [budget_id])
  organization         LiteLLM_OrganizationTable? @relation(fields: [organization_id], references: [organization_id])
}
```

#### 6.5 花費日誌表 (LiteLLM_SpendLogs)

```prisma
model LiteLLM_SpendLogs {
  request_id              String  @id
  call_type               String
  api_key                 String  @default("")
  spend                   Float   @default(0.0)
  total_tokens            Int     @default(0)
  prompt_tokens           Int     @default(0)
  completion_tokens       Int     @default(0)
  startTime               DateTime
  endTime                 DateTime
  completionStartTime     DateTime?
  model                   String  @default("")
  model_id                String?
  model_group             String?
  custom_llm_provider     String?
  api_base                String?
  user                    String?
  metadata                Json?
  cache_hit               String?
  cache_key               String?
  request_tags            Json?
  team_id                 String?
  organization_id         String?
  end_user                String?
  requester_ip_address    String?
  messages                Json?
  response                Json?
  session_id              String?
  status                  String?
  agent_id                String?
  
  @@index([startTime])
  @@index([end_user])
  @@index([session_id])
}
```

#### 6.6 每日花費表 (LiteLLM_DailyUserSpend)

```prisma
model LiteLLM_DailyUserSpend {
  id                      String   @id @default(uuid())
  user_id                 String?
  date                    String
  api_key                 String
  model                   String?
  model_group             String?
  custom_llm_provider     String?
  mcp_namespaced_tool_name String?
  endpoint                String?
  prompt_tokens           BigInt   @default(0)
  completion_tokens       BigInt   @default(0)
  cache_read_input_tokens BigInt   @default(0)
  cache_creation_input_tokens BigInt @default(0)
  spend                   Float    @default(0.0)
  api_requests           BigInt   @default(0)
  successful_requests    BigInt   @default(0)
  failed_requests        BigInt   @default(0)
  created_at              DateTime @default(now())
  updated_at              DateTime @updatedAt
  
  @@unique([user_id, date, api_key, model, custom_llm_provider, mcp_namespaced_tool_name, endpoint])
  @@index([date])
  @@index([user_id])
}
```

### 資料庫關聯圖

```mermaid
erDiagram
    LiteLLM_OrganizationTable ||--o{ LiteLLM_TeamTable : contains
    LiteLLM_OrganizationTable ||--o{ LiteLLM_UserTable : contains
    LiteLLM_OrganizationTable ||--o{ LiteLLM_VerificationToken : owns
    LiteLLM_OrganizationTable ||--|| LiteLLM_BudgetTable : has_budget
    LiteLLM_TeamTable ||--o{ LiteLLM_UserTable : has_members
    LiteLLM_TeamTable ||--o{ LiteLLM_VerificationToken : has_keys
    LiteLLM_UserTable ||--o{ LiteLLM_VerificationToken : creates
    LiteLLM_VerificationToken ||--|| LiteLLM_BudgetTable : limited_by
    LiteLLM_BudgetTable ||--o{ LiteLLM_TagTable : budgets
```

---

## 7. 資料庫軟體建置與部署

### 7.1 Docker Compose 部署

#### docker-compose.yml

```yaml
version: '3.8'

services:
  litellm:
    image: ghcr.io/berriai/litellm:main-stable
    ports:
      - "4000:4000"
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/litellm
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - LITELLM_MASTER_KEY=sk-admin-key
    depends_on:
      - db
      - redis
    volumes:
      - ./proxy_server_config.yaml:/app/proxy_server_config.yaml

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=litellm
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### 7.2 手動資料庫設定

#### 建立資料庫

```sql
-- 連線到 PostgreSQL
psql -U postgres

-- 建立資料庫
CREATE DATABASE litellm;

-- 連線到 litellm 資料庫
\c litellm

-- 執行 Prisma Migration
npx prisma migrate deploy
```

#### 環境變數

```bash
# 資料庫連線字串
export DATABASE_URL="postgresql://postgres:postgres@localhost:5432/litellm"

# Redis 連線
export REDIS_HOST="localhost"
export REDIS_PORT="6379"

# 管理員金鑰
export LITELLM_MASTER_KEY="sk-admin-key"
```

### 7.3 初始化資料庫指令

```bash
# 執行 Prisma Client 生成
npx prisma generate

# 執行 Migration
npx prisma migrate dev

# 檢視資料
npx prisma studio

# 種子資料
npx prisma db seed
```

### 7.4 資料庫備份與恢復

#### 備份

```bash
# 完整備份
pg_dump -h localhost -U postgres -d litellm > litellm_backup_$(date +%Y%m%d).sql

# 壓縮備份
pg_dump -h localhost -U postgres -d litellm | gzip > litellm_backup_$(date +%Y%m%d).sql.gz
```

#### 恢復

```bash
# 恢復資料
psql -h localhost -U postgres -d litellm < litellm_backup_20240119.sql

# 從壓縮檔恢復
zcat litellm_backup_20240119.sql.gz | psql -h localhost -U postgres -d litellm
```

### 7.5 健康檢查

```mermaid
flowchart TD
    A[健康檢查腳本] --> B[檢查 PostgreSQL]
    B --> C{PostgreSQL 可用?}
    C -->|否| D[記錄錯誤]
    C -->|是| E[檢查 Redis]
    
    E --> F{Redis 可用?}
    F -->|否| G[記錄警告]
    F -->|是| H[檢查連線數]
    
    H --> I{連線數正常?}
    I -->|否| J[發送警報]
    I -->|是| K[返回健康狀態]
    
    D --> L[生成報告]
    G --> L
    J --> L
    K --> L
```

---

## 8. Complete API Endpoint Reference

This section provides a comprehensive list of all API endpoints available in LiteLLM Proxy, verified against the live Swagger documentation at http://localhost:4000/docs.

### Summary Statistics
- **Total Endpoints**: 397
- **Categories Documented**: 10 major categories

---

### 8.1 User Management

| Endpoint | HTTP Methods | Description |
|----------|--------------|-------------|
| `/user/available_users` | GET | Available Enterprise Users |
| `/user/bulk_update` | POST | Bulk User Update |
| `/user/daily/activity` | GET | Get User Daily Activity |
| `/user/daily/activity/aggregated` | GET | Get User Daily Activity Aggregated |
| `/user/delete` | POST | Delete User |
| `/user/info` | GET | User Info |
| `/user/list` | GET | Get Users |
| `/user/new` | POST | New User |
| `/user/update` | POST | User Update |

### 8.2 Team Management

| Endpoint | HTTP Methods | Description |
|----------|--------------|-------------|
| `/team/available` | GET | List Available Teams |
| `/team/block` | POST | Block Team |
| `/team/bulk_member_add` | POST | Bulk Team Member Add |
| `/team/daily/activity` | GET | Get Team Daily Activity |
| `/team/delete` | POST | Delete Team |
| `/team/info` | GET | Team Info |
| `/team/list` | GET | List Team |
| `/team/member_add` | POST | Team Member Add |
| `/team/member_delete` | POST | Team Member Delete |
| `/team/member_update` | POST | Team Member Update |
| `/team/model/add` | POST | Team Model Add |
| `/team/model/delete` | POST | Team Model Delete |
| `/team/new` | POST | New Team |
| `/team/permissions_list` | GET | Team Member Permissions |
| `/team/permissions_update` | POST | Update Team Member Permissions |
| `/team/unblock` | POST | Unblock Team |
| `/team/update` | POST | Update Team |
| `/team/{team_id}/callback` | POST, GET | Add Team Callbacks |
| `/team/{team_id}/disable_logging` | POST | Disable Team Logging |

### 8.3 API Key Management

| Endpoint | HTTP Methods | Description |
|----------|--------------|-------------|
| `/key/aliases` | GET | Key Aliases |
| `/key/block` | POST | Block Key |
| `/key/delete` | POST | Delete Key Fn |
| `/key/generate` | POST | Generate Key Fn |
| `/key/health` | POST | Key Health |
| `/key/info` | GET | Info Key Fn |
| `/key/list` | GET | List Keys |
| `/key/regenerate` | POST | Regenerate Key Fn |
| `/key/service-account/generate` | POST | Generate Service Account Key Fn |
| `/key/unblock` | POST | Unblock Key |
| `/key/update` | POST | Update Key Fn |
| `/key/{key}/regenerate` | POST | Regenerate Key Fn |

### 8.4 Budget Management

| Endpoint | HTTP Methods | Description |
|----------|--------------|-------------|
| `/budget/delete` | POST | Delete Budget |
| `/budget/info` | POST | Info Budget |
| `/budget/list` | GET | List Budget |
| `/budget/new` | POST | New Budget |
| `/budget/settings` | GET | Budget Settings |
| `/budget/update` | POST | Update Budget |

### 8.5 Customer (End User) Management

| Endpoint | HTTP Methods | Description |
|----------|--------------|-------------|
| `/customer/block` | POST | Block User |
| `/customer/daily/activity` | GET | Get Customer Daily Activity |
| `/customer/delete` | POST | Delete End User |
| `/customer/info` | GET | End User Info |
| `/customer/list` | GET | List End User |
| `/customer/new` | POST | New End User |
| `/customer/unblock` | POST | Unblock User |
| `/customer/update` | POST | Update End User |

### 8.6 Tag Management

| Endpoint | HTTP Methods | Description |
|----------|--------------|-------------|
| `/tag/daily/activity` | GET | Get Tag Daily Activity |
| `/tag/dau` | GET | Get Daily Active Users |
| `/tag/delete` | POST | Delete Tag |
| `/tag/distinct` | GET | Get Distinct User Agent Tags |
| `/tag/info` | POST | Info Tag |
| `/tag/list` | GET | List Tags |
| `/tag/mau` | GET | Get Monthly Active Users |
| `/tag/new` | POST | New Tag |
| `/tag/summary` | GET | Get Tag Summary |
| `/tag/update` | POST | Update Tag |
| `/tag/user-agent/per-user-analytics` | GET | Get Per User Analytics |
| `/tag/wau` | GET | Get Weekly Active Users |

### 8.7 Guardrails Management

| Endpoint | HTTP Methods | Description |
|----------|--------------|-------------|
| `/guardrails` | POST | Create Guardrail |
| `/guardrails/apply_guardrail` | POST | Apply Guardrail |
| `/guardrails/list` | GET | List Guardrails |
| `/guardrails/ui/add_guardrail_settings` | GET | Get Guardrail Ui Settings |
| `/guardrails/ui/category_yaml/{category_name}` | GET | Get Category Yaml |
| `/guardrails/ui/provider_specific_params` | GET | Get Provider Specific Params |
| `/guardrails/validate_blocked_words_file` | POST | Validate Blocked Words File |
| `/guardrails/{guardrail_id}` | PUT, DELETE, PATCH, GET | Update Guardrail |
| `/guardrails/{guardrail_id}/info` | GET | Get Guardrail Info |

### 8.8 Cache Management

| Endpoint | HTTP Methods | Description |
|----------|--------------|-------------|
| `/cache/delete` | POST | Cache Delete |
| `/cache/flushall` | POST | Cache Flushall |
| `/cache/ping` | GET | Cache Ping |
| `/cache/redis/info` | GET | Cache Redis Info |
| `/cache/settings` | GET, POST | Get Cache Settings |
| `/cache/settings/test` | POST | Test Cache Connection |

### 8.9 Health Check

| Endpoint | HTTP Methods | Description |
|----------|--------------|-------------|
| `/health` | GET | Health Endpoint |
| `/health/history` | GET | Health Check History Endpoint |
| `/health/latest` | GET | Latest Health Checks Endpoint |
| `/health/license` | GET | Health License Endpoint |
| `/health/liveliness` | GET, OPTIONS | Health Liveliness |
| `/health/liveness` | GET, OPTIONS | Health Liveliness |
| `/health/readiness` | GET, OPTIONS | Health Readiness |
| `/health/services` | GET | Health Services Endpoint |
| `/health/shared-status` | GET | Shared Health Check Status Endpoint |
| `/health/test_connection` | POST | Test Model Connection |

### 8.10 Model Management

| Endpoint | HTTP Methods | Description |
|----------|--------------|-------------|
| `/model/delete` | POST | Delete Model |
| `/model/info` | GET | Model Info V1 |
| `/model/new` | POST | Add New Model |
| `/model/update` | POST | Update Model |
| `/model/{model_id}/update` | PATCH | Patch Model |
| `/model_group/info` | GET | Model Group Info |
| `/model_group/make_public` | POST | Update Public Model Groups |
| `/models` | GET | Model List |
| `/models/{model_id}` | GET | Model Info |
| `/models/{model_name}:countTokens` | POST | Google Count Tokens |
| `/models/{model_name}:generateContent` | POST | Google Generate Content |
| `/models/{model_name}:streamGenerateContent` | POST | Google Stream Generate Content |

---

## 總結

LiteLLM 是一個成熟且功能完整的開源專案，為開發者和企業提供了：
1. 統一的 LLM 調用介面
2. 強大的 AI Gateway 功能
3. 廣泛的 LLM 提供者支援
4. 完善的監控與管理能力
5. 活躍的社群支援

適合用於：
- 需要統一管理多個 LLM 提供者的應用
- 建構內部 LLM API 網關
- 需要負載均衡和備援的生產環境
- 需要詳細使用追蹤和成本控制的場景
