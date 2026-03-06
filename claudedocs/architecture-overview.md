# Sub2API 架构概览

## 1. 项目定位

Sub2API 是一个 AI API 网关平台，用于分发和管理 AI 产品订阅（如 Claude Code $200/月）的 API 配额。用户通过平台生成的 API Key 调用上游 AI 服务，平台负责鉴权、计费、负载均衡和请求转发。

**核心价值**：

- 多账号聚合管理
- 精确 Token 级计费
- 智能调度与负载均衡
- 并发控制与速率限制
- 完整的 SaaS 运营能力

## 2. 技术栈

| 组件      | 技术                             |
| --------- | -------------------------------- |
| 后端框架  | Go 1.25.7 + Gin (HTTP)           |
| ORM       | Ent (Code First, 类型安全)       |
| 数据库    | PostgreSQL 15+                   |
| 缓存/队列 | Redis 7+                         |
| 前端      | Vue 3.4+ + Vite 5+ + TailwindCSS |
| 依赖注入  | Wire (编译时注入)                |
| 部署      | Docker / 脚本安装 / Systemd      |

## 3. 分层架构

Sub2API 遵循经典的分层架构模式，从外到内依次为：

```
HTTP Request
    ↓
【Middleware 中间件层】
    - Recovery (恐慌恢复)
    - CORS (跨域控制)
    - SecurityHeaders (CSP/安全头)
    - JWTAuth / AdminAuth / APIKeyAuth (认证鉴权)
    - RequestBodyLimit (请求体大小限制)
    - ClientRequestID (请求追踪)
    ↓
【Handler 处理层】
    - 参数校验与绑定
    - 业务编排调用
    - 响应格式化
    ↓
【Service 业务层】
    - 核心业务逻辑
    - 多 Repository 协调
    - 事务管理
    - 缓存策略
    ↓
【Repository 数据访问层】
    - Ent ORM 操作
    - Redis 缓存操作
    - 第三方 API 调用
    ↓
【数据存储层】
    - PostgreSQL (持久化)
    - Redis (缓存/会话)
```

### 3.1 依赖注入流程

使用 Wire 进行编译时依赖注入，主要流程：

```
wire.go (定义依赖关系)
    ↓
wire gen (生成 wire_gen.go)
    ↓
wire_gen.go (initializeApplication)
    - Config 加载
    - Ent Client 初始化
    - Redis Client 初始化
    - Repository 实例化
    - Service 实例化
    - Handler 实例化
    - Middleware 实例化
    - Router 配置
    - HTTP Server 启动
```

## 4. 数据库实体关系

### 4.1 核心实体

#### 4.1.1 User (用户)

- **字段**: id, email, password_hash, role, balance, concurrency, status, username, notes
- **TOTP**: totp_secret_encrypted, totp_enabled, totp_enabled_at
- **关系**:
  - 1:N → APIKey (api_keys)
  - 1:N → UserSubscription (subscriptions)
  - 1:N → UsageLog (usage_logs)
  - M:N → Group (allowed_groups via user_allowed_groups)
  - 1:N → UserAttributeValue (attribute_values)

#### 4.1.2 Account (上游账号)

- **字段**: name, notes, platform, type, credentials (jsonb), extra (jsonb)
- **配置**: proxy_id, concurrency, priority, rate_multiplier
- **状态**: status, error_message, schedulable, rate_limited_at, overload_until, expires_at
- **关系**:
  - M:N → Group (groups via account_groups)
  - 1:N → UsageLog (usage_logs)

**Platform 支持**:

- `anthropic` (Claude)
- `gemini` (Gemini Code Assist / AI Studio)
- `openai` (OpenAI / Azure)
- `antigravity` (反重力平台)

**Type 认证类型**:

- `api_key`: API 密钥
- `oauth`: OAuth 2.0 (refresh_token 自动刷新)
- `cookie`: Session Cookie

#### 4.1.3 APIKey (用户 API 密钥)

- **字段**: user_id, key, name, group_id, status, last_used_at
- **配额**: quota (美元), quota_used, expires_at
- **IP 控制**: ip_whitelist, ip_blacklist (CIDR 支持)
- **速率限制**: rate*limit_5h, rate_limit_1d, rate_limit_7d; usage_5h, usage_1d, usage_7d; window*\*\_start
- **关系**:
  - N:1 → User
  - N:1 → Group
  - 1:N → UsageLog

#### 4.1.4 Group (分组)

- **字段**: name, description, rate_multiplier, is_exclusive, status
- **订阅**: platform, subscription_type, daily_limit_usd, weekly_limit_usd, monthly_limit_usd
- **图片计费**: image_price_1k, image_price_2k, image_price_4k
- **路由**: model_routing (jsonb: 模型模式 → 账号 ID 列表), model_routing_enabled
- **高级**: claude_code_only, fallback_group_id, mcp_xml_inject, supported_model_scopes
- **关系**:
  - M:N → Account (accounts via account_groups)
  - 1:N → APIKey
  - 1:N → UserSubscription

#### 4.1.5 UsageLog (使用日志)

- **关联**: user_id, api_key_id, account_id, group_id, subscription_id, request_id
- **Token**: input_tokens, output_tokens, cache_creation_tokens, cache_read_tokens, cache_creation_5m_tokens, cache_creation_1h_tokens
- **成本**: input_cost, output_cost, cache_creation_cost, cache_read_cost, total_cost, actual_cost, rate_multiplier, account_rate_multiplier
- **元数据**: model, billing_type, stream, duration_ms, first_token_ms, user_agent, ip_address
- **图片**: image_count, image_size
- **时间**: created_at (只追加，不可修改)

#### 4.1.6 UserSubscription (用户订阅)

- **字段**: user_id, group_id, assigner_id, status, expires_at
- **配额**: daily_quota_usd, weekly_quota_usd, monthly_quota_usd
- **关系**:
  - N:1 → User (user)
  - N:1 → User (assigner)
  - N:1 → Group

### 4.2 辅助实体

- **Proxy**: 代理配置 (支持 http/socks5)
- **RedeemCode**: 卡密兑换
- **PromoCode**: 优惠码
- **Announcement**: 公告系统
- **Setting**: 系统设置 (KV 存储)
- **UserAttributeDefinition/Value**: 用户自定义属性
- **ErrorPassthroughRule**: 错误透传规则
- **UsageCleanupTask**: 使用记录清理任务
- **DashboardAggregation**: 仪表盘预聚合表（预聚合服务，非 Ent 实体）
- **AccountGroup/UserAllowedGroup/AnnouncementRead/PromoCodeUsage**: 关联表
- **IdempotencyRecord**: 幂等请求记录 (scope, idempotency_key_hash, status, expires_at)
- **SecuritySecret**: 系统级安全密钥 (JWT 签名、TOTP 加密等)

### 4.3 软删除机制

核心实体 (User, Account, APIKey, Group 等) 使用软删除：

- `deleted_at` 字段标记删除时间
- 唯一约束通过部分索引实现 (`WHERE deleted_at IS NULL`)
- 支持删除后重用相同 email/key

## 5. 配置项说明

### 5.1 Server 配置

```yaml
server:
  host: "0.0.0.0" # 监听地址
  port: 8080 # 监听端口
  mode: "release" # debug/release
  max_request_body_size: 100MB # 全局请求体大小限制
  h2c:
    enabled: true # HTTP/2 Cleartext 支持
    max_concurrent_streams: 50 # 最大并发流
```

### 5.2 Gateway 配置

```yaml
gateway:
  response_header_timeout: 600 # 上游响应头超时 (秒)
  max_body_size: 100MB # 请求体大小限制
  connection_pool_isolation: "account_proxy" # 连接池隔离策略
  max_idle_conns: 240 # 最大空闲连接数
  concurrency_slot_ttl_minutes: 30 # 并发槽位过期时间
  stream_data_interval_timeout: 180 # 流数据间隔超时
  inject_beta_for_apikey: false # 自动注入 anthropic-beta 头
  failover_on_400: false # 允许 400 错误故障转移
```

### 5.3 Scheduling 配置 (调度缓存)

```yaml
gateway.scheduling:
  sticky_session_max_waiting: 3 # 粘性会话最大排队长度
  sticky_session_wait_timeout: 120s # 粘性会话等待超时
  fallback_wait_timeout: 30s # 兜底排队超时
  load_batch_enabled: true # 启用批量负载计算
  db_fallback_enabled: true # 允许受控回源到 DB
  outbox_poll_interval_seconds: 1 # Outbox 轮询周期
  full_rebuild_interval_seconds: 300 # 全量重建周期
```

### 5.4 缓存配置

```yaml
api_key_auth_cache:
  l1_size: 65535 # L1 缓存容量 (LRU)
  l1_ttl_seconds: 15 # L1 TTL
  l2_ttl_seconds: 300 # L2 TTL (Redis)
  singleflight: true # 合并回源

dashboard_cache:
  enabled: true
  stats_fresh_ttl_seconds: 15 # 新鲜阈值
  stats_ttl_seconds: 30 # Redis TTL
```

### 5.5 预聚合配置

```yaml
dashboard_aggregation:
  enabled: true
  interval_seconds: 60 # 刷新间隔
  lookback_seconds: 120 # 回看窗口 (处理迟到数据)
  recompute_days: 2 # 启动时重算天数
  retention:
    usage_logs_days: 90 # 原始日志保留
    hourly_days: 180 # 小时聚合保留
    daily_days: 730 # 日聚合保留
```

### 5.6 Gemini OAuth

```yaml
gemini.oauth:
  client_id: "681255809395-..." # 默认使用 Gemini CLI 公开凭证
  client_secret: "GOCSPX-..."
  scopes: "" # 留空自动选择 (Code Assist / AI Studio)

gemini.quota: # 本地配额模拟 (Code Assist)
  tiers:
    LEGACY: { pro_rpd: 50, flash_rpd: 1500, cooldown_minutes: 30 }
    PRO: { pro_rpd: 1500, flash_rpd: 4000, cooldown_minutes: 5 }
    ULTRA: { pro_rpd: 2000, flash_rpd: 0, cooldown_minutes: 5 }
```

### 5.7 Security 配置

```yaml
security:
  url_allowlist:
    enabled: false # URL 白名单验证
    allow_insecure_http: true # 允许 http (仅开发环境)
    allow_private_hosts: true # 允许私有 IP
  csp:
    enabled: true # CSP 响应头
    policy: "default-src 'self'; script-src 'self' __CSP_NONCE__ ..."
```

## 6. API 路由总览

### 6.1 网关路由 (API Key 认证)

| 路由                           | 方法 | 说明                                 |
| ------------------------------ | ---- | ------------------------------------ |
| `/v1/messages`                 | POST | Claude 消息接口 (兼容 Anthropic API) |
| `/v1/messages/count_tokens`    | POST | Token 计数                           |
| `/v1/models`                   | GET  | 模型列表                             |
| `/v1/usage`                    | GET  | 用量查询                             |
| `/v1/responses`                | POST | OpenAI Responses API (实时 API)      |
| `/v1beta/models`               | GET  | Gemini 模型列表                      |
| `/v1beta/models/:model`        | GET  | Gemini 模型详情                      |
| `/v1beta/models/*modelAction`  | POST | Gemini 生成接口                      |
| `/antigravity/v1/messages`     | POST | Antigravity 专用 (强制平台)          |
| `/antigravity/v1beta/models/*` | POST | Antigravity Gemini 接口              |
| `/sora/v1/chat/completions`    | POST | Sora 视频生成 (强制平台)             |
| `/sora/v1/models`              | GET  | Sora 模型列表                        |
| `/sora/media/*`                | GET  | Sora 媒体代理                        |
| `/sora/media-signed/*`         | GET  | Sora 签名媒体代理                    |

### 6.2 用户路由 (JWT 认证)

| 路由                             | 方法           | 说明         |
| -------------------------------- | -------------- | ------------ |
| `/api/v1/user/profile`           | GET            | 获取用户信息 |
| `/api/v1/user/password`          | PUT            | 修改密码     |
| `/api/v1/user`                   | PUT            | 更新个人资料 |
| `/api/v1/user/totp/status`       | GET            | TOTP 状态    |
| `/api/v1/user/totp/setup`        | POST           | 初始化 TOTP  |
| `/api/v1/user/totp/enable`       | POST           | 启用 TOTP    |
| `/api/v1/user/totp/disable`      | POST           | 禁用 TOTP    |
| `/api/v1/keys`                   | GET/POST       | API Key 管理 |
| `/api/v1/keys/:id`               | GET/PUT/DELETE | API Key 操作 |
| `/api/v1/groups/available`       | GET            | 可用分组列表 |
| `/api/v1/usage`                  | GET            | 使用记录列表 |
| `/api/v1/usage/stats`            | GET            | 统计数据     |
| `/api/v1/usage/dashboard/*`      | GET            | 用户仪表盘   |
| `/api/v1/subscriptions`          | GET            | 订阅列表     |
| `/api/v1/subscriptions/progress` | GET            | 订阅进度     |
| `/api/v1/redeem`                 | POST           | 卡密兑换     |
| `/api/v1/announcements`          | GET            | 公告列表     |

### 6.3 管理员路由 (Admin 认证)

| 路由组                                 | 说明                                          |
| -------------------------------------- | --------------------------------------------- |
| `/api/v1/admin/dashboard/*`            | 仪表盘统计 (实时/趋势/模型)                   |
| `/api/v1/admin/users/*`                | 用户管理 (CRUD/余额/并发)                     |
| `/api/v1/admin/groups/*`               | 分组管理 (CRUD/账号关联/路由配置)             |
| `/api/v1/admin/accounts/*`             | 账号管理 (CRUD/OAuth/测试/用量)               |
| `/api/v1/admin/proxies/*`              | 代理管理 (CRUD/探测/延迟测试)                 |
| `/api/v1/admin/redeem-codes/*`         | 卡密管理 (生成/导出/批量)                     |
| `/api/v1/admin/promo-codes/*`          | 优惠码管理                                    |
| `/api/v1/admin/announcements/*`        | 公告管理                                      |
| `/api/v1/admin/settings/*`             | 系统设置 (Turnstile/SMTP/LinuxDo)             |
| `/api/v1/admin/ops/*`                  | 运维监控 (错误日志/告警/实时监控/WebSocket)   |
| `/api/v1/admin/system/*`               | 系统管理 (版本检测/在线更新)                  |
| `/api/v1/admin/subscriptions/*`        | 订阅管理 (分配/撤销/批量)                     |
| `/api/v1/admin/usage/*`                | 使用记录管理 (清理任务)                       |
| `/api/v1/admin/oauth/*`                | OAuth 管理 (Claude/OpenAI/Gemini/Antigravity) |
| `/api/v1/admin/data-management/*`      | 数据管理 (备份/S3/源配置)                     |
| `/api/v1/admin/api-keys/*`             | API Key 管理 (分组更新)                       |
| `/api/v1/admin/scheduled-test-plans/*` | 定时测试计划                                  |
| `/api/v1/admin/sora/*`                 | Sora OAuth 集成                               |

### 6.4 认证路由 (公开)

| 路由                                  | 方法 | 说明                  |
| ------------------------------------- | ---- | --------------------- |
| `/api/v1/auth/register`               | POST | 用户注册              |
| `/api/v1/auth/login`                  | POST | 登录                  |
| `/api/v1/auth/refresh`                | POST | 刷新 Token            |
| `/api/v1/auth/logout`                 | POST | 登出                  |
| `/api/v1/auth/forgot-password`        | POST | 请求重置密码          |
| `/api/v1/auth/reset-password`         | POST | 重置密码              |
| `/api/v1/auth/oauth/linuxdo/*`        | GET  | LinuxDo Connect OAuth |
| `/api/v1/auth/me`                     | GET  | 获取当前用户          |
| `/api/v1/auth/login/2fa`              | POST | 2FA 登录验证          |
| `/api/v1/auth/revoke-all-sessions`    | POST | 撤销所有会话          |
| `/api/v1/auth/send-verify-code`       | POST | 发送验证码            |
| `/api/v1/auth/validate-promo-code`    | POST | 验证优惠码            |
| `/api/v1/auth/validate-invitation-code` | POST | 验证邀请码          |

## 7. 请求流转流程

### 7.1 网关请求流程

```
Client (API Key)
    ↓
Middleware: APIKeyAuth
    - 解析 Authorization Header
    - 两层缓存验证 (L1: 进程内 LRU, L2: Redis)
    - 加载 User/APIKey/Group/Subscription 信息
    - IP 白名单/黑名单校验
    - 配额校验 (quota/expires_at)
    ↓
Handler: GatewayHandler.Messages
    - 请求体解析 (Claude/OpenAI/Gemini 格式)
    - 平台适配检测 (claude_code_only, model_routing)
    - 并发槽位获取 (用户级 + 账号级)
    ↓
Service: GatewayService.ForwardMessages
    - 调度快照查询 (优先内存缓存，降级 DB)
    - 账号选择算法:
      1. 粘性会话 (conversation_id)
      2. 模型路由 (model_routing 配置)
      3. 负载均衡 (并发槽位 + 优先级 + 随机)
    - 账号可用性检查 (schedulable, rate_limited, overload, expires_at)
    ↓
Repository: HTTPUpstream
    - Token 获取 (OAuth 自动刷新)
    - HTTP 连接池 (按 account_proxy 隔离)
    - 代理配置 (HTTP/SOCKS5)
    - 流式请求转发 (SSE/HTTP2)
    ↓
Upstream AI Service (Claude/OpenAI/Gemini/Antigravity)
    ↓
Response Stream Processing
    - 实时流式转发
    - Token 计数 (input/output/cache)
    - 成本计算 (rate_multiplier * account_rate_multiplier)
    ↓
Service: BillingService.DeductBalance
    - 扣除用户余额
    - 更新 APIKey.quota_used
    - 写入 UsageLog (异步)
    ↓
Middleware: 释放并发槽位
    ↓
Client 收到完整响应
```

### 7.2 调度缓存机制

Sub2API 实现了高性能的调度缓存系统，避免高频查询数据库：

```
【Outbox 模式】
DB 写操作 (Account/Group CRUD)
    ↓
写入 scheduler_outbox 表 (事务内)
    ↓
后台轮询服务 (1s 间隔)
    - 读取增量变更
    - 触发缓存更新
    ↓
Redis: scheduler_snapshot
    - 账号列表 (按分组索引)
    - 并发槽位状态
    - 速率限制状态
    ↓
调度器查询 (O(1) Redis 查询)
```

**降级策略**:

- 缓存未命中 → 受控回源 DB (QPS 限流)
- Outbox 积压 → 触发全量重建
- 定期全量重建 (5 分钟)

### 7.3 多平台适配架构

Sub2API 支持多种 AI 平台，通过统一的 Gateway 接口对外暴露：

```
【平台抽象层】
Client Request (OpenAI/Claude/Gemini 格式)
    ↓
GatewayService (统一入口)
    ↓
分发到具体平台服务:
    - AnthropicGateway (Claude Code)
    - OpenAIGateway (OpenAI Responses API)
    - GeminiMessagesCompat (Gemini Messages 兼容层)
    - AntigravityGateway (反重力平台)
    ↓
TokenProvider (平台特定 Token 提供)
    - ClaudeTokenProvider (session_key)
    - OpenAITokenProvider (access_token)
    - GeminiTokenProvider (access_token + project_id)
    - AntigravityTokenProvider (access_token)
    ↓
HTTPUpstream (统一 HTTP 客户端)
    ↓
各平台 API 端点:
    - https://api.anthropic.com
    - https://api.openai.com
    - https://generativelanguage.googleapis.com
    - https://cloudcode-pa.googleapis.com
    - (Antigravity 平台地址)
```

**平台特性**:

| 平台               | 认证类型                  | Token 刷新           | 特殊功能               |
| ------------------ | ------------------------- | -------------------- | ---------------------- |
| Claude             | session_key               | 自动 Refresh (OAuth) | 粘性会话, Digest Auth  |
| OpenAI             | access_token              | 自动 Refresh (OAuth) | Responses API (实时)   |
| Gemini Code Assist | access_token + project_id | 自动 Refresh         | RPD 配额模拟           |
| Gemini AI Studio   | access_token              | 自动 Refresh         | generativelanguage API |
| Antigravity        | access_token              | 自动 Refresh         | MCP XML 协议注入       |

### 7.4 计费流程

```
【Token 计数】
Upstream Response (流式/非流式)
    ↓
BillingService.CalculateCost
    - 解析 usage 字段 (input/output/cache tokens)
    - 查询 model_prices (LiteLLM 定价数据)
    - 计算分项成本:
      * input_cost = (input_tokens / 1M) * input_price_per_1m
      * output_cost = (output_tokens / 1M) * output_price_per_1m
      * cache_creation_cost = ...
      * cache_read_cost = ...
    - 应用倍率:
      * total_cost = sum(分项成本)
      * actual_cost = total_cost * group.rate_multiplier
      * account_cost = total_cost * account.rate_multiplier (仅统计)
    ↓
【余额扣除】
BillingService.DeductBalance
    - User.balance -= actual_cost
    - APIKey.quota_used += actual_cost
    - 写入 UsageLog:
      {
        user_id, api_key_id, account_id, group_id,
        input_tokens, output_tokens, cache_*_tokens,
        input_cost, output_cost, cache_*_cost,
        total_cost, actual_cost,
        rate_multiplier (分组), account_rate_multiplier (账号)
      }
    ↓
【缓存失效】
BillingCacheService.InvalidateUserCache
    - 清空 L1/L2 认证缓存 (触发下次请求重新加载余额)
```

## 8. 关键设计模式

### 8.1 两层缓存 (L1 + L2)

- **L1**: 进程内 LRU/TTL (15s)
- **L2**: Redis (5min)
- **用途**: API Key 认证、仪表盘统计
- **优势**: 减少 DB 查询 90%+

### 8.2 Outbox 模式

- **场景**: 调度缓存同步
- **实现**: DB 写入同时写 outbox 表，后台轮询增量更新 Redis
- **优势**: 事务一致性 + 高性能查询

### 8.3 Singleflight 合并回源

- **场景**: 缓存击穿保护
- **实现**: 同一 key 并发请求合并为单次 DB 查询
- **用途**: API Key 认证、仪表盘缓存

### 8.4 Circuit Breaker 熔断器

- **场景**: 计费服务异常保护
- **实现**: 连续失败 5 次 → 熔断 30s
- **优势**: 防止雪崩，保障网关可用

### 8.5 软删除 + 部分唯一索引

- **实现**: `WHERE deleted_at IS NULL` 唯一约束
- **优势**: 支持删除后重用 email/key

### 8.6 Repository 抽象层

- **职责**: 封装 Ent/Redis/HTTP 操作
- **优势**: 业务层无感知底层存储切换

### 8.7 依赖注入 (Wire)

- **方式**: 编译时生成依赖图
- **优势**: 类型安全、无运行时反射开销

## 9. 并发控制

### 9.1 用户级并发

- **限制**: User.concurrency (默认 5)
- **实现**: Redis 计数器 + TTL
- **粒度**: 每用户独立限制

### 9.2 账号级并发

- **限制**: Account.concurrency (默认 3)
- **实现**: Redis 集合 (存储活跃 request_id)
- **粒度**: 每账号独立限制

### 9.3 并发槽位清理

- **周期**: 30s
- **策略**: 清理过期槽位 (TTL: 30min)

## 10. 速率限制

### 10.1 429 速率限制

- **触发**: 上游返回 429
- **动作**: 设置 `rate_limited_at`
- **恢复**: 不自动恢复 (手动管理)

### 10.2 529 过载保护

- **触发**: 上游返回 529
- **动作**: 设置 `overload_cooldown_until` (默认 10 分钟)
- **恢复**: 自动恢复 (时间窗口过后)

### 10.3 Gemini RPD 配额

- **实现**: 本地模拟 (pro_rpd, flash_rpd)
- **策略**: 按 tier 限制每日请求数
- **冷却**: 触发后冷却 5-30 分钟

## 11. 运维监控

### 11.1 实时监控

- 并发统计 (用户/账号)
- 账号可用性
- 实时流量 (QPS/TPS WebSocket)

### 11.2 告警系统

- 规则配置 (阈值/条件)
- 告警事件 (触发/恢复/静默)
- 邮件通知 (SMTP)

### 11.3 错误追踪

- Request Errors (客户端可见)
- Upstream Errors (上游失败)
- 重试机制 (手动/自动)

### 11.4 数据清理

- 使用日志清理 (90 天)
- 聚合数据保留 (小时 180 天，日 730 天)
- 后台任务调度 (定时清理)

## 12. 部署架构

### 12.1 Docker Compose (推荐)

```
sub2api (Go 后端)
    ↓ 连接
postgresql (数据持久化)
redis (缓存/会话)
    ↓ 访问
用户浏览器/API 客户端
```

### 12.2 Systemd 服务 (生产)

```
/opt/sub2api/sub2api-server (二进制)
    ↓ 读取
/etc/sub2api/config.yaml (配置)
    ↓ 连接
外部 PostgreSQL + Redis
    ↓
systemctl start sub2api (自动重启/日志)
```

### 12.3 在线更新

- 检测新版本 (GitHub Releases API)
- 下载二进制 (支持代理)
- 原地替换 + 重启服务
- 支持回滚

## 13. 性能优化

### 13.1 缓存策略

- **L1 缓存**: 进程内 LRU (极速)
- **L2 缓存**: Redis (跨实例共享)
- **调度缓存**: 内存快照 (避免高频 DB 查询)
- **Singleflight**: 防缓存击穿

### 13.2 连接池优化

- **隔离策略**: account_proxy (最细粒度)
- **连接复用**: HTTP/2 多路复用
- **池大小**: 240 连接 (支持高并发)

### 13.3 数据库优化

- **索引**: 覆盖高频查询字段
- **Ent 预加载**: 减少 N+1 查询
- **批量操作**: 聚合/清理任务批量执行
- **只读副本**: 可扩展读写分离

### 13.4 流式处理

- **SSE**: 实时转发流式响应
- **Keepalive**: 定时发送心跳 (10s)
- **超时控制**: 数据间隔超时 (180s)

## 14. 安全机制

### 14.1 认证鉴权

- **JWT**: 用户登录认证 (24h 过期)
- **API Key**: SHA256 哈希存储
- **Admin**: 角色校验 (role=admin)
- **TOTP**: 双因素认证 (AES 加密)

### 14.2 访问控制

- **IP 白名单/黑名单**: CIDR 支持
- **CORS**: 可配置跨域策略
- **CSP**: 内容安全策略 (防 XSS)
- **Trusted Proxies**: 限制可信代理

### 14.3 数据安全

- **密码**: Bcrypt 哈希
- **TOTP Secret**: AES-256-GCM 加密
- **OAuth Token**: 加密存储 (credentials jsonb)
- **软删除**: 保留审计记录

### 14.4 速率保护

- **并发限制**: 用户/账号双层限制
- **配额管理**: 美元级精确控制
- **过载保护**: 529 自动冷却

## 15. 扩展性

### 15.1 水平扩展

- **无状态设计**: 所有状态存 Redis/PostgreSQL
- **多实例部署**: 负载均衡器前置
- **Redis 集群**: 支持 Sentinel/Cluster
- **PostgreSQL 读写分离**: 支持主从复制

### 15.2 垂直扩展

- **连接池调优**: 根据并发需求调整
- **缓存容量**: L1/L2 容量可配置
- **调度缓存**: 内存快照可扩展

### 15.3 功能扩展

- **新平台接入**: 实现 TokenProvider + GatewayService
- **新模型支持**: 更新 model_prices 数据
- **新计费规则**: 扩展 BillingService

---

**文档版本**: 1.1
**生成时间**: 2026-03-06
**项目仓库**: <https://github.com/Wei-Shaw/sub2api>
