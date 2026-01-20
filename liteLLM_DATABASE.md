# LiteLLM 資料庫與快取格式文件

本文檔詳細說明 LiteLLM 專案中 PostgreSQL 和 Redis 的資料格式、使用範例及資料結構。

---

## 目錄

1. [PostgreSQL 資料庫 Schema](#1-postgresql-資料庫-schema)
2. [Prisma ORM 資料操作範例](#2-prisma-orm-資料操作範例)
3. [Redis 快取資料格式](#3-redis-快取資料格式)
4. [使用量分析資料格式](#4-使用量分析資料格式)
5. [RBAC 權限資料格式](#5-rbac-權限資料格式)
6. [多租戶資料格式](#6-多租戶資料格式)
7. [花費追蹤資料格式](#7-花費追蹤資料格式)
8. [速率限制資料格式](#8-速率限制資料格式)

---

## 1. PostgreSQL 資料庫 Schema

### 1.1 資料庫連線配置

```yaml
# docker-compose.yml
version: '3.8'
services:
  litellm:
    image: ghcr.io/berriai/litellm:main-stable
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/litellm
      - REDIS_HOST=redis
      - REDIS_PORT=6379

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=litellm
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine

volumes:
  postgres_data:
```

### 1.2 環境變數

```bash
# 資料庫連線字串
export DATABASE_URL="postgresql://postgres:postgres@localhost:5432/litellm"

# Redis 連線
export REDIS_HOST="localhost"
export REDIS_PORT="6379"
export REDIS_PASSWORD="your_redis_password"
```

---

## 2. Prisma ORM 資料操作範例

### 2.1 Prisma Schema 定義

```prisma
// schema.prisma

datasource client {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-py"
}

// 用戶表
model LiteLLM_UserTable {
  user_id              String   @id
  user_alias           String?
  team_id              String?
  organization_id      String?
  user_email           String?  @unique
  user_role            String?
  max_budget           Float?
  spend                Float    @default(0.0)
  models               String[]
  tpm_limit            BigInt?
  rpm_limit            BigInt?
  metadata             Json     @default("{}")
  created_at           DateTime? @default(now())
  updated_at           DateTime? @updatedAt

  organization         LiteLLM_OrganizationTable? @relation(fields: [organization_id], references: [organization_id])
}

// 團隊表
model LiteLLM_TeamTable {
  team_id              String   @id @default(uuid())
  team_alias           String?
  organization_id      String?
  admins               String[]
  members              String[]
  members_with_roles   Json     @default("{}")
  max_budget           Float?
  spend                Float    @default(0.0)
  models               String[]
  tpm_limit            BigInt?
  rpm_limit            BigInt?
  blocked              Boolean  @default(false)
  created_at           DateTime @default(now())
  updated_at           DateTime @updatedAt

  organization         LiteLLM_OrganizationTable? @relation(fields: [organization_id], references: [organization_id])
}

// 組織表
model LiteLLM_OrganizationTable {
  organization_id      String   @id @default(uuid())
  organization_alias   String
  budget_id            String
  metadata             Json     @default("{}")
  models               String[]
  spend                Float    @default(0.0)
  created_at           DateTime @default(now())

  budget               LiteLLM_BudgetTable? @relation(fields: [budget_id], references: [budget_id])
  teams                LiteLLM_TeamTable[]
  users                LiteLLM_UserTable[]
}

// API 金鑰表
model LiteLLM_VerificationToken {
  token                String   @id
  key_name             String?
  key_alias            String?
  spend                Float    @default(0.0)
  expires              DateTime?
  models               String[]
  user_id              String?
  team_id              String?
  permissions          Json     @default("{}")
  max_budget           Float?
  tpm_limit            BigInt?
  rpm_limit            BigInt?
  max_parallel_requests Int?
  blocked              Boolean?
  budget_id            String?
  organization_id      String?
  auto_rotate          Boolean? @default(false)
  rotation_interval    String?
  created_at           DateTime? @default(now())
}

// 預算表
model LiteLLM_BudgetTable {
  budget_id            String   @id @default(uuid())
  max_budget           Float?
  soft_budget          Float?
  tpm_limit            BigInt?
  rpm_limit            BigInt?
  model_max_budget     Json?
  budget_duration      String?
  created_at           DateTime @default(now())
  created_by           String

  organization         LiteLLM_OrganizationTable?
  keys                 LiteLLM_VerificationToken[]
}

// 花費日誌表
model LiteLLM_SpendLogs {
  request_id           String   @id
  call_type            String
  api_key              String   @default("")
  spend                Float    @default(0.0)
  total_tokens         Int      @default(0)
  prompt_tokens        Int      @default(0)
  completion_tokens    Int      @default(0)
  startTime            DateTime
  endTime              DateTime
  model                String   @default("")
  model_group          String?
  user                 String?
  metadata             Json?
  team_id              String?
  organization_id      String?
  end_user             String?

  @@index([startTime])
  @@index([end_user])
}

// 每日花費表 (用戶)
model LiteLLM_DailyUserSpend {
  id                      String   @id @default(uuid())
  user_id                 String?
  date                    String
  api_key                 String
  model                   String?
  model_group             String?
  custom_llm_provider     String?
  prompt_tokens           BigInt   @default(0)
  completion_tokens       BigInt   @default(0)
  spend                   Float    @default(0.0)
  api_requests            BigInt   @default(0)
  successful_requests     BigInt   @default(0)
  failed_requests         BigInt   @default(0)
  created_at              DateTime @default(now())

  @@unique([user_id, date, api_key, model])
  @@index([date])
}

// 每日花費表 (團隊)
model LiteLLM_DailyTeamSpend {
  id                      String   @id @default(uuid())
  team_id                 String?
  date                    String
  api_key                 String
  model                   String?
  spend                   Float    @default(0.0)
  prompt_tokens           BigInt   @default(0)
  completion_tokens       BigInt   @default(0)
  api_requests            BigInt   @default(0)

  @@unique([team_id, date, api_key, model])
  @@index([date])
}

// 每日花費表 (組織)
model LiteLLM_DailyOrganizationSpend {
  id                      String   @id @default(uuid())
  organization_id         String?
  date                    String
  api_key                 String
  model                   String?
  spend                   Float    @default(0.0)
  prompt_tokens           BigInt   @default(0)
  completion_tokens       BigInt   @default(0)

  @@unique([organization_id, date, api_key, model])
  @@index([date])
}

// 審計日誌表
model LiteLLM_AuditLog {
  id              String   @id @default(uuid())
  updated_at      DateTime @default(now())
  changed_by      String
  action          String
  table_name      String
  object_id       String
  before_value    Json?
  updated_values  Json?
}
```

### 2.2 Python Prisma 查詢範例

#### 2.2.1 查詢用戶

```python
# 查詢單一用戶
user = await prisma_client.db.litellm_usertable.find_unique(
    where={"user_id": "user-uuid-123"}
)

# 查詢所有用戶
users = await prisma_client.db.litellm_usertable.find_many()

# 條件查詢
users = await prisma_client.db.litellm_usertable.find_many(
    where={
        "user_role": "internal_user",
        "spend": {"gte": 100.0}
    }
)

# 關聯查詢
user_with_org = await prisma_client.db.litellm_usertable.find_first(
    where={"user_id": "user-uuid-123"},
    include={"organization": True}
)
```

#### 2.2.2 創建用戶

```python
new_user = await prisma_client.db.litellm_usertable.create(
    data={
        "user_id": "user-uuid-456",
        "user_alias": "John Doe",
        "user_email": "john@example.com",
        "user_role": "internal_user",
        "max_budget": 500.00,
        "models": ["gpt-4", "claude-3-sonnet"],
        "metadata": {"department": "Engineering"},
        "created_by": "admin-user-id"
    }
)
```

#### 2.2.3 更新用戶

```python
updated_user = await prisma_client.db.litellm_usertable.update(
    where={"user_id": "user-uuid-123"},
    data={
        "max_budget": 1000.00,
        "models": ["gpt-4", "claude-3-opus"]
    }
)
```

#### 2.2.4 查詢 API 金鑰

```python
# 查詢金鑰
key = await prisma_client.db.litellm_verificationtoken.find_unique(
    where={"token": "sk-1234567890abcdef..."}
)

# 查詢用戶的所有金鑰
keys = await prisma_client.db.litellm_verificationtoken.find_many(
    where={"user_id": "user-uuid-123"}
)

# 批量查詢
keys = await prisma_client.db.litellm_verificationtoken.find_many(
    where={"token": {"in": ["sk-key1", "sk-key2", "sk-key3"]}}
)
```

#### 2.2.5 創建 API 金鑰

```python
new_key = await prisma_client.db.litellm_verificationtoken.create(
    data={
        "token": "sk-" + secrets.token_hex(32),
        "key_name": "Production API Key",
        "user_id": "user-uuid-123",
        "team_id": "team-uuid-456",
        "models": ["gpt-4", "claude-3-sonnet"],
        "max_budget": 1000.00,
        "rpm_limit": 1000,
        "tpm_limit": 1000000,
        "expires": datetime.now() + timedelta(days=30),
        "created_by": "admin-user-id"
    }
)
```

#### 2.2.6 統計查詢

```python
# 計算總花費
result = await prisma_client.db.query_raw(
    query="""
        SELECT SUM(spend) as total_spend 
        FROM "LiteLLM_SpendLogs" 
        WHERE startTime >= NOW() - INTERVAL '30 days'
    """
)

# 按模型分組統計
result = await prisma_client.db.query_raw(
    query="""
        SELECT model, SUM(spend) as total_spend, COUNT(*) as request_count
        FROM "LiteLLM_SpendLogs"
        WHERE startTime >= NOW() - INTERVAL '7 days'
        GROUP BY model
        ORDER BY total_spend DESC
    """
)

# 按團隊統計
result = await prisma_client.db.query_raw(
    query="""
        SELECT t.team_id, t.team_alias, SUM(t.spend) as total_spend
        FROM "LiteLLM_TeamTable" t
        LEFT JOIN "LiteLLM_SpendLogs" s ON t.team_id = s.team_id
        WHERE s.startTime >= NOW() - INTERVAL '30 days'
        GROUP BY t.team_id, t.team_alias
        ORDER BY total_spend DESC
    """
)
```

---

## 3. Redis 快取資料格式

### 3.1 DualCache 結構

```python
from litellm.caching import DualCache

# 初始化 DualCache (記憶體 + Redis)
user_api_key_cache = DualCache(
    in_memory_cache=None,  # 使用預設記憶體快取
    redis_cache=RedisCache(
        host="localhost",
        port=6379,
        password="your_redis_password"
    ),
    default_in_memory_ttl=60,   # 記憶體 TTL: 60秒
    default_redis_ttl=300       # Redis TTL: 300秒
)
```

### 3.2 快取操作範例

#### 3.2.1 設置快取

```python
# 同步設置
user_api_key_cache.set_cache(
    key="user:user-uuid-123",
    value={
        "user_id": "user-uuid-123",
        "user_alias": "John Doe",
        "max_budget": 500.00,
        "spend": 150.00,
        "models": ["gpt-4"]
    },
    ttl=300  # 5分鐘過期
)

# 非同步設置
await user_api_key_cache.async_set_cache(
    key="key:sk-1234...ABCD",
    value={
        "token": "sk-1234...ABCD",
        "key_alias": "Production Key",
        "team_id": "team-uuid-456",
        "max_budget": 1000.00,
        "models": ["gpt-4", "claude-3-sonnet"]
    },
    ttl=600
)
```

#### 3.2.2 獲取快取

```python
# 同步獲取
user_data = user_api_key_cache.get_cache(key="user:user-uuid-123")
if user_data:
    print(f"User: {user_data['user_alias']}")
    print(f"Budget: {user_data['max_budget']}")

# 非同步獲取
user_data = await user_api_key_cache.async_get_cache(key="user:user-uuid-123")

# 批量獲取
results = await user_api_key_cache.async_batch_get_cache(
    keys=["user:1", "user:2", "user:3"]
)
```

#### 3.2.3 批量設置

```python
cache_list = [
    {"key": "model:gpt-4", "value": {"name": "GPT-4", "cost": 0.03}},
    {"key": "model:claude-3-opus", "value": {"name": "Claude 3 Opus", "cost": 0.015}},
    {"key": "model:gpt-3.5-turbo", "value": {"name": "GPT-3.5 Turbo", "cost": 0.0015}}
]

await user_api_key_cache.async_set_cache_pipeline(
    cache_list=cache_list,
    ttl=3600
)
```

### 3.3 Redis 鍵命名規範

```python
# API 金鑰快取
"key:{hashed_token}"

# 用戶快取
"user:{user_id}"

# 團隊快取
"team:{team_id}"

# 組織快取
"org:{organization_id}"

# 速率限制計數器
"ratelimit:{token}:rpm"   # 每分鐘請求數
"ratelimit:{token}:tpm"   # 每分鐘 Token 數

# 花費追蹤
"spend:{user_id}"         # 用戶總花費
"spend:{team_id}"         # 團隊總花費
"spend:{key}"             # 金鑰花費

# 全域花費
"{admin_name}:spend"
```

### 3.4 Redis 資料結構範例

#### 3.4.1 API 金鑰快取資料格式

```json
{
  "token_id": "sk-1234567890abcdef...",
  "key_name": "Production API Key",
  "key_alias": "My Production Key",
  "user_id": "user-uuid-123",
  "team_id": "team-uuid-456",
  "organization_id": "org-uuid-789",
  "models": ["gpt-4", "claude-3-sonnet", "gpt-3.5-turbo"],
  "max_budget": 1000.00,
  "spend": 250.50,
  "rpm_limit": 1000,
  "tpm_limit": 1000000,
  "max_parallel_requests": 10,
  "permissions": {
    "allowed_routes": ["llm_api_routes"],
    "allowed_models": ["gpt-4", "claude-3-opus"]
  },
  "metadata": {
    "created_by": "admin@example.com",
    "department": "Engineering"
  },
  "expires": "2024-02-19T10:00:00Z",
  "blocked": false,
  "auto_rotate": true,
  "rotation_interval": "90d"
}
```

#### 3.4.2 用戶快取資料格式

```json
{
  "user_id": "user-uuid-123",
  "user_alias": "John Doe",
  "user_email": "john@example.com",
  "user_role": "internal_user",
  "max_budget": 500.00,
  "spend": 150.00,
  "models": ["gpt-4", "claude-3-sonnet"],
  "tpm_limit": 500000,
  "rpm_limit": 500,
  "teams": ["team-uuid-456", "team-uuid-789"],
  "organization_id": "org-uuid-111",
  "metadata": {
    "department": "Engineering",
    "project": "AI Research"
  }
}
```

#### 3.4.3 速率限制計數器

```python
# RPM (Requests Per Minute) 計數器
key = "ratelimit:sk-1234...ABCD:rpm"

# 使用 Redis INCR 命令
await redis_cache.increment_cache(key=key, value=1)

# 設置過期時間 (1分鐘)
await redis_cache.async_set_cache(key=key, value=current_count, ttl=60)

# TPM (Tokens Per Minute) 計數器
key = "ratelimit:sk-1234...ABCD:tpm"
await redis_cache.increment_cache(key=key, value=num_tokens)
```

---

## 4. 使用量分析資料格式

### 4.1 花費日誌資料格式

```python
# spend_tracking_utils.py

spend_log_payload = {
    "request_id": "req-uuid-123",
    "call_type": "acompletion",
    "api_key": "sk-1234567890abcdef...",  # Hash 後的 token
    "spend": 0.025,  # 花費金額 (USD)
    "total_tokens": 1500,
    "prompt_tokens": 1000,
    "completion_tokens": 500,
    "startTime": "2024-01-19T10:00:00Z",
    "endTime": "2024-01-19T10:00:01Z",
    "model": "gpt-4",
    "model_group": "gpt-4",
    "custom_llm_provider": "openai",
    "api_base": "https://api.openai.com/v1",
    "user": "user-uuid-123",
    "metadata": {
        "project": "AI Research",
        "environment": "production"
    },
    "team_id": "team-uuid-456",
    "organization_id": "org-uuid-789",
    "end_user": "end-user-123",
    "cache_hit": "false",
    "request_tags": ["project-alpha", "feature-xyz"],
    "session_id": "session-uuid-123"
}
```

### 4.2 每日花費聚合資料格式

```python
# LiteLLM_DailyUserSpend
daily_spend_record = {
    "id": "daily-uuid-123",
    "user_id": "user-uuid-123",
    "date": "2024-01-19",
    "api_key": "sk-1234...ABCD",
    "model": "gpt-4",
    "model_group": "gpt-4",
    "custom_llm_provider": "openai",
    "prompt_tokens": 150000,
    "completion_tokens": 75000,
    "cache_read_input_tokens": 10000,
    "cache_creation_input_tokens": 5000,
    "spend": 125.50,
    "api_requests": 1500,
    "successful_requests": 1480,
    "failed_requests": 20
}
```

### 4.3 花費分析 API Response

```json
{
  "results": [
    {
      "date": "2024-01-19",
      "metrics": {
        "spend": 125.50,
        "prompt_tokens": 150000,
        "completion_tokens": 75000,
        "total_tokens": 225000,
        "cache_read_input_tokens": 10000,
        "cache_creation_input_tokens": 5000,
        "api_requests": 1525,
        "successful_requests": 1500,
        "failed_requests": 25
      },
      "breakdown": {
        "models": {
          "gpt-4": {
            "metrics": {
              "spend": 100.00,
              "prompt_tokens": 120000,
              "completion_tokens": 60000
            },
            "metadata": {
              "display_name": "GPT-4"
            },
            "api_key_breakdown": {
              "sk-abc123...": {
                "metrics": {
                  "spend": 50.00
                },
                "metadata": {
                  "key_alias": "Production Key",
                  "team_id": "team-123"
                }
              }
            }
          },
          "claude-3-sonnet": {
            "metrics": {
              "spend": 25.50,
              "prompt_tokens": 30000,
              "completion_tokens": 15000
            }
          }
        },
        "model_groups": {
          "gpt-4": {
            "metrics": {
              "spend": 100.00
            }
          }
        },
        "providers": {
          "openai": {
            "metrics": {
              "spend": 100.00
            }
          },
          "anthropic": {
            "metrics": {
              "spend": 25.50
            }
          }
        },
        "api_keys": {
          "sk-abc123...": {
            "metrics": {
              "spend": 50.00
            },
            "metadata": {
              "key_alias": "Production Key"
            }
          }
        }
      }
    }
  ],
  "metadata": {
    "total_spend": 125.50,
    "total_prompt_tokens": 150000,
    "total_completion_tokens": 75000,
    "total_tokens": 225000,
    "total_api_requests": 1525,
    "total_successful_requests": 1500,
    "total_failed_requests": 25,
    "total_cache_read_input_tokens": 10000,
    "total_cache_creation_input_tokens": 5000,
    "page": 1,
    "total_pages": 1,
    "has_more": false
  }
}
```

### 4.4 花費追蹤佇列資料格式

```python
# SpendUpdateQueueItem
spend_update_item = {
    "entity_type": "user",  # user, team, key, organization, end_user, team_member, tag
    "entity_id": "user-uuid-123",
    "response_cost": 0.025
}

# DBSpendUpdateTransactions
db_transactions = {
    "user_list_transactions": {
        "user-uuid-1": 100.00,
        "user-uuid-2": 50.00
    },
    "end_user_list_transactions": {
        "end-user-1": 25.00
    },
    "key_list_transactions": {
        "hashed-key-1": 75.00,
        "hashed-key-2": 25.00
    },
    "team_list_transactions": {
        "team-uuid-1": 150.00
    },
    "team_member_list_transactions": {
        "team-uuid-1::user-uuid-1": 50.00
    },
    "org_list_transactions": {
        "org-uuid-1": 200.00
    },
    "tag_list_transactions": {
        "project-alpha": 100.00
    }
}
```

---

## 5. RBAC 權限資料格式

### 5.1 角色定義

```python
from enum import Enum

class LitellmUserRoles(str, Enum):
    """角色定義"""
    PROXY_ADMIN = "proxy_admin"
    PROXY_ADMIN_VIEW_ONLY = "proxy_admin_viewer"
    ORG_ADMIN = "org_admin"
    INTERNAL_USER = "internal_user"
    INTERNAL_USER_VIEW_ONLY = "internal_user_viewer"
    TEAM = "team"
    CUSTOMER = "customer"

class LiteLLMTeamRoles(str, Enum):
    """團隊角色"""
    TEAM_ADMIN = "admin"
    TEAM_MEMBER = "user"
```

### 5.2 角色權限對照

| 角色 | 查看所有金鑰 | 查看所有花費 | 創建用戶 | 創建團隊 | 管理模型 |
|------|-------------|-------------|---------|---------|---------|
| PROXY_ADMIN | ✅ | ✅ | ✅ | ✅ | ✅ |
| PROXY_ADMIN_VIEW_ONLY | ✅ | ✅ | ❌ | ❌ | ❌ |
| ORG_ADMIN | 組織內 | 組織內 | ✅ | ✅ | ✅ |
| INTERNAL_USER | 自己的 | 自己的 | ❌ | ❌ | ❌ |
| INTERNAL_USER_VIEW_ONLY | 自己的 | 自己的 | ❌ | ❌ | ❌ |

### 5.3 API 金鑰權限資料格式

```python
# GenerateKeyRequest
key_request = {
    "user_id": "user-uuid-123",
    "team_id": "team-uuid-456",
    "models": ["gpt-4", "claude-3-sonnet"],
    "max_budget": 1000.00,
    "budget_duration": "30d",
    "rpm_limit": 1000,
    "tpm_limit": 1000000,
    "max_parallel_requests": 10,
    "key_alias": "Production API Key",
    "allowed_routes": ["llm_api_routes"],
    "permissions": {
        "allowed_models": ["gpt-4", "claude-3-opus"],
        "blocked_models": ["gpt-3.5-turbo"],
        "allowed_end_users": ["user-1", "user-2"],
        "deny_all_end_users": False
    },
    "metadata": {
        "environment": "production",
        "department": "Engineering"
    }
}
```

### 5.4 JWT 權限資料格式

```python
# JWT Token Payload
jwt_payload = {
    "sub": "user-uuid-123",
    "email": "john@example.com",
    "role": "internal_user",
    "teams": ["team-uuid-456", "team-uuid-789"],
    "organization": "org-uuid-111",
    "scopes": ["read:keys", "write:keys", "read:spend"],
    "permissions": {
        "can_create_keys": True,
        "can_delete_keys": False,
        "can_view_all_spend": False,
        "can_manage_users": False
    },
    "model_access": {
        "allowed_models": ["gpt-4", "claude-3-sonnet"],
        "denied_models": []
    },
    "rate_limits": {
        "rpm": 500,
        "tpm": 500000
    },
    "budget": {
        "max_budget": 500.00,
        "soft_budget": 450.00
    },
    "exp": 1705670400,
    "iat": 1705584000
}
```

### 5.5 團隊成員權限資料格式

```python
# Member 結構
member = {
    "user_id": "user-uuid-123",
    "role": "admin",  # "admin" 或 "user"
    "permissions": {
        "can_manage_team": True,
        "can_add_members": True,
        "can_remove_members": True,
        "can_view_spend": True,
        "can_create_keys": True,
        "can_delete_keys": False
    },
    "budget": {
        "max_budget": 500.00,
        "rpm_limit": 100,
        "tpm_limit": 100000
    },
    "model_access": {
        "allowed_models": ["gpt-4", "claude-3-sonnet"],
        "denied_models": ["gpt-3.5-turbo"]
    }
}

# Team Table members_with_roles
{
    "members_with_roles": [
        {
            "user_id": "user-uuid-1",
            "role": "admin",
            "permissions": {...},
            "budget": {...},
            "model_access": {...}
        },
        {
            "user_id": "user-uuid-2",
            "role": "user",
            "permissions": {...},
            "budget": {...},
            "model_access": {...}
        }
    ]
}
```

---

## 6. 多租戶資料格式

### 6.1 組織結構資料

```python
# LiteLLM_OrganizationTable
organization = {
    "organization_id": "org-uuid-123",
    "organization_alias": "Acme Corporation",
    "budget_id": "budget-uuid-456",
    "models": ["gpt-4", "claude-3-opus", "gpt-3.5-turbo"],
    "spend": 2500.00,
    "metadata": {
        "industry": "Technology",
        "size": "Enterprise",
        "plan": "premium"
    },
    "settings": {
        "default_model": "gpt-4",
        "max_tokens_default": 4096,
        "temperature_default": 0.7,
        "allowed_regions": ["us-east-1", "eu-west-1"]
    }
}
```

### 6.2 組織成員關聯

```python
# LiteLLM_OrganizationMembership
membership = {
    "user_id": "user-uuid-123",
    "organization_id": "org-uuid-456",
    "user_role": "org_admin",  # org_admin, internal_user, viewer
    "spend": 150.00,
    "budget_id": "budget-uuid-789",
    "permissions": {
        "can_manage_team": True,
        "can_manage_budget": True,
        "can_view_analytics": True,
        "can_manage_integrations": True
    },
    "created_at": "2024-01-01T10:00:00Z",
    "updated_at": "2024-01-19T10:00:00Z"
}
```

### 6.3 團隊結構資料

```python
# LiteLLM_TeamTable
team = {
    "team_id": "team-uuid-123",
    "team_alias": "AI Research Team",
    "organization_id": "org-uuid-456",
    "admins": ["user-uuid-1", "user-uuid-2"],
    "members": ["user-uuid-1", "user-uuid-2", "user-uuid-3"],
    "members_with_roles": [...],  # 詳細成員資料
    "max_budget": 5000.00,
    "spend": 1250.00,
    "models": ["gpt-4", "claude-3-opus"],
    "tpm_limit": 1000000,
    "rpm_limit": 1000,
    "blocked": False,
    "metadata": {
        "department": "AI Research",
        "project": "LLM Optimization"
    },
    "router_settings": {
        "routing_strategy": "usage-based-routing-v2",
        "fallback_models": ["gpt-4", "claude-3-sonnet"]
    },
    "model_max_budget": {
        "gpt-4": 2000.00,
        "claude-3-opus": 3000.00
    }
}
```

### 6.4 多租戶隔離查詢

```python
# 查詢特定組織的所有團隊
teams = await prisma_client.db.litellm_teamtable.find_many(
    where={"organization_id": "org-uuid-123"}
)

# 查詢特定組織的所有用戶
users = await prisma_client.db.litellm_usertable.find_many(
    where={"organization_id": "org-uuid-123"}
)

# 查詢特定組織的所有金鑰
keys = await prisma_client.db.litellm_verificationtoken.find_many(
    where={"organization_id": "org-uuid-123"}
)

# 查詢用戶所屬組織
user_org = await prisma_client.db.litellm_usertable.find_first(
    where={"user_id": "user-uuid-123"},
    include={"organization": True}
)
```

---

## 7. 花費追蹤資料格式

### 7.1 花費更新佇列

```python
# SpendUpdateQueueItem
{
    "entity_type": "user",  # user, team, key, organization, end_user, team_member, tag
    "entity_id": "user-uuid-123",
    "response_cost": 0.025
}

# DailyUserSpendTransaction
daily_spend_transaction = {
    "user_keyhash_model_provider_endpoint_date": "user-uuid-1_sk-key_gpt-4_openai_chat_completions_2024-01-19",
    "spend": 0.025,
    "prompt_tokens": 1000,
    "completion_tokens": 500,
    "api_requests": 1,
    "successful_requests": 1,
    "failed_requests": 0,
    "cache_read_input_tokens": 0,
    "cache_creation_input_tokens": 0
}
```

### 7.2 Redis 緩衝區資料格式

```python
# Redis 鍵常量
REDIS_UPDATE_BUFFER_KEY = "litellm_spend_update_buffer"
REDIS_DAILY_SPEND_UPDATE_BUFFER_KEY = "litellm_daily_spend_update_buffer"
REDIS_DAILY_TEAM_SPEND_UPDATE_BUFFER_KEY = "litellm_daily_team_spend_update_buffer"
REDIS_DAILY_ORG_SPEND_UPDATE_BUFFER_KEY = "litellm_daily_org_spend_update_buffer"
REDIS_DAILY_END_USER_SPEND_UPDATE_BUFFER_KEY = "litellm_daily_end_user_spend_update_buffer"
REDIS_DAILY_TAG_SPEND_UPDATE_BUFFER_KEY = "litellm_daily_tag_spend_update_buffer"

# Redis List 儲存格式
# spend_update_buffer:
# [
#   "{\"user_list_transactions\":{\"user-1\":100.0},\"key_list_transactions\":{}}",
#   "{\"user_list_transactions\":{\"user-2\":50.0},\"key_list_transactions\":{}}"
# ]

# daily_spend_update_buffer:
# [
#   "{\"user-uuid-1_gpt-4_2024-01-19\":{\"spend\":0.025,\"prompt_tokens\":1000}}"
# ]
```

### 7.3 花費批次寫入

```python
# 聚合花費更新
async def aggregate_spend_updates(updates: List[SpendUpdateQueueItem]) -> DBSpendUpdateTransactions:
    aggregated = {
        "user_list_transactions": {},
        "end_user_list_transactions": {},
        "key_list_transactions": {},
        "team_list_transactions": {},
        "team_member_list_transactions": {},
        "org_list_transactions": {},
        "tag_list_transactions": {}
    }
    
    for update in updates:
        entity_type = update["entity_type"]
        entity_id = update["entity_id"]
        cost = update["response_cost"]
        
        if entity_type == "user":
            if entity_id not in aggregated["user_list_transactions"]:
                aggregated["user_list_transactions"][entity_id] = 0
            aggregated["user_list_transactions"][entity_id] += cost
        
        # ... 其他實體類型
    
    return aggregated
```

### 7.4 花費警報資料格式

```python
# CallInfo - 預算檢查
call_info = {
    "user_id": "user-uuid-123",
    "max_budget": 500.00,
    "spend": 450.00,
    "token": "sk-1234...ABCD",
    "event_group": "user"  # user, team, key, organization, end_user
}

# 警報閾值
budget_alert_threshold = {
    "percentage": 0.8,  # 80%
    "channels": ["email", "slack", "webhook"],
    "recipients": ["admin@example.com"],
    "slack_webhook_url": "https://hooks.slack.com/..."
}
```

---

## 8. 速率限制資料格式

### 8.1 速率限制配置

```python
# API 金鑰速率限制
rate_limit_config = {
    "rpm_limit": 1000,      # 每分鐘請求數
    "tpm_limit": 1000000,   # 每分鐘 Token 數
    "max_parallel_requests": 10
}

# 團隊速率限制
team_rate_limit = {
    "rpm_limit": 5000,
    "tpm_limit": 5000000,
    "max_parallel_requests": 50
}

# 組織速率限制
org_rate_limit = {
    "rpm_limit": 10000,
    "tpm_limit": 10000000,
    "max_parallel_requests": 100
}
```

### 8.2 速率限制檢查資料格式

```python
# 速率限制鍵格式
ratelimit_keys = {
    "rpm": "ratelimit:{token}:rpm",
    "tpm": "ratelimit:{token}:tpm",
    "team_rpm": "ratelimit:team:{team_id}:rpm",
    "team_tpm": "ratelimit:team:{team_id}:tpm",
    "org_rpm": "ratelimit:org:{org_id}:rpm",
    "org_tpm": "ratelimit:org:{org_id}:tpm"
}

# 速率限制檢查結果
rate_limit_result = {
    "allowed": True,
    "remaining": 950,
    "reset_at": "2024-01-19T10:01:00Z",
    "limit": 1000,
    "type": "rpm"  # rpm 或 tpm
}

# 速率限制拒絕響應
rate_limit_response = {
    "error": {
        "message": "Rate limit exceeded. Please retry in 30 seconds.",
        "type": "rate_limit_error",
        "param": None,
        "code": 429
    },
    "headers": {
        "x-ratelimit-limit": "1000",
        "x-ratelimit-remaining": "0",
        "x-ratelimit-reset": "30",
        "retry-after": "30"
    }
}
```

### 8.3 Redis 速率限制 Lua 腳本

```python
# 速率限制 Lua 腳本
RATE_LIMIT_SCRIPT = """
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local current = tonumber(redis.call('GET', key) or '0')

if current >= limit then
    local ttl = redis.call('TTL', key)
    return {0, current, limit, ttl}
end

local new_count = redis.call('INCR', key)
if new_count == 1 then
    redis.call('EXPIRE', key, window)
end

return {1, new_count, limit, window}
"""

# 註冊腳本
redis_cache.async_register_script(
    name="rate_limit",
    script=RATE_LIMIT_SCRIPT
)

# 使用腳本
result = await redis_cache.evalsha(
    script_sha="...",  # 腳本 SHA
    keys=["ratelimit:sk-1234...ABCD:rpm"],
    args=[1000, 60]    # limit=1000, window=60秒
)
```

### 8.4 模型特定速率限制

```python
# 模型特定 RPM 限制
model_rpm_limit = {
    "gpt-4": 500,
    "gpt-4-turbo": 300,
    "claude-3-opus": 200,
    "claude-3-sonnet": 1000,
    "gpt-3.5-turbo": 5000
}

# 模型特定 TPM 限制
model_tpm_limit = {
    "gpt-4": 500000,
    "gpt-4-turbo": 1000000,
    "claude-3-opus": 200000,
    "claude-3-sonnet": 500000,
    "gpt-3.5-turbo": 2000000
}

# 使用配置
model_config = {
    "model": "gpt-4",
    "rpm_limit": model_rpm_limit["gpt-4"],
    "tpm_limit": model_tpm_limit["gpt-4"]
}
```

---

## 附錄 A: 常用資料庫操作函數

### A.1 用戶管理

```python
async def create_user(prisma_client, user_data):
    """創建用戶"""
    return await prisma_client.db.litellm_usertable.create(data=user_data)

async def get_user(prisma_client, user_id):
    """查詢用戶"""
    return await prisma_client.db.litellm_usertable.find_unique(
        where={"user_id": user_id}
    )

async def update_user(prisma_client, user_id, update_data):
    """更新用戶"""
    return await prisma_client.db.litellm_usertable.update(
        where={"user_id": user_id},
        data=update_data
    )

async def delete_user(prisma_client, user_id):
    """刪除用戶"""
    return await prisma_client.db.litellm_usertable.delete(
        where={"user_id": user_id}
    )

async def list_users(prisma_client, organization_id=None):
    """列出用戶"""
    where = {}
    if organization_id:
        where["organization_id"] = organization_id
    return await prisma_client.db.litellm_usertable.find_many(where=where)
```

### A.2 API 金鑰管理

```python
async def create_key(prisma_client, key_data):
    """創建 API 金鑰"""
    return await prisma_client.db.litellm_verificationtoken.create(
        data=key_data
    )

async def get_key(prisma_client, token):
    """查詢 API 金鑰"""
    return await prisma_client.db.litellm_verificationtoken.find_unique(
        where={"token": token}
    )

async def update_key_spend(prisma_client, token, spend_increment):
    """更新金鑰花費"""
    key = await prisma_client.db.litellm_verificationtoken.find_unique(
        where={"token": token}
    )
    new_spend = (key.spend or 0) + spend_increment
    return await prisma_client.db.litellm_verificationtoken.update(
        where={"token": token},
        data={"spend": new_spend}
    )

async def revoke_key(prisma_client, token):
    """撤銷金鑰"""
    return await prisma_client.db.litellm_verificationtoken.update(
        where={"token": token},
        data={"blocked": True}
    )
```

### A.3 花費查詢

```python
async def get_user_spend(prisma_client, user_id, start_date, end_date):
    """查詢用戶花費"""
    return await prisma_client.db.query_raw(
        query="""
            SELECT 
                DATE(startTime) as date,
                SUM(spend) as total_spend,
                SUM(prompt_tokens) as total_prompt_tokens,
                SUM(completion_tokens) as total_completion_tokens,
                COUNT(*) as request_count
            FROM "LiteLLM_SpendLogs"
            WHERE user = $1 AND startTime BETWEEN $2 AND $3
            GROUP BY DATE(startTime)
            ORDER BY date DESC
        """,
        args=[user_id, start_date, end_date]
    )

async def get_team_spend(prisma_client, team_id, start_date, end_date):
    """查詢團隊花費"""
    return await prisma_client.db.query_raw(
        query="""
            SELECT 
                DATE(startTime) as date,
                model,
                SUM(spend) as total_spend,
                COUNT(*) as request_count
            FROM "LiteLLM_SpendLogs"
            WHERE team_id = $1 AND startTime BETWEEN $2 AND $3
            GROUP BY DATE(startTime), model
            ORDER BY date DESC, total_spend DESC
        """,
        args=[team_id, start_date, end_date]
    )
```

---

## 附錄 B: 資料庫遷移指令

```bash
# 執行 Prisma Migration
npx prisma migrate dev

# 生成 Prisma Client
npx prisma generate

# 檢視資料庫內容
npx prisma studio

# 備份資料庫
pg_dump -h localhost -U postgres -d litellm > litellm_backup.sql

# 恢復資料庫
psql -h localhost -U postgres -d litellm < litellm_backup.sql
```

---

## 附錄 C: 健康檢查

```python
# 資料庫連線檢查
async def check_database_connection(prisma_client):
    try:
        await prisma_client.db.query_raw("SELECT 1")
        return {"status": "healthy", "database": "connected"}
    except Exception as e:
        return {"status": "unhealthy", "database": str(e)}

# Redis 連線檢查
async def check_redis_connection(redis_cache):
    try:
        await redis_cache.async_ping()
        return {"status": "healthy", "redis": "connected"}
    except Exception as e:
        return {"status": "unhealthy", "redis": str(e)}

# 花費追蹤檢查
async def check_spend_tracking(prisma_client):
    try:
        recent_spend = await prisma_client.db.litellm_spendlogs.find_first(
            order={"startTime": "desc"}
        )
        if recent_spend:
            return {
                "status": "healthy",
                "last_spend_log": recent_spend.startTime.isoformat()
            }
        return {"status": "warning", "message": "No spend logs found"}
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}
```
