# Sub2API 后端架构文档：Handler / Repository / Server 层

本文档详细说明 Sub2API 后端的三层架构设计，包括 Handler 层（HTTP 处理）、Repository 层（数据访问）和 Server 层（HTTP 服务器配置）。

---

## 目录

1. [架构概览](#1-架构概览)
2. [Handler 层](#2-handler-层)
3. [Repository 层](#3-repository-层)
4. [Server 层](#4-server-层)
5. [Ent Schema（数据模型）](#5-ent-schema数据模型)
6. [依赖注入（Wire）](#6-依赖注入wire)
7. [总结](#7-总结)
8. [附录](#8-附录)

---

## 1. 架构概览

### 1.1 三层架构模式

```
┌─────────────────────────────────────────────────┐
│           Client Requests (HTTP/SSE)            │
└────────────────┬────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│  Server 层：HTTP 服务器配置、路由注册、中间件   │
│  - http.go: 服务器初始化                        │
│  - router.go: 路由设置                          │
│  - routes/: 路由分组                            │
│  - middleware/: 认证、CORS、日志等中间件         │
└────────────────┬────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│  Handler 层：HTTP 请求处理、业务编排             │
│  - gateway_handler.go: API 网关核心逻辑          │
│  - auth_handler.go: 认证登录                    │
│  - user_handler.go: 用户管理                    │
│  - admin/: 管理后台 handlers                    │
│  - dto/: 数据传输对象                           │
└────────────────┬────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│  Service 层：业务逻辑（未包含在本文档）          │
└────────────────┬────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│  Repository 层：数据持久化、缓存、外部服务       │
│  - *_repo.go: 数据库 CRUD                       │
│  - *_cache.go: Redis 缓存                       │
│  - *_service.go: HTTP 上游服务（OAuth、定价等） │
│  - ent.go: Ent ORM 客户端初始化                 │
│  - redis.go: Redis 客户端初始化                 │
└────────────────┬────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│  Infrastructure: PostgreSQL + Redis + 外部 API  │
└─────────────────────────────────────────────────┘
```

### 1.2 设计原则

1. **职责分离**：
   - Handler：HTTP 协议处理、请求验证、响应格式化
   - Service：业务逻辑、流程编排
   - Repository：数据访问、缓存策略、外部服务集成

2. **依赖注入**：使用 Google Wire 自动生成依赖注入代码，避免手动管理依赖。

3. **错误处理**：统一的错误响应格式，支持透传上游错误、业务错误转换。

4. **并发控制**：用户级 + 账户级并发槽位管理，防止 API 过载。

---

## 2. Handler 层

Handler 层负责处理 HTTP 请求，执行参数验证、业务逻辑编排、响应格式化。

### 2.1 核心文件结构

```
internal/handler/
├── handler.go              # Handler 聚合结构体定义
├── wire.go                 # Wire 依赖注入配置
├── gateway_handler.go      # Claude/Gemini API 网关核心
├── openai_gateway_handler.go  # OpenAI API 网关
├── gemini_v1beta_handler.go   # Gemini v1beta 兼容层
├── auth_handler.go         # 用户认证（登录/注册）
├── auth_linuxdo_oauth.go   # LinuxDo OAuth 集成
├── user_handler.go         # 用户信息管理
├── api_key_handler.go      # API Key 管理
├── usage_handler.go        # 用量查询
├── subscription_handler.go # 订阅管理
├── redeem_handler.go       # 兑换码
├── setting_handler.go      # 系统设置
├── totp_handler.go         # 双因素认证
├── announcement_handler.go # 公告管理
├── gateway_helper.go       # 网关辅助函数
├── sora_client_handler.go  # Sora 客户端 API (生成/配额/模型)
├── sora_gateway_handler.go # Sora 网关 (chat/completions, 媒体代理)
├── ops_error_logger.go     # 错误日志记录
├── request_body_limit.go   # 请求体大小限制
├── dto/                    # 数据传输对象
│   ├── types.go            # 通用类型定义
│   ├── mappers.go          # Entity ↔ DTO 转换
│   ├── settings.go         # 设置相关 DTO
│   └── announcement.go     # 公告相关 DTO
└── admin/                  # 管理后台 Handlers
    ├── account_handler.go      # 账户管理
    ├── account_data.go         # 账户数据处理
    ├── group_handler.go        # 分组管理
    ├── user_handler.go         # 用户管理（后台）
    ├── proxy_handler.go        # 代理配置
    ├── proxy_data.go           # 代理数据处理
    ├── redeem_handler.go       # 兑换码管理
    ├── promo_handler.go        # 推广码管理
    ├── subscription_handler.go # 订阅管理
    ├── usage_handler.go        # 用量统计
    ├── setting_handler.go      # 系统设置
    ├── announcement_handler.go # 公告管理
    ├── dashboard_handler.go    # 仪表盘
    ├── ops_handler.go          # 运维监控聚合
    ├── ops_dashboard_handler.go    # 运维仪表盘
    ├── ops_realtime_handler.go     # 实时流量监控
    ├── ops_alerts_handler.go       # 告警配置
    ├── ops_settings_handler.go     # 运维设置
    ├── ops_ws_handler.go           # WebSocket 实时数据
    ├── system_handler.go           # 系统信息/更新检查
    ├── user_attribute_handler.go   # 用户属性
    ├── error_passthrough_handler.go  # 错误透传规则
    ├── data_management_handler.go   # 数据管理（备份/S3/源配置）
    ├── apikey_handler.go           # API Key 管理（分组更新）
    ├── scheduled_test_handler.go   # 定时测试计划
    ├── idempotency_helper.go       # 幂等辅助（订阅分配等）
    ├── antigravity_oauth_handler.go  # Antigravity OAuth
    ├── openai_oauth_handler.go       # OpenAI OAuth
    └── gemini_oauth_handler.go       # Gemini OAuth
```

### 2.2 关键 Handler 详解

#### 2.2.1 `gateway_handler.go` - API 网关核心

**功能**：代理 Claude/Gemini API 请求，实现账户调度、并发控制、计费、故障转移。

**核心流程**（Messages 接口）：

```go
func (h *GatewayHandler) Messages(c *gin.Context)
```

1. **认证与上下文获取**：
   - 从中间件获取 `APIKey` 和 `AuthSubject`（用户信息）
   - 读取请求体，解析模型、stream 等参数

2. **并发控制**（三级检查）：

   ```
   ┌─────────────────────────────────────────────┐
   │ 0. Wait 队列检查                             │
   │    - 计算 maxWait = 并发数 * 2               │
   │    - IncrementWaitCount：队列满则 429        │
   └──────────────┬──────────────────────────────┘
                  │
   ┌──────────────▼──────────────────────────────┐
   │ 1. 用户并发槽位获取                          │
   │    - AcquireUserSlotWithWait                │
   │    - 等待可用槽位（最多 30 秒）              │
   │    - 获取后 DecrementWaitCount               │
   └──────────────┬──────────────────────────────┘
                  │
   ┌──────────────▼──────────────────────────────┐
   │ 2. 余额/订阅二次检查                         │
   │    - CheckBillingEligibility                │
   │    - 防止等待期间余额耗尽                    │
   └──────────────┬──────────────────────────────┘
                  │
   ┌──────────────▼──────────────────────────────┐
   │ 3. 账户并发槽位获取（Service 层）            │
   │    - 选择可用账户                            │
   │    - 获取账户级并发槽位                      │
   └──────────────┬──────────────────────────────┘
                  │
   ┌──────────────▼──────────────────────────────┐
   │ 4. 上游 API 调用 + Failover                  │
   │    - 最多尝试 MaxAccountSwitches 次          │
   │    - 429/529 错误时切换账户                  │
   └─────────────────────────────────────────────┘
   ```

3. **流式响应处理**：
   - 设置 SSE 响应头（`text/event-stream`）
   - 实时转发上游 events
   - 心跳保活（ping interval）
   - 完成后记录用量、释放槽位

4. **错误处理**：
   - 流式未开始：返回标准 JSON 错误
   - 流式已开始：发送 SSE error event
   - 错误透传：根据配置规则透传上游错误

**关键数据结构**：

```go
type GatewayHandler struct {
    gatewayService            *service.GatewayService          // 账户调度、上游调用
    geminiCompatService       *service.GeminiMessagesCompatService  // Gemini 兼容层
    antigravityGatewayService *service.AntigravityGatewayService    // Antigravity 网关
    userService               *service.UserService
    billingCacheService       *service.BillingCacheService     // 计费检查缓存
    usageService              *service.UsageService            // 用量记录
    apiKeyService             *service.APIKeyService
    errorPassthroughService   *service.ErrorPassthroughService // 错误透传规则
    concurrencyHelper         *ConcurrencyHelper               // 并发控制辅助
    maxAccountSwitches        int                              // 最大账户切换次数
    maxAccountSwitchesGemini  int
}
```

#### 2.2.2 `openai_gateway_handler.go` - OpenAI API 网关

**功能**：代理 OpenAI Responses API（聊天对话），实现与 GatewayHandler 类似的并发控制和计费。

**与 GatewayHandler 的区别**：

- 支持 `instructions` 字段自动注入（OpenCode 场景）
- 验证 `function_call_output` 必须有 `call_id` 或 `previous_response_id`
- 心跳格式不同（SSE comment 而非 Claude 的 ping event）

#### 2.2.3 `auth_handler.go` - 用户认证

**核心接口**：

| 方法                         | 路径                  | 功能                         |
| ---------------------------- | --------------------- | ---------------------------- |
| `POST /auth/login`           | Login                 | 邮箱密码登录，返回 JWT Token |
| `POST /auth/register`        | Register              | 用户注册（需检查是否开放）   |
| `POST /auth/logout`          | Logout                | 注销（清除 refresh token）   |
| `POST /auth/refresh`         | RefreshToken          | 刷新 JWT Token               |
| `GET /auth/linuxdo`          | InitiateLinuxDoOAuth  | 发起 LinuxDo OAuth 授权      |
| `GET /auth/linuxdo/callback` | HandleLinuxDoCallback | OAuth 回调处理               |

**TOTP 双因素认证流程**（Login 方法中）：

1. 验证邮箱密码
2. 检查用户是否启用 TOTP（`totp_enabled`）
3. 如果启用：
   - 验证请求中的 `totp_code`
   - 使用 AES 解密 `totp_secret_encrypted`
   - 验证 TOTP 码有效性
4. 通过后生成 JWT

#### 2.2.4 `admin/` - 管理后台 Handlers

管理后台 Handlers 提供完整的 CRUD 接口，主要特点：

- **权限控制**：通过 `AdminAuthMiddleware` 验证管理员角色
- **批量操作**：账户/分组支持批量导入导出
- **运维监控**：
  - `ops_dashboard_handler.go`：统计数据聚合（请求数、错误率、账户健康度）
  - `ops_realtime_handler.go`：实时流量监控（每秒请求数、活跃用户）
  - `ops_alerts_handler.go`：告警配置（错误率、账户失败等）
  - `ops_ws_handler.go`：WebSocket 推送实时数据

**示例：账户管理 Handler**

```go
// admin/account_handler.go
type AccountHandler struct {
    accountService *service.AccountService
    groupService   *service.GroupService
}

// 核心方法
List(c *gin.Context)           // 分页查询账户列表，支持过滤
Get(c *gin.Context)            // 获取账户详情
Create(c *gin.Context)         // 创建账户
Update(c *gin.Context)         // 更新账户配置
Delete(c *gin.Context)         // 软删除账户
BatchCreate(c *gin.Context)    // 批量创建（JSON/CSV）
BatchUpdate(c *gin.Context)    // 批量更新状态/分组
RefreshToken(c *gin.Context)   // 刷新 OAuth Token
```

### 2.3 DTO（数据传输对象）

位于 `handler/dto/`，负责 Entity ↔ API 响应的转换。

#### 2.3.1 `dto/types.go` - 通用类型

```go
// 分页响应
type PaginatedResponse[T any] struct {
    Items      []T   `json:"items"`
    Total      int64 `json:"total"`
    Page       int   `json:"page"`
    PageSize   int   `json:"page_size"`
}

// 用户信息响应
type UserResponse struct {
    ID                int64     `json:"id"`
    Email             string    `json:"email"`
    Username          string    `json:"username"`
    Role              string    `json:"role"`
    Balance           float64   `json:"balance"`
    Concurrency       int       `json:"concurrency"`
    Status            string    `json:"status"`
    TotpEnabled       bool      `json:"totp_enabled"`
    SubscriptionInfo  *SubscriptionInfo `json:"subscription_info,omitempty"`
    CreatedAt         time.Time `json:"created_at"`
}

// 账户响应
type AccountResponse struct {
    ID                int64                  `json:"id"`
    Name              string                 `json:"name"`
    Platform          string                 `json:"platform"`
    Type              string                 `json:"type"`
    Credentials       map[string]interface{} `json:"credentials,omitempty"`
    Groups            []GroupSummary         `json:"groups"`
    Concurrency       int                    `json:"concurrency"`
    Priority          int                    `json:"priority"`
    Schedulable       bool                   `json:"schedulable"`
    Status            string                 `json:"status"`
    // ...
}
```

#### 2.3.2 `dto/mappers.go` - 转换函数

```go
func ToUserResponse(u *ent.User, sub *ent.UserSubscription) *UserResponse
func ToAccountResponse(acc *ent.Account, includeGroups bool) *AccountResponse
func ToGroupResponse(g *ent.Group) *GroupResponse
// ...
```

---

## 3. Repository 层

Repository 层封装数据访问逻辑，包括数据库 CRUD、Redis 缓存、外部 HTTP 服务集成。

### 3.1 核心文件结构

```
internal/repository/
├── ent.go                   # Ent ORM 客户端初始化
├── wire.go                  # Wire 依赖注入配置
├── redis.go                 # Redis 客户端初始化
├── db_pool.go               # 数据库连接池配置
├── migrations_runner.go     # SQL 迁移执行器
├── error_translate.go       # 数据库错误转换
├── pagination.go            # 分页辅助函数
├── sql_scan.go              # 原生 SQL 扫描辅助
│
├── *_repo.go                # 数据库 Repository
│   ├── user_repo.go
│   ├── api_key_repo.go
│   ├── account_repo.go
│   ├── group_repo.go
│   ├── proxy_repo.go
│   ├── redeem_code_repo.go
│   ├── promo_code_repo.go
│   ├── usage_log_repo.go
│   ├── usage_cleanup_repo.go
│   ├── setting_repo.go
│   ├── announcement_repo.go
│   ├── announcement_read_repo.go
│   ├── user_subscription_repo.go
│   ├── user_attribute_repo.go
│   ├── user_group_rate_repo.go
│   ├── error_passthrough_repo.go
│   ├── dashboard_aggregation_repo.go
│   ├── scheduler_outbox_repo.go
│   └── ops_repo*.go         # 运维数据仓库（9个文件）
│
├── *_cache.go               # Redis 缓存
│   ├── gateway_cache.go         # 账户选择缓存
│   ├── billing_cache.go         # 计费检查缓存
│   ├── api_key_cache.go         # API Key 缓存
│   ├── concurrency_cache.go     # 并发槽位管理
│   ├── session_limit_cache.go   # 会话限制
│   ├── timeout_counter_cache.go # 超时计数
│   ├── temp_unsched_cache.go    # 临时不可调度标记
│   ├── dashboard_cache.go       # 仪表盘缓存
│   ├── email_cache.go           # 邮件发送限流
│   ├── identity_cache.go        # 身份验证缓存
│   ├── redeem_cache.go          # 兑换码缓存
│   ├── update_cache.go          # 更新检查缓存
│   ├── gemini_token_cache.go    # Gemini Token 缓存
│   ├── scheduler_cache.go       # 调度器缓存
│   ├── proxy_latency_cache.go   # 代理延迟缓存
│   ├── totp_cache.go            # TOTP 验证缓存
│   ├── refresh_token_cache.go   # Refresh Token 缓存
│   └── error_passthrough_cache.go # 错误透传缓存
│
├── *_service.go             # HTTP 上游服务
│   ├── http_upstream.go         # 通用 HTTP 客户端
│   ├── turnstile_service.go     # Cloudflare Turnstile 验证
│   ├── pricing_service.go       # 定价数据获取（GitHub）
│   ├── github_release_service.go # GitHub Release 检查
│   ├── proxy_probe_service.go   # 代理出口探测
│   ├── claude_usage_service.go  # Claude 用量查询
│   ├── claude_oauth_service.go  # Claude OAuth
│   ├── openai_oauth_service.go  # OpenAI OAuth
│   ├── gemini_oauth_client.go   # Gemini OAuth
│   └── geminicli_codeassist_client.go # Gemini CLI Code Assist
│
├── aes_encryptor.go         # AES 加密（TOTP secret）
├── req_client_pool.go       # HTTP 客户端池
└── simple_mode_default_groups.go # Simple 模式默认分组
```

### 3.2 关键 Repository 详解

#### 3.2.1 `ent.go` - Ent ORM 初始化

**功能**：初始化 Ent 客户端、连接池、自动迁移。

```go
func InitEnt(cfg *config.Config) (*ent.Client, *sql.DB, error)
```

**执行步骤**：

1. **时区初始化**：确保全局时区一致（`timezone.Init`）
2. **数据库连接**：使用 PostgreSQL 驱动建立连接
3. **连接池配置**：

   ```go
   db.SetMaxOpenConns(cfg.Database.MaxOpenConns)     // 最大连接数
   db.SetMaxIdleConns(cfg.Database.MaxIdleConns)     // 最大空闲连接
   db.SetConnMaxLifetime(time.Hour)                  // 连接最长存活时间
   ```

4. **自动迁移**：执行 `migrations/` 目录下的 SQL 文件
5. **Simple 模式分组初始化**：为 anthropic/openai/gemini 创建默认分组

#### 3.2.2 `redis.go` - Redis 客户端初始化

```go
func InitRedis(cfg *config.Config) *redis.Client
```

- 支持单机模式和哨兵模式
- 连接池配置（PoolSize, MinIdleConns）
- 健康检查（Ping）

#### 3.2.3 `*_repo.go` - 数据库 Repository

##### 3.2.3.1 `user_repo.go` - 用户数据访问

```go
type UserRepository struct {
    client *ent.Client
}

// 核心方法
Create(ctx, email, passwordHash string) (*ent.User, error)
FindByEmail(ctx, email string) (*ent.User, error)
FindByID(ctx, userID int64) (*ent.User, error)
List(ctx, page, pageSize int, filters UserFilters) ([]*ent.User, int, error)
UpdateBalance(ctx, userID int64, delta float64) error
UpdateTOTPSecret(ctx, userID int64, encryptedSecret string, enabled bool) error
SoftDelete(ctx, userID int64) error
// ...
```

**特点**：

- 软删除支持（`deleted_at IS NULL` 条件）
- 余额更新使用 `ent.Raw` 原子递增
- 分页查询复用 `pagination.go` 辅助函数

##### 3.2.3.2 `account_repo.go` - 账户数据访问

```go
type AccountRepository struct {
    client *ent.Client
}

// 调度相关
FindSchedulableByGroupID(ctx, groupID int64) ([]*ent.Account, error)
MarkRateLimited(ctx, accountID int64, resetAt time.Time) error
MarkOverloaded(ctx, accountID int64, until time.Time) error
UpdateSchedulable(ctx, accountID int64, schedulable bool) error

// CRUD
Create(ctx, req *CreateAccountRequest) (*ent.Account, error)
Update(ctx, accountID int64, updates map[string]interface{}) error
BatchUpdateStatus(ctx, accountIDs []int64, status string) error
// ...
```

##### 3.2.3.3 `ops_repo*.go` - 运维数据仓库

运维监控使用了 **9 个独立的 Repository 文件**，高度模块化：

| 文件                                    | 功能                                     |
| --------------------------------------- | ---------------------------------------- |
| `ops_repo.go`                           | 主仓库，聚合其他子仓库                   |
| `ops_repo_dashboard.go`                 | 仪表盘统计（今日/本周/本月请求数、费用） |
| `ops_repo_metrics.go`                   | 核心指标（错误率、平均延迟、P95/P99）    |
| `ops_repo_trends.go`                    | 趋势分析（请求量、费用、错误率曲线）     |
| `ops_repo_realtime_traffic.go`          | 实时流量（每秒请求数、活跃用户）         |
| `ops_repo_histograms.go`                | 直方图分桶（延迟分布）                   |
| `ops_repo_latency_histogram_buckets.go` | 延迟分桶配置                             |
| `ops_repo_request_details.go`           | 请求详情查询（用于调试）                 |
| `ops_repo_alerts.go`                    | 告警检测（错误率阈值、账户失败）         |
| `ops_repo_preagg.go`                    | 预聚合数据（加速仪表盘查询）             |
| `ops_repo_window_stats.go`              | 窗口统计（滑动窗口内的汇总）             |

**设计亮点**：

- 使用原生 SQL 执行复杂聚合查询（Ent ORM 不适合）
- 时间窗口查询优化（索引 `idx_usage_logs_created_at`）
- 预聚合表减少实时计算压力

#### 3.2.4 `*_cache.go` - Redis 缓存

##### 3.2.4.1 `concurrency_cache.go` - 并发槽位管理

```go
type ConcurrencyCache struct {
    rdb              *redis.Client
    slotTTLMinutes   int  // 槽位 TTL（防止泄漏）
    waitTTLSeconds   int  // 等待队列 TTL
}

// 用户并发槽位
AcquireUserSlot(ctx, userID int64, maxConcurrency int) (bool, error)
ReleaseUserSlot(ctx, userID int64) error
GetUserSlotCount(ctx, userID int64) (int, error)

// 账户并发槽位
AcquireAccountSlot(ctx, accountID int64, maxConcurrency int) (bool, error)
ReleaseAccountSlot(ctx, accountID int64) error

// 等待队列计数
IncrementWaitCount(ctx, userID int64, maxWait int) (bool, error)
DecrementWaitCount(ctx, userID int64) error
```

**实现原理**：

- Redis Hash 存储 `concurrency:user:{userID}` → `{timestamp: slot_id}`
- `HLEN` 获取当前槽位数
- `HSETNX` 原子抢占槽位
- `HDEL` 释放槽位
- `EXPIRE` 设置 TTL 防止客户端断连泄漏

##### 3.2.4.2 `gateway_cache.go` - 账户选择缓存

```go
type GatewayCache struct {
    rdb *redis.Client
}

// 粘性会话（Sticky Session）
GetStickyAccount(ctx, userID, groupID int64) (int64, error)
SetStickyAccount(ctx, userID, groupID, accountID int64, ttl time.Duration) error
ClearStickyAccount(ctx, userID, groupID int64) error

// 账户健康度记录
IncrementAccountSuccess(ctx, accountID int64) error
IncrementAccountError(ctx, accountID int64) error
GetAccountHealthScore(ctx, accountID int64) (float64, error)
```

**Sticky Session 策略**：

- 同一用户短期内（TTL）复用同一账户
- 减少账户切换开销，提升响应速度
- 账户失败时自动清除 Sticky，允许重新选择

#### 3.2.5 `*_service.go` - HTTP 上游服务

##### 3.2.5.1 `claude_oauth_service.go` - Claude OAuth 集成

```go
type ClaudeOAuthClient struct {
    upstreamClient service.HTTPUpstream
}

// OAuth 令牌刷新
RefreshToken(ctx, refreshToken string, proxy *ent.Proxy) (*RefreshTokenResponse, error)

// 令牌撤销
RevokeToken(ctx, token string, proxy *ent.Proxy) error
```

##### 3.2.5.2 `pricing_service.go` - 定价数据获取

```go
type PricingRemoteClient struct {
    client *req.Client
}

// 从 GitHub 拉取最新定价配置
FetchPricing(ctx) (*PricingData, error)
```

**数据源**：`https://raw.githubusercontent.com/Wei-Shaw/sub2api/main/pricing.json`

---

## 4. Server 层

Server 层负责 HTTP 服务器初始化、路由注册、中间件配置。

### 4.1 核心文件

```
internal/server/
├── http.go                 # HTTP 服务器初始化
├── router.go               # 路由配置主函数
├── routes/                 # 路由分组
│   ├── common.go           # 健康检查等通用路由
│   ├── auth.go             # 认证路由
│   ├── user.go             # 用户路由
│   ├── admin.go            # 管理后台路由
│   ├── gateway.go          # API 网关路由
│   └── sora_client.go      # Sora 客户端 API 路由
└── middleware/             # 中间件
    ├── middleware.go       # 中间件聚合
    ├── wire.go             # Wire 配置
    ├── logger.go           # 请求日志
    ├── recovery.go         # Panic 恢复
    ├── cors.go             # 跨域配置
    ├── security_headers.go # 安全头设置
    ├── jwt_auth.go         # JWT 认证
    ├── admin_auth.go       # 管理员认证
    ├── admin_only.go       # 管理员权限检查
    ├── api_key_auth.go     # API Key 认证
    ├── api_key_auth_google.go # Google API Key 认证
    ├── auth_subject.go     # 认证主体提取
    ├── client_request_id.go # 客户端请求 ID
    └── request_body_limit.go # 请求体大小限制
```

### 4.2 `http.go` - 服务器初始化

```go
func ProvideHTTPServer(cfg *config.Config, router *gin.Engine) *http.Server
```

**配置要点**：

1. **HTTP/2 Cleartext (h2c) 支持**：

   ```go
   if cfg.Server.H2C.Enabled {
       httpHandler = h2c.NewHandler(router, &http2.Server{
           MaxConcurrentStreams:         300,
           IdleTimeout:                  120s,
           MaxReadFrameSize:             1MB,
           MaxUploadBufferPerConnection: 1MB,
           MaxUploadBufferPerStream:     1MB,
       })
   }
   ```

2. **请求体大小限制**：

   ```go
   httpHandler = http.MaxBytesHandler(httpHandler, cfg.Gateway.MaxBodySize)
   ```

3. **超时配置**：
   - `ReadHeaderTimeout`：防止慢速请求头攻击
   - `IdleTimeout`：释放空闲连接
   - **不设置** `WriteTimeout` 和 `ReadTimeout`：因为流式响应可能持续十几分钟

### 4.3 `router.go` - 路由配置

```go
func SetupRouter(r *gin.Engine, handlers *handler.Handlers, ...) *gin.Engine
```

**中间件应用顺序**：

```go
r.Use(middleware2.Recovery())       // Panic 恢复（最高优先级）
r.Use(middleware2.Logger())         // 请求日志
r.Use(middleware2.CORS(cfg.CORS))   // 跨域
r.Use(middleware2.SecurityHeaders(cfg.Security.CSP)) // 安全头
```

**前端静态资源服务**：

```go
if web.HasEmbeddedFrontend() {
    frontendServer, err := web.NewFrontendServer(settingService)
    // 注册设置缓存失效回调
    settingService.SetOnUpdateCallback(frontendServer.InvalidateCache)
    r.Use(frontendServer.Middleware())
}
```

### 4.4 `routes/` - 路由分组

#### 4.4.1 `routes/gateway.go` - API 网关路由

```go
func RegisterGatewayRoutes(r *gin.Engine, h *handler.Handlers, apiKeyAuth, ...) {
    // Claude/Gemini API 网关
    r.POST("/v1/messages",
        apiKeyAuth.ApiKeyAuth(),
        middleware2.SubscriptionMiddleware(subscriptionService),
        middleware2.OpsMiddleware(opsService),
        h.Gateway.Messages,
    )

    // Gemini v1beta 兼容层
    r.POST("/gemini/v1beta/chat/completions",
        apiKeyAuth.ApiKeyAuthGoogle(),
        // ...
        h.Gateway.GeminiV1Beta,
    )

    // OpenAI Responses API
    r.POST("/openai/v1/responses",
        apiKeyAuth.ApiKeyAuth(),
        // ...
        h.OpenAIGateway.Responses,
    )
}
```

**中间件链**：

```
Request → ApiKeyAuth（验证 API Key）
        → SubscriptionMiddleware（检查订阅状态）
        → OpsMiddleware（运维监控埋点）
        → Handler（业务逻辑）
```

#### 4.4.2 `routes/admin.go` - 管理后台路由

```go
func RegisterAdminRoutes(v1 *gin.RouterGroup, h *handler.Handlers, adminAuth) {
    admin := v1.Group("/admin", adminAuth.AdminAuth())
    {
        // 用户管理
        admin.GET("/users", h.Admin.User.List)
        admin.POST("/users", h.Admin.User.Create)
        // ...

        // 账户管理
        admin.GET("/accounts", h.Admin.Account.List)
        admin.POST("/accounts", h.Admin.Account.Create)
        admin.POST("/accounts/batch", h.Admin.Account.BatchCreate)
        // ...

        // 分组管理
        admin.GET("/groups", h.Admin.Group.List)
        // ...

        // 运维监控
        ops := admin.Group("/ops")
        {
            ops.GET("/dashboard", h.Admin.Ops.GetDashboard)
            ops.GET("/metrics", h.Admin.Ops.GetMetrics)
            ops.GET("/realtime", h.Admin.Ops.GetRealtimeTraffic)
            ops.GET("/ws", h.Admin.Ops.HandleWebSocket)
            // ...
        }
    }
}
```

### 4.5 `middleware/` - 中间件详解

#### 4.5.1 `api_key_auth.go` - API Key 认证

```go
func (m *apiKeyAuthMiddleware) ApiKeyAuth() gin.HandlerFunc {
    return func(c *gin.Context) {
        // 1. 从请求头提取 API Key
        apiKeyStr := extractAPIKey(c)

        // 2. 查询 API Key（含缓存）
        apiKey, err := m.apiKeyService.GetByKey(c.Request.Context(), apiKeyStr)

        // 3. 验证 API Key 状态、IP 白名单、过期时间、Quota

        // 4. 查询 User、Group、Subscription

        // 5. 计算 AuthSubject（用户并发数、分组费率等）
        subject := middleware2.AuthSubject{
            UserID:      apiKey.User.ID,
            Concurrency: apiKey.User.Concurrency,
            GroupID:     apiKey.GroupID,
            // ...
        }

        // 6. 存入 Context
        c.Set(middleware2.CtxAPIKey, apiKey)
        c.Set(middleware2.CtxAuthSubject, subject)

        c.Next()
    }
}
```

**缓存策略**：

- API Key 元数据缓存 5 分钟（`api_key_cache.go`）
- 减少数据库查询压力

#### 4.5.2 `jwt_auth.go` - JWT 认证

```go
func (m *jwtAuthMiddleware) JWTAuth() gin.HandlerFunc {
    return func(c *gin.Context) {
        // 1. 从 Authorization 头提取 Token
        token := extractToken(c)

        // 2. 验证 JWT 签名和有效期
        claims, err := m.jwtService.ValidateToken(token)

        // 3. 查询用户（验证是否仍然存在/启用）
        user, err := m.userService.GetByID(c.Request.Context(), claims.UserID)

        // 4. 存入 Context
        c.Set(middleware2.CtxUser, user)

        c.Next()
    }
}
```

#### 4.5.3 `logger.go` - 请求日志

```go
func Logger() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()

        c.Next()

        latency := time.Since(start)
        log.Printf("[%s] %s %s %d %s",
            c.Request.Method,
            c.Request.URL.Path,
            c.ClientIP(),
            c.Writer.Status(),
            latency,
        )
    }
}
```

---

## 5. Ent Schema（数据模型）

Ent 是 Facebook 开源的 ORM 框架，Schema 定义在 `backend/ent/schema/` 目录。

### 5.1 核心 Schema

#### 5.1.1 `user.go` - 用户

```go
type User struct {
    ent.Schema
}

Fields() []ent.Field {
    field.String("email").MaxLen(255).NotEmpty(),
    field.String("password_hash").MaxLen(255).NotEmpty(),
    field.String("role").Default(domain.RoleUser),        // "user" | "admin"
    field.Float("balance").Default(0),                    // 余额（USD）
    field.Int("concurrency").Default(5),                  // 并发数
    field.String("status").Default(domain.StatusActive),  // "active" | "disabled"
    field.String("username").MaxLen(100).Default(""),
    field.String("notes").SchemaType("text").Default(""),

    // TOTP 双因素认证
    field.String("totp_secret_encrypted").Optional().Nillable(),
    field.Bool("totp_enabled").Default(false),
    field.Time("totp_enabled_at").Optional().Nillable(),
}

Edges() []ent.Edge {
    edge.To("api_keys", APIKey.Type),
    edge.To("subscriptions", UserSubscription.Type),
    edge.To("allowed_groups", Group.Type).Through("user_allowed_groups", UserAllowedGroup.Type),
    edge.To("usage_logs", UsageLog.Type),
    // ...
}
```

**软删除 Mixin**：

- `mixins.SoftDeleteMixin`：提供 `deleted_at` 字段
- 唯一索引通过部分索引实现（`WHERE deleted_at IS NULL`）

#### 5.1.2 `api_key.go` - API Key

```go
Fields() []ent.Field {
    field.Int64("user_id"),
    field.String("key").MaxLen(128).Unique(),
    field.String("name").MaxLen(100),
    field.Int64("group_id").Optional().Nillable(),
    field.String("status").Default(domain.StatusActive),
    field.JSON("ip_whitelist", []string{}).Optional(),    // IP 白名单
    field.JSON("ip_blacklist", []string{}).Optional(),    // IP 黑名单

    // Quota 限制
    field.Float("quota").Default(0),                      // 额度限制（USD，0 = 无限）
    field.Float("quota_used").Default(0),                 // 已用额度
    field.Time("expires_at").Optional().Nillable(),       // 过期时间
}
```

#### 5.1.3 `account.go` - AI API 账户

```go
Fields() []ent.Field {
    field.String("name").MaxLen(100),
    field.String("platform"),                             // "claude" | "gemini" | "openai"
    field.String("type"),                                 // "api_key" | "oauth" | "cookie"
    field.JSON("credentials", map[string]any{}),          // 凭证（JSONB）
    field.JSON("extra", map[string]any{}),                // 扩展数据
    field.Int64("proxy_id").Optional().Nillable(),
    field.Int("concurrency").Default(3),                  // 账户并发数
    field.Int("priority").Default(50),                    // 优先级
    field.Float("rate_multiplier").Default(1.0),          // 计费倍率
    field.String("status").Default(domain.StatusActive),

    // 调度相关
    field.Bool("schedulable").Default(true),
    field.Time("rate_limited_at").Optional().Nillable(),
    field.Time("rate_limit_reset_at").Optional().Nillable(),
    field.Time("overload_until").Optional().Nillable(),
    field.Time("session_window_start").Optional().Nillable(),
    field.Time("session_window_end").Optional().Nillable(),

    // 过期管理
    field.Time("expires_at").Optional().Nillable(),
    field.Bool("auto_pause_on_expired").Default(true),
}

Edges() []ent.Edge {
    edge.To("groups", Group.Type).Through("account_groups", AccountGroup.Type),
    edge.To("proxy", Proxy.Type).Field("proxy_id").Unique(),
    edge.To("usage_logs", UsageLog.Type),
}
```

#### 5.1.4 `group.go` - 分组

```go
Fields() []ent.Field {
    field.String("name").MaxLen(100),
    field.String("description").Optional().Nillable(),
    field.Float("rate_multiplier").Default(1.0),          // 费率倍数
    field.Bool("is_exclusive").Default(false),            // 独占分组
    field.String("status").Default(domain.StatusActive),

    // 订阅相关
    field.String("platform").Default(domain.PlatformAnthropic),
    field.String("subscription_type").Default(domain.SubscriptionTypeStandard),
    field.Float("daily_limit_usd").Optional().Nillable(),
    field.Float("weekly_limit_usd").Optional().Nillable(),
    field.Float("monthly_limit_usd").Optional().Nillable(),
    field.Int("default_validity_days").Default(30),

    // 图片生成计费
    field.Float("image_price_1k").Optional().Nillable(),
    field.Float("image_price_2k").Optional().Nillable(),
    field.Float("image_price_4k").Optional().Nillable(),

    // Claude Code 限制
    field.Bool("claude_code_only").Default(false),
    field.Int64("fallback_group_id").Optional().Nillable(),
    field.Int64("fallback_group_id_on_invalid_request").Optional().Nillable(),

    // 模型路由
    field.JSON("model_routing", map[string][]int64{}).Optional(),
    field.Bool("model_routing_enabled").Default(false),
    field.Bool("mcp_xml_inject").Default(true),

    // 支持的模型系列
    field.JSON("supported_model_scopes", []string{}).Default([]string{"claude", "gemini_text", "gemini_image"}),

    field.Int("sort_order").Default(0),                   // 显示排序
}
```

#### 5.1.5 `usage_log.go` - 用量日志

```go
Fields() []ent.Field {
    field.Int64("user_id"),
    field.Int64("api_key_id"),
    field.Int64("account_id"),
    field.String("request_id").MaxLen(64).NotEmpty(),
    field.String("model").MaxLen(100).NotEmpty(),
    field.Int64("group_id").Optional().Nillable(),
    field.Int64("subscription_id").Optional().Nillable(),

    field.Int("input_tokens").Default(0),
    field.Int("output_tokens").Default(0),
    field.Int("cache_creation_tokens").Default(0),
    field.Int("cache_read_tokens").Default(0),
    field.Int("cache_creation_5m_tokens").Default(0),
    field.Int("cache_creation_1h_tokens").Default(0),

    field.Float("input_cost").Default(0),
    field.Float("output_cost").Default(0),
    field.Float("cache_creation_cost").Default(0),
    field.Float("cache_read_cost").Default(0),
    field.Float("total_cost").Default(0),
    field.Float("actual_cost").Default(0),
    field.Float("rate_multiplier").Default(1),
    field.Float("account_rate_multiplier").Optional().Nillable(),

    field.Int8("billing_type").Default(0),
    field.Bool("stream").Default(false),
    field.Int("duration_ms").Optional().Nillable(),
    field.Int("first_token_ms").Optional().Nillable(),
    field.String("user_agent").MaxLen(512).Optional().Nillable(),
    field.String("ip_address").MaxLen(45).Optional().Nillable(),
    field.Int("image_count").Default(0),
    field.String("image_size").MaxLen(10).Optional().Nillable(),

    field.Time("created_at"),
}

Indexes() []ent.Index {
    index.Fields("user_id"),
    index.Fields("api_key_id"),
    index.Fields("account_id"),
    index.Fields("group_id"),
    index.Fields("created_at"),
}
```

### 5.2 Mixin（可复用组件）

#### 5.2.1 `mixins/time.go` - 时间戳 Mixin

```go
type TimeMixin struct {
    mixin.Schema
}

func (TimeMixin) Fields() []ent.Field {
    return []ent.Field{
        field.Time("created_at").
            Default(time.Now).
            Immutable().
            SchemaType(map[string]string{dialect.Postgres: "timestamptz"}),
        field.Time("updated_at").
            Default(time.Now).
            UpdateDefault(time.Now).
            SchemaType(map[string]string{dialect.Postgres: "timestamptz"}),
    }
}
```

#### 5.2.2 `mixins/soft_delete.go` - 软删除 Mixin

```go
type SoftDeleteMixin struct {
    mixin.Schema
}

func (SoftDeleteMixin) Fields() []ent.Field {
    return []ent.Field{
        field.Time("deleted_at").
            Optional().
            Nillable().
            SchemaType(map[string]string{dialect.Postgres: "timestamptz"}),
    }
}

func (SoftDeleteMixin) Hooks() []ent.Hook {
    return []ent.Hook{
        // 查询时自动过滤已删除记录
        hook.On(func(next ent.Mutator) ent.Mutator {
            return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
                if q, ok := m.(interface{ WhereP(...func(*sql.Selector)) }); ok {
                    q.WhereP(sql.FieldIsNull("deleted_at"))
                }
                return next.Mutate(ctx, m)
            })
        }, ent.OpQuery),
    }
}
```

---

## 6. 依赖注入（Wire）

项目使用 [Google Wire](https://github.com/google/wire) 实现依赖注入，避免手动管理复杂依赖关系。

### 6.1 Wire 配置文件

```
backend/
├── cmd/server/wire.go              # 应用主 Wire 配置
├── internal/handler/wire.go        # Handler 层 ProviderSet
├── internal/repository/wire.go     # Repository 层 ProviderSet
├── internal/server/middleware/wire.go  # Middleware ProviderSet
└── internal/service/wire.go        # Service 层 ProviderSet
```

### 6.2 主 Wire 配置（`cmd/server/wire.go`）

```go
//go:build wireinject
// +build wireinject

package main

import (
    "github.com/Wei-Shaw/sub2api/internal/config"
    "github.com/Wei-Shaw/sub2api/internal/handler"
    "github.com/Wei-Shaw/sub2api/internal/repository"
    "github.com/Wei-Shaw/sub2api/internal/server"
    "github.com/Wei-Shaw/sub2api/internal/service"
    middleware2 "github.com/Wei-Shaw/sub2api/internal/server/middleware"

    "github.com/google/wire"
    "net/http"
)

func InitializeServer(cfg *config.Config) (*http.Server, error) {
    wire.Build(
        // BuildInfo
        wire.Value(handler.BuildInfo{
            Version:   Version,
            BuildType: BuildType,
        }),

        // Infrastructure
        repository.ProviderSet,    // Ent, Redis, Repos, Caches

        // Services
        service.ProviderSet,       // 业务逻辑层

        // Handlers
        handler.ProviderSet,       // HTTP Handlers

        // Middleware
        middleware2.ProviderSet,   // 中间件

        // Server
        server.ProviderSet,        // HTTP Server 和 Router
    )
    return nil, nil
}
```

### 6.3 ProviderSet 示例

#### 6.3.1 `repository/wire.go`

```go
var ProviderSet = wire.NewSet(
    // Ent Client and DB
    ProvideEnt,
    ProvideSQLDB,
    ProvideRedis,

    // Repositories
    NewUserRepository,
    NewAPIKeyRepository,
    NewAccountRepository,
    NewGroupRepository,
    NewUsageLogRepository,
    // ...

    // Caches
    NewGatewayCache,
    NewBillingCache,
    ProvideConcurrencyCache,
    // ...

    // HTTP Services
    NewHTTPUpstream,
    NewTurnstileVerifier,
    NewPricingRemoteClient,
    NewClaudeOAuthClient,
    // ...
)
```

#### 6.3.2 `handler/wire.go`

```go
var ProviderSet = wire.NewSet(
    // Top-level handlers
    NewAuthHandler,
    NewUserHandler,
    NewAPIKeyHandler,
    NewGatewayHandler,
    NewOpenAIGatewayHandler,
    // ...

    // Admin handlers
    admin.NewDashboardHandler,
    admin.NewUserHandler,
    admin.NewAccountHandler,
    // ...

    // Aggregators
    ProvideAdminHandlers,
    ProvideHandlers,
)
```

### 6.4 Wire 工作流程

1. **编写 Provider 函数**：每个依赖项有一个构造函数（如 `NewUserRepository`）
2. **定义 ProviderSet**：组织相关 Providers
3. **编写 Injector 函数**：在 `wire.go` 中定义 `InitializeServer`
4. **生成代码**：运行 `wire` 命令生成 `wire_gen.go`
5. **使用**：`main.go` 调用生成的 `InitializeServer`

**优势**：

- 编译时检查依赖循环
- 自动推导依赖顺序
- 类型安全
- 减少样板代码

---

## 7. 总结

### 7.1 关键设计模式

1. **分层架构**：
   - Handler → Service → Repository → Infrastructure
   - 职责清晰，易于测试和维护

2. **依赖注入**：
   - 使用 Wire 自动生成依赖图
   - 避免全局变量，提升可测试性

3. **缓存策略**：
   - 多级缓存（Redis + 内存）
   - 缓存失效机制（TTL + 手动清除）

4. **并发控制**：
   - 用户级 + 账户级双层限流
   - Redis 原子操作保证一致性
   - TTL 防止槽位泄漏

5. **错误处理**：
   - 统一错误响应格式
   - 错误透传规则配置
   - 详细的运维日志

6. **可扩展性**：
   - 模块化设计（Ops 监控拆分 9 个文件）
   - 插件式 OAuth 集成
   - 灵活的分组和路由配置

### 7.2 性能优化要点

1. **数据库查询优化**：
   - 索引覆盖（`usage_logs.created_at`, `accounts.platform` 等）
   - 预聚合表减少实时计算
   - 批量查询减少往返

2. **缓存策略**：
   - API Key 缓存 5 分钟
   - 账户选择 Sticky Session
   - 预热关键缓存

3. **并发控制**：
   - 非阻塞槽位获取（`SETNX`）
   - 等待队列限流防止雪崩
   - TTL 自动清理

4. **流式响应**：
   - SSE 低延迟推送
   - 心跳保活机制
   - 客户端断连检测

### 7.3 安全机制

1. **认证与授权**：
   - JWT Token 认证
   - API Key + IP 白名单
   - 管理员角色隔离

2. **速率限制**：
   - 用户并发数限制
   - API Key Quota 限制
   - 邮件发送频率限制

3. **数据加密**：
   - TOTP Secret AES 加密
   - 密码 bcrypt 哈希
   - 敏感字段加密存储

4. **输入验证**：
   - 请求体大小限制
   - 参数类型验证
   - SQL 注入防护（Ent ORM）

---

## 8. 附录

### 8.1 常用命令

```bash
# 生成 Wire 依赖注入代码
cd cmd/server && wire

# 运行 Ent 代码生成
go generate ./ent

# 数据库迁移（自动执行）
# 见 repository/ent.go:InitEnt()

# 运行测试
go test ./...

# 构建
go build -o sub2api cmd/server/main.go
```

### 8.2 参考文档

- [Ent ORM 文档](https://entgo.io/docs/getting-started)
- [Google Wire 文档](https://github.com/google/wire/blob/main/docs/guide.md)
- [Gin Web Framework](https://gin-gonic.com/docs/)
- [Redis Go Client](https://redis.uptrace.dev/)
