# LiteLLM 功能技術文檔

本文檔描述 LiteLLM 主要功能的 API 請求/響應範例及流程圖。

## 目錄

1. [可觀測性與 FinOps](#1-可觀測性與-finops)
2. [存取控制](#2-存取控制)
3. [流量管理](#3-流量管理)
4. [安全護欄](#4-安全護欄)
5. [支援功能](#5-支援功能)
6. [企業級功能](#6-企業級功能)

---

## 1. 可觀測性與 FinOps

### 1.1 預算管理 (Budgeting)

預算管理功能允許管理員設定和管理不同層級的預算限制，包括團隊、組織、API 金鑰和最終用戶。

#### 主要檔案
- `litellm/budget_manager.py` - 預算管理器
- `litellm/proxy/management_endpoints/budget_management_endpoints.py` - 預算管理 API 端點

#### API 端點

**建立預算**
```
POST /budget/new
```

**請求範例:**
```json
{
  "budget_id": "budget-001",
  "max_budget": 100.0,
  "soft_budget": 80.0,
  "budget_duration": "30d",
  "tpm_limit": 100000,
  "rpm_limit": 10000,
  "max_parallel_requests": 100,
  "model_max_budget": {
    "openai/gpt-4o": {
      "max_budget": 50.0,
      "budget_duration": "7d"
    }
  }
}
```

**響應範例:**
```json
{
  "budget_id": "budget-001",
  "max_budget": 100.0,
  "soft_budget": 80.0,
  "budget_duration": "30d",
  "tpm_limit": 100000,
  "rpm_limit": 10000,
  "max_parallel_requests": 100,
  "model_max_budget": {
    "openai/gpt-4o": {
      "max_budget": 50.0,
      "budget_duration": "7d"
    }
  },
  "created_by": "admin",
  "updated_by": "admin"
}
```

**更新預算**
```
POST /budget/update
```

**請求範例:**
```json
{
  "budget_id": "budget-001",
  "max_budget": 150.0,
  "soft_budget": 120.0
}
```

**查詢預算資訊**
```
POST /budget/info
```

**請求範例:**
```json
{
  "budgets": ["budget-001", "budget-002"]
}
```

**列出所有預算**
```
GET /budget/list
```

**刪除預算**
```
POST /budget/delete
```

**請求範例:**
```json
{
  "id": "budget-001"
}
```

#### 預算管理流程圖

```mermaid
flowchart TD
    A[開始] --> B{管理員操作}
    
    B --> C[建立預算]
    B --> D[更新預算]
    B --> E[查詢預算]
    B --> F[刪除預算]
    
    C --> G[驗證請求參數]
    D --> G
    G --> H{參數有效?}
    
    H -->|否| I[返回錯誤]
    H -->|是| J{預算存在?}
    
    J -->|否| K[創建新預算]
    J -->|是| L[更新現有預算]
    
    K --> M[寫入資料庫]
    L --> M
    M --> N[返回成功響應]
    
    E --> O[從資料庫查詢]
    O --> P[返回預算資訊]
    
    F --> Q[從資料庫刪除]
    Q --> R[返回成功響應]
    
    I --> S[結束]
    N --> S
    P --> S
    R --> S
```

#### BudgetManager 類別

```python
class BudgetManager:
    def create_budget(
        self,
        total_budget: float,
        user: str,
        duration: Optional[Literal["daily", "weekly", "monthly", "yearly"]] = None,
        created_at: float = time.time(),
    ):
        """建立預算"""
        
    def update_cost(
        self,
        user: str,
        completion_obj: Optional[ModelResponse] = None,
        model: Optional[str] = None,
        input_text: Optional[str] = None,
        output_text: Optional[str] = None,
    ):
        """更新成本"""
        
    def get_current_cost(self, user):
        """取得當前成本"""
        
    def reset_cost(self, user):
        """重置成本"""
```

---

### 1.2 帳單 (Billing)

帳單功能追蹤和報告 API 使用成本。

#### 主要檔案
- `litellm/litellm_core_utils/llm_cost_calc/usage_object_transformation.py` - 使用量計算

#### API 使用量追蹤

**請求範例:**
```json
{
  "model": "gpt-4o",
  "messages": [{"role": "user", "content": "Hello!"}],
  "stream": false
}
```

**響應範例:**
```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1699999999,
  "model": "gpt-4o",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Hello! How can I help you?"
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 12,
    "completion_tokens": 10,
    "total_tokens": 22,
    "completion_tokens_details": {
      "audio_tokens": null,
      "reasoning_tokens": 0,
      "accepted_prediction_tokens": 0,
      "rejected_prediction_tokens": null
    },
    "prompt_tokens_details": {
      "audio_tokens": null,
      "cached_tokens": 0
    }
  }
}
```

#### 成本計算

```python
# 計算成本
prompt_cost, completion_cost = litellm.cost_per_token(
    model="gpt-4o",
    prompt_tokens=1000,
    completion_tokens=500
)

total_cost = prompt_cost + completion_cost
```

---

### 1.3 AI 模型使用監控 (Token Dashboard)

Token Dashboard 提供即時的使用量監控和分析。

#### 主要檔案
- `cookbook/litellm_proxy_server/cli_token_usage.py` - Token 使用量 CLI

#### 使用量監控 API

**查詢使用量**
```
GET /spend/logs
```

**響應範例:**
```json
{
  "spend_logs": [
    {
      "id": "log-001",
      "api_key": "sk-****1234",
      "model": "gpt-4o",
      "prompt_tokens": 1000,
      "completion_tokens": 500,
      "total_tokens": 1500,
      "spend": 0.03,
      "startTime": "2024-01-21T10:00:00Z",
      "endTime": "2024-01-21T10:00:01Z"
    }
  ],
  "total_spend": 0.03,
  "total_tokens": 1500
}
```

#### 使用量監控流程圖

```mermaid
flowchart LR
    A[API 請求] --> B[Router]
    B --> C[執行 LLM 調用]
    C --> D[計算 Token 使用量]
    D --> E[記錄到資料庫]
    E --> F[更新緩存]
    F --> G[返回響應]
    
    H[管理員] --> I[查詢使用量 API]
    I --> J[聚合數據]
    J --> K[返回使用量報告]
```

---

## 2. 存取控制

### 2.1 角色權限控制 (RBAC)

LiteLLM 支援基於角色的存取控制。

#### 主要檔案
- `litellm/proxy/auth/user_api_key_auth.py` - 使用者 API 金鑰認證
- `litellm/proxy/_types.py` - 類型定義

#### 使用者角色

```python
class LitellmUserRoles(str, enum.Enum):
    """
    Admin Roles:
    PROXY_ADMIN: admin over the platform
    PROXY_ADMIN_VIEW_ONLY: can login, view all own keys, view all spend
    ORG_ADMIN: admin over a specific organization

    Internal User Roles:
    INTERNAL_USER: can login, view/create/delete their own keys, view their spend
    INTERNAL_USER_VIEW_ONLY: can login, view their own keys, view their own spend

    Team Roles:
    TEAM: used for JWT auth

    Customer Roles:
    CUSTOMER: External users -> these are customers
    """

    # Admin Roles
    PROXY_ADMIN = "proxy_admin"
    PROXY_ADMIN_VIEW_ONLY = "proxy_admin_viewer"

    # Organization admins
    ORG_ADMIN = "org_admin"

    # Internal User Roles
    INTERNAL_USER = "internal_user"
    INTERNAL_USER_VIEW_ONLY = "internal_user_viewer"

    # Team Roles
    TEAM = "team"

    # Customer Roles - External users of proxy
    CUSTOMER = "customer"
```

#### API 金鑰認證流程

```mermaid
sequenceDiagram
    participant Client
    participant Proxy
    participant Auth
    participant DB
    
    Client->>Proxy: 攜帶 API Key 的請求
    Proxy->>Auth: 驗證 API Key
    Auth->>DB: 查詢金鑰資訊
    DB-->>Auth: 返回金鑰權限
    Auth-->>Proxy: 返回 UserAPIKeyAuth 物件
    Proxy->>Proxy: 檢查角色權限
    Proxy->>Proxy: 執行請求
    Proxy-->>Client: 返回響應
```

#### 認證請求範例

```python
from fastapi import HTTPException

async def user_api_key_auth(request: Request, api_key: str):
    """驗證 API 金鑰"""
    
    # 提取 Bearer token
    api_key = _get_bearer_token_or_received_api_key(api_key)
    
    # 查詢金鑰物件
    key_object = await get_key_object(api_key=api_key)
    
    # 檢查角色權限
    user_role = _get_user_role(key_object)
    
    # 驗證是否能調用模型
    can_access = can_key_call_model(key_object, model_name)
    
    return UserAPIKeyAuth(
        api_key=api_key,
        user_id=key_object.user_id,
        team_id=key_object.team_id,
        user_role=user_role,
        permissions=key_object.permissions
    )
```

---

### 2.2 虛擬金鑰管理 (Virtual Keys)

虛擬金鑰提供安全且可控的 API 存取方式。

#### 主要檔案
- `litellm/proxy/management_endpoints/key_management_endpoints.py` - 金鑰管理端點

#### 生成虛擬金鑰

**生成金鑰**
```
POST /key/generate
```

**請求範例:**
```json
{
  "models": ["gpt-4o", "claude-3-opus"],
  "budget_id": "budget-001",
  "team_id": "team-001",
  "user_id": "user-001",
  "max_budget": 100.0,
  "tpm_limit": 10000,
  "rpm_limit": 1000,
  "duration": "30d",
  "auto_rotate": true,
  "rotation_interval": "90d",
  "allowed_cache_controls": ["no-cache", "no-store"]
}
```

**響應範例:**
```json
{
  "api_key": "sk-1234567890abcdef",
  "api_key_prefix": "sk-1234",
  "budget_id": "budget-001",
  "team_id": "team-001",
  "user_id": "user-001",
  "max_budget": 100.0,
  "tpm_limit": 10000,
  "rpm_limit": 1000,
  "expires_at": "2024-02-20T10:00:00Z",
  "auto_rotate": true,
  "rotation_interval": "90d",
  "key_rotation_at": "2024-04-20T10:00:00Z"
}
```

#### 金鑰管理流程圖

```mermaid
flowchart TD
    A[開始] --> B{操作類型}
    
    B --> C[生成金鑰]
    B --> D[查詢金鑰]
    B --> E[更新金鑰]
    B --> F[刪除金鑰]
    
    C --> G[驗證權限]
    D --> G
    E --> G
    F --> G
    
    G --> H{權限檢查通過?}
    
    H -->|否| I[返回 403 錯誤]
    H -->|是| J{驗證參數?}
    
    J -->|否| K[返回 400 錯誤]
    J -->|是| L[處理請求]
    
    L --> M[寫入資料庫]
    M --> N[設置緩存]
    N --> O[返回結果]
    
    I --> P[結束]
    K --> P
    O --> P
```

#### 金鑰輪換

```python
def _calculate_key_rotation_time(rotation_interval: str) -> datetime:
    """計算下一次輪換時間"""
    now = datetime.now(timezone.utc)
    interval_seconds = duration_in_seconds(rotation_interval)
    return now + timedelta(seconds=interval_seconds)

def _set_key_rotation_fields(
    data: dict, 
    auto_rotate: bool, 
    rotation_interval: Optional[str]
) -> None:
    """設置金鑰輪換欄位"""
    if auto_rotate and rotation_interval:
        data.update({
            "auto_rotate": auto_rotate,
            "rotation_interval": rotation_interval,
            "key_rotation_at": _calculate_key_rotation_time(rotation_interval),
        })
```

---

## 3. 流量管理

### 3.1 負載均衡 (Load Balancing)

LiteLLM Router 提供智能負載均衡功能。

#### 主要檔案
- `litellm/router.py` - Router 實作

#### Router 初始化

```python
from litellm import Router

router = Router(
    model_list=[
        {
            "model_name": "gpt-4o",
            "litellm_params": {
                "model": "openai/gpt-4o",
                "api_key": "sk-..."
            },
            "tpm": 100000,
            "rpm": 10000,
        },
        {
            "model_name": "gpt-4o",
            "litellm_params": {
                "model": "azure/gpt-4o",
                "api_base": "https://...",
                "api_key": "..."
            },
            "tpm": 80000,
            "rpm": 8000,
        }
    ],
    routing_strategy="least-busy",
    default_fallbacks=["claude-3-sonnet"]
)
```

#### 負載均衡策略

```mermaid
flowchart TD
    A[請求進入] --> B[選擇路由策略]
    
    B --> C[simple-shuffle]
    B --> D[least-busy]
    B --> E[usage-based-routing]
    B --> F[latency-based-routing]
    B --> G[cost-based-routing]
    B --> H[usage-based-routing-v2]
    
    C --> I[隨機選擇部署]
    D --> J[計算每個部署的繁忙程度]
    E --> K[基於使用量選擇]
    F --> L[基於延遲選擇]
    G --> M[基於成本選擇]
    H --> N[使用量路由 v2]
    
    I --> O[選擇部署]
    J --> O
    K --> O
    L --> O
    M --> O
    N --> O
    
    K --> L[執行請求]
    L --> M{成功?}
    
    M -->|是| N[返回響應]
    M -->|否| O[嘗試 fallback]
    
    O --> P{有 fallback?}
    P -->|是| Q[選擇下一個部署]
    P -->|否| R[返回錯誤]
    
    Q --> L
    N --> S[結束]
    R --> S
```

---

### 3.2 路由與重試 (Routing & Retries)

#### Router 重試機制

```python
from litellm import Router

router = Router(
    model_list=[...],
    retry_policy={
        "gpt-4o": {
            "retries": 3,
            "timeout": 60,
            "allowed_fails": 3,
        }
    },
    default_fallbacks=["claude-3-sonnet"],
    fallback_models=[
        {"model_name": "claude-3-haiku"},
        {"model_name": "gemini-pro"}
    ]
)
```

#### 重試流程圖

```mermaid
flowchart TD
    A[發送請求] --> B[執行 LLM 調用]
    
    B --> C{成功?}
    
    C -->|是| D[返回響應]
    C -->|否| E[檢查是否可重試]
    
    E --> F{重試次數 < 上限?}
    
    F -->|是| G[等待延遲]
    G --> H[指數退避]
    H --> B
    
    F -->|否| I{有 fallback?}
    
    I -->|是| J[切換到 fallback 模型]
    J --> K[執行 fallback]
    K --> L{成功?}
    
    I -->|否| M[返回最終錯誤]
    
    L -->|是| D
    L -->|否| N[繼續 fallback 鏈]
    N --> K
    
    D --> O[結束]
    M --> O
```

---

### 3.3 模型 Token 限流 (Rate Limiting)

#### 主要檔案
- `litellm/proxy/hooks/dynamic_rate_limiter.py` - 動態限流器
- `litellm/proxy/hooks/rate_limiter_utils.py` - 限流工具

#### 限流配置

```python
class DynamicRateLimiterCache:
    """追蹤每分鐘的活躍項目數量"""
    
    async def async_get_cache(self, model: str) -> Optional[int]:
        """取得當前分鐘的活躍項目數"""
        
    async def async_set_cache_sadd(self, model: str, value: List):
        """添加項目到集合"""
```

#### 限流檢查流程

```mermaid
flowchart TD
    A[請求進入] --> B[提取 API Key]
    B --> C[查詢 Key 限流配置]
    C --> D[查詢模型限流配置]
    D --> E{檢查 TPM 限制}
    
    E -->|超過| F[返回 429 錯誤]
    E -->|否| G{檢查 RPM 限制}
    
    G -->|超過| F
    G -->|否| H{檢查並發請求數}
    
    H -->|超過| I[等待或返回 429]
    H -->|否| J[執行請求]
    
    J --> K[更新使用量計數]
    K --> L[返回響應]
```

#### 限流配置範例

```python
# 在 config.yaml 中配置
model_list:
  - model_name: gpt-4o
    litellm_params:
      model: openai/gpt-4o
    tpm: 100000      # 每分鐘 token 限制
    rpm: 10000       # 每分鐘請求限制
    max_parallel_requests: 100  # 最大並發請求數

# 預設用戶限流
general_settings:
  default_user_params:
    tpm_limit: 50000
    rpm_limit: 5000
```

---

## 4. 安全護欄

### 4.1 PII/DLP Redaction - 自動遮蔽個資

#### 主要檔案
- `litellm/litellm_core_utils/redact_messages.py` - 訊息脫敏
- `tests/guardrails_tests/test_presidio_pii.py` - Presidio PII 測試

#### 脫敏功能

```python
def _redact_choice_content(choice):
    """脫敏選擇內容"""
    if isinstance(choice, litellm.Choices):
        choice.message.content = "redacted-by-litellm"
        if hasattr(choice.message, "reasoning_content"):
            choice.message.reasoning_content = "redacted-by-litellm"

def perform_redaction(model_call_details: dict, result):
    """執行脫敏"""
    # 脫敏輸入
    model_call_details["messages"] = [
        {"role": "user", "content": "redacted-by-litellm"}
    ]
    
    # 脫敏輸出
    if isinstance(result, litellm.ModelResponse):
        for choice in result.choices:
            _redact_choice_content(choice)
```

#### 脫敏流程圖

```mermaid
flowchart TD
    A[LLM 響應] --> B{脫敏開關?}
    
    B -->|否| C[直接記錄日誌]
    C --> D[返回響應]
    
    B -->|是| E[識別 PII]
    E --> F[遮蔽敏感資訊]
    
    F --> G[遮蔽身分證號]
    F --> H[遮蔽信用卡號]
    F --> I[遮蔽電話號碼]
    F --> J[遮蔽電子郵件]
    
    G --> K
    H --> K
    I --> K
    J --> K
    
    K[記錄脫敏後的日誌]
    K --> L[返回脫敏後的響應]
```

#### PII 遮蔽配置

```yaml
# config.yaml
general_settings:
  disable_message_redaction: false
  
litellm_omit_messages: true
litellm_omit_completion: true
```

---

### 4.2 Prompt Injection Defense - 防止惡意攻擊

#### 主要檔案
- `litellm/proxy/guardrails/` - Guardrail 實作

#### Prompt Injection 保護

```python
# 初始化 guardrails
from litellm import litellm_name_config_map

litellm_name_config_map = {
    "prompt_injection": {
        "callbacks": ["lakera_prompt_injection", "prompt_injection_api"],
        "default_on": True,
        "enabled_roles": ["user"]
    }
}
```

#### Guardrail 流程圖

```mermaid
flowchart TD
    A[用戶請求] --> B[Guardrail Pre-Call 檢查]
    
    B --> C{檢測到注入?}
    
    C -->|是| D[阻擋請求]
    D --> E[返回安全警告]
    
    C -->|否| F[執行 LLM 調用]
    
    F --> G[Guardrail Post-Call 檢查]
    G --> H{檢測到有害內容?}
    
    H -->|是| I[過濾輸出]
    H -->|否| J[正常返回]
    
    I --> K[返回過濾後的內容]
    
    E --> L[結束]
    J --> L
    K --> L
```

#### Guardrail 配置

```yaml
# config.yaml
guardrails:
  - prompt_injection:
      callbacks:
        - presidio
        - lakera_prompt_injection
      default_on: true
      enabled_roles:
        - user
        - assistant
```

---

### 4.3 Toxicity/Content Filtering - 內容過濾

#### 主要檔案
- `litellm/proxy/guardrails/guardrail_hooks/` - Guardrail hooks

#### 內容過濾配置

```python
# 初始化內容過濾 guardrails
all_guardrails = [
    {
        "content_safety": {
            "callbacks": ["bedrock_content_safety"],
            "default_on": True,
            "enabled_roles": ["user", "assistant"]
        }
    }
]
```

#### 內容過濾流程

```mermaid
flowchart TD
    A[輸入內容] --> B[毒性檢測]
    B --> C{仇恨言論?}
    
    C -->|是| D[標記並過濾]
    D --> E[記錄違規]
    E --> F[阻擋或消毒]
    
    C -->|否| G{暴力內容?}
    
    G -->|是| D
    G -->|否| H{不當內容?}
    
    H -->|是| D
    H -->|否| I[允許通過]
    
    F --> J[結束]
    I --> J
```

---

## 5. 支援功能

### 5.1 A2A (Agent-to-Agent)

#### 主要檔案
- `litellm/proxy/agent_endpoints/a2a_endpoints.py` - A2A 端點

#### A2A 端點

```python
# 獲取 Agent Card
GET /a2a/{agent_id}/.well-known/agent-card.json

# 發送消息
POST /a2a/{agent_id}/message/send

# 發送消息（流式）
POST /a2a/{agent_id}/message/stream
```

#### A2A 請求範例

```json
// POST /a2a/{agent_id}/message/send
{
  "jsonrpc": "2.0",
  "id": "req-001",
  "method": "message/send",
  "params": {
    "message": {
      "role": "user",
      "parts": [
        {
          "kind": "text",
          "text": "Hello, agent!"
        }
      ],
      "messageId": "msg-001"
    }
  }
}
```

#### A2A 響應範例

```json
{
  "jsonrpc": "2.0",
  "id": "req-001",
  "result": {
    "id": "msg-response-001",
    "status": {
      "state": "completed"
    },
    "role": "agent",
    "parts": [
      {
        "kind": "text",
        "text": "Hello! How can I help you today?"
      }
    ]
  }
}
```

#### A2A 流程圖

```mermaid
flowchart TD
    A[Agent Client] --> B[獲取 Agent Card]
    B --> C[發現 Agent 能力]
    
    C --> D[發送 A2A 請求]
    D --> E[LiteLLM Proxy 認證]
    
    E --> F{認證成功?}
    
    F -->|否| G[返回 403 錯誤]
    F -->|是| H[路由到目標 Agent]
    
    H --> I[Agent 處理請求]
    I --> J[返回響應]
    
    J --> K[記錄使用量]
    K --> L[返回給 Client]
    
    G --> M[結束]
    L --> M
```

---

### 5.2 AI API 標準化 (Input/Output)

#### OpenAI 格式標準化

```python
# LiteLLM 統一格式
response = litellm.completion(
    model="claude-3-opus",
    messages=[{"role": "user", "content": "Hello!"}]
)

# 返回標準 OpenAI 格式
# {
#   "id": "chatcmpl-abc123",
#   "object": "chat.completion",
#   "created": 1699999999,
#   "model": "claude-3-opus-20240307",
#   "choices": [...],
#   "usage": {...}
# }
```

#### 支援的端點

| 端點 | 說明 |
|------|------|
| `/v1/chat/completions` | 對話補全 |
| `/v1/embeddings` | 嵌入向量 |
| `/v1/images/generations` | 圖像生成 |
| `/v1/audio/transcriptions` | 語音轉文字 |
| `/v1/audio/speech` | 文字轉語音 |
| `/v1/moderations` | 內容審核 |
| `/v1/batches` | 批次處理 |
| `/v1/rerank` | 重排序 |
| `/v1/responses` | 回應 API |
| `/v1/messages` | 訊息 API |
| `/v1/a2a` | Agent-to-Agent |

---

### 5.3 雲地 AI 模型支援

#### 主要檔案
- `litellm/llms/` - 各提供商實作

#### 支援的雲端提供商

```python
# OpenAI
model="openai/gpt-4o"

# Azure
model="azure/gpt-4o"

# Anthropic
model="anthropic/claude-3-opus-20240307"

# Google Vertex AI
model="vertex_ai/gemini-pro"

# AWS Bedrock
model="bedrock/anthropic.claude-3-sonnet-20240307"

# Huggingface
model="huggingface/meta-llama/Meta-Llama-3-70B-Instruct"

# Ollama (本地)
model="ollama/llama2"
```

#### 模型路由流程

```mermaid
flowchart TD
    A[統一 API 請求] --> B[解析模型名稱]
    B --> C[識別提供商]
    
    C --> D{雲端?}
    
    D -->|是| E[獲取 API Key]
    E --> F[構建提供商特定請求]
    
    D -->|否| G{本地部署?}
    
    G --> H[連接到本地伺服器]
    G --> I[使用本地模型]
    
    F --> J[執行請求]
    H --> J
    I --> J
    
    J --> K[轉換為統一格式]
    K --> L[返回 OpenAI 格式響應]
```

---

## 6. 企業級功能

### 6.1 SSO (單一登入)

#### 主要檔案
- `litellm/proxy/management_endpoints/ui_sso.py` - SSO 端點
- `litellm/proxy/custom_sso.py` - 自訂 SSO

#### SSO 流程

```mermaid
flowchart TD
    A[用戶訪問 UI] --> B{已登入?}
    
    B -->|否| C[重定向到 SSO 提供商]
    C --> D[用戶登入]
    D --> E[SSO 提供商返回 JWT]
    
    E --> F[驗證 JWT]
    F --> G[提取用戶資訊]
    
    G --> H{映射到 LiteLLM 角色?}
    
    H -->|是| I[創建/更新用戶]
    H -->|否| J[使用預設角色]
    
    I --> K[創建會話]
    J --> K
    
    K --> L[重定向到 UI]
    L --> M[登入成功]
    
    B -->|是| N[顯示 UI]
```

#### SSO 配置

```python
# config.yaml
auth_settings:
  google_oauth2:
    client_id: "your-client-id"
    client_secret: "your-client-secret"
    redirect_uri: "http://localhost:4000/sso/callback"
    
  microsoft:
    client_id: "your-client-id"
    client_secret: "your-client-secret"
    redirect_uri: "http://localhost:4000/sso/callback"
    tenant_id: "your-tenant-id"
    
  # 自訂 OpenID
  custom_openid:
    openid_configuration_url: "https://your-idp/.well-known/openid-configuration"
    client_id: "your-client-id"
    client_secret: "your-client-secret"
```

#### 角色映射

```python
def determine_role_from_groups(
    user_groups: List[str],
    role_mappings: "RoleMappings",
) -> Optional[LitellmUserRoles]:
    """根據用戶組確定角色"""
    role_hierarchy = [
        LitellmUserRoles.PROXY_ADMIN,
        LitellmUserRoles.PROXY_ADMIN_VIEW_ONLY,
        LitellmUserRoles.INTERNAL_USER,
        LitellmUserRoles.INTERNAL_USER_VIEW_ONLY,
    ]
    
    for role in role_hierarchy:
        if role in role_mappings.roles:
            role_groups = role_mappings.roles[role]
            if set(user_groups).intersection(set(role_groups)):
                return role
    
    return role_mappings.default_role
```

---

### 6.2 Audit Logs (稽核日誌)

#### 主要檔案
- `litellm/proxy/management_helpers/audit_logs.py` - 稽核日誌

#### 稽核日誌 API

```python
async def create_audit_log_for_update(request_data: LiteLLM_AuditLogs):
    """創建稽核日誌"""
    await prisma_client.db.litellm_auditlog.create(
        data={
            "id": request_data.id,
            "updated_at": request_data.updated_at,
            "changed_by": request_data.changed_by,
            "changed_by_api_key": request_data.changed_by_api_key,
            "table_name": request_data.table_name,
            "object_id": request_data.object_id,
            "action": request_data.action,
            "updated_values": request_data.updated_values,
            "before_value": request_data.before_value,
        }
    )
```

#### 稽核日誌資料結構

```python
class LiteLLM_AuditLogs(BaseModel):
    id: str
    updated_at: datetime
    changed_by: str  # 執行操作的用戶
    changed_by_api_key: str  # 使用的 API Key
    table_name: str  # 操作的表格
    object_id: str  # 操作的物件 ID
    action: AUDIT_ACTIONS  # 操作類型
    updated_values: Optional[str]  # 更新後的值
    before_value: Optional[str]  # 更新前的值
```

#### 稽核日誌流程圖

```mermaid
flowchart TD
    A[管理操作] --> B[創建稽核日誌]
    
    B --> C[檢查是否啟用稽核]
    C --> D{已啟用?}
    
    D -->|否| E[跳過]
    D -->|是| F{是企業用戶?}
    
    F -->|否| E
    F -->|是| G[提取操作資訊]
    
    G --> H[記錄操作者]
    H --> I[記錄操作類型]
    I --> J[記錄物件 ID]
    J --> K[記錄變更前後值]
    
    K --> L[寫入資料庫]
    L --> M[返回操作結果]
    
    E --> M
```

#### 稽核日誌查詢

```python
# 查詢稽核日誌
GET /audit/logs

# 請求範例
{
  "table_name": "LiteLLM_VerificationToken",
  "object_id": "key-001",
  "start_date": "2024-01-01T00:00:00Z",
  "end_date": "2024-01-31T23:59:59Z"
}
```

#### 稽核日誌記錄的動作

```python
# 稽核日誌記錄的動作 (實際實作為 Literal 類型)
AUDIT_ACTIONS = Literal["created", "updated", "deleted", "blocked", "rotated"]
```

---

## 附錄

### 配置範例

```yaml
# 完整配置範例 - config.yaml

model_list:
  - model_name: gpt-4o
    litellm_params:
      model: openai/gpt-4o
      api_key: os.environ/OPENAI_API_KEY
    tpm: 100000
    rpm: 10000
  - model_name: claude-3-opus
    litellm_params:
      model: anthropic/claude-3-opus-20240307
      api_key: os.environ/ANTHROPIC_API_KEY
    tpm: 80000
    rpm: 5000

general_settings:
  # 資料庫連接
  database_url: "postgresql://user:password@localhost:5432/litellm"
  
  # 認證
  master_key: "sk-admin-master-key"
  
  # 限流
  default_user_params:
    tpm_limit: 50000
    rpm_limit: 5000
  
  # 脫敏
  disable_message_redaction: false

# Guardrails
guardrails:
  - presidio:
      callbacks:
        - presidio
      default_on: true
      enabled_roles:
        - user

# 稽核日誌
store_audit_logs: true
LITELLM_STORE_AUDIT_LOGS: true

# SSO 配置
auth_settings:
  google_oauth2:
    client_id: os.environ/GOOGLE_CLIENT_ID
    client_secret: os.environ/GOOGLE_CLIENT_SECRET
```

### 環境變數

| 變數名稱 | 說明 |
|----------|------|
| `LITELLM_STORE_AUDIT_LOGS` | 啟用稽核日誌 |
| `LITELLM_LICENSE` | 企業版授權碼 |
| `LITELLM_TURN_OFF_MESSAGE_LOGGING` | 關閉訊息日誌 |
| `LITELLM_DISABLE_MESSAGE_REDACTION` | 禁用訊息脫敏 |

---

**最後更新**: 2026年1月21日
**版本**: LiteLLM Proxy
