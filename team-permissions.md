# Maxun 项目与团队权限模型分析

## 1. 总体架构概览

Maxun 的权限模型是一个 **单用户隔离模型**（Single-User Isolation Model），当前版本不存在团队/组织的概念。所有资源（Robot、Run、API Key、Proxy 配置等）都以 `userId` 作为唯一归属标识，通过 **JWT Cookie 认证** 或 **API Key 认证** 两条路径进入系统，在数据查询层强制执行 `userId` 过滤，实现用户间数据隔离。

权限执行的三个层次：

| 层次 | 机制 | 代码位置 |
|------|------|----------|
| 身份认证 | `requireSignIn` / `requireAPIKey` 中间件 | [auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/middlewares/auth.ts), [api.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/middlewares/api.ts) |
| 访问判断 | 每个路由内部 `userId` 条件检查 | 各 `routes/*.ts` 文件 |
| 数据范围限制 | Sequelize 查询中 `where: { userId }` 过滤 | 各路由的数据库查询语句 |

---

## 2. 成员身份模型（Membership）

### 2.1 数据模型

系统只有 `User` 一个身份实体，没有 Team/Organization/Role 模型。

**User 模型** — [User.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/models/User.ts#L18-L28)

```typescript
class User {
  id: number;              // 自增主键
  email: string;           // 唯一，用于登录
  password: string;        // bcrypt 哈希
  api_key_name?: string;   // API Key 名称
  api_key?: string;        // API Key（用于 SDK/API 认证）
  api_key_created_at?: Date;
  proxy_url?: string;      // 代理 URL（AES-256-CBC 加密存储）
  proxy_username?: string; // 代理用户名（加密存储）
  proxy_password?: string; // 代理密码（加密存储）
}
```

**Robot 模型** — [Robot.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/models/Robot.ts#L77-L95)

```typescript
class Robot {
  id: string;                  // UUID
  userId: number;              // 外键 → User.id，核心归属字段
  recording_meta: RobotMeta;   // JSONB：元信息（name, type, url, formats 等）
  recording: RobotWorkflow;    // JSONB：工作流定义
  google_sheet_*: string;      // Google Sheets 集成凭据
  airtable_*: string;          // Airtable 集成凭据
  schedule?: ScheduleConfig;   // JSONB：调度配置
  webhooks?: WebhookConfig[];  // JSONB：Webhook 配置
}
```

**Run 模型** — [Run.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/models/Run.ts#L39-L60)

```typescript
class Run {
  id: string;                    // UUID
  robotId: string;               // 外键 → Robot.id
  robotMetaId: string;           // 冗余存储的 Robot meta.id
  runByUserId?: string;          // 触发者
  runByScheduleId?: string;
  runByAPI?: boolean;
  runBySDK?: boolean;
  runByMCP?: boolean;
  runByCLI?: boolean;
  status: string;                // running / queued / success / failed / aborted
  // ... 其他运行时字段
}
```

**模型关联** — [associations.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/models/associations.ts)

```typescript
Run.belongsTo(Robot, { foreignKey: 'robotId' });
Robot.hasMany(Run, { foreignKey: 'robotId' });
```

> **关键发现**：User 与 Robot 之间没有 Sequelize 外键关联定义，仅在 Robot 表的 `userId` 字段上做逻辑过滤。User ↔ Robot 的归属完全依赖应用层查询约束。

### 2.2 成员身份关系图

```
User (1) ──owns──▶ Robot (N)
                    │
                    └──has──▶ Run (N)

User (1) ──authenticates──▶ API Key (1)     // api_key 字段直接存在 User 表
User (1) ──configures──▶ Proxy Config (1)    // proxy_* 字段直接存在 User 表
```

---

## 3. 访问判断（Access Decision）

### 3.1 认证中间件

系统有两条认证路径，互斥使用：

#### 路径 A：Web UI 认证（JWT Cookie）

**`requireSignIn`** — [auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/middlewares/auth.ts#L8-L31)

```
请求 → Cookie 中取 token → JWT verify → req.user = { id } → next()
                                           ↓ 失败
                                      401 / 403
```

- Token 来源：`req.cookies.token`
- 签发时机：注册/登录时 `jwt.sign({ id: user.id }, JWT_SECRET)`
- 分发方式：`res.cookie("token", token, { httpOnly: true })`
- 有效期：无过期时间（由浏览器 Cookie 生命周期控制）
- 载荷：`{ id: number }` — 仅有用户 ID，无角色/权限信息

#### 路径 B：API / SDK 认证（API Key Header）

**`requireAPIKey`** — [api.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/middlewares/api.ts#L5-L23)

```
请求 → Header 取 x-api-key → User.findOne({ where: { api_key } }) → req.user = user → next()
                                  ↓ 未找到
                              403 Invalid API key
```

- Key 来源：`req.headers['x-api-key']`
- 验证方式：直接在数据库中查找匹配 `api_key` 的 User
- 注意：`req.user` 被赋值为完整的 User 实例（含 password 等敏感字段），而非仅 `{ id }`

### 3.2 路由级访问控制

所有路由在挂载时统一应用 `requireSignIn`，部分路由在每个 handler 内部再增加二次检查：

| 路由文件 | 中间件 | 二次检查模式 |
|----------|--------|-------------|
| [record.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/routes/record.ts) | `router.all('/', requireSignIn, ...)` | `if (!req.user) return 401` |
| [workflow.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/routes/workflow.ts) | `router.all('/', requireSignIn, ...)` | `if (!req.user) return 401` |
| [storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/routes/storage.ts) | `router.all('/', requireSignIn, ...)` | `if (!req.user) return 401` |
| [auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/routes/auth.ts) | 逐路由 `requireSignIn` | N/A |
| [proxy.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/routes/proxy.ts) | 逐路由 `requireSignIn` | `if (!authenticatedReq.user)` |
| [webhook.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/routes/webhook.ts) | 逐路由 `requireSignIn` | `if (!authenticatedReq.user)` |
| [sdk.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/api/sdk.ts) | 逐路由 `requireAPIKey` | `if (!req.user)` |
| [record.ts (api)](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/api/record.ts) | 逐路由 `requireAPIKey` | `if (!authenticatedReq.user)` |

**前端路由守卫** — [userRoute.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/src/routes/userRoute.tsx)

```typescript
return state.user ? <Outlet /> : <Navigate to="/login" />;
```

前端仅检查 `AuthContext` 中是否有 `user` 对象，无角色判断。

### 3.3 WebSocket 连接的认证

**Recording Socket** — [connection.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/socket-connection/connection.ts)

- `createSocketConnection` 接受 `userId` 参数，注册 Input Handler 时绑定该 userId
- 无 Token 验证：Socket 连接时不校验客户端身份，仅依赖 HTTP 层已有的认证

**Queued-Run Namespace** — [server.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/server.ts#L179-L201)

```typescript
io.of('/queued-run').on('connection', (socket) => {
    const userId = socket.handshake.query.userId;
    if (userId) {
        socket.join(`user-${userId}`);
    } else {
        socket.disconnect(); // 无 userId 则断开
    }
});
```

- 客户端在 `handshake.query.userId` 中传递 userId
- 服务端按 userId 加入 room `user-${userId}`，实现按用户推送
- **安全风险**：未验证客户端传来的 userId 是否真实对应已认证用户，恶意客户端可加入任意用户的 room

---

## 4. 数据范围限制（Data Scope Limitation）

### 4.1 核心过滤模式

所有数据访问都遵循 **"先认证，再按 userId 过滤"** 的模式：

#### 模式一：直接过滤（最常见）

```typescript
// 列出当前用户的所有 Robot
Robot.findAll({ where: { userId: req.user.id } })

// 获取单个 Robot（双重条件：meta.id + userId）
Robot.findOne({ where: { 'recording_meta.id': robotId, userId: req.user.id } })
```

#### 模式二：间接过滤（Run 通过 Robot 归属校验）

```typescript
// 列出当前用户的所有 Run
const userRobotIds = (await Robot.findAll({
    where: { userId: req.user.id }, attributes: ['id'], raw: true
})).map(r => r.id);
Run.findAll({ where: { robotId: { [Op.in]: userRobotIds } } });

// 获取单个 Run（先找 Run，再验证其 Robot 归属）
const run = await Run.findOne({ where: { runId: req.params.id } });
const robot = await Robot.findOne({
    where: { 'recording_meta.id': run.robotMetaId, userId: req.user.id }
});
if (!robot) return 404;
```

#### 模式三：资源属主校验（更新/删除操作）

```typescript
// 更新 Robot
await Robot.update(updates, {
    where: { 'recording_meta.id': id, userId: req.user.id }
});

// 删除 Robot
await Robot.destroy({
    where: { 'recording_meta.id': req.params.id, userId: req.user.id }
});
```

### 4.2 各资源类型的数据范围

| 资源 | 查询过滤方式 | 写入归属 | 代码示例 |
|------|-------------|---------|----------|
| **Robot** | `where: { userId }` | 创建时 `userId: req.user.id` | [storage.ts#L181](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/routes/storage.ts#L181) |
| **Run** | 通过 Robot 的 userId 间接过滤 | 创建时 `runByUserId: req.user.id` | [storage.ts#L941-L944](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/routes/storage.ts#L941) |
| **API Key** | `User.findByPk(req.user.id)` | 更新当前 User 的 api_key 字段 | [auth.ts#L246](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/routes/auth.ts#L246) |
| **Proxy** | `User.findByPk(req.user.id)` | 更新当前 User 的 proxy_* 字段 | [proxy.ts#L23](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/routes/proxy.ts#L23) |
| **Webhook** | `Robot.findOne({ where: { ..., userId } })` | 更新归属 Robot 的 webhooks 字段 | [webhook.ts#L74](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/routes/webhook.ts#L74) |
| **Google Sheets** | `Robot.findOne({ where: { ..., userId } })` | 更新归属 Robot 的 google_* 字段 | [auth.ts#L425-L426](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/routes/auth.ts#L425) |
| **Airtable** | `Robot.findOne({ where: { ..., userId } })` | 更新归属 Robot 的 airtable_* 字段 | [auth.ts#L749-L750](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/routes/auth.ts#L749) |
| **Schedule** | `Robot.findOne({ where: { ..., userId } })` | 更新归属 Robot 的 schedule 字段 | [storage.ts#L1234](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/routes/storage.ts#L1234) |
| **Browser** | `browserPool.getActiveBrowserId(userId, state)` | 创建时绑定 userId | [BrowserPool.ts#L93](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/browser-management/classes/BrowserPool.ts#L93) |

### 4.3 浏览器资源池的隔离

**BrowserPool** — [BrowserPool.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/browser-management/classes/BrowserPool.ts)

BrowserPool 实现了 **"1 User - 2 Browser"** 策略：

```typescript
private pool: PoolDictionary = {};                    // browserId → BrowserPoolInfo
private userToBrowserMap: Map<string, string[]> = {}; // userId → [browserId, ...]
```

- 每个用户最多 2 个浏览器实例（1 recording + 1 run）
- `addRemoteBrowser` 检查用户已有浏览器数
- `getActiveBrowserId` 按 userId + state 过滤
- `getUserForBrowser` 可反向查询 browserId → userId
- `reserveBrowserSlotAtomic` 原子化预留槽位防竞态

### 4.4 Robot 名称唯一性约束

Robot 名称在 **同一用户范围内** 唯一：

```typescript
// storage.ts - isRobotNameTaken
const robots = await Robot.findAll({
    where: {
        userId,
        [Op.and]: sequelizeInstance.where(
            sequelizeInstance.fn('trim', sequelizeInstance.literal("recording_meta->>'name'")),
            trimmed
        ),
    },
});
```

数据库层有唯一索引约束（迁移 [20250612000000-add-robot-name-unique-index.js](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/db/migrations/20250612000000-add-robot-name-unique-index.js)），但应用层已做 userId 限定。

---

## 5. 权限协作流程图

### 5.1 Web UI 请求的完整权限链路

```
┌──────────┐     ┌────────────────┐     ┌──────────────────┐     ┌───────────────┐
│  Browser  │────▶│  Cookie(token) │────▶│  requireSignIn   │────▶│  Route Handler│
│  Request  │     │  JWT verify    │     │  req.user = {id} │     │  userId 过滤   │
└──────────┘     └────────────────┘     └──────────────────┘     └───────────────┘
                         ↓ 失败                                    ↓
                    401 / 403                              Robot/Run/...
                                                           where: { userId }
```

### 5.2 API / SDK 请求的完整权限链路

```
┌──────────┐     ┌────────────────┐     ┌──────────────────┐     ┌───────────────┐
│  SDK/API  │────▶│  Header        │────▶│  requireAPIKey   │────▶│  Route Handler│
│  Request  │     │  x-api-key     │     │  req.user = User │     │  userId 过滤   │
└──────────┘     └────────────────┘     └──────────────────┘     └───────────────┘
                         ↓ 失败                                    ↓
                    401 / 403                              Robot/Run/...
                                                           where: { userId }
```

### 5.3 WebSocket 连接的权限链路

```
┌──────────┐     ┌────────────────────────┐     ┌─────────────────┐
│  Socket   │────▶│  handshake.query.userId │────▶│  socket.join(   │
│  Client   │     │  （未认证！）            │     │    user-${id}   │
└──────────┘     └────────────────────────┘     └─────────────────┘
                         ↓ 无 userId
                    socket.disconnect()
```

---

## 6. 特殊场景与安全分析

### 6.1 已识别的安全风险

| # | 风险 | 详情 | 严重度 |
|---|------|------|--------|
| 1 | **WebSocket userId 伪造** | `/queued-run` namespace 不验证客户端传来的 userId 是否为已认证用户，恶意客户端可加入其他用户的 room 接收通知 | 中 |
| 2 | **API Key 认证泄露密码** | `requireAPIKey` 将完整 User 实例（含 password 哈希）赋值给 `req.user`，若后续代码意外返回 `req.user` 则泄露密码 | 中 |
| 3 | **Token 无过期** | JWT 签发时未设置 `expiresIn`，Token 永久有效（前端有 4h 自动登出，但仅限前端） | 低 |
| 4 | **runByUserId 类型不一致** | Run 的 `runByUserId` 是 `INTEGER`，但 API 路由中以 `string` 传入 (`user.id.toString()`)，可能存在类型隐式转换 | 低 |
| 5 | **二次认证检查不一致** | 部分路由在 `requireSignIn` 之后还做 `if (!req.user)` 检查，部分不做，不统一 | 低 |

### 6.2 缺失的权限能力

| 能力 | 现状 | 影响 |
|------|------|------|
| 团队/组织 | ❌ 不存在 | 无法多人协作 |
| 角色（admin/viewer/editor） | ❌ 不存在 | 无法区分权限等级 |
| 资源共享 | ❌ 不存在 | Robot 无法被其他用户查看/运行 |
| 审计日志 | ❌ 仅 telemetry 上报 | 无法追踪谁在何时做了什么 |
| 细粒度权限 | ❌ 全有或全无 | 用户对自己的所有资源有完全控制权 |

### 6.3 自动登出机制

前端实现了基于活动检测的自动登出：

**[auth.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/src/context/auth.tsx#L70-L79)**

- 超时时间：4 小时（`AUTO_LOGOUT_TIME = 4 * 60 * 60 * 1000`）
- 检测事件：mousedown / keydown / scroll / touchstart
- 检测频率：每 60 秒检查一次
- 触发行为：调用 `/auth/logout` → 清除 Cookie → 跳转 `/login`

---

## 7. 敏感数据保护

### 7.1 密码存储

- 使用 bcrypt 哈希，salt rounds = 12 — [auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/utils/auth.ts#L5-L19)
- 注册/登录时分别调用 `hashPassword` / `comparePassword`

### 7.2 代理凭据加密

- 使用 AES-256-CBC 加密存储 — [auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/116-maxun/server/src/utils/auth.ts#L26-L43)
- IV 随机生成，与密文拼接存储（格式：`iv:encrypted`）
- 读取时解密，展示时脱敏（仅显示前3后3字符）

### 7.3 OAuth Token 存储

- Google/Airtable 的 access_token / refresh_token 直接以明文存储在 Robot 表中
- Airtable token 使用 TEXT 类型（因 token 较长），Google token 使用 STRING 类型

---

## 8. 总结

Maxun 当前的权限模型是一个简洁的 **用户级隔离模型**：

1. **身份认证**：双通道（JWT Cookie + API Key），无角色概念
2. **访问判断**：中间件层统一拦截，路由层二次确认
3. **数据隔离**：所有查询强制 `where: { userId }` 过滤，浏览器池按 userId 隔离

该模型适合单人使用场景，但不支持多人协作。如需引入团队/组织概念，需要新增 Team 模型、Member 关联表、角色枚举，并在所有查询中从 `where: { userId }` 扩展为 `where: { teamId }` + 角色判断。
