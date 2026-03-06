# sub2api 后端服务层文档

> 路径: `backend/internal/service/`
> 模块: `github.com/Wei-Shaw/sub2api`
> 依赖注入: Google Wire (`wire.go`)

---

## 目录

- [1. 项目概述](#1-项目概述)
- [2. 架构总览](#2-架构总览)
- [3. 领域常量与基础类型](#3-领域常量与基础类型)
- [4. 网关服务 (Gateway)](#4-网关服务-gateway)
- [5. 认证与授权服务 (Auth)](#5-认证与授权服务-auth)
- [6. OAuth 与令牌管理](#6-oauth-与令牌管理)
- [7. 账号管理服务 (Account)](#7-账号管理服务-account)
- [8. 用户管理服务 (User)](#8-用户管理服务-user)
- [9. API Key 服务](#9-api-key-服务)
- [10. 分组管理服务 (Group)](#10-分组管理服务-group)
- [11. 计费与定价服务 (Billing & Pricing)](#11-计费与定价服务-billing--pricing)
- [12. 订阅服务 (Subscription)](#12-订阅服务-subscription)
- [13. 调度与并发控制](#13-调度与并发控制)
- [14. 速率限制服务 (Rate Limit)](#14-速率限制服务-rate-limit)
- [15. 使用记录服务 (Usage)](#15-使用记录服务-usage)
- [16. 仪表盘服务 (Dashboard)](#16-仪表盘服务-dashboard)
- [17. 运维监控服务 (Ops)](#17-运维监控服务-ops)
- [18. 系统设置服务 (Settings)](#18-系统设置服务-settings)
- [19. 邮件服务 (Email)](#19-邮件服务-email)
- [20. 公告服务 (Announcement)](#20-公告服务-announcement)
- [21. 兑换码与推广码](#21-兑换码与推广码)
- [22. 代理管理 (Proxy)](#22-代理管理-proxy)
- [23. 更新服务 (Update)](#23-更新服务-update)
- [24. 外部同步服务 (CRS Sync)](#24-外部同步服务-crs-sync)
- [25. 用户自定义属性](#25-用户自定义属性)
- [26. 错误透传服务 (Error Passthrough)](#26-错误透传服务-error-passthrough)
- [27. 身份指纹服务 (Identity)](#27-身份指纹服务-identity)
- [28. Turnstile 验证服务](#28-turnstile-验证服务)
- [29. TOTP 双因素认证](#29-totp-双因素认证)
- [30. 定时任务与延迟服务](#30-定时任务与延迟服务)
- [31. 幂等服务 (Idempotency)](#31-幂等服务-idempotency)
- [32. 安全密钥服务 (SecuritySecret)](#32-安全密钥服务-securitysecret)
- [33. Wire 依赖注入配置](#33-wire-依赖注入配置)

---

## 1. 项目概述

sub2api 是一个多平台 AI API 网关/代理系统，支持以下平台:

| 平台 | 常量 | 支持账号类型 |
|------|------|-------------|
| Anthropic (Claude) | `PlatformAnthropic` | OAuth, SetupToken, APIKey |
| OpenAI | `PlatformOpenAI` | OAuth, APIKey, Upstream |
| Gemini (Google) | `PlatformGemini` | OAuth, APIKey |
| Antigravity | `PlatformAntigravity` | OAuth |

核心功能: 将来自用户 API Key 的请求，通过调度器选择合适的上游账号，代理转发至对应平台的 API，并进行计费、限流、监控等处理。

---

## 2. 架构总览

### 2.1 分层架构

```
Handler (HTTP/Gin) → Service (业务逻辑) → Repository (数据访问)
                        ↓
                   Cache (Redis/内存)
```

### 2.2 核心设计模式

- **Repository 模式**: 所有数据访问通过 interface 抽象
- **依赖注入**: 使用 Google Wire 进行编译时依赖注入
- **领域模型**: Account, User, Group 等核心实体定义在 service 层
- **Outbox 模式**: 调度器快照通过 outbox 事件保持最终一致性
- **熔断器模式**: 计费缓存服务使用状态机熔断器 (Closed/Open/HalfOpen)
- **Singleflight 模式**: API Key 认证查询去重

---

## 3. 领域常量与基础类型

### 3.1 `domain_constants.go`

从 `internal/domain` 包重新导出所有领域常量，集中管理以便 service 层直接引用。

**平台常量:**

- `PlatformAnthropic`, `PlatformOpenAI`, `PlatformGemini`, `PlatformAntigravity`

**账号类型:**

- `AccountTypeOAuth` — OAuth 授权账号
- `AccountTypeSetupToken` — Setup Token 账号 (仅 Anthropic)
- `AccountTypeAPIKey` — API Key 账号
- `AccountTypeUpstream` — 上游转发账号 (仅 OpenAI)

**状态常量:**

- `StatusActive`, `StatusDisabled`, `StatusError`, `StatusUnused`, `StatusUsed`, `StatusExpired`

**角色常量:**

- `RoleAdmin`, `RoleUser`

**订阅类型:**

- `SubscriptionTypeStandard` — 标准分组
- `SubscriptionTypeSubscription` — 订阅分组

**设置键 (40+ 个):**

- 注册、SMTP、Turnstile、TOTP、LinuxDo Connect、OEM、Ops 监控等

### 3.2 `settings_view.go`

定义系统设置的视图模型:

| 结构体 | 说明 | 字段数 |
|-------|------|--------|
| `SystemSettings` | 管理员完整系统设置视图 | 62 |
| `PublicSettings` | 公开设置 (不含敏感信息) | 17 |
| `StreamTimeoutSettings` | 流式超时处理配置 | 5 |

**流超时处理方式:**

- `StreamTimeoutActionTempUnsched` — 临时不可调度
- `StreamTimeoutActionError` — 标记为错误状态
- `StreamTimeoutActionNone` — 不处理

---

## 4. 网关服务 (Gateway)

网关是系统核心，负责将用户请求代理转发到上游 AI 平台。

### 4.1 `gateway_service.go` — Anthropic 网关

**核心职责:** 处理 Anthropic (Claude) API 请求的完整代理流程。

**关键结构体:**

- `GatewayService` — Anthropic 网关主服务
- `accountWithLoad` — 带负载信息的账号，用于感知负载的调度

**核心流程:**

1. 解析请求 → 2. 认证/授权 → 3. 账号调度 → 4. 请求转发 → 5. 响应处理 → 6. 计费记录

**特性:**

- Sticky Session 支持 (基于 digest chain)
- Claude Code CLI 请求验证
- 模型映射与路由
- 身份指纹伪装

### 4.2 `openai_gateway_service.go` — OpenAI 网关

**核心职责:** 处理 OpenAI API 请求 (包含 Chat Completions 和 Responses API)。

**关键特性:**

- 工具调用修正 (`openai_tool_corrector.go`)
- 工具调用续传 (`openai_tool_continuation.go`)
- Codex 格式转换 (`openai_codex_transform.go`)

### 4.3 `antigravity_gateway_service.go` — Antigravity 网关

**核心职责:** 处理 Antigravity 平台请求。

**特性:**

- Google RPC 状态码处理
- 配额管理与查询 (`antigravity_quota_fetcher.go`, `antigravity_quota_scope.go`)
- 混合调度支持 (可同时参与 Anthropic/Gemini 调度)

### 4.4 `gemini_messages_compat_service.go` — Gemini Messages 兼容层

**核心职责:** 将 Anthropic Messages API 格式转换为 Gemini API 格式。

**特性:**

- 最大重试次数: `geminiMaxRetries = 5`
- Gemini Code Assist 支持 (`geminicli_codeassist.go`)

### 4.5 `gateway_request.go` — 网关请求模型

定义通用网关请求结构和上下文键常量。

### 4.6 网关辅助文件

| 文件 | 说明 |
|------|------|
| `openai_tool_corrector.go` | 修正 OpenAI 工具调用格式错误 |
| `openai_tool_continuation.go` | 处理工具调用的连续执行 |
| `openai_codex_transform.go` | Codex API 格式与标准格式互转 |
| `geminicli_codeassist.go` | Gemini CLI Code Assist 适配 |
| `anthropic_session.go` | Anthropic 会话摘要链管理 |
| `claude_code_validator.go` | Claude Code CLI 请求验证 |
| `http_upstream_port.go` | 上游 HTTP 端口定义 |

---

## 5. 认证与授权服务 (Auth)

### 5.1 `auth_service.go`

**核心职责:** 用户认证 (注册/登录/JWT)

**结构体:**

```go
type AuthService struct {
    userRepo       UserRepository
    emailService   *EmailService
    settingService *SettingService
    totpService    *TotpService
    jwtSecret      string
    jwtExpiry      time.Duration
}
```

**核心方法:**

| 方法 | 说明 |
|------|------|
| `Register` | 用户注册 (支持邮箱验证、邀请码) |
| `Login` | 用户登录 (支持 TOTP 二步验证) |
| `GenerateToken` | 生成 JWT Token (含 TokenVersion) |
| `ValidateToken` | 验证 JWT Token |
| `ResetPassword` | 密码重置 |

**安全机制:**

- JWT Token 包含 `TokenVersion`，密码修改后自动失效旧 Token
- LinuxDo Connect OAuth 第三方登录支持
- 邮箱验证码 (constant-time 比较防止时序攻击)

---

## 6. OAuth 与令牌管理

### 6.1 `oauth_service.go` — Claude OAuth

**核心职责:** Anthropic Claude OAuth 2.0 认证流程

**接口:**

```go
type ClaudeOAuthClient interface {
    ExchangeCode(ctx, code, codeVerifier, redirectURI string) (*ClaudeTokenInfo, error)
    RefreshToken(ctx, refreshToken string) (*ClaudeTokenInfo, error)
    BuildAuthorizationURL(state, codeChallenge, redirectURI string) string
}
```

**特性:**

- PKCE (Proof Key for Code Exchange) 支持
- 完整授权范围 vs 仅推理 (inference only) 两种 scope
- 代理转发支持

### 6.2 `openai_oauth_service.go` — OpenAI OAuth

**核心职责:** OpenAI OAuth 2.0 认证流程

**接口:**

```go
type OpenAIOAuthClient interface {
    ExchangeCode(ctx, code, codeVerifier, redirectURI string) (*OpenAITokenInfo, error)
    RefreshToken(ctx, refreshToken string) (*OpenAITokenInfo, error)
    BuildAuthorizationURL(state, codeChallenge, redirectURI string) string
}
```

### 6.3 `gemini_oauth_service.go` — Gemini OAuth

**核心职责:** Google Gemini OAuth 认证与 Tier 检测

**Tier 类型:**

| Tier ID | 说明 | 配额类型 |
|---------|------|---------|
| `google_one_free` | Google One 免费版 | 共享池 |
| `google_ai_pro` | Google AI Pro | 共享池 |
| `google_ai_ultra` | Google AI Ultra | 共享池 |
| `gcp_standard` | GCP Code Assist 标准版 | 共享池 |
| `gcp_enterprise` | GCP Code Assist 企业版 | 共享池 |
| `aistudio_free` | AI Studio 免费版 | 按模型 |
| `aistudio_paid` | AI Studio 付费版 | 按模型 |

### 6.4 `antigravity_oauth_service.go` — Antigravity OAuth

**核心职责:** Antigravity 平台 OAuth 认证

### 6.5 令牌提供者 (Token Provider)

每个平台有对应的令牌提供者，负责缓存管理和自动刷新:

| 文件 | 平台 | 缓存 TTL 偏移 |
|------|------|---------------|
| `claude_token_provider.go` | Anthropic | 刷新偏移 3min, 缓存偏移 5min |
| `openai_token_provider.go` | OpenAI | 类似机制 |
| `gemini_token_provider.go` | Gemini | 含 Tier 检测 |
| `antigravity_token_provider.go` | Antigravity | 含配额查询 |

### 6.6 `token_refresh_service.go` — 令牌刷新服务

**核心职责:** 后台定时刷新所有平台的 OAuth 令牌

**结构体:**

```go
type TokenRefreshService struct {
    accountRepo    AccountRepository
    refreshers     map[string]TokenRefresher
    cacheInvalidator TokenCacheInvalidator
    schedulerCache SchedulerCache
}
```

**Refresher 模式:**

```go
type TokenRefresher interface {
    RefreshAccountToken(ctx, account *Account) (*TokenInfo, error)
}
```

**支持的 Refresher:**

- `token_refresher.go` — 通用 refresher 接口
- `claude_token_provider.go` — Claude refresher
- `antigravity_token_refresher.go` — Antigravity refresher
- `gemini_token_refresher.go` — Gemini refresher

### 6.7 `token_cache_invalidator.go` — 令牌缓存失效

**核心职责:** 跨平台令牌缓存失效机制

```go
type TokenCacheInvalidator interface {
    InvalidateTokenCache(ctx, accountID int64) error
}

type CompositeTokenCacheInvalidator struct {
    claudeCache      ClaudeTokenCache
    geminiCache      GeminiTokenCache
    openaiCache      OpenAITokenCache
    antigravityCache AntigravityTokenCache
}
```

**Token Version 检测:**

- `CheckTokenVersion` 通过 `_token_version` 凭据字段检测过期令牌

### 6.8 `gemini_token_cache.go` — Gemini 令牌缓存

定义 `GeminiTokenCache` 接口。

---

## 7. 账号管理服务 (Account)

### 7.1 `account.go` — 账号领域模型

**核心结构体 (30+ 字段):**

```go
type Account struct {
    ID           int64
    Name         string
    Platform     string           // anthropic/openai/gemini/antigravity
    Type         string           // oauth/setup_token/apikey/upstream
    Credentials  map[string]any   // 凭据 (JSONB)
    Extra        map[string]any   // 扩展信息 (JSONB)
    ProxyID      *int64           // 代理关联
    Concurrency  int              // 并发数
    Priority     int              // 调度优先级
    Status       string           // active/disabled/error
    Schedulable  bool             // 是否可调度
    GroupIDs     []int64          // 关联分组
    // ... 更多字段
}
```

**核心方法:**

| 方法 | 说明 |
|------|------|
| `GetCredential(key)` | 安全获取凭据值 |
| `ModelMap()` | 获取模型映射配置 |
| `ResolveModel(model)` | 解析请求模型到实际模型 (支持通配符) |
| `CheckWindowCostSchedulability()` | 检查 5 小时窗口成本调度能力 |
| `IsMixedSchedulingEnabled()` | 是否启用混合调度 |
| `GeminiOAuthType()` | 获取 Gemini OAuth 类型 |
| `GeminiTierID()` | 获取 Gemini Tier ID |

**模型映射规则:**

- 精确匹配 → 通配符匹配 (`*` → 目标模型)
- 支持多目标映射 (负载均衡)

### 7.2 `account_service.go` — 账号服务

**仓库接口 (20+ 方法):**

```go
type AccountRepository interface {
    Create, GetByID, Update, Delete
    List, ListSchedulable
    ListSchedulableByGroupIDAndPlatform
    ListSchedulableByPlatforms
    BatchUpdateLastUsed
    GetByCRSAccountID         // CRS 同步
    UpdateCredentials
    UpdateWindowCost          // 窗口成本
    // ... 更多
}
```

**服务方法:**

| 方法 | 说明 |
|------|------|
| `Create/Update/Delete` | CRUD 操作 |
| `List/ListWithFilters` | 列表查询 |
| `UpdateCredentials` | 更新凭据 |
| `BulkUpdate` | 批量更新 |
| `TestAccount` | 测试账号连通性 (`account_test_service.go`) |

### 7.3 `account_group.go` — 账号-分组关联

```go
type AccountGroup struct {
    AccountID int64
    GroupID   int64
    Priority  int
    CreatedAt time.Time
}
```

### 7.4 `account_expiry_service.go` — 账号过期服务

后台定时检查并处理过期账号。

### 7.5 `model_rate_limit.go` — 模型级速率限制

**核心职责:** 基于账号的模型级别速率限制

**存储位置:** `Account.Extra["model_rate_limits"]`

**核心方法:**

| 方法 | 说明 |
|------|------|
| `isModelRateLimitedWithContext` | 检查模型是否被限流 (支持 Antigravity thinking 后缀) |
| `GetModelRateLimitRemainingTime` | 获取限流剩余时间 |
| `SetModelRateLimit` | 设置模型限流 |

---

## 8. 用户管理服务 (User)

### 8.1 `user.go` — 用户领域模型

```go
type User struct {
    ID            int64
    Email         string
    Username      string
    PasswordHash  string
    Role          string              // admin/user
    Balance       float64             // 余额
    Concurrency   int                 // 并发限制
    Status        string
    AllowedGroups []int64             // 允许绑定的专属分组
    TokenVersion  int64               // 密码修改时递增，失效旧 Token
    GroupRates    map[int64]float64   // 用户专属分组倍率
    // TOTP 双因素认证
    TotpSecretEncrypted *string
    TotpEnabled         bool
    TotpEnabledAt       *time.Time
}
```

**核心方法:**

- `IsAdmin()` — 是否管理员
- `CanBindGroup(groupID, isExclusive)` — 是否可绑定分组
- `SetPassword/CheckPassword` — 密码管理 (bcrypt)

### 8.2 `user_service.go` — 用户服务

```go
type UserRepository interface {
    Create, GetByID, GetByEmail, GetFirstAdmin
    Update, Delete
    List, ListWithFilters
    UpdateBalance, DeductBalance
    UpdateConcurrency
    ExistsByEmail
    RemoveGroupFromAllowedGroups
    // TOTP 相关
    UpdateTotpSecret, EnableTotp, DisableTotp
}
```

**服务方法:**

| 方法 | 说明 |
|------|------|
| `GetProfile` | 获取用户资料 |
| `UpdateProfile` | 更新资料 (支持邮箱唯一性检查) |
| `ChangePassword` | 修改密码 (自动递增 TokenVersion) |
| `UpdateBalance` | 更新余额 (管理员) |
| `UpdateStatus` | 更新状态 (管理员) |
| `Delete` | 删除用户 |

---

## 9. API Key 服务

### 9.1 `api_key_service.go`

**核心职责:** 用户 API Key 的完整生命周期管理

**仓库接口:**

```go
type APIKeyRepository interface {
    Create, GetByID, Update, Delete
    GetByHashedKey              // 通过哈希查找
    ListByUserID
    CountByUserID
    IncrementUsage
    // 认证缓存
    GetAuthSnapshot, SetAuthSnapshot
}
```

**缓存接口:**

```go
type APIKeyCache interface {
    GetAuthCache, SetAuthCache       // Redis L2 缓存
    PublishInvalidation              // Pub/Sub 发布失效通知
    SubscribeInvalidation            // Pub/Sub 订阅失效通知
}
```

**性能优化:**

- **L1 缓存**: 使用 `ristretto` 本地内存缓存
- **L2 缓存**: Redis 缓存
- **Singleflight**: `golang.org/x/sync/singleflight` 避免并发查询重复
- **Pub/Sub**: Redis Pub/Sub 跨实例 L1 缓存失效

### 9.2 `api_key_auth_cache.go` — 认证缓存快照

```go
type APIKeyAuthSnapshot struct {
    UserID      int64
    Status      string
    Balance     float64
    Concurrency int
    GroupRates  map[int64]float64
    Groups      []APIKeyAuthGroupSnapshot
}

type APIKeyAuthCacheEntry struct {
    Snapshot *APIKeyAuthSnapshot
    NotFound bool    // 支持负缓存
    CachedAt int64
}
```

### 9.3 `api_key_auth_cache_invalidate.go` — 认证缓存失效

定义 `APIKeyAuthCacheInvalidator` 接口和实现。

---

## 10. 分组管理服务 (Group)

### 10.1 `group.go` — 分组领域模型

```go
type Group struct {
    ID             int64
    Name           string
    Platform       string
    Type           string               // standard/subscription
    IsExclusive    bool                  // 是否专属分组
    ClaudeCodeOnly bool                  // 仅 Claude Code
    SortOrder      int                   // 排序
    // 订阅限额
    DailyLimitUSD   *float64
    WeeklyLimitUSD  *float64
    MonthlyLimitUSD *float64
    // 图片定价
    ImagePricing1K  *float64
    ImagePricing2K  *float64
    ImagePricing4K  *float64
    // 模型路由 map[model][]groupID
    ModelRouting    map[string][]int64
    // Fallback 分组
    FallbackGroups  []int64
    // MCP XML 注入
    MCPXMLInject    bool
    // 支持的模型范围
    SupportedModelScopes []string
}
```

**核心方法:**

- `IsSubscriptionType()` — 是否订阅类型
- `HasDailyLimit/HasWeeklyLimit/HasMonthlyLimit` — 是否有使用限额
- `GetRateMultiplier(userGroupRates)` — 获取倍率

### 10.2 `group_service.go` — 分组服务

```go
type GroupRepository interface {
    Create, GetByID, Update, Delete
    List, ListActive
    ListByPlatform
    UpdateSortOrders
}
```

**服务方法:**

| 方法 | 说明 |
|------|------|
| `Create/Update/Delete` | CRUD (变更后失效认证缓存) |
| `List/GetByID` | 查询 |
| `UpdateSortOrders` | 批量更新排序 |

---

## 11. 计费与定价服务 (Billing & Pricing)

### 11.1 `pricing_service.go` — 动态定价服务

**核心职责:** 基于 LiteLLM 格式的动态模型定价

**核心结构体:**

```go
type PricingService struct {
    prices        map[string]*ModelPricing
    remoteClient  PricingRemoteClient
    hash          string           // 远程数据哈希
}
```

**模型匹配优先级 (5 级):**

1. 精确匹配 (如 `claude-sonnet-4-20250514`)
2. 变体匹配 (日期后缀去除)
3. 模糊匹配 (前缀匹配)
4. 家族匹配 (模型家族)
5. OpenAI 回退 (添加 `openai/` 前缀)

**远程同步:**

- 基于 hash 的增量更新，仅在定价数据变化时更新
- `PricingRemoteClient` 接口用于获取远程定价数据

### 11.2 `billing_service.go` — 计费服务

**核心职责:** 计算请求成本

**核心结构体:**

```go
type UsageTokens struct {
    InputTokens, OutputTokens   int64
    CacheCreateTokens           int64    // 缓存创建令牌
    CacheReadTokens             int64    // 缓存读取令牌
}

type CostBreakdown struct {
    InputCost, OutputCost    float64
    CacheCreateCost          float64
    CacheReadCost            float64
    TotalCost                float64
    RateMultiplier           float64  // 倍率
    ActualCost               float64  // 实际扣费
}
```

**计费流程:**

1. 查找模型定价 → 2. 计算各项成本 → 3. 应用倍率 → 4. 计算实际扣费

**Gemini 200K 长上下文双倍计费:**

- `CalculateCostWithLongContext` — 超过 200K token 的请求双倍收费

### 11.3 `billing_cache_service.go` — 计费缓存服务

**核心职责:** 高性能异步计费资格检查

**关键组件:**

- **异步 Worker Pool**: 10 个 worker，1000 缓冲区
- **熔断器**: `billingCircuitBreaker` (Closed → Open → HalfOpen)
- **缓存策略**: Redis 缓存 + 内存缓存

```go
type BillingCacheService struct {
    cache          BillingCachePort
    breaker        *billingCircuitBreaker
    workerCh       chan billingAsyncTask
}
```

### 11.4 `billing_cache_port.go` — 计费缓存端口

定义 `BillingCachePort` 接口。

---

## 12. 订阅服务 (Subscription)

### 12.1 `user_subscription.go` — 订阅领域模型

```go
type UserSubscription struct {
    ID, UserID, GroupID      int64
    StartsAt, ExpiresAt      time.Time
    Status                   string
    // 使用窗口
    DailyWindowStart         *time.Time
    WeeklyWindowStart        *time.Time
    MonthlyWindowStart       *time.Time
    DailyUsageUSD            float64
    WeeklyUsageUSD           float64
    MonthlyUsageUSD          float64
}
```

**核心方法:**

- `IsActive()`, `IsExpired()`, `DaysRemaining()`
- `NeedsDailyReset()`, `NeedsWeeklyReset()`, `NeedsMonthlyReset()`
- `CheckDailyLimit()`, `CheckWeeklyLimit()`, `CheckMonthlyLimit()`
- `CheckAllLimits()` — 同时检查所有限额

### 12.2 `subscription_service.go` — 订阅服务

**核心方法:**

| 方法 | 说明 |
|------|------|
| `AssignSubscription` | 分配订阅 (不允许重复) |
| `AssignOrExtendSubscription` | 分配或续期订阅 |
| `BulkAssignSubscription` | 批量分配订阅 |
| `RevokeSubscription` | 撤销订阅 |
| `ExtendSubscription` | 调整订阅时长 (正数延长/负数缩短) |
| `CheckAndActivateWindow` | 首次使用时激活窗口 |
| `CheckAndResetWindows` | 检查并重置过期窗口 |
| `CheckUsageLimits` | 检查使用限额 |
| `RecordUsage` | 记录使用量 |
| `GetSubscriptionProgress` | 获取订阅使用进度 |

**订阅进度模型:**

```go
type SubscriptionProgress struct {
    Daily   *UsageWindowProgress
    Weekly  *UsageWindowProgress
    Monthly *UsageWindowProgress
}
```

### 12.3 `subscription_expiry_service.go` — 订阅过期服务

后台定时检查并处理过期订阅。

---

## 13. 调度与并发控制

### 13.1 `scheduler_snapshot_service.go` — 调度器快照服务

**核心职责:** 基于 Outbox 模式维护调度器的 Redis 缓存快照

**架构:**

```
DB (Outbox 事件) → SnapshotService → Redis Cache → Scheduler
```

**三个后台 Worker:**

1. **Initial Rebuild** — 启动时重建缓存
2. **Outbox Poller** — 轮询 outbox 事件并增量更新
3. **Full Rebuild** — 定期全量重建

**核心方法:**

```go
func (s *SchedulerSnapshotService) ListSchedulableAccounts(
    ctx context.Context, groupID *int64, platform string, hasForcePlatform bool,
) ([]Account, bool, error)
```

**混合调度模式:**

- `PlatformAnthropic` 和 `PlatformGemini` 请求可以使用 `PlatformAntigravity` 账号
- 通过 `SchedulerModeMixed` 实现

**Outbox 事件类型:**

| 事件 | 说明 |
|------|------|
| `AccountChanged` | 账号变更 |
| `AccountGroupsChanged` | 账号分组变更 |
| `AccountLastUsed` | 账号最后使用时间批量更新 |
| `AccountBulkChanged` | 批量账号变更 |
| `GroupChanged` | 分组变更 |
| `FullRebuild` | 触发全量重建 |

**容灾机制:**

- DB Fallback: 缓存未命中时回退到数据库查询
- `fallbackLimiter`: 基于 QPS 的限流器，防止 DB 过载
- Outbox Lag 检测: 延迟过大时自动触发全量重建

### 13.2 `scheduler_cache.go` — 调度器缓存接口

```go
type SchedulerBucket struct {
    GroupID  int64
    Platform string
    Mode     string  // single/mixed/forced
}

type SchedulerCache interface {
    GetSnapshot(ctx, bucket SchedulerBucket) ([]*Account, bool, error)
    SetSnapshot(ctx, bucket SchedulerBucket, accounts []Account) error
    GetAccount(ctx, accountID int64) (*Account, error)
    SetAccount(ctx, account *Account) error
    TryLockBucket(ctx, bucket, ttl) (bool, error)
    ListBuckets(ctx) ([]SchedulerBucket, error)
    GetOutboxWatermark(ctx) (int64, error)
    SetOutboxWatermark(ctx, watermark int64) error
    UpdateLastUsed(ctx, updates map[int64]time.Time) error
}
```

**Bucket 模式:**

- `SchedulerModeSingle` — 单平台调度
- `SchedulerModeMixed` — 混合调度 (含 Antigravity)
- `SchedulerModeForced` — 强制平台调度

### 13.3 `scheduler_outbox.go` — Outbox 事件模型

### 13.4 `scheduler_events.go` — 调度器事件定义

### 13.5 `concurrency_service.go` — 并发控制服务

**核心职责:** 基于 Redis Sorted Set 的账号/用户级并发槽管理

```go
type ConcurrencyCache interface {
    AcquireAccountSlot(ctx, accountID, slotID, maxSlots)
    ReleaseAccountSlot(ctx, accountID, slotID)
    AcquireUserSlot(ctx, userID, slotID, maxSlots)
    ReleaseUserSlot(ctx, userID, slotID)
    CountAccountSlots(ctx, accountID) int
    BatchCountAccountSlots(ctx, accountIDs) map[int64]int
    CleanExpiredSlots(ctx, maxAge) int64
}
```

**核心概念:**

- **AcquireResult**: 包含 `ReleaseFunc`，调用后自动释放槽位
- **Extra Wait Slots**: `defaultExtraWaitSlots = 20`，允许额外等待槽位
- **槽位清理 Worker**: 定期清理过期槽位

### 13.6 `session_limit_cache.go` — 会话限制缓存

### 13.7 `temp_unsched.go` — 临时不可调度缓存

用于标记因错误临时不可调度的账号。

```go
type TempUnschedCache interface {
    IsTempUnscheduled(ctx, accountID) bool
    SetTempUnscheduled(ctx, accountID, duration) error
}
```

---

## 14. 速率限制服务 (Rate Limit)

### 14.1 `ratelimit_service.go`

**核心职责:** 多层速率限制与错误策略处理

```go
type RateLimitService struct {
    accountRepo         AccountRepository
    usageRepo           UsageLogRepository
    cfg                 *config.Config
    geminiQuotaService  *GeminiQuotaService
    tempUnschedCache    TempUnschedCache
    timeoutCounterCache TimeoutCounterCache
    settingService      *SettingService
    tokenCacheInvalidator TokenCacheInvalidator
}
```

**错误策略结果:**

```go
type ErrorPolicyResult int
const (
    ErrorPolicyNone            // 无匹配策略
    ErrorPolicySkipped         // 策略被跳过
    ErrorPolicyMatched         // 策略匹配
    ErrorPolicyTempUnscheduled // 临时不可调度
)
```

**核心方法:**

| 方法 | 说明 |
|------|------|
| `CheckErrorPolicy` | 检查上游错误是否匹配速率限制策略 |
| `HandleUpstreamError` | 处理上游错误 (可能触发临时不可调度) |

### 14.2 `gemini_quota.go` — Gemini 配额服务

**核心职责:** Gemini 平台的分层配额管理

**默认配额策略:**

| Tier | Pro RPD/RPM | Flash RPD/RPM | 冷却时间 |
|------|-------------|---------------|---------|
| aistudio_free | 50/2 | 1500/15 | 30 min |
| aistudio_paid | 无限/1000 | 无限/2000 | 5 min |
| google_one_free | 共享 1000/60 | | 30 min |
| google_ai_pro | 共享 1500/120 | | 5 min |
| gcp_standard | 共享 1500/120 | | 5 min |

**配额覆盖:**

- V1 格式: 简单 tier 覆盖
- V2 格式: `quota_rules` 支持更细粒度的模型级配额

### 14.3 `gemini_oauth.go` — Gemini OAuth 辅助

Gemini Tier 常量和规范化函数。

### 14.4 `quota_fetcher.go` — 配额查询器

通用配额查询接口。

---

## 15. 使用记录服务 (Usage)

### 15.1 `usage_service.go`

**核心职责:** 使用记录的创建与查询

```go
type CreateUsageLogRequest struct {
    UserID, APIKeyID, AccountID, GroupID    int64
    Platform, Model, RequestModel          string
    InputTokens, OutputTokens              int64
    CacheCreateTokens, CacheReadTokens     int64
    TotalCost, ActualCost                  float64
    RateMultiplier                         float64
    IsStream                               bool
    BillingType                            int8
    // ... 更多字段
}
```

**事务性创建:**

- 使用 ent Client 事务 (tx)
- 创建使用记录的同时扣减余额

### 15.2 `usage_cleanup_service.go` — 使用记录清理

**核心职责:** 定时清理过期使用记录

- 基于 `TimingWheelService` 调度
- 支持按 user/apikey/account/group/model/stream/billing_type 过滤

### 15.3 `usage_cache.go` — 使用缓存

用于 Gemini 配额检查的使用统计缓存。

---

## 16. 仪表盘服务 (Dashboard)

### 16.1 `dashboard_service.go`

**核心职责:** 管理员仪表盘统计数据服务

```go
type DashboardService struct {
    usageRepo      UsageLogRepository
    aggRepo        DashboardAggregationRepository
    cache          DashboardStatsCache
    cacheFreshTTL  time.Duration  // 默认 15s
    cacheTTL       time.Duration  // 默认 30s
}
```

**缓存策略:**

- **Fresh TTL**: 15s 内认为数据新鲜
- **Cache TTL**: 30s 内可用 (非新鲜时异步刷新)
- **异步刷新**: 使用 `atomic.CompareAndSwapInt32` 保证只有一个 goroutine 刷新

**核心方法:**

| 方法 | 说明 |
|------|------|
| `GetDashboardStats` | 获取仪表盘统计 |
| `GetUsageTrendWithFilters` | 使用趋势 (支持多维过滤) |
| `GetModelStatsWithFilters` | 模型统计 |
| `GetAPIKeyUsageTrend` | API Key 使用趋势 |
| `GetUserUsageTrend` | 用户使用趋势 |
| `GetBatchUserUsageStats` | 批量用户使用统计 |
| `GetBatchAPIKeyUsageStats` | 批量 API Key 使用统计 |

### 16.2 `dashboard_aggregation_service.go` — 仪表盘聚合

后台定时聚合使用数据，减轻实时查询压力。

---

## 17. 运维监控服务 (Ops)

Ops 监控是一个完整的子系统，包含多个组件:

### 17.1 `ops_service.go` — Ops 主服务

```go
type OpsService struct {
    repo            OpsRepository
    settingRepo     SettingRepository
    gatewayService  *GatewayService
    openaiGateway   *OpenAIGatewayService
    antigravityGw   *AntigravityGatewayService
    cfg             *config.Config
}
```

**核心常量:**

- `opsMaxStoredRequestBodyBytes = 10KB`
- `opsMaxStoredErrorBodyBytes = 20KB`

### 17.2 Ops 子组件

| 文件 | 说明 |
|------|------|
| `ops_metrics_collector.go` | 指标采集器 (Leader 选举, gopsutil CPU/内存) |
| `ops_aggregation_service.go` | 小时/天预聚合服务 |
| `ops_alert_evaluator_service.go` | 告警评估服务 |
| `ops_scheduled_report_service.go` | 定时报告服务 |
| `ops_cleanup_service.go` | 数据清理服务 (cron) |
| `ops_advisory_lock.go` | Redis 分布式咨询锁 |
| `ops_concurrency.go` | 分页账号加载与并发查询 |
| `ops_dashboard.go` | 运维仪表盘数据 |
| `ops_realtime.go` | 实时监控数据 |
| `ops_realtime_traffic.go` | 实时流量数据 |
| `ops_health_score.go` | 健康分数计算 |
| `ops_trends.go` | 趋势分析 |
| `ops_histograms.go` | 直方图统计 |
| `ops_errors.go` | 错误分析 |
| `ops_window_stats.go` | 窗口统计 |
| `ops_request_details.go` | 请求详情 |
| `ops_query_mode.go` | 查询模式 |
| `ops_settings.go` | Ops 设置管理 |
| `ops_upstream_context.go` | 上游上下文 |
| `ops_models.go` | Ops 数据模型 |
| `ops_dashboard_models.go` | 仪表盘数据模型 |
| `ops_realtime_traffic_models.go` | 实时流量数据模型 |
| `ops_settings_models.go` | 设置数据模型 |
| `ops_alerts.go` | 告警逻辑 |
| `ops_alert_models.go` | 告警数据模型 |

---

## 18. 系统设置服务 (Settings)

### 18.1 `setting_service.go`

```go
type SettingRepository interface {
    Get(ctx, key string) (*Setting, error)
    GetValue(ctx, key string) (string, error)
    GetMultiple(ctx, keys []string) (map[string]string, error)
    Set(ctx, key, value string) error
    SetMultiple(ctx, settings map[string]string) error
    Delete(ctx, key string) error
}
```

**核心方法:**

| 方法 | 说明 |
|------|------|
| `GetPublicSettings` | 获取公开设置 (~20 个键) |
| `GetSystemSettings` | 获取完整系统设置 (管理员) |
| `UpdateSettings` | 批量更新设置 |
| `IsTurnstileEnabled` | Turnstile 是否启用 |
| `IsRegistrationEnabled` | 注册是否启用 |
| `IsEmailVerifyEnabled` | 邮箱验证是否启用 |
| `IsTotpEnabled` | TOTP 是否启用 |

### 18.2 `setting.go` — 设置领域模型

---

## 19. 邮件服务 (Email)

### 19.1 `email_service.go`

**核心职责:** SMTP 邮件发送、验证码管理、密码重置

```go
type EmailCache interface {
    GetVerificationCode, SetVerificationCode, DeleteVerificationCode
    GetPasswordResetToken, SetPasswordResetToken, DeletePasswordResetToken
    IsPasswordResetEmailInCooldown, SetPasswordResetEmailCooldown
}
```

**常量:**

- 验证码 TTL: 15 分钟
- 验证码冷却: 1 分钟
- 最大尝试次数: 5
- 密码重置令牌 TTL: 30 分钟
- 密码重置邮件冷却: 30 秒

**安全特性:**

- Constant-time 比较防止时序攻击
- TLS 1.2+ 强制要求
- 邮件轰炸防护 (cooldown)

### 19.2 `email_queue_service.go` — 邮件队列

异步邮件发送队列，默认 3 个 worker。

---

## 20. 公告服务 (Announcement)

### 20.1 `announcement.go` — 公告模型

```go
type Announcement = domain.Announcement

type AnnouncementTargeting = domain.AnnouncementTargeting
```

**公告状态:** `Draft`, `Active`, `Archived`

**定向条件类型:**

- `Subscription` — 按订阅分组
- `Balance` — 按余额条件 (GT/GTE/LT/LTE/EQ)

### 20.2 `announcement_service.go` — 公告服务

**核心方法:**

| 方法 | 说明 |
|------|------|
| `Create/Update/Delete` | CRUD |
| `ListForUser` | 获取用户可见公告 (含定向过滤) |
| `MarkRead` | 标记已读 (安全检查可见性) |
| `ListUserReadStatus` | 查看公告阅读状态 |

**可见性规则:**

1. 公告必须是 Active 状态
2. 当前时间在 StartsAt ~ EndsAt 范围内
3. 用户匹配定向条件 (余额/订阅)

**排序规则:** 未读优先，同状态按创建时间倒序

---

## 21. 兑换码与推广码

### 21.1 `redeem_code.go` — 兑换码模型

```go
type RedeemCode struct {
    ID        int64
    Code      string
    Type      string       // 类型
    Value     float64      // 面值
    Status    string       // unused/used
    UsedBy    *int64
    GroupID   *int64       // 可关联分组 (用于订阅兑换)
    ValidityDays int       // 有效天数
}
```

- `GenerateRedeemCode()` — 生成 32 字符十六进制兑换码

### 21.2 `promo_code.go` — 推广码模型

### 21.3 `promo_code_repository.go` — 推广码仓库接口

### 21.4 `promo_service.go` — 推广服务

---

## 22. 代理管理 (Proxy)

### 22.1 `proxy.go` — 代理模型

```go
type Proxy struct {
    ID       int64
    Name     string
    Protocol string    // http/https/socks5
    Host     string
    Port     int
    Username string
    Password string
    Status   string
}
```

- `URL()` — 生成代理 URL (支持认证格式)

### 22.2 `proxy_latency_cache.go` — 代理延迟缓存

---

## 23. 更新服务 (Update)

### 23.1 `update_service.go`

**核心职责:** 从 GitHub Release 检查并执行软件更新

```go
type UpdateService struct {
    cache          UpdateCache
    githubClient   GitHubReleaseClient
    currentVersion string
    buildType      string  // "source" | "release"
}
```

**核心方法:**

| 方法 | 说明 |
|------|------|
| `CheckUpdate` | 检查更新 (支持缓存, TTL 20min) |
| `PerformUpdate` | 下载并执行更新 |
| `Rollback` | 回滚到上一版本 |

**安全特性:**

- URL 白名单验证 (仅允许 `github.com` 和 `objects.githubusercontent.com`)
- SHA256 校验和验证
- Zip Slip / Path Traversal 防护
- 最大文件大小限制: 500MB
- 原子替换 (rename 模式)
- 仅 HTTPS

---

## 24. 外部同步服务 (CRS Sync)

### 24.1 `crs_sync_service.go`

**核心职责:** 从 CRS (Claude Resource Scheduler) 系统同步账号

**支持的账号类型同步:**

| CRS 类型 | sub2api 平台 | sub2api 类型 |
|----------|-------------|-------------|
| Claude OAuth/SetupToken | anthropic | oauth/setup_token |
| Claude Console API Key | anthropic | apikey |
| OpenAI OAuth | openai | oauth |
| OpenAI Responses | openai | apikey |
| Gemini OAuth | gemini | oauth |
| Gemini API Key | gemini | apikey |

**同步流程:**

1. CRS 登录获取 adminToken
2. 导出所有账号
3. 逐个同步 (创建/更新/跳过)
4. OAuth 账号同步后自动刷新令牌
5. 可选同步代理

**URL 安全:**

- 支持 URL 白名单
- SSRF 防护 (IP 验证)

---

## 25. 用户自定义属性

### 25.1 `user_attribute.go` — 属性模型

```go
type UserAttributeDefinition struct {
    ID           int64
    Key          string
    Name         string
    Type         UserAttributeType  // text/textarea/number/email/url/date/select/multi_select
    Options      []UserAttributeOption
    Required     bool
    Validation   UserAttributeValidation
}
```

**验证规则:**

```go
type UserAttributeValidation struct {
    MinLength, MaxLength *int
    Min, Max             *int
    Pattern              *string    // 正则表达式
    Message              *string    // 错误消息
}
```

### 25.2 `user_attribute_service.go` — 属性服务

**核心方法:**

| 方法 | 说明 |
|------|------|
| `CreateDefinition` | 创建属性定义 |
| `UpdateDefinition` | 更新属性定义 |
| `DeleteDefinition` | 删除属性定义 (级联删除值) |
| `ReorderDefinitions` | 重排序 |
| `GetUserAttributes` | 获取用户属性值 |
| `UpdateUserAttributes` | 批量更新用户属性值 (含验证) |
| `GetBatchUserAttributes` | 批量获取多用户属性 |

**验证逻辑:**

- 字符串长度验证
- 数字范围验证
- 正则表达式验证
- Select/MultiSelect 选项验证

---

## 26. 错误透传服务 (Error Passthrough)

### 26.1 `error_passthrough_service.go`

**核心职责:** 基于规则的上游错误透传到客户端

```go
type ErrorPassthroughService struct {
    repo     ErrorPassthroughRepository
    cache    ErrorPassthroughCache     // 含 Pub/Sub
    rules    []ErrorPassthroughRule    // 本地内存缓存
}
```

**接口:**

```go
type ErrorPassthroughCache interface {
    GetRules, SetRules
    PublishInvalidation
    SubscribeInvalidation
}
```

**核心方法:**

- `MatchRule(statusCode, errorType, message)` — 匹配规则
- CRUD 操作 (变更后发布失效通知)

### 26.2 `error_passthrough_runtime.go`

**核心职责:** Gin 上下文中的错误透传运行时绑定

- `applyErrorPassthroughRule` — 检查规则并重写 HTTP 状态码/错误类型/错误消息

---

## 27. 身份指纹服务 (Identity)

### 27.1 `identity_service.go`

**核心职责:** OAuth 账号的指纹伪装和会话 ID 掩码

```go
type IdentityService struct {
    cache IdentityCache
    cfg   *config.Config
}
```

**核心结构体:**

```go
type Fingerprint struct {
    Cookie       string
    UserAgent    string
    AcceptLang   string
    SecChUaPlatform string
}
```

**核心方法:**

- `RewriteUserID` — 重写 User ID
- `RewriteUserIDWithMasking` — 带会话 ID 掩码的重写
- 指纹缓存管理

### 27.2 `anthropic_session.go` — Anthropic 会话管理

**核心职责:** Anthropic 会话回退的摘要链管理

```go
func BuildAnthropicDigestChain(system, userMsg string, accountID int64) string
// 生成格式: s:<systemHash>-u:<userMsgHash>-a:<accountHash>
```

- TTL: 5 分钟
- `GenerateAnthropicDigestSessionKey` — 组合 prefixHash + uuid

### 27.3 `digest_session_store.go` — 摘要会话存储

**核心职责:** 基于 `go-cache` 的内存摘要会话存储

```go
type DigestSessionStore struct {
    cache *cache.Cache  // TTL 5 分钟
}
```

- `Find` — 最长前缀匹配 (逐步截断摘要链段)

---

## 28. Turnstile 验证服务

### 28.1 `turnstile_service.go`

**核心职责:** Cloudflare Turnstile 人机验证

```go
type TurnstileVerifier interface {
    VerifyToken(ctx, secretKey, token, remoteIP string) (*TurnstileVerifyResponse, error)
}
```

**核心方法:**

- `VerifyToken` — 验证 Turnstile Token
- `IsEnabled` — 检查是否启用
- `ValidateSecretKey` — 验证 Secret Key 有效性

---

## 29. TOTP 双因素认证

### 29.1 `totp_service.go`

TOTP (Time-based One-Time Password) 双因素认证服务。

**用户相关字段 (在 `user.go`):**

- `TotpSecretEncrypted` — AES-256-GCM 加密的 TOTP 密钥
- `TotpEnabled` — 是否启用
- `TotpEnabledAt` — 启用时间

---

## 30. 定时任务与延迟服务

### 30.1 `timing_wheel_service.go` — 时间轮服务

**核心职责:** 基于 go-zero TimingWheel 的任务调度

```go
type TimingWheelService struct {
    tw *collection.TimingWheel  // 1秒精度, 3600个槽 (最长1小时)
}
```

**方法:**

| 方法 | 说明 |
|------|------|
| `Schedule(name, delay, fn)` | 单次延迟任务 |
| `ScheduleRecurring(name, interval, fn)` | 循环任务 |
| `Cancel(name)` | 取消任务 |

**使用者:**

- `DeferredService` — 延迟批量更新
- `UsageCleanupService` — 使用记录清理
- `DashboardAggregationService` — 仪表盘聚合

### 30.2 `deferred_service.go` — 延迟批量更新

**核心职责:** 将高频的 LastUsed 更新合并为批量写入

```go
type DeferredService struct {
    lastUsedUpdates sync.Map  // 内存聚合
    interval        time.Duration  // 默认 10s
}
```

**流程:**

1. `ScheduleLastUsedUpdate(accountID)` — 写入内存 Map
2. 定时 (10s) flush — 批量写入数据库
3. 失败时回写内存 Map (重试)

---

## 31. 幂等服务 (Idempotency)

### 31.1 `idempotency.go` — 幂等协调器

**核心职责:** 保证重复请求的幂等性，避免重复执行（如订阅分配、备份创建等）。

**关键结构体:**

- `IdempotencyRecord` — 幂等记录 (scope, idempotency_key_hash, status, expires_at)
- `IdempotencyCoordinator` — 协调器，提供 `ExecuteIdempotent` 等接口
- `IdempotencyRepository` — 数据访问接口

**状态流转:**

- `processing` → `succeeded` / `failed_retryable`
- 支持锁续期、过期清理

**使用场景:** API Key 创建、订阅分配、数据管理备份等需幂等的操作。

### 31.2 `idempotency_cleanup_service.go`

**核心职责:** 定期清理过期的幂等记录，避免表膨胀。

---

## 32. 安全密钥服务 (SecuritySecret)

### 32.1 `security_secret_bootstrap.go` (repository)

**核心职责:** 系统启动时引导 SecuritySecret 表，存储 JWT 签名密钥、TOTP 加密密钥等敏感配置。

**存储位置:** `security_secrets` 表 (key-value)

**用途:** 将原先环境变量/配置文件中的敏感密钥迁移到数据库，支持运行时轮换。

---

## 33. Wire 依赖注入配置

### 33.1 `wire.go`

**核心职责:** 使用 Google Wire 的编译时依赖注入配置

**Provider 函数 (生命周期管理):**

| Provider | 说明 | 特殊行为 |
|----------|------|---------|
| `ProvidePricingService` | 定价服务 | 初始化失败不阻塞启动 |
| `ProvideUpdateService` | 更新服务 | 注入 BuildInfo |
| `ProvideEmailQueueService` | 邮件队列 | 3 个 worker |
| `ProvideTokenRefreshService` | 令牌刷新 | 自动 Start() |
| `ProvideDashboardAggregationService` | 仪表盘聚合 | 自动 Start() |
| `ProvideUsageCleanupService` | 使用清理 | 自动 Start() |
| `ProvideAccountExpiryService` | 账号过期 | 自动 Start(), 1min 间隔 |
| `ProvideSubscriptionExpiryService` | 订阅过期 | 自动 Start(), 1min 间隔 |
| `ProvideTimingWheelService` | 时间轮 | 自动 Start() |
| `ProvideDeferredService` | 延迟服务 | 自动 Start(), 10s 间隔 |
| `ProvideConcurrencyService` | 并发控制 | 启动槽清理 Worker |
| `ProvideSchedulerSnapshotService` | 调度快照 | 自动 Start() |
| `ProvideRateLimitService` | 速率限制 | 设置可选依赖 |
| `ProvideOpsMetricsCollector` | Ops 指标 | 自动 Start() |
| `ProvideOpsAggregationService` | Ops 聚合 | 自动 Start() |
| `ProvideOpsAlertEvaluatorService` | Ops 告警 | 自动 Start() |
| `ProvideOpsCleanupService` | Ops 清理 | 自动 Start() |
| `ProvideOpsScheduledReportService` | Ops 报告 | 自动 Start() |
| `ProvideAPIKeyAuthCacheInvalidator` | 认证缓存失效 | 启动 Pub/Sub 订阅 |

**ProviderSet (40+ 服务):**
包含所有服务的构造函数和 Provider 函数，通过 Wire 自动解析依赖关系。

**关键绑定:**

```go
wire.Bind(new(TokenCacheInvalidator), new(*CompositeTokenCacheInvalidator))
```

---

## 附录: 文件索引

### 33.2 领域模型文件

| 文件 | 说明 |
|------|------|
| `account.go` | 账号领域模型 (891 行) |
| `account_group.go` | 账号-分组关联 |
| `user.go` | 用户领域模型 |
| `user_subscription.go` | 用户订阅模型 |
| `group.go` | 分组领域模型 |
| `proxy.go` | 代理模型 |
| `redeem_code.go` | 兑换码模型 |
| `promo_code.go` | 推广码模型 |
| `announcement.go` | 公告模型 |
| `user_attribute.go` | 用户属性模型 |
| `setting.go` | 设置模型 |
| `domain_constants.go` | 领域常量 |
| `settings_view.go` | 设置视图模型 |

### 33.3 服务文件

| 文件 | 说明 |
|------|------|
| `auth_service.go` | 认证服务 |
| `user_service.go` | 用户服务 |
| `account_service.go` | 账号服务 |
| `api_key_service.go` | API Key 服务 |
| `group_service.go` | 分组服务 |
| `billing_service.go` | 计费服务 |
| `billing_cache_service.go` | 计费缓存服务 |
| `pricing_service.go` | 定价服务 |
| `subscription_service.go` | 订阅服务 |
| `announcement_service.go` | 公告服务 |
| `email_service.go` | 邮件服务 |
| `email_queue_service.go` | 邮件队列服务 |
| `setting_service.go` | 设置服务 |
| `turnstile_service.go` | Turnstile 服务 |
| `update_service.go` | 更新服务 |
| `dashboard_service.go` | 仪表盘服务 |
| `user_attribute_service.go` | 用户属性服务 |
| `promo_service.go` | 推广服务 |
| `admin_service.go` | 管理员服务 |
| `crs_sync_service.go` | CRS 同步服务 |
| `data_management_service.go` | 数据管理（备份/S3/源配置） |
| `system_operation_lock_service.go` | 系统操作锁服务 |

### 33.3.1 Sora 相关服务

| 文件 | 说明 |
|------|------|
| `sora_client.go` | Sora 客户端接口 |
| `sora_gateway_service.go` | Sora 网关服务 |
| `sora_account_service.go` | Sora 账号服务 |
| `sora_quota_service.go` | Sora 配额服务 |
| `sora_generation_service.go` | Sora 生成服务 |
| `sora_models.go` | Sora 模型配置 |
| `sora_sdk_client.go` | Sora SDK 客户端 |
| `sora_media_storage.go` | Sora 媒体存储 |
| `sora_media_sign.go` | Sora 媒体签名 |
| `sora_media_cleanup_service.go` | Sora 媒体清理 |
| `sora_s3_storage.go` | Sora S3 存储 |
| `sora_upstream_forwarder.go` | Sora 上游转发 |

### 33.4 网关文件

| 文件 | 说明 |
|------|------|
| `gateway_service.go` | Anthropic 网关 |
| `openai_gateway_service.go` | OpenAI 网关 |
| `antigravity_gateway_service.go` | Antigravity 网关 |
| `gemini_messages_compat_service.go` | Gemini 兼容层 |
| `gateway_request.go` | 网关请求模型 |
| `openai_tool_corrector.go` | 工具调用修正 |
| `openai_tool_continuation.go` | 工具调用续传 |
| `openai_codex_transform.go` | Codex 转换 |
| `geminicli_codeassist.go` | Gemini CLI Code Assist |
| `http_upstream_port.go` | 上游端口 |

### 33.5 OAuth / 令牌文件

| 文件 | 说明 |
|------|------|
| `oauth_service.go` | Claude OAuth |
| `openai_oauth_service.go` | OpenAI OAuth |
| `gemini_oauth_service.go` | Gemini OAuth |
| `antigravity_oauth_service.go` | Antigravity OAuth |
| `claude_token_provider.go` | Claude 令牌提供者 |
| `openai_token_provider.go` | OpenAI 令牌提供者 |
| `gemini_token_provider.go` | Gemini 令牌提供者 |
| `antigravity_token_provider.go` | Antigravity 令牌提供者 |
| `token_refresh_service.go` | 令牌刷新服务 |
| `token_cache_invalidator.go` | 令牌缓存失效 |
| `token_refresher.go` | Refresher 接口 |
| `antigravity_token_refresher.go` | Antigravity Refresher |
| `gemini_token_refresher.go` | Gemini Refresher |
| `gemini_token_cache.go` | Gemini 令牌缓存 |

### 33.6 调度 / 并发文件

| 文件 | 说明 |
|------|------|
| `scheduler_snapshot_service.go` | 调度器快照服务 |
| `scheduler_cache.go` | 调度器缓存接口 |
| `scheduler_outbox.go` | Outbox 事件模型 |
| `scheduler_events.go` | 调度器事件 |
| `concurrency_service.go` | 并发控制服务 |
| `session_limit_cache.go` | 会话限制缓存 |
| `temp_unsched.go` | 临时不可调度 |

### 33.7 速率限制文件

| 文件 | 说明 |
|------|------|
| `ratelimit_service.go` | 速率限制服务 |
| `model_rate_limit.go` | 模型级速率限制 |
| `gemini_quota.go` | Gemini 配额 |
| `gemini_oauth.go` | Gemini OAuth 辅助 |
| `quota_fetcher.go` | 配额查询器 |
| `antigravity_quota_fetcher.go` | Antigravity 配额查询 |
| `antigravity_quota_scope.go` | Antigravity 配额范围 |

### 33.8 运维监控文件

| 文件 | 说明 |
|------|------|
| `ops_service.go` | Ops 主服务 |
| `ops_metrics_collector.go` | 指标采集 |
| `ops_aggregation_service.go` | 聚合服务 |
| `ops_alert_evaluator_service.go` | 告警评估 |
| `ops_scheduled_report_service.go` | 定时报告 |
| `ops_cleanup_service.go` | 数据清理 |
| `ops_advisory_lock.go` | 分布式锁 |
| `ops_concurrency.go` | 并发查询 |
| `ops_dashboard.go` | 运维仪表盘 |
| `ops_realtime.go` | 实时监控 |
| `ops_realtime_traffic.go` | 实时流量 |
| `ops_health_score.go` | 健康分数 |
| `ops_trends.go` | 趋势分析 |
| `ops_histograms.go` | 直方图 |
| `ops_errors.go` | 错误分析 |
| `ops_window_stats.go` | 窗口统计 |
| `ops_request_details.go` | 请求详情 |
| `ops_query_mode.go` | 查询模式 |
| `ops_settings.go` | 设置管理 |
| `ops_upstream_context.go` | 上游上下文 |

### 33.9 幂等与安全密钥文件

| 文件 | 说明 |
|------|------|
| `idempotency.go` | 幂等协调器 (IdempotencyCoordinator) |
| `idempotency_cleanup_service.go` | 幂等记录清理服务 |
| `idempotency_observability.go` | 幂等可观测性 |
| `security_secret_bootstrap.go` (repository) | SecuritySecret 启动引导 |

### 33.10 其他文件

| 文件 | 说明 |
|------|------|
| `identity_service.go` | 身份指纹服务 |
| `anthropic_session.go` | Anthropic 会话 |
| `claude_code_validator.go` | Claude Code 验证 |
| `digest_session_store.go` | 摘要会话存储 |
| `error_passthrough_service.go` | 错误透传服务 |
| `error_passthrough_runtime.go` | 错误透传运行时 |
| `deferred_service.go` | 延迟批量更新 |
| `timing_wheel_service.go` | 时间轮服务 |
| `usage_service.go` | 使用记录服务 |
| `usage_cleanup_service.go` | 使用清理服务 |
| `usage_cache.go` | 使用缓存 |
| `account_expiry_service.go` | 账号过期服务 |
| `subscription_expiry_service.go` | 订阅过期服务 |
| `dashboard_aggregation_service.go` | 仪表盘聚合 |
| `proxy_latency_cache.go` | 代理延迟缓存 |
| `billing_cache_port.go` | 计费缓存端口 |
| `api_key_auth_cache.go` | API Key 认证缓存 |
| `api_key_auth_cache_invalidate.go` | 缓存失效 |
| `wire.go` | Wire DI 配置 |
