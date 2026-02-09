# Central Platform 登录与权限系统设计文档

**文档日期**: 2026-02-09  
**项目名称**: central-platform  
**技术栈**: Go + Gin + GORM + Redis

---

## 1. 系统概述

### 1.1 架构层次

```
┌─────────────────────────────────────────────────────────────┐
│                        前端 (Frontend)                       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway (Gin)                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │   Handler   │  │  Middleware │  │      Router         │ │
│  │   Layer     │  │   Layer     │  │      Group          │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Business Layer                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │   UseCase   │  │    Module   │  │    Repository       │ │
│  │   Layer     │  │   Layer     │  │      Layer          │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Data Layer                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │   MySQL     │  │    Redis    │  │    MongoDB          │ │
│  │  (Primary)  │  │   (Cache)   │  │   (Optional)        │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 代码规范

根据项目 README，代码分层规范如下：

1. **Handler 层**: 可调用 usecase, module 层
2. **UseCase 层**: 可调用 module 层, repository 层
3. **Module 层**: 可调用 repository 层
4. **初始化**: 皆需使用 dig (依赖注入) 创建

---

## 2. 登录机制设计

### 2.1 API 层级定义

系统支持四种 API 访问级别，定义在 `service/internal/constant/api_level.go`：

```go
const (
    API_Level        = "api.level"
    API_Level_Member = "member"  // 会员级别
    API_Level_User   = "user"    // 用户级别
    API_Level_Agent  = "agent"   // 代理级别
    API_Level_App    = "app"     // 应用级别
)
```

### 2.2 登录流程

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   客户端      │────▶│   Handler    │────▶│   Module     │
│  (Frontend)  │     │  (member.go) │     │(member.go)   │
└──────────────┘     └──────────────┘     └──────┬───────┘
                                                  │
                                                  ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Token      │◀────│  Repository  │◀────│   MySQL      │
│   Redis      │     │(member_login)│     │  Member      │
└──────────────┘     └──────────────┘     └──────────────┘
```

### 2.3 登录接口

**端点**: `POST /member/login`

**请求结构** (`dto.MemberLoginReq`):
```go
type MemberLoginReq struct {
    Login    string `json:"login"`    // 登录账号
    Password string `json:"password"` // 密码
}
```

**登录验证逻辑**:
1. 接收登录请求，绑定 JSON 参数
2. 验证参数（账号、密码不能为空）
3. 查询数据库验证账号密码
4. 生成 Token 并存储到数据库和 Redis
5. 返回 Token 给客户端

### 2.4 Token 机制

#### 2.4.1 Token 结构

```go
type Token struct {
    Token        string    `gorm:"column:token"`
    UserLogin    string    `gorm:"column:user_login"`    // 用户登录名
    AgentLogin   string    `gorm:"column:agent_login"`   // 代理登录名
    MemberLogin  string    `gorm:"column:member_login"`  // 会员登录名
    IssuedTime   time.Time `gorm:"column:issued_time"`   // 发行时间
    LastUsedTime time.Time `gorm:"column:last_used_time"`// 最后使用时间
    VerifyType   int8      `gorm:"column:verify_type"`   // 0=普通, 1=白名单
}
```

#### 2.4.2 Token 存储策略

- **数据库存储**: Token 持久化存储在 MySQL `token` 表中
- **Redis 缓存**: Token 同时缓存到 Redis，用于快速验证，设置过期时间

```go
// Token 缓存 Key 格式: token:{level}:{token}
// 示例: token:member:abc123def456
```

#### 2.4.3 Token 超时配置

```go
type TimeOutConfig struct {
    MemberTimeout   int    // 会员 Token 超时时间（秒）
    UserTimeout     int    // 用户 Token 超时时间（秒）
    ResellerTimeout int    // 代理 Token 超时时间（秒）
}
```

默认超时时间为 7000 秒，可从数据库 `config` 表动态配置。

---

## 3. 权限授权机制

### 3.1 授权架构

```
┌────────────────────────────────────────────────────────────┐
│                    Middleware Layer                        │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐ │
│  │ MemberAuthorizer│ │ UserAuthorizer │ │ AgentAuthorizer│ │
│  │  (会员授权)      │ │   (用户授权)    │ │   (代理授权)   │ │
│  └────────────────┘ └────────────────┘ └────────────────┘ │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────┐
│                   Authorizer Module                        │
│  ┌──────────────────────────────────────────────────────┐ │
│  │              IAuthorizer Interface                    │ │
│  │         IsAuthorized(ctx, token) bool                 │ │
│  └──────────────────────────────────────────────────────┘ │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐       │
│  │MemberLevel   │ │ UserLevel    │ │ AgentLevel   │       │
│  │Authorizer    │ │ Authorizer   │ │ Authorizer   │       │
│  └──────────────┘ └──────────────┘ └──────────────┘       │
└────────────────────────────────────────────────────────────┘
```

### 3.2 授权中间件

#### 3.2.1 MemberAuthorizer (会员授权)

**文件**: `service/internal/controller/middleware/memberAuthorizer.go`

**授权流程**:
1. 从请求头获取 `token`
2. 检查 Redis 缓存中是否存在该 Token
3. 如缓存未命中，查询数据库验证 Token 有效性
4. 验证通过则设置 Token 到缓存并放行
5. 验证失败返回 401 Unauthorized

```go
func (m memberAuthorizer) IsAuthorized(ctx *gin.Context) {
    token := ctx.GetHeader("token")
    if token == "" {
        ctxs.SetError(ctx, errs.UnauthorizedError())
        ctx.Abort()
        return
    }

    // 1. 检查 Redis 缓存
    hasToken := m.in.UseCase.TokenUseCase.CheckToken(ctx, constant.API_Level_Member, token)
    if hasToken {
        ctx.Next()
        return
    }

    // 2. 查询数据库验证
    if isAuthorized := m.in.Module.Authorizer.Member.IsAuthorized(ctx, token); isAuthorized {
        m.in.UseCase.TokenUseCase.SetToken(ctx, constant.API_Level_Member, token)
        ctx.Next()
        return
    }

    ctxs.SetError(ctx, errs.UnauthorizedError())
    ctx.Abort()
}
```

#### 3.2.2 UserAuthorizer (用户授权)

**文件**: `service/internal/controller/middleware/userAuthorizer.go`

与 MemberAuthorizer 逻辑相同，但验证 `user` 级别的 Token。

#### 3.2.3 AgentAuthorizer (代理授权)

**文件**: `service/internal/controller/middleware/agentAuthorizer.go`

与 MemberAuthorizer 逻辑相同，但验证 `agent` 级别的 Token。

### 3.3 授权验证实现

**文件**: `service/internal/core/module/authorizer.go`

```go
type Authorizer struct {
    Member IAuthorizer
    User   IAuthorizer
    Agent  IAuthorizer
}

type IAuthorizer interface {
    IsAuthorized(ctx context.Context, token string) bool
}
```

#### 3.3.1 会员级别验证

```go
func (m *MemberLevelAuthorizer) IsAuthorized(ctx context.Context, token string) bool {
    db := m.in.DB.GetDB(ctx)
    memberByToken, err := m.in.Repository.MemberLoginRepo.GetMemberLoginByToken(ctx, db, token)
    
    if err != nil {
        m.in.Logger.Error(ctx, err)
        return false
    }
    
    if memberByToken != nil {
        return true
    }
    return false
}
```

### 3.4 路由分组与权限绑定

**文件**: `service/internal/app/apis/app.go`

```go
func (s *Server) Run(ctx context.Context) error {
    router := gin.New()
    
    // 基础路由（无需授权）
    s.setBaseRoute(router)
    
    baseGroup := router.Group("central-platform")
    
    // 公开路由
    s.setPublicRouter(baseGroup)
    
    // 会员路由（需会员授权）
    s.setMemberRouter(baseGroup)
    
    // 用户路由（需用户授权）
    s.setUserRouter(baseGroup)
    
    // 代理路由（需代理授权）
    s.setAgentRouter(baseGroup)
}
```

#### 3.4.1 会员路由组

```go
func (s *Server) setMemberRouter(router *gin.RouterGroup) {
    // 应用会员授权中间件
    router.Use(s.In.Middleware.MemberAuthorizer.IsAuthorized)
    
    // 会员相关 API
    s.setMemberApiRoute(router)
    s.setPromoRoute(router)
}
```

#### 3.4.2 会员 API 路由

**文件**: `service/internal/app/apis/router_member.go`

```go
func (s *Server) setMemberApiRoute(router *gin.RouterGroup) {
    // 公开登录接口（无需授权）
    router.POST("/member/login", s.In.Handler.MemberHandler.Login)
    
    // 需授权的会员接口
    router.GET("/member", s.In.Handler.MemberHandler.GetMemberByID)
    router.GET("/member/list", s.In.Handler.MemberHandler.GetMember)
    router.POST("/member", s.In.Handler.MemberHandler.CreateMember)
    router.PUT("/member", s.In.Handler.MemberHandler.UpdateMember)
    router.DELETE("/member", s.In.Handler.MemberHandler.DeleteMember)
}
```

---

## 4. 数据库表结构

### 4.1 表关系图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          登录与权限相关表                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌──────────────┐         ┌──────────────┐         ┌──────────────┐   │
│   │   member     │         │     token    │         │    level     │   │
│   │   (会员表)    │◀───────▶│   (令牌表)   │◀───────▶│  (级别表)    │   │
│   └──────────────┘         └──────────────┘         └──────────────┘   │
│          │                                                           │
│          │                                                            │
│          ▼                                                            │
│   ┌──────────────┐         ┌──────────────┐         ┌──────────────┐   │
│   │  member_login│         │    user      │         │    agent     │   │
│   │  (会员登录信息)│         │   (用户表)   │         │  (代理表)    │   │
│   └──────────────┘         └──────────────┘         └──────────────┘   │
│                                                                         │
│   ┌──────────────┐                                                     │
│   │    config    │                                                     │
│   │  (系统配置表) │                                                     │
│   └──────────────┘                                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 核心表结构

#### 4.2.1 member (会员表)

**用途**: 存储会员基本信息

| 字段名 | 类型 | 说明 |
|--------|------|------|
| login | string (PK) | 会员登录名（主键） |
| name | string | 会员名称 |
| level_code | int | 会员级别代码 |
| status | int | 用户状态 (0:冻结, 1:可用, 2:密码错误锁定) |
| suspension_reason | int8 | 禁用原因 |
| locked | int | 1=停权, 0=正常 |
| passwd | string | 密码 Hash |
| security_code | string | 安全密码 |
| salt | string | 密码盐 |
| currency_code | string | 币种 |
| credit | float64 | 可用信用额 |
| balance | float64 | 可用余额 |
| agent_login | string | 所属代理登录名 |
| referral_login | string | 推荐人登录名 |
| added_time | datetime | 创建时间 |
| last_login_time | datetime | 最后登录时间 |
| failed_attempt_count | int | 登录失败次数 |
| lang_code | string | 语言代码 (zh_CN, en_US) |
| is_trial | int8 | 是否试用会员 |
| frozen_status | int8 | 支付冻结状态 |
| name_passed | int8 | 实名认证状态 |
| offsite_count | int | 异地错误次数 |

#### 4.2.2 token (令牌表)

**用途**: 存储用户登录 Token

| 字段名 | 类型 | 说明 |
|--------|------|------|
| token | string | Token 字符串 |
| user_login | string | 用户登录名 |
| agent_login | string | 代理登录名 |
| member_login | string | 会员登录名 |
| issued_time | datetime | Token 发行时间 |
| last_used_time | datetime | 最后使用时间 |
| verify_type | int8 | 验证类型 (0=普通, 1=白名单) |

**关联关系**:
- 通过 `member_login` 关联 `member` 表
- 通过 `user_login` 关联 `user` 表
- 通过 `agent_login` 关联 `agent` 表

#### 4.2.3 member_login (会员登录信息表)

**用途**: 存储会员登录相关的安全信息

| 字段名 | 类型 | 说明 |
|--------|------|------|
| member_login | string | 会员登录账号 |
| mobile | string | 会员手机号 |
| mobile_mask | string | 手机末四码 |
| added_time | datetime | 新增时间 |
| sms_locked | int | SMS 锁定状态 (0=正常, 1=锁定) |
| sms_locked_time | datetime | SMS 锁定时间 |
| sms_failure_count | int | SMS 验证失败次数 |

#### 4.2.4 level (会员级别表)

**用途**: 定义会员级别和权益

| 字段名 | 类型 | 说明 |
|--------|------|------|
| code | int (PK) | 会员级别代码 |
| name | string | 会员级别名称 |
| point_total | float64 | 升级所需有效投注 |
| deposit_total | float64 | 升级所需存款总额 |
| deposit_times | int | 升级所需存款次数 |
| locked | int | 锁定后不可自动升级 |
| feature | int | 功能类型 |
| verification_type | int8 | 会员验证类型 (0=安全码, 1=SMS) |
| enable_deposit_otp | int8 | 启用入款 OTP |
| is_bot | int8 | 层级状态 (0=一般, 1=机器人) |
| sort | int8 | 排序 |
| thirdparty_transfer_limit | int64 | 三方转账限额 |

#### 4.2.5 config (系统配置表)

**用途**: 存储系统全局配置，包含 Token 超时配置

| 字段名 | 类型 | 说明 |
|--------|------|------|
| code | string (PK) | 配置代码 |
| member_timeout | int | 会员 Token 超时时间（秒） |
| user_timeout | int | 用户 Token 超时时间（秒） |
| reseller_timeout | int | 代理 Token 超时时间（秒） |
| lock_timeout | int | 锁定超时时间 |
| ... | ... | 其他配置项 |

---

## 5. Token 验证 SQL 查询

### 5.1 会员 Token 验证

**文件**: `service/internal/repository/sql/token/get_member_login_by_token.sql`

```sql
SELECT `token`.`member_login` AS login, `member`.`name` AS name 
FROM `token`
LEFT JOIN `member` ON `token`.`member_login` = `member`.`login`
WHERE `token`.`token` = ?
```

### 5.2 用户 Token 验证

**文件**: `service/internal/repository/sql/token/get_user_login_by_token.sql`

```sql
SELECT `token`.`user_login` AS login, `user`.`login` AS name 
FROM `token`
LEFT JOIN `user` ON `token`.`user_login` = `user`.`login`
WHERE `token`.`token` = ?
```

### 5.3 代理 Token 验证

**文件**: `service/internal/repository/sql/token/get_agent_login_by_token.sql`

```sql
SELECT `token`.`agent_login` AS login, `agent`.`login` AS name 
FROM `token`
LEFT JOIN `agent` ON `token`.`agent_login` = `agent`.`login`
WHERE `token`.`token` = ?
```

### 5.4 App Token 验证

**文件**: `service/internal/repository/sql/token/get_app_login_by_token.sql`

```sql
SELECT `app`.`key` AS login, `app`.`name` AS name 
FROM `app`
WHERE `app`.`secret` = ? AND `app`.`name` = ?
```

---

## 6. RBAC 权限管理设计 (authz_permission)

### 6.1 概述

当前系统仅实现了基于 Token 的身份认证（Authentication），但缺少细粒度的权限控制（Authorization）。以下为建议的 RBAC (Role-Based Access Control) 权限管理模块设计。

### 6.2 RBAC 架构

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         RBAC 权限管理架构                                    │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│   ┌──────────────────────────────────────────────────────────────────┐    │
│   │                        用户层 (User)                              │    │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐          │    │
│   │  │  member  │  │   user   │  │  agent   │  │   app    │          │    │
│   │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘          │    │
│   │       │             │             │             │                │    │
│   │       └─────────────┴──────┬──────┴─────────────┘                │    │
│   │                            │                                      │    │
│   │                            ▼                                      │    │
│   │                   ┌────────────────┐                              │    │
│   │                   │   user_role    │                              │    │
│   │                   │ (用户角色关联表)  │                              │    │
│   │                   └───────┬────────┘                              │    │
│   │                           │                                       │    │
│   └───────────────────────────┼───────────────────────────────────────┘    │
│                               │                                            │
│                               ▼                                            │
│   ┌──────────────────────────────────────────────────────────────────┐    │
│   │                        角色层 (Role)                              │    │
│   │  ┌──────────────────────────────────────────────────────────┐   │    │
│   │  │                        role                               │   │    │
│   │  │                   (角色定义表)                             │   │    │
│   │  └─────────────────────────┬────────────────────────────────┘   │    │
│   │                            │                                    │    │
│   │                            ▼                                    │    │
│   │                   ┌────────────────┐                            │    │
│   │                   │ role_permission│                            │    │
│   │                   │(角色权限关联表)  │                            │    │
│   │                   └───────┬────────┘                            │    │
│   │                           │                                     │    │
│   └───────────────────────────┼─────────────────────────────────────┘    │
│                               │                                          │
│                               ▼                                          │
│   ┌──────────────────────────────────────────────────────────────────┐  │
│   │                        权限层 (Permission)                        │  │
│   │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │  │
│   │  │  permission  │  │     menu     │  │   resource   │            │  │
│   │  │   (权限表)    │  │   (菜单表)   │  │   (资源表)   │            │  │
│   │  └──────────────┘  └──────────────┘  └──────────────┘            │  │
│   │                                                                  │  │
│   │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │  │
│   │  │     api      │  │   action     │  │    policy    │            │  │
│   │  │   (API表)    │  │   (操作表)   │  │  (策略规则表) │            │  │
│   │  └──────────────┘  └──────────────┘  └──────────────┘            │  │
│   └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 6.3 权限管理表结构

#### 6.3.1 authz_role (角色表)

**用途**: 定义系统中的角色

| 字段名 | 类型 | 说明 |
|--------|------|------|
| id | bigint (PK) | 角色ID |
| code | varchar(50) | 角色编码（唯一） |
| name | varchar(100) | 角色名称 |
| description | varchar(255) | 角色描述 |
| level | varchar(20) | 适用级别 (member/user/agent/app) |
| status | tinyint | 状态 (0=禁用, 1=启用) |
| created_at | datetime | 创建时间 |
| updated_at | datetime | 更新时间 |
| created_by | varchar(50) | 创建人 |

**示例数据**:
| code | name | level | description |
|------|------|-------|-------------|
| SUPER_ADMIN | 超级管理员 | user | 系统超级管理员 |
| OPERATOR | 运营人员 | user | 日常运营人员 |
| VIP_MEMBER | VIP会员 | member | 高级会员角色 |
| NORMAL_MEMBER | 普通会员 | member | 普通会员角色 |
| GENERAL_AGENT | 总代理 | agent | 总代理角色 |

#### 6.3.2 authz_user_role (用户角色关联表)

**用途**: 建立用户与角色的多对多关系（N:M），一个用户可以拥有多个角色，一个角色可以分配给多个用户。

**表结构**:

| 字段名 | 类型 | 说明 |
|--------|------|------|
| id | bigint (PK) | 关联ID，自增主键 |
| user_type | varchar(20) | 用户类型 (member/user/agent/app) |
| user_login | varchar(50) | 用户登录名，与 user_type 组合确定唯一用户 |
| role_id | bigint | 角色ID，外键关联 authz_role 表 |
| assigned_at | datetime | 角色分配时间 |
| assigned_by | varchar(50) | 分配人，记录操作者 |
| expire_at | datetime | 角色过期时间，NULL 表示永久有效 |

**索引设计**:
- `idx_user`: (user_type, user_login) - 快速查询用户的所有角色
- `idx_role_id`: (role_id) - 快速查询拥有某角色的所有用户
- `idx_expire`: (expire_at) - 用于清理过期角色

**示例数据**:

| id | user_type | user_login | role_id | assigned_at | assigned_by | expire_at |
|----|-----------|------------|---------|-------------|-------------|-----------|
| 1 | user | admin01 | 1 | 2026-01-01 10:00:00 | system | NULL |
| 2 | user | operator01 | 2 | 2026-01-05 14:30:00 | admin01 | NULL |
| 3 | member | john_doe | 4 | 2026-01-10 09:15:00 | system | NULL |
| 4 | member | vip_user | 3 | 2026-01-15 16:45:00 | admin01 | 2026-12-31 23:59:59 |
| 5 | agent | agent001 | 5 | 2026-01-20 11:20:00 | admin01 | NULL |

**数据说明**:
- 第 1 行: 后台用户 admin01 拥有超级管理员角色(role_id=1)，永久有效
- 第 2 行: 后台用户 operator01 拥有运营人员角色(role_id=2)
- 第 3 行: 会员 john_doe 拥有普通会员角色(role_id=4)
- 第 4 行: 会员 vip_user 拥有 VIP 会员角色(role_id=3)，但有过期时间
- 第 5 行: 代理 agent001 拥有总代理角色(role_id=5)

**典型查询场景**:

```sql
-- 1. 查询某个用户的所有角色
SELECT r.* 
FROM authz_role r
INNER JOIN authz_user_role ur ON r.id = ur.role_id
WHERE ur.user_type = 'member' 
  AND ur.user_login = 'john_doe'
  AND (ur.expire_at IS NULL OR ur.expire_at > NOW());

-- 2. 查询拥有某角色的所有用户
SELECT ur.user_type, ur.user_login, m.name as user_name
FROM authz_user_role ur
LEFT JOIN member m ON ur.user_login = m.login AND ur.user_type = 'member'
LEFT JOIN `user` u ON ur.user_login = u.login AND ur.user_type = 'user'
WHERE ur.role_id = 3
  AND (ur.expire_at IS NULL OR ur.expire_at > NOW());

-- 3. 检查用户是否拥有特定角色
SELECT COUNT(*) as has_role
FROM authz_user_role
WHERE user_type = 'user' 
  AND user_login = 'admin01'
  AND role_id = 1
  AND (expire_at IS NULL OR expire_at > NOW());

-- 4. 查询即将过期的角色分配
SELECT ur.*, r.name as role_name
FROM authz_user_role ur
JOIN authz_role r ON ur.role_id = r.id
WHERE ur.expire_at IS NOT NULL 
  AND ur.expire_at BETWEEN NOW() AND DATE_ADD(NOW(), INTERVAL 7 DAY);
```

#### 6.3.3 authz_permission (权限表)

**用途**: 定义系统中的权限项

| 字段名 | 类型 | 说明 |
|--------|------|------|
| id | bigint (PK) | 权限ID |
| code | varchar(100) | 权限编码（唯一） |
| name | varchar(100) | 权限名称 |
| type | varchar(20) | 权限类型 (menu/api/resource/action) |
| resource | varchar(100) | 资源标识 |
| action | varchar(50) | 操作类型 (create/read/update/delete/execute) |
| description | varchar(255) | 权限描述 |
| status | tinyint | 状态 (0=禁用, 1=启用) |
| created_at | datetime | 创建时间 |

**示例数据**:
| code | name | type | resource | action |
|------|------|------|----------|--------|
| member:create | 创建会员 | api | /member | POST |
| member:read | 查看会员 | api | /member | GET |
| member:update | 修改会员 | api | /member | PUT |
| member:delete | 删除会员 | api | /member | DELETE |
| promo:view | 查看优惠 | menu | promo | read |
| deposit:execute | 存款操作 | action | deposit | execute |

#### 6.3.4 authz_role_permission (角色权限关联表)

**用途**: 建立角色与权限的多对多关系

| 字段名 | 类型 | 说明 |
|--------|------|------|
| id | bigint (PK) | 关联ID |
| role_id | bigint | 角色ID |
| permission_id | bigint | 权限ID |
| granted_at | datetime | 授权时间 |
| granted_by | varchar(50) | 授权人 |
| conditions | json | 权限条件 (可选，如数据范围) |

**索引**:
- `idx_role_id`: (role_id) - 快速查询角色权限
- `idx_permission_id`: (permission_id) - 快速查询权限分配的角色

#### 6.3.5 authz_menu (菜单表)

**用途**: 定义前端菜单结构

| 字段名 | 类型 | 说明 |
|--------|------|------|
| id | bigint (PK) | 菜单ID |
| parent_id | bigint | 父菜单ID (0=根菜单) |
| code | varchar(50) | 菜单编码 |
| name | varchar(100) | 菜单名称 |
| icon | varchar(50) | 图标 |
| path | varchar(200) | 路由路径 |
| component | varchar(200) | 前端组件路径 |
| sort_order | int | 排序顺序 |
| level | varchar(20) | 适用级别 (member/user/agent/app) |
| status | tinyint | 状态 (0=禁用, 1=启用) |
| created_at | datetime | 创建时间 |
| updated_at | datetime | 更新时间 |

#### 6.3.6 authz_role_menu (角色菜单关联表)

**用途**: 建立角色与菜单的可见性关系

| 字段名 | 类型 | 说明 |
|--------|------|------|
| id | bigint (PK) | 关联ID |
| role_id | bigint | 角色ID |
| menu_id | bigint | 菜单ID |
| granted_at | datetime | 授权时间 |

#### 6.3.7 authz_api (API接口表)

**用途**: 定义系统中所有API接口

| 字段名 | 类型 | 说明 |
|--------|------|------|
| id | bigint (PK) | API ID |
| path | varchar(200) | API路径 |
| method | varchar(10) | HTTP方法 (GET/POST/PUT/DELETE) |
| name | varchar(100) | API名称 |
| description | varchar(255) | API描述 |
| level | varchar(20) | 适用级别 (member/user/agent/app) |
| status | tinyint | 状态 (0=禁用, 1=启用) |
| created_at | datetime | 创建时间 |

**示例数据**:
| path | method | name | level |
|------|--------|------|-------|
| /member | GET | 获取会员信息 | member |
| /member/list | GET | 获取会员列表 | user |
| /member | POST | 创建会员 | user |
| /promo | GET | 获取优惠活动 | member |

#### 6.3.8 authz_permission_api (权限API关联表)

**用途**: 关联权限与具体的API接口

| 字段名 | 类型 | 说明 |
|--------|------|------|
| id | bigint (PK) | 关联ID |
| permission_id | bigint | 权限ID |
| api_id | bigint | API ID |

### 6.4 权限关系图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         权限管理表关系图                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────┐       ┌─────────────────┐       ┌─────────────┐          │
│   │    user     │◀─────▶│  authz_user_role│◀─────▶│  authz_role │          │
│   │   (用户表)   │  N:M  │  (用户角色关联)  │  N:M  │  (角色表)    │          │
│   └─────────────┘       └─────────────────┘       └──────┬──────┘          │
│                                                          │                  │
│                                                          │ N:M              │
│                                                          ▼                  │
│   ┌─────────────┐       ┌─────────────────┐       ┌─────────────┐          │
│   │  authz_api  │◀─────▶│authz_permission_│◀─────▶│authz_permiss│          │
│   │   (API表)   │  N:M  │    api          │  N:M  │  ion        │          │
│   └─────────────┘       │  (权限API关联)   │       │  (权限表)    │          │
│                         └─────────────────┘       └─────────────┘          │
│                                                          │                  │
│                                                          │ N:M              │
│                                                          ▼                  │
│   ┌─────────────┐       ┌─────────────────┐       ┌─────────────┐          │
│   │  authz_menu │◀─────▶│ authz_role_menu │◀─────▶│authz_role_  │          │
│   │   (菜单表)   │  N:M  │ (角色菜单关联)   │  N:M  │permission   │          │
│   └─────────────┘       └─────────────────┘       │(角色权限关联)│          │
│                                                   └─────────────┘          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.5 权限检查流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           权限检查流程图                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   请求进入                                                                    │
│      │                                                                       │
│      ▼                                                                       │
│   ┌─────────────────────────────────────┐                                   │
│   │  1. Token 认证 (Authentication)      │                                   │
│   │     - 验证 Token 有效性               │                                   │
│   │     - 获取用户信息                    │                                   │
│   └──────────────┬──────────────────────┘                                   │
│                  │                                                           │
│                  ▼                                                           │
│   ┌─────────────────────────────────────┐                                   │
│   │  2. 获取用户角色                      │                                   │
│   │     - 查询 user_type + user_login    │                                   │
│   │     - 获取 authz_user_role           │                                   │
│   │     - 获取所有生效的角色              │                                   │
│   └──────────────┬──────────────────────┘                                   │
│                  │                                                           │
│                  ▼                                                           │
│   ┌─────────────────────────────────────┐                                   │
│   │  3. 获取角色权限                      │                                   │
│   │     - 查询 authz_role_permission     │                                   │
│   │     - 获取所有权限编码                │                                   │
│   └──────────────┬──────────────────────┘                                   │
│                  │                                                           │
│                  ▼                                                           │
│   ┌─────────────────────────────────────┐                                   │
│   │  4. 权限验证 (Authorization)         │                                   │
│   │     - 匹配请求路径 + 方法            │                                   │
│   │     - 查询 authz_api                 │                                   │
│   │     - 查询 authz_permission_api      │                                   │
│   │     - 检查权限是否在用户权限列表      │                                   │
│   └──────────────┬──────────────────────┘                                   │
│                  │                                                           │
│        ┌─────────┴─────────┐                                                │
│        │                   │                                                │
│        ▼                   ▼                                                │
│   ┌─────────┐         ┌─────────┐                                          │
│   │  允许   │         │  拒绝   │                                          │
│   │ 访问   │         │ 403     │                                          │
│   └─────────┘         └─────────┘                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.6 Go 实体定义示例

#### 6.6.1 Role (角色实体)

```go
// service/internal/model/po/authz_role.go
package po

import "time"

type AuthzRole struct {
    ID          int64     `gorm:"primaryKey;column:id"`
    Code        string    `gorm:"uniqueIndex;column:code"`
    Name        string    `gorm:"column:name"`
    Description string    `gorm:"column:description"`
    Level       string    `gorm:"column:level"` // member/user/agent/app
    Status      int8      `gorm:"column:status"`
    CreatedAt   time.Time `gorm:"column:created_at"`
    UpdatedAt   time.Time `gorm:"column:updated_at"`
    CreatedBy   string    `gorm:"column:created_by"`
}

func (*AuthzRole) TableName() string {
    return "authz_role"
}
```

#### 6.6.2 UserRole (用户角色关联实体)

```go
// service/internal/model/po/authz_user_role.go
package po

import "time"

type AuthzUserRole struct {
    ID         int64      `gorm:"primaryKey;column:id"`
    UserType   string     `gorm:"index:idx_user;column:user_type"`   // member/user/agent/app
    UserLogin  string     `gorm:"index:idx_user;column:user_login"`  // 用户登录名
    RoleID     int64      `gorm:"index:idx_role_id;column:role_id"`  // 角色ID
    AssignedAt time.Time  `gorm:"column:assigned_at"`                // 分配时间
    AssignedBy string     `gorm:"column:assigned_by"`                // 分配人
    ExpireAt   *time.Time `gorm:"column:expire_at"`                  // 过期时间，NULL=永久
}

func (*AuthzUserRole) TableName() string {
    return "authz_user_role"
}

// IsExpired 检查角色是否已过期
func (ur *AuthzUserRole) IsExpired() bool {
    if ur.ExpireAt == nil {
        return false
    }
    return time.Now().After(*ur.ExpireAt)
}

// IsValid 检查角色是否有效（未过期）
func (ur *AuthzUserRole) IsValid() bool {
    return !ur.IsExpired()
}
```

#### 6.6.3 Permission (权限实体)

```go
// service/internal/model/po/authz_permission.go
package po

import "time"

type AuthzPermission struct {
    ID          int64     `gorm:"primaryKey;column:id"`
    Code        string    `gorm:"uniqueIndex;column:code"`
    Name        string    `gorm:"column:name"`
    Type        string    `gorm:"column:type"` // menu/api/resource/action
    Resource    string    `gorm:"column:resource"`
    Action      string    `gorm:"column:action"`
    Description string    `gorm:"column:description"`
    Status      int8      `gorm:"column:status"`
    CreatedAt   time.Time `gorm:"column:created_at"`
}

func (*AuthzPermission) TableName() string {
    return "authz_permission"
}
```

#### 6.6.3 UserRole (用户角色关联实体)

```go
// service/internal/model/po/authz_user_role.go
package po

import "time"

type AuthzUserRole struct {
    ID         int64      `gorm:"primaryKey;column:id"`
    UserType   string     `gorm:"index:idx_user;column:user_type"`
    UserLogin  string     `gorm:"index:idx_user;column:user_login"`
    RoleID     int64      `gorm:"index:idx_role_id;column:role_id"`
    AssignedAt time.Time  `gorm:"column:assigned_at"`
    AssignedBy string     `gorm:"column:assigned_by"`
    ExpireAt   *time.Time `gorm:"column:expire_at"`
}

func (*AuthzUserRole) TableName() string {
    return "authz_user_role"
}
```

#### 6.6.4 RolePermission (角色权限关联实体)

```go
// service/internal/model/po/authz_role_permission.go
package po

import "time"

type AuthzRolePermission struct {
    ID            int64     `gorm:"primaryKey;column:id"`
    RoleID        int64     `gorm:"index:idx_role_id;column:role_id"`
    PermissionID  int64     `gorm:"index:idx_permission_id;column:permission_id"`
    GrantedAt     time.Time `gorm:"column:granted_at"`
    GrantedBy     string    `gorm:"column:granted_by"`
    Conditions    string    `gorm:"column:conditions"` // JSON格式
}

func (*AuthzRolePermission) TableName() string {
    return "authz_role_permission"
}
```

### 6.7 权限中间件实现示例

```go
// service/internal/controller/middleware/permission.go

package middleware

import (
    "central-platform/service/internal/constant"
    "central-platform/service/internal/errs"
    "central-platform/service/internal/util/ctxs"
    "github.com/gin-gonic/gin"
    "strings"
)

type permissionMiddleware struct {
    in digIn
}

// CheckPermission 检查用户是否有权限访问该 API
func (p *permissionMiddleware) CheckPermission(ctx *gin.Context) {
    // 1. 获取当前用户信息和 Token
    token := ctx.GetHeader("token")
    if token == "" {
        ctxs.SetError(ctx, errs.UnauthorizedError())
        ctx.Abort()
        return
    }

    // 2. 获取请求信息
    path := ctx.Request.URL.Path
    method := ctx.Request.Method
    
    // 3. 从 Token 获取用户类型和登录名
    userType, userLogin, err := p.getUserFromToken(ctx, token)
    if err != nil {
        ctxs.SetError(ctx, errs.UnauthorizedError())
        ctx.Abort()
        return
    }

    // 4. 检查权限
    hasPermission, err := p.in.Module.AuthzModule.CheckPermission(ctx, userType, userLogin, path, method)
    if err != nil {
        ctxs.SetError(ctx, errs.InternalServerError().Log(ctx, err))
        ctx.Abort()
        return
    }

    if !hasPermission {
        ctxs.SetError(ctx, errs.ForbiddenError()) // 403 Forbidden
        ctx.Abort()
        return
    }

    // 5. 设置用户上下文
    ctx.Set(constant.UserTypeKey, userType)
    ctx.Set(constant.UserLoginKey, userLogin)
    
    ctx.Next()
}

// getUserFromToken 从 Token 解析用户信息
func (p *permissionMiddleware) getUserFromToken(ctx *gin.Context, token string) (userType, userLogin string, err error) {
    // 根据 token 查询数据库获取用户类型和登录名
    // ...
    return
}
```

### 6.8 权限缓存策略

为提高权限检查性能，建议采用多级缓存：

```go
// 1. Redis 缓存用户权限列表
// Key: authz:user:{user_type}:{user_login}:permissions
// Value: ["member:read", "member:update", "promo:view", ...]
// TTL: 5分钟

// 2. Redis 缓存 API 与权限的映射
// Key: authz:api:{path}:{method}
// Value: permission_code
// TTL: 10分钟

// 3. 本地缓存 (可选)
// 使用 sync.Map 缓存热点权限数据
```

---

## 7. 核心组件说明

### 7.1 Repository 层

**文件**: `service/internal/repository/member_login.go`

```go
type IMemberLoginRepository interface {
    GetMemberLoginByToken(cc context.Context, db *gorm.DB, token string) (*po.LoginByToken, error)
    GetUserLoginByToken(cc context.Context, db *gorm.DB, token string) (*po.LoginByToken, error)
    GetAppLoginByToken(cc context.Context, db *gorm.DB, token, tokenLogin string) (*po.LoginByToken, error)
    GetAgentLoginByToken(cc context.Context, db *gorm.DB, token string) (*po.LoginByToken, error)
    SetToken(ctx context.Context, level, token string, expiration time.Duration) error
    GetToken(ctx context.Context, level, token string) error
}
```

### 7.2 UseCase 层

**文件**: `service/internal/core/usecase/token.go`

```go
type TokenUseCase struct {
    in digIn
}

// SetToken 设置 Token 到 Redis 缓存
func (t *TokenUseCase) SetToken(ctx context.Context, level, token string) {
    // 根据 level 获取对应的超时配置
    // 存储到 Redis: key = token:{level}:{token}
}

// CheckToken 检查 Token 是否存在于 Redis 缓存
func (t *TokenUseCase) CheckToken(ctx context.Context, level, token string) bool {
    // 查询 Redis 验证 Token
}
```

### 7.3 Module 层

**文件**: `service/internal/core/module/authorizer.go`

```go
type Authorizer struct {
    Member IAuthorizer
    User   IAuthorizer
    Agent  IAuthorizer
}

// 不同级别的授权器实现
- MemberLevelAuthorizer
- UserLevelAuthorizer
- AgentLevelAuthorizer
```

---

## 8. 安全机制

### 8.1 Token 安全

1. **Token 生成**: 使用安全随机算法生成
2. **Token 存储**: 数据库 + Redis 双重存储
3. **Token 过期**: 可配置的过期时间
4. **Token 验证**: 双层验证（缓存+数据库）

### 8.2 登录安全

1. **密码加密**: 使用 Salt + Hash 存储
2. **失败锁定**: 登录失败次数限制
3. **异地检测**: offsite_count 异地登录错误计数
4. **SMS 锁定**: sms_locked SMS 验证失败锁定

### 8.3 访问控制

1. **中间件授权**: 每个路由组应用对应的授权中间件
2. **分层权限**: member / user / agent / app 四级权限
3. **Token 刷新**: 每次验证更新 last_used_time

---

## 9. 表清单 (Table List)

### 9.1 现有表

| 序号 | 表名 | 说明 | 实体文件 |
|------|------|------|----------|
| 1 | member | 会员主表 | service/internal/model/po/member.go |
| 2 | member_login | 会员登录信息表 | service/internal/model/po/member_login.go |
| 3 | token | 登录令牌表 | service/internal/model/po/token.go |
| 4 | level | 会员级别表 | service/internal/model/po/level.go |
| 5 | user | 用户表 | 运营后台用户 |
| 6 | agent | 代理表 | 代理账号信息 |
| 7 | app | 应用表 | App 授权信息 |
| 8 | config | 系统配置表 | Token 超时配置等 |

### 9.2 建议新增权限管理表

| 序号 | 表名 | 说明 | 用途 |
|------|------|------|------|
| 1 | authz_role | 角色表 | 定义角色 |
| 2 | authz_user_role | 用户角色关联表 | 用户-角色多对多关系 |
| 3 | authz_permission | 权限表 | 定义权限项 |
| 4 | authz_role_permission | 角色权限关联表 | 角色-权限多对多关系 |
| 5 | authz_menu | 菜单表 | 前端菜单定义 |
| 6 | authz_role_menu | 角色菜单关联表 | 角色-菜单可见性关系 |
| 7 | authz_api | API接口表 | 定义所有API |
| 8 | authz_permission_api | 权限API关联表 | 权限-API映射关系 |

### 9.3 关联关系

```
现有表关系:
member (1) ────── (N) token
member (N) ────── (1) level (通过 level_code)
token  (N) ────── (1) user (通过 user_login)
token  (N) ────── (1) agent (通过 agent_login)
member (N) ────── (1) agent (通过 agent_login)

新增权限表关系:
user (N) ────── (N) authz_role ────── (N) authz_permission
                                    │
                                    └─── (N) authz_menu
                                    
authz_permission (N) ────── (N) authz_api
```

---

## 10. 总结

### 10.1 当前系统特点

1. **分层架构**: 清晰的 handler → usecase → module → repository 分层
2. **多级权限**: 支持 member / user / agent / app 四级访问控制
3. **Token 机制**: Redis + MySQL 双存储，支持过期和缓存
4. **安全设计**: Salt + Hash 密码、登录失败锁定、异地检测

### 10.2 权限管理增强建议

当前系统仅实现了基于 Token 的身份认证，建议增加 RBAC 权限管理模块：

1. **角色管理**: 定义不同角色（超级管理员、运营人员、VIP会员等）
2. **权限定义**: 细粒度控制 API、菜单、资源的访问权限
3. **权限缓存**: Redis 缓存权限数据，提高检查性能
4. **权限中间件**: 在现有授权中间件基础上增加权限检查

### 10.3 扩展性

1. 可通过添加新的 Authorizer 实现扩展权限级别
2. 可通过 config 表动态调整 Token 超时配置
3. 模块化设计便于添加新的登录方式（OAuth、短信登录等）
4. RBAC 设计支持未来 ABAC (Attribute-Based Access Control) 扩展

---

**文档结束**
