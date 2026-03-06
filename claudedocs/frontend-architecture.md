# Sub2API 前端架构文档

## 1. 技术栈

- **框架**: Vue 3 + TypeScript + Vite
- **状态管理**: Pinia
- **路由**: Vue Router
- **HTTP 客户端**: Axios
- **UI**: TailwindCSS + 自定义组件
- **国际化**: Vue I18n

---

## 2. 目录结构

```
frontend/src/
├── api/                # API 接口层
│   ├── admin/          # 管理员 API
│   ├── client.ts       # HTTP 客户端配置
│   ├── auth.ts         # 认证 API
│   └── ...             # 其他用户 API
├── stores/             # Pinia 状态管理
├── views/              # 页面视图
│   ├── admin/          # 管理员页面
│   ├── auth/           # 认证页面
│   └── user/           # 用户页面
├── components/         # Vue 组件
│   ├── admin/          # 管理组件
│   ├── auth/           # 认证组件
│   ├── common/         # 通用组件
│   ├── layout/         # 布局组件
│   └── user/           # 用户组件
├── composables/        # 组合式函数
├── router/             # 路由配置
├── types/              # TypeScript 类型定义
├── utils/              # 工具函数
└── i18n/               # 国际化配置
```

---

## 3. 核心模块

### 3.1 API 层 (`src/api/`)

#### 3.1.1 HTTP 客户端 (`client.ts`)

- **基础配置**: Axios 实例，baseURL: `/api/v1`，超时 30s
- **请求拦截器**:
  - 自动附加 localStorage 中的 `auth_token` (Bearer Token)
  - 附加用户语言 (`Accept-Language`)
  - GET 请求自动附加时区参数
- **响应拦截器**:
  - 自动解包 API 响应格式 `{ code, message, data }`
  - Token 刷新机制：401 错误自动刷新 token，队列化待刷新请求
  - 统一错误处理

#### 3.1.2 认证 API (`auth.ts`)

核心函数：

- `login(req)` - 登录，支持 2FA
- `register(req)` - 注册
- `logout()` - 登出
- `getCurrentUser()` - 获取当前用户信息
- `refreshToken()` - 刷新访问令牌
- `getPublicSettings()` - 获取公开配置
- `sendVerifyCode(req)` - 发送验证码
- Token 管理工具：`setAuthToken()`, `getAuthToken()`, `clearAuthToken()`

#### 3.1.3 用户 API

- **keys.ts**: API 密钥 CRUD (`list`, `create`, `update`, `delete`, `getById`)
- **usage.ts**: 用量记录查询 (`list`, `export`)
- **groups.ts**: 分组管理 (`list`, `getById`)
- **subscriptions.ts**: 订阅管理 (`list`, `create`, `update`, `delete`, `testConnection`)
- **user.ts**: 用户信息 (`updateProfile`, `changePassword`, `updateRunMode`)
- **redeem.ts**: 兑换码 (`redeem`)
- **totp.ts**: 双因素认证 (`setupTotp`, `verifyTotp`, `disableTotp`)

#### 3.1.4 管理员 API (`admin/`)

- **users.ts**: 用户管理 (`list`, `create`, `update`, `delete`, `getById`, `updateBalance`, `resetPassword`)
- **groups.ts**: 分组管理 (`list`, `create`, `update`, `delete`, `getById`, `updatePriority`)
- **proxies.ts**: 代理管理 (`list`, `create`, `update`, `delete`, `getById`, `testConnection`, `importData`)
- **usage.ts**: 用量统计 (`list`, `export`, `cleanup`, `stats`)
- **settings.ts**: 系统配置 (`getSettings`, `updateSettings`)
- **dashboard.ts**: 仪表盘统计 (`getDashboardStats`)
- **announcements.ts**: 公告管理 (`list`, `create`, `update`, `delete`, `getById`)
- **promo.ts**: 促销码管理 (`list`, `create`, `delete`, `stats`)
- **redeem.ts**: 兑换码管理 (`list`, `create`, `delete`, `stats`)
- **ops.ts**: 运维操作 (`ops/*`)
- **accounts.ts**: 账号管理 (OpenAI/Gemini/Antigravity OAuth)
- **system.ts**: 系统管理 (`checkUpdates`, `backupDatabase`, `restoreDatabase`)
- **antigravity.ts**: Antigravity 集成
- **gemini.ts**: Gemini 集成
- **errorPassthrough.ts**: 错误透传规则
- **subscriptions.ts**: 订阅管理 (管理员视角)
- **userAttributes.ts**: 用户属性管理
- **dataManagement.ts**: 数据管理 (备份/S3/源配置)
- **apiKeys.ts**: API Key 管理 (分组更新)
- **scheduledTests.ts**: 定时测试计划
- **sora.ts** (api/): Sora 用户端 API (生成/配额/模型，需认证)
- **setup.ts**: 初始化向导 API

---

### 3.2 状态管理 (`src/stores/`)

#### 3.2.1 认证 Store (`auth.ts`)

状态：

- `user: User | null` - 当前用户
- `token: string | null` - 访问令牌
- `refreshTokenValue: string | null` - 刷新令牌
- `tokenExpiresAt: number | null` - 令牌过期时间戳
- `runMode: 'standard' | 'simple'` - 运行模式

计算属性：

- `isAuthenticated` - 是否已认证
- `isAdmin` - 是否管理员
- `isSimpleMode` - 是否简单模式

核心方法：

- `checkAuth()` - 初始化认证状态（从 localStorage 恢复）
- `login(req)` - 登录
- `logout()` - 登出
- `refreshUser()` - 刷新用户信息
- `startAutoRefresh()` - 启动自动刷新（60s 间隔）
- `scheduleTokenRefreshAt(expiresAt)` - 定时刷新 token（过期前 120s）

#### 3.2.2 应用 Store (`app.ts`)

状态：

- `sidebarCollapsed: boolean` - 侧边栏折叠状态
- `mobileOpen: boolean` - 移动端侧边栏打开状态
- `loading: boolean` - 全局加载状态
- `toasts: Toast[]` - Toast 通知队列
- `publicSettings: PublicSettings | null` - 公开配置缓存
- 版本信息缓存 (`currentVersion`, `latestVersion`, `hasUpdate`)

核心方法：

- `toggleSidebar()` - 切换侧边栏
- `setLoading(isLoading)` - 设置加载状态
- `showToast(type, message, duration?)` - 显示 Toast
- `showSuccess/showError/showInfo/showWarning(message)` - 快捷通知方法
- `fetchPublicSettings()` - 获取并缓存公开配置
- `checkUpdates()` - 检查系统更新

#### 3.2.3 管理员设置 Store (`adminSettings.ts`)

管理员配置缓存和更新。

#### 3.2.4 订阅 Store (`subscriptions.ts`)

用户订阅数据管理。

#### 3.2.5 引导 Store (`onboarding.ts`)

用户引导状态管理。

---

### 3.3 类型定义 (`src/types/index.ts`)

共 1328 行，包含所有 TypeScript 接口定义。

#### 3.3.1 核心类型

- **用户认证**: `User`, `AdminUser`, `LoginRequest`, `RegisterRequest`, `AuthResponse`
- **API 密钥**: `ApiKey`, `CreateApiKeyRequest`, `UpdateApiKeyRequest`
- **订阅**: `Subscription`, `CreateSubscriptionRequest`, `UpdateSubscriptionRequest`
- **分组**: `Group`, `CreateGroupRequest`, `UpdateGroupRequest`
- **代理**: `Proxy`, `CreateProxyRequest`, `UpdateProxyRequest`
- **用量**: `Usage`, `UsageStats`, `ExportUsageRequest`
- **公告**: `Announcement`, `AnnouncementCondition`, `AnnouncementStatus`
- **通用**: `PaginatedResponse<T>`, `ApiResponse<T>`, `SelectOption`, `Toast`
- **配置**: `PublicSettings`, `AdminSettings`
- **促销码**: `PromoCode`, `RedeemCode`
- **用户属性**: `UserAttribute`, `UserAttributeValue`

---

### 3.4 路由配置 (`src/router/index.ts`)

#### 3.4.1 路由分组

1. **公开路由** (无需认证):
   - `/home` - 首页
   - `/login` - 登录
   - `/register` - 注册
   - `/email-verify` - 邮箱验证
   - `/forgot-password` - 忘记密码
   - `/reset-password` - 重置密码
   - `/auth/callback` - OAuth 回调
   - `/auth/linuxdo/callback` - LinuxDo OAuth 回调

2. **用户路由** (需要认证):
   - `/dashboard` - 用户仪表盘
   - `/keys` - API 密钥管理
   - `/usage` - 用量记录
   - `/subscriptions` - 订阅管理
   - `/redeem` - 兑换码
   - `/profile` - 个人资料
   - `/purchase` - 购买订阅

- `/sora` - Sora 视频生成
- `/custom/:id` - 自定义页面
- `/key-usage` - Key 用量（公开）

1. **管理员路由** (需要管理员权限):
   - `/admin/dashboard` - 管理仪表盘
   - `/admin/users` - 用户管理
   - `/admin/groups` - 分组管理
   - `/admin/proxies` - 代理管理
   - `/admin/usage` - 用量统计
   - `/admin/announcements` - 公告管理
   - `/admin/promo-codes` - 促销码管理
   - `/admin/redeem` - 兑换码管理
   - `/admin/settings` - 系统设置
   - `/admin/subscriptions` - 订阅管理
   - `/admin/accounts` - 账号管理
   - `/admin/ops` - 运维操作

- `/admin/data-management` - 数据管理

1. **特殊路由**:
   - `/setup` - 初始化向导
   - `/sora` - Sora 视频生成 (需认证)

#### 3.4.2 路由守卫

- 认证检查：未登录重定向到 `/login`
- 权限检查：非管理员访问管理页面重定向到 `/dashboard`
- 导航加载状态：自动显示加载条
- 路由预加载：优化页面切换性能

---

### 3.5 组合式函数 (`src/composables/`)

#### 3.5.1 核心 Composables

- **useForm.ts**: 表单提交逻辑封装，统一处理加载状态、错误捕获、Toast 通知
- **useTableLoader.ts**: 表格数据加载，处理分页、筛选、搜索防抖、请求取消
- **useClipboard.ts**: 剪贴板复制功能
- **useNavigationLoading.ts**: 路由导航加载状态管理
- **useRoutePrefetch.ts**: 路由预加载优化
- **useOnboardingTour.ts**: 用户引导功能
- **useModelWhitelist.ts**: 模型白名单管理

#### 3.5.2 OAuth Composables

- **useOpenAIOAuth.ts**: OpenAI OAuth 集成
- **useGeminiOAuth.ts**: Gemini OAuth 集成
- **useAntigravityOAuth.ts**: Antigravity OAuth 集成
- **useAccountOAuth.ts**: 通用账号 OAuth 逻辑

---

### 3.6 视图层 (`src/views/`)

#### 3.6.1 认证视图 (`auth/`)

- `LoginView.vue` - 登录页面
- `RegisterView.vue` - 注册页面
- `EmailVerifyView.vue` - 邮箱验证
- `ForgotPasswordView.vue` - 忘记密码
- `ResetPasswordView.vue` - 重置密码
- `OAuthCallbackView.vue` - OAuth 回调处理
- `LinuxDoCallbackView.vue` - LinuxDo OAuth 回调

#### 3.6.2 用户视图 (`user/`)

- `DashboardView.vue` - 用户仪表盘
- `KeysView.vue` - API 密钥管理
- `UsageView.vue` - 用量记录
- `SubscriptionsView.vue` - 订阅管理
- `RedeemView.vue` - 兑换码
- `ProfileView.vue` - 个人资料
- `PurchaseSubscriptionView.vue` - 购买订阅
- `SoraView.vue` - Sora 视频生成
- `CustomPageView.vue` - 自定义页面

#### 3.6.3 管理员视图 (`admin/`)

- `DashboardView.vue` - 管理仪表盘（系统统计、用户统计、用量统计）
- `UsersView.vue` - 用户管理（CRUD、余额管理、重置密码）
- `GroupsView.vue` - 分组管理（CRUD、优先级排序）
- `ProxiesView.vue` - 代理管理（CRUD、连接测试、数据导入）
- `UsageView.vue` - 用量统计（数据导出、清理）
- `AnnouncementsView.vue` - 公告管理（CRUD、目标用户条件设置）
- `PromoCodesView.vue` - 促销码管理
- `RedeemView.vue` - 兑换码管理
- `SettingsView.vue` - 系统设置（全局配置、邮件配置、Turnstile 配置等）
- `SubscriptionsView.vue` - 订阅管理（管理员视角）
- `AccountsView.vue` - 账号管理（OpenAI/Gemini OAuth）
- `ops/` - 运维操作相关页面
- `DataManagementView.vue` - 数据管理（备份/S3 配置）

#### 3.6.4 其他视图

- `HomeView.vue` - 首页
- `KeyUsageView.vue` - Key 用量（公开，views/ 根目录）
- `setup/SetupWizardView.vue` - 初始化向导

---

### 3.7 组件层 (`src/components/`)

#### 3.7.1 布局组件 (`layout/`)

- `AppLayout.vue` - 应用主布局（侧边栏 + 顶栏 + 内容区）
- `AuthLayout.vue` - 认证页面布局
- `AppHeader.vue` - 顶部导航栏
- `AppSidebar.vue` - 侧边栏菜单
- `TablePageLayout.vue` - 表格页面通用布局

#### 3.7.2 管理员组件 (`admin/`)

**用户管理**:

- `user/UserCreateModal.vue` - 创建用户弹窗
- `user/UserEditModal.vue` - 编辑用户弹窗
- `user/UserBalanceModal.vue` - 余额管理弹窗
- `user/UserBalanceHistoryModal.vue` - 余额历史弹窗
- `user/UserApiKeysModal.vue` - 用户 API 密钥弹窗
- `user/UserAllowedGroupsModal.vue` - 用户分组权限弹窗

**代理管理**:

- `proxy/ImportDataModal.vue` - 导入数据弹窗

**公告管理**:

- `announcements/AnnouncementTargetingEditor.vue` - 公告目标用户编辑器
- `announcements/AnnouncementReadStatusDialog.vue` - 公告阅读状态弹窗

**用量统计**:

- `usage/UsageStatsCards.vue` - 用量统计卡片
- `usage/UsageTable.vue` - 用量表格
- `usage/UsageFilters.vue` - 用量筛选器
- `usage/UsageExportProgress.vue` - 用量导出进度
- `usage/UsageCleanupDialog.vue` - 用量清理弹窗

**账号管理**:

- `account/AccountCreateModal.vue` - 创建账号弹窗
- `account/AccountEditModal.vue` - 编辑账号弹窗

**其他**:

- `ErrorPassthroughRulesModal.vue` - 错误透传规则配置

#### 3.7.3 用户组件 (`user/`)

- `dashboard/` - 仪表盘组件（统计卡片、快捷操作）
- `profile/` - 个人资料组件

#### 3.7.4 密钥组件 (`keys/`)

API 密钥相关组件。

#### 3.7.5 认证组件 (`auth/`)

- `TotpLoginModal.vue` - 双因素认证弹窗
- `LinuxDoOAuthSection.vue` - LinuxDo OAuth 登录区域

#### 3.7.6 通用组件 (`common/`)

通用 UI 组件（按钮、表单、对话框等）。

#### 3.7.7 图表组件 (`charts/`)

数据可视化图表。

#### 3.7.8 图标组件 (`icons/`)

SVG 图标组件。

#### 3.7.9 引导组件 (`Guide/`)

- `steps.ts` - 引导步骤配置

#### 3.7.10 其他

- `TurnstileWidget.vue` - Cloudflare Turnstile 验证组件

---

### 3.8 工具函数 (`src/utils/`)

#### 3.8.1 `format.ts`

格式化工具函数：

- `formatRelativeTime(date)` - 相对时间（5m ago, 2h ago）
- `formatNumber(num)` - 数字格式化（1.2K, 3.5M）
- `formatCurrency(amount, currency)` - 货币格式化（$1.25）
- `formatBytes(bytes)` - 字节格式化（1.2 MB）
- `formatPercent(value)` - 百分比格式化（12.5%）
- `formatDateTime(date)` - 日期时间格式化

#### 3.8.2 `url.ts`

URL 工具函数（推测）。

---

## 4. 核心流程

### 4.1 认证流程

1. 用户访问 `/login`
2. 输入凭据 → `authAPI.login()` → 后端验证
3. 后端返回 `access_token`, `refresh_token`, `expires_in`, `user`
4. 前端存储到 localStorage (`auth_token`, `refresh_token`, `token_expires_at`, `auth_user`)
5. `authStore.setAuth()` 更新状态
6. 启动自动刷新定时器（60s 刷新用户信息，过期前 120s 刷新 token）
7. 路由导航守卫检查认证状态，重定向到 `/dashboard`

### 4.2 Token 刷新流程

1. **被动刷新**（401 错误）：
   - Axios 响应拦截器检测到 401 错误
   - 调用 `authAPI.refreshToken()` 使用 `refresh_token` 获取新 token
   - 更新 localStorage 和状态
   - 重放原始请求
   - 队列化等待的请求（防止重复刷新）

2. **主动刷新**（定时）：
   - `authStore.scheduleTokenRefreshAt()` 在过期前 120s 触发
   - 调用 `authAPI.refreshToken()` 获取新 token
   - 更新 localStorage 和状态
   - 重新调度下次刷新

### 4.3 数据加载流程（以用户列表为例）

1. 页面组件使用 `useTableLoader` composable
2. 传入 `fetchFn: adminUsersAPI.list`，初始参数，页面大小
3. Composable 初始化分页状态、筛选参数、加载状态
4. 组件调用 `load()` 或 `reload()` 加载数据
5. Composable 调用 `fetchFn(page, pageSize, params, { signal })`
6. API 函数发送请求，Axios 自动附加 token 和语言头
7. 后端返回 `{ items, total, page, page_size, pages }`
8. Composable 更新 `items` 和分页状态
9. 组件渲染表格
10. 用户切换页码 → `handlePageChange()` → `load()`
11. 用户修改筛选 → 触发 `debouncedReload()` → 防抖后重新加载

### 4.4 表单提交流程（以创建 API 密钥为例）

1. 组件使用 `useForm` composable
2. 传入表单数据、提交函数、成功/错误消息
3. 用户填写表单 → 点击提交 → `submit()`
4. Composable 设置 `loading = true`
5. 调用 `submitFn(form)` → `keysAPI.create(...)`
6. API 函数发送 POST 请求
7. 成功 → `appStore.showSuccess(successMsg)` → 组件可选处理
8. 失败 → `appStore.showError(errorMsg)` → 抛出错误供组件处理
9. 最终 `loading = false`

---

## 5. 特色功能

### 5.1 自动 Token 刷新

- **被动刷新**：401 错误自动刷新
- **主动刷新**：过期前 120s 自动刷新
- **请求队列化**：刷新期间的请求等待后重放

### 5.2 国际化支持

- Vue I18n 集成
- 自动附加 `Accept-Language` 头
- 动态语言切换

### 5.3 时区感知

- GET 请求自动附加用户时区参数
- 后端可根据时区设置默认日期范围

### 5.4 表格优化

- 通用 `useTableLoader` 封装
- 自动分页、筛选、搜索防抖
- 请求取消（防止重复请求）
- 中止控制器（AbortController）

### 5.5 OAuth 集成

- OpenAI OAuth
- Gemini OAuth
- Antigravity OAuth
- LinuxDo OAuth

### 5.6 双因素认证 (2FA)

- TOTP 支持
- 登录时 2FA 验证
- 2FA 设置/禁用

### 5.7 用户引导

- 新手引导流程
- 步骤化教程

### 5.8 版本更新检测

- 自动检查系统更新
- 显示最新版本和更新日志

### 5.9 路由优化

- 懒加载（代码分割）
- 路由预加载（`useRoutePrefetch`）
- 导航加载状态（`useNavigationLoading`）

### 5.10 错误处理

- 全局错误拦截
- Toast 通知
- 统一错误消息展示

---

## 6. 数据流图

```
┌─────────────┐
│  View 组件  │
└──────┬──────┘
       │ 调用
       ▼
┌─────────────┐
│ Composable  │ (useForm, useTableLoader)
└──────┬──────┘
       │ 调用
       ▼
┌─────────────┐
│  API 函数   │ (authAPI.login, usersAPI.list)
└──────┬──────┘
       │ HTTP 请求
       ▼
┌─────────────┐
│ Axios Client│ (自动附加 token、语言、时区)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   后端 API  │
└──────┬──────┘
       │ 响应
       ▼
┌─────────────┐
│ 响应拦截器  │ (解包数据、token 刷新、错误处理)
└──────┬──────┘
       │ 返回数据
       ▼
┌─────────────┐
│  Pinia Store│ (auth, app)
└──────┬──────┘
       │ 更新状态
       ▼
┌─────────────┐
│  View 组件  │ (响应式更新)
└─────────────┘
```

---

## 7. 技术亮点

1. **完全类型化**：所有 API、Store、Composable 均有完整 TypeScript 类型定义
2. **组合式 API**：全面使用 Vue 3 Composition API，代码复用性高
3. **统一抽象**：通过 `useForm` 和 `useTableLoader` 统一常见业务逻辑
4. **性能优化**：路由懒加载、预加载、请求防抖、请求取消
5. **用户体验**：全局加载状态、Toast 通知、错误处理、国际化
6. **安全性**：Token 自动刷新、请求签名、TOTP 2FA

---

## 8. 最佳实践

### 8.1 添加新 API 端点

1. 在 `types/index.ts` 定义请求/响应类型
2. 在 `api/` 下创建 API 函数
3. 使用 `apiClient.get/post/put/delete`
4. 导出函数供组件使用

### 8.2 添加新页面

1. 在 `views/` 下创建 `.vue` 文件
2. 在 `router/index.ts` 添加路由配置
3. 设置 `meta.requiresAuth` 和 `meta.requiresAdmin`
4. 组件中使用 `useTableLoader` 或 `useForm` 处理数据

### 8.3 添加新 Store

1. 在 `stores/` 下创建文件
2. 使用 `defineStore` 定义 store
3. 使用组合式 API 风格（`ref`, `computed`, 函数）
4. 在组件中 `import { use<Name>Store } from '@/stores/<name>'`

### 8.4 添加新 Composable

1. 在 `composables/` 下创建 `.ts` 文件
2. 导出 `use<Name>` 函数
3. 返回响应式状态和方法
4. 在组件 `setup()` 中调用

---

## 9. 环境变量

- `VITE_API_BASE_URL`: API 基础 URL（默认 `/api/v1`）

---

## 10. 构建与部署

```bash
# 开发
npm run dev

# 构建
npm run build

# 预览
npm run preview

# 类型检查
npm run type-check

# 代码检查
npm run lint
```

---

## 11. 总结

Sub2API 前端采用现代化的 Vue 3 技术栈，具有清晰的分层架构：

- **API 层**：封装所有后端交互，自动处理认证和错误
- **状态管理**：Pinia 管理全局状态（认证、应用配置）
- **视图层**：功能明确的页面组件，用户/管理员分离
- **组件层**：高度复用的 UI 组件和业务组件
- **Composables**：统一的业务逻辑抽象（表单、表格、OAuth）
- **工具层**：格式化、URL 处理等通用工具

代码质量高，类型安全，性能优化充分，用户体验友好。适合中大型企业级应用。
