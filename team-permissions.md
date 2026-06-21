# Maxun 项目与团队权限模型分析

## 1. 总体架构概览

Maxun 的权限模型是一个 **单用户隔离模型**（Single-User Isolation Model），当前版本不存在团队/组织的概念。所有资源（Robot、Run、API Key、Proxy 配置等）都以 `userId` 作为唯一归属标识，但权限执行的强度在 **高度依赖请求路径而异**：

- **同步 HTTP 路径**（Web UI / API / SDK）** 经过完整的三层权限链：**JWT Cookie 认证** 或 **API Key 认证** → 路由 handler 中间件注入身份 → 数据查询层强制执行 `userId` 过滤 → 用户间数据隔离严格。
- **异步路径**（队列轮询 / graphile-worker / 调度 worker / 集成后处理 / Socket.IO）** 完全绕过中间件**，userId 作为 payload 或 closure 中透传，下游查询大多 **不做 userId 联合校验**，隔离强度弱。

权限执行的三个层次（仅适用于同步 HTTP 路径）：

| 层次 | 机制 | 代码位置 | 异步路径是否绕过 |
|------|------|----------|:---------:|
| 身份认证 | `requireSignIn` / `requireAPIKey` 中间件 | [auth.ts](server/src/middlewares/auth.ts), [api.ts](server/src/middlewares/api.ts) | ❌ 完全绕过 |
| 访问判断 | 每个路由内部 `userId` 条件检查 | 各 `routes/*.ts` 文件 | ❌ 异步函数大多无 |
| 数据范围限制 | Sequelize 查询中 `where: { userId }` 过滤 | 各路由的数据库查询语句 | ❌ 大多查询无联合条件 |

> ⚠️ **关键注意**：下文 §8 详细分析的 4 条同步→异步边界（B1–B4）显示，系统中大量后台进程（`processQueuedRuns`、`schedule-worker`、`sendWebhook`、`gsheet/airtable` 后处理等）在数据库层面做全表扫描，不区分用户。**userId 过滤不是全局规则，仅是同步 HTTP 路径的规则**。

---

## 2. 成员身份模型（Membership）

### 2.1 数据模型

系统只有 `User` 一个身份实体，没有 Team/Organization/Role 模型。

**User 模型** — [User.ts](server/src/models/User.ts#L18-L28)

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

**Robot 模型** — [Robot.ts](server/src/models/Robot.ts#L77-L95)

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

**Run 模型** — [Run.ts](server/src/models/Run.ts#L39-L60)

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

**模型关联** — [associations.ts](server/src/models/associations.ts)

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

**`requireSignIn`** — [auth.ts](server/src/middlewares/auth.ts#L8-L31)

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

**`requireAPIKey`** — [api.ts](server/src/middlewares/api.ts#L5-L23)

```
请求 → Header 取 x-api-key → User.findOne({ where: { api_key } }) → req.user = user → next()
                                  ↓ 未找到
                              403 Invalid API key
```

- Key 来源：`req.headers['x-api-key']`
- 验证方式：直接在数据库中查找匹配 `api_key` 的 User
- 注意：`req.user` 被赋值为完整的 User 实例（含 password 等敏感字段），而非仅 `{ id }`

### 3.2 路由级访问控制

> 本节描述 HTTP 路由层的访问控制。**异步路径（队列、调度、集成后处理等）不经过路由层，完全绕过本节所有机制**，详见 §8。

所有 HTTP 路由在挂载时统一应用 `requireSignIn` 或 `requireAPIKey`，部分路由在每个 handler 内部再增加二次检查：

| 路由文件 | 中间件 | 二次检查模式 |
|----------|--------|-------------|
| [record.ts](server/src/routes/record.ts) | `router.all('/', requireSignIn, ...)` | `if (!req.user) return 401` |
| [workflow.ts](server/src/routes/workflow.ts) | `router.all('/', requireSignIn, ...)` | `if (!req.user) return 401` |
| [storage.ts](server/src/routes/storage.ts) | `router.all('/', requireSignIn, ...)` | `if (!req.user) return 401` |
| [auth.ts](server/src/routes/auth.ts) | 逐路由 `requireSignIn` | N/A |
| [proxy.ts](server/src/routes/proxy.ts) | 逐路由 `requireSignIn` | `if (!authenticatedReq.user)` |
| [webhook.ts](server/src/routes/webhook.ts) | 逐路由 `requireSignIn` | `if (!authenticatedReq.user)` |
| [sdk.ts](server/src/api/sdk.ts) | 逐路由 `requireAPIKey` | `if (!req.user)` |
| [record.ts (api)](server/src/api/record.ts) | 逐路由 `requireAPIKey` | `if (!authenticatedReq.user)` |

**前端路由守卫** — [userRoute.tsx](src/routes/userRoute.tsx)

```typescript
return state.user ? <Outlet /> : <Navigate to="/login" />;
```

前端仅检查 `AuthContext` 中是否有 `user` 对象，无角色判断。

### 3.3 WebSocket 连接的认证

**Recording Socket** — [connection.ts](server/src/socket-connection/connection.ts)

- `createSocketConnection` 接受 `userId` 参数，注册 Input Handler 时绑定该 userId
- 无 Token 验证：Socket 连接时不校验客户端身份，仅依赖 HTTP 层已有的认证

**Queued-Run Namespace** — [server.ts](server/src/server.ts#L179-L201)

```typescript
io.of('/queued-run').on('connection', (socket) => {
    const userId = socket.handshake.query.userId as string;
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

> ⚠️ **范围声明**：本节描述的三种过滤模式和资源范围表，**仅适用于同步 HTTP 路径**（受 `requireSignIn` / `requireAPIKey` 保护的路由 handler）。异步路径（队列、调度、集成后处理、Webhook 发送、Socket 通道等）的数据范围限制与本节描述差异很大，详细分析见 §8–§11。

### 4.1 同步路径的核心过滤模式

在同步 HTTP 路径中，所有数据访问都遵循 **"先认证，再按 userId 过滤"** 的模式，有三种典型模式：

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

### 4.2 同步路径下各资源类型的数据范围

> 下表描述的是**同步 HTTP 路由 handler 中的数据访问模式**。异步路径下同一资源的访问方式差异显著（见 §8 联合过滤缺口矩阵）。

| 资源 | 查询过滤方式 | 写入归属 | 代码示例 |
|------|-------------|---------|----------|
| **Robot** | `where: { userId }` | 创建时 `userId: req.user.id` | [storage.ts#L181](server/src/routes/storage.ts#L181) |
| **Run** | 通过 Robot 的 userId 间接过滤 | 创建时 `runByUserId: req.user.id` | [storage.ts#L941-L944](server/src/routes/storage.ts#L941) |
| **API Key** | `User.findByPk(req.user.id)` | 更新当前 User 的 api_key 字段 | [auth.ts#L246](server/src/routes/auth.ts#L246) |
| **Proxy** | `User.findByPk(req.user.id)` | 更新当前 User 的 proxy_* 字段 | [proxy.ts#L23](server/src/routes/proxy.ts#L23) |
| **Webhook** | `Robot.findOne({ where: { ..., userId } })` | 更新归属 Robot 的 webhooks 字段 | [webhook.ts#L74](server/src/routes/webhook.ts#L74) |
| **Google Sheets** | `Robot.findOne({ where: { ..., userId } })` | 更新归属 Robot 的 google_* 字段 | [auth.ts#L425-L426](server/src/routes/auth.ts#L425) |
| **Airtable** | `Robot.findOne({ where: { ..., userId } })` | 更新归属 Robot 的 airtable_* 字段 | [auth.ts#L749-L750](server/src/routes/auth.ts#L749) |
| **Schedule** | `Robot.findOne({ where: { ..., userId } })` | 更新归属 Robot 的 schedule 字段 | [storage.ts#L1234](server/src/routes/storage.ts#L1234) |
| **Browser** | `browserPool.getActiveBrowserId(userId, state)` | 创建时绑定 userId | [BrowserPool.ts#L93](server/src/browser-management/classes/BrowserPool.ts#L93) |

### 4.3 浏览器资源池的隔离

**BrowserPool** — [BrowserPool.ts](server/src/browser-management/classes/BrowserPool.ts)

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

> ⚠️ **异步路径缺口**：`getRemoteBrowser(id)` 方法**仅按 browserId 查找，不校验 userId**。在异步任务路径中（如 `processRunExecution`），如果 browserId 来源不可信，可直接获取到任意用户的浏览器实例。详见 §11.6 和 N4 风险。

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

数据库层有唯一索引约束（迁移 [20250612000000-add-robot-name-unique-index.js](server/src/db/migrations/20250612000000-add-robot-name-unique-index.js)），但应用层已做 userId 限定。

---

## 5. 同步路径权限协作流程图

> 本节展示的三条链路均为**同步 HTTP / WebSocket 路径**。异步队列、调度 worker、集成后处理等路径的权限流转差异很大，详见 §8–§11。

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

**[auth.tsx](src/context/auth.tsx#L70-L79)**

- 超时时间：4 小时（`AUTO_LOGOUT_TIME = 4 * 60 * 60 * 1000`）
- 检测事件：mousedown / keydown / scroll / touchstart
- 检测频率：每 60 秒检查一次
- 触发行为：调用 `/auth/logout` → 清除 Cookie → 跳转 `/login`

---

## 7. 敏感数据保护

### 7.1 密码存储

- 使用 bcrypt 哈希，salt rounds = 12 — [auth.ts](server/src/utils/auth.ts#L5-L19)
- 注册/登录时分别调用 `hashPassword` / `comparePassword`

### 7.2 代理凭据加密

- 使用 AES-256-CBC 加密存储 — [auth.ts](server/src/utils/auth.ts#L26-L43)
- IV 随机生成，与密文拼接存储（格式：`iv:encrypted`）
- 读取时解密，展示时脱敏（仅显示前3后3字符）

### 7.3 OAuth Token 存储

- Google/Airtable 的 access_token / refresh_token 直接以明文存储在 Robot 表中
- Airtable token 使用 TEXT 类型（因 token 较长），Google token 使用 STRING 类型

---

## 8. 同步 / 异步边界与 userId 传递链路

在 Maxun 中，「同步路径」指受 `requireSignIn` / `requireAPIKey` 保护的 HTTP 请求，所有身份都是 **中间件强制注入** 的；而异步路径包括 setInterval 定时器、graphile-worker 任务、内存中的集成任务队列等，**身份完全依赖上游调用者传入的参数**。一旦同步到异步的边界处 userId 没有被正确封装、或异步下游没有进行「联合过滤」，就会产生越权或归属不一致问题。

### 8.1 四条同步 → 异步边界总览

| 边界编号 | 同步发起位置 | 触发机制 | 异步执行位置 | userId 传递方式 | 是否跨进程 |
|---------|------------|---------|------------|----------------|----------|
| B1 | HTTP 路由内（`/storage/...` `/api/...`）| `setInterval(5s)` 全局轮询 | [storage.ts#L1493](server/src/routes/storage.ts#L1493) `processQueuedRuns()` | 从 `Run.runByUserId` 字段回读 | 同进程 |
| B2 | HTTP 路由内调用 `addJob()` | graphile-worker（PG 通知 / 长轮询） | [task-runner.ts#L130](server/src/task-runner.ts#L130) `processRunExecution()` 等 7 个任务 | 任务 JSON payload 的 `data.userId` 字段 | 同进程（也可跨进程部署 worker）|
| B3 | 任意路径调用 `scheduleWorkflow()` | `setInterval(30s)` + DB 建议锁 | [schedule-worker.ts#L122](server/src/schedule-worker.ts#L122) `processDueSchedules()` → `addJob(SCHEDULED_WORKFLOW)` | 从 `Robot.userId` 列读出后写入 payload | 同进程 |
| B4 | 三条主路径（scheduler / task-runner / api/record）| `add*UpdateTask()` 推入内存 Map | [gsheet.ts#L382](server/src/workflow-management/integrations/gsheet.ts#L382) `processGoogleSheetUpdates()`、[airtable.ts#L634](server/src/workflow-management/integrations/airtable.ts#L634) `processAirtableUpdates()` | **完全没有传递 userId**，任务对象只有 robotId / runId | 同进程内存 |

### 8.2 边界 B2：graphile-worker 任务类型与 userId 字段完整性

[task-runner.ts#L631-L675](server/src/task-runner.ts#L631-L675) 定义了 7 种任务，各任务 payload 中 userId 字段的使用方式如下：

| 任务名 | payload 携带 userId？| 下游查询是否做 userId 联合过滤？ | 备注 |
|--------|--------------------|-------------------------------|------|
| `INITIALIZE_BROWSER_RECORDING` | ✅ `data.userId` | ❌ 仅传给 `initializeRemoteBrowserForRecording(userId)` 用于 BrowserPool 分配 | 查询 User 代理配置时正确按 userId 查 |
| `DESTROY_BROWSER` | ✅ `data.userId` | ⚠️ 仅 BrowserPool 清理，不查 DB | `destroyRemoteBrowser(browserId, userId)` 的 userId 仅用于 userToBrowserMap 清理 |
| `INTERPRET_WORKFLOW` | ✅ `data.userId` | ⚠️ 内存操作，无 DB 查询 | `interpretWholeWorkflow(userId)` |
| `STOP_INTERPRETATION` | ✅ `data.userId` | ⚠️ 同上 | 仅停止解释器状态机 |
| `EXECUTE_RUN` | ✅ `data.userId` | ❌ **关键缺口**：`Run.findOne({runId})` 和 `Robot.findOne({recording_meta.id})` **都不带 userId** | 见 Q2、Q3 风险 |
| `ABORT_RUN` | ✅ `data.userId` | ❌ **关键缺口**：同上 `Run.findOne({runId})` 无 userId，`destroyRemoteBrowser` 仅依赖内存 | 若 runId 被伪造可 abort 任意 run |
| `SCHEDULED_WORKFLOW` | ✅ `data.userId` | ❌ `createWorkflowAndStoreMetadata()` 中 `Robot.findOne({recording_meta.id})` 无 userId | 见 B3 → scheduler 链路分析 |

### 8.3 边界 B3：调度 worker 三层链路（DB poll → graphile → socket loopback）

调度系统有一个独特的「三次身份传递」过程，userId 在每一步都有被替换或丢失的风险：

**第 1 层：DB 轮询** ([schedule-worker.ts#L29-L85](server/src/schedule-worker.ts#L29-L85) `claimDueDbSchedules()`)

```typescript
// 每 30s 由 setInterval 触发（DB_SCHEDULER_POLL_MS = 30000）
const dueRobots = await Robot.findAll({
  where: {
    schedule: { [Op.ne]: null },
    schedule->>'nextRunAt' <= now,
    // ❌ 全局扫描：没有任何 userId 条件（正确，全局调度器要处理所有用户的计划）
  },
  attributes: ['id', 'userId', 'schedule',
    [Sequelize.literal(`recording_meta->>'id'`), 'robotMetaId']],
  lock: transaction.LOCK.UPDATE, skipLocked: true,
});

// 写入任务 payload（这一步是「从可信源读出 userId 后封装」）
claimed.push({ id: robot.id, robotMetaId, userId: String(robot.userId) });
await addJob(QUEUE_NAMES.SCHEDULED_WORKFLOW,
  { robotMetaId: robot.robotMetaId, userId: robot.userId },
  { maxAttempts: 6 });
```

- **此处是 userId 的可信起点**：从 Robot 表的 `userId` 列读出，真实性有 DB 保证
- 但后续 graphile-worker 反序列化 payload 时，`userId` 是 JSON 字符串，**不会再回到 Robot 表二次校验**

**第 2 层：graphile-worker 执行** ([task-runner.ts#L670-L674](server/src/task-runner.ts#L670-L674))

```typescript
[QUEUE_NAMES.SCHEDULED_WORKFLOW]: async (payload) => {
  const data = payload as ScheduledWorkflowData;
  // ⚠️ data.userId 直接透传，不校验 data.userId === Robot.userId
  await handleRunRecording(data.robotMetaId, data.userId);
},
```

**第 3 层：handleRunRecording → createWorkflowAndStoreMetadata → Socket 回连** ([scheduler/index.ts#L42-L138](server/src/workflow-management/scheduler/index.ts#L42-L138) + [scheduler/index.ts#L860-L909](server/src/workflow-management/scheduler/index.ts#L860-L909))

```typescript
// createWorkflowAndStoreMetadata 内部
const recording = await Robot.findOne({
  where: { 'recording_meta.id': id }
  // ❌ 无 userId 联合条件
});
// ...
await Run.create({
  runByUserId: userId, // ✅ 把上游 userId 写入 Run 表
  robotId: recording.id,
  robotMetaId: recording.recording_meta.id,
  // ⚠️ 如果 recording.userId 与 userId 不一致，Run 的 robotMetaId 与 runByUserId 将属于不同用户
});
```

**调度 worker 的 userId 归属一致性缺口**（Q3 的一个特例）：
- 当 graphile-worker 队列被外部写入假 payload，或 DB 中的 `Robot.userId` 记录被单独更改但 `Run.runByUserId` 仍为旧值时，会产生「代理配置取自 A 用户、Robot 工作流取自 B 用户」的混合场景
- `getDecryptedProxyConfig(userId)` ([scheduler/index.ts#L58](server/src/workflow-management/scheduler/index.ts#L58)) 按 payload 的 userId 取代理，而 Robot 工作流按 recording_meta.id 取，两者独立查询

### 8.4 边界 B4：辅助链（集成同步 / 文档处理）的 userId 丢失

这是 **最严重的 userId 传递断层**——从 run 完成后触发 `triggerIntegrationUpdates()` 起，userId 就被完全丢弃。

**触发入口（三处副本）**：

| 位置 | 触发场景 | 代码片段 |
|------|---------|---------|
| [scheduler/index.ts#L149-L173](server/src/workflow-management/scheduler/index.ts#L149-L173) | 调度方式 run 完成 | `triggerIntegrationUpdates(runId, robotMetaId)` （仅传两个参数，**无 userId**） |
| [task-runner.ts#L102-L110](server/src/task-runner.ts#L102-L110) | graphile EXECUTE_RUN 任务完成 | 同上，`triggerIntegrationUpdates(runId, robotMetaId)` |
| [api/record.ts#L665-L686](server/src/api/record.ts#L665-L686) | API/SDK 方式 run 完成 | 同上 |

**任务对象定义（无 userId 字段）**：

```typescript
// gsheet.ts#L6-L11 — GoogleSheetUpdateTask
{ robotId: string; runId: string; status: 'pending'|'completed'|'failed'; retries: number; }
// ❌ 缺少 userId 字段

// airtable.ts#L7-L12 — AirtableUpdateTask
{ robotId: string; runId: string; status: 'pending'|'completed'|'failed'; retries: number; }
// ❌ 缺少 userId 字段
```

**process*Updates 内部查询（完全无 userId 过滤）**：

| 集成类型 | 查询位置 | 查询语句 | 是否有 userId 联合条件 |
|---------|---------|---------|---------------------|
| Google Sheets | [gsheet.ts#L46](server/src/workflow-management/integrations/gsheet.ts#L46) | `Run.findOne({ where: { runId } })` | ❌ |
| Google Sheets | [gsheet.ts#L55](server/src/workflow-management/integrations/gsheet.ts#L55) | `Robot.findOne({ where: {'recording_meta.id': robotId} })` | ❌ |
| Airtable | [airtable.ts#L325](server/src/workflow-management/integrations/airtable.ts#L325) | `Run.findOne({ where: { runId } })` | ❌ |
| Airtable | [airtable.ts#L334](server/src/workflow-management/integrations/airtable.ts#L334) | `Robot.findOne({ where: {'recording_meta.id': robotId} })` | ❌ |

**后果**：
- 只要攻击者能猜到 `runId`（UUID v4，虽然难度高但纯算法层面），就能触发 `add*UpdateTask()` 让系统去同步该 run 的数据到 **任意目标的 Google Sheet / Airtable**（目标凭据存储在 Robot 表，不由 userId 过滤）
- 内存队列 `googleSheetUpdateTasks` / `airtableUpdateTasks` 是全局共享 Map，按 runId 去重，没有按 userId 分区或限流

**文档处理辅助链**：

[executeDocumentRun](server/src/utils/document/executeDocumentRun.ts#L7-L102) 和 [executeDocumentParseRun](server/src/utils/document/executeDocumentParseRun.ts#L8-L88) 接收 `(recording, run, userId, serverIo)` 参数：
- `userId` 仅用于 Socket emit 目标 (`serverIo.of('/queued-run').to('user-${userId}')`)
- Run 更新、Robot 查询、sendWebhook 调用 **完全不带 userId 条件**
- sendWebhook 同样只传 `robotMeta.id`，下游查询无 userId

### 8.5 总览：联合过滤缺口矩阵（Joint-Filtering Gap Matrix）

下表列出 **异步路径中所有按 robotMetaId / runId / browserId 查询，但未附加「userId 联合条件」的关键数据库操作**。

✅ 已做联合过滤 / ⚠️ 部分场景 / ❌ 完全缺失

| 查询操作 | HTTP 同步路径 | graphile-worker (B2) | schedule-worker (B3) | 集成辅助链 (B4) | 文档处理链 |
|---------|:----------:|:-------:|:-------:|:-------:|:-------:|
| `Robot.findByPk(...)` / `Robot.findOne({where:{id:...}})` | ✅ 全有 `userId` | — | — | — | — |
| `Robot.findOne({where:{'recording_meta.id':...}})` | ✅ 全有 `userId` | ❌ | ❌ `createWorkflowAndStoreMetadata` | ❌ `updateGoogleSheet` / `updateAirtable` | ❌（参数传入 recording） |
| `Run.findOne({where:{runId:...}})` | ✅ 仅少数端点 | ❌ `processRunExecution` `abortRun` | ❌ `executeRun`（L190 无，L219 ✅ 有） | ❌ `updateGoogleSheet` `updateAirtable` | — |
| `Run.findAll({where:{status:'queued'/'running'}})` | — | — | — | — | — |
| `Robot.update({schedule:...})` (`finalizeSchedule`) | — | — | ❌ 按 `recording_meta.id`，无 userId | — | — |
| `BrowserPool.getRemoteBrowser(id)` | ⚠️ 仅同步 handler 可信传入 | ❌ 按 id 取 | ❌ 按 id 取 | — | — |
| `sendWebhook(robotId, ...)` → Robot 查询 | — | ❌ | ❌ | — | ❌ |
| `Run.destroy` / `Run.update` | ✅ 全有 `userId`（通过 Run → Robot 归属校验）| ❌ 直接按 runId 改 | ❌ 直接按 runId 改 | — | — |

### 8.6 每一条 userId 传递路径的「信任链」总结

```
【信任链 A：HTTP → DB → Browser（同步强路径）】
JWT Cookie → req.user.id → 所有查询 where:{userId} → 返回结果
信任度：★★★★★  中间件强制注入，DB 查询强制过滤

【信任链 B：HTTP → addJob() → graphile → DB（B2 边界）】
JWT Cookie → req.user.id → JSON payload → data.userId → DB 查询（❌ 无联合条件）
信任度：★★☆☆☆  payload 反序列化后信任完全依赖上游正确性，下游不再验证

【信任链 C：scheduleWorkflow() → DB → setInterval → addJob() → handleRunRecording（B3）】
Robot.userId (DB 可信列) → claimDueDbSchedules 读出 → payload → data.userId
  → createWorkflowAndStoreMetadata（❌ Robot 查询无联合）
  → Run.create(runByUserId = data.userId)  ✅ 写入 Run 表
信任度：★★★☆☆  起点可信，但中途不校验 Robot.userId === data.userId

【信任链 D：run 完成 → triggerIntegrationUpdates → 内存队列 → 外部服务（B4）】
data.userId（上游携带） → ❌ 直接丢弃 → addXxxUpdateTask(robotId, runId)
  → process*Updates() → Run/Robot 查询（❌ 无 userId 联合）
  → Robot.google_sheet_* / airtable_* 凭据 → 外部 API 调用
信任度：★☆☆☆☆  userId 在这条链路中完全不可见

【信任链 E：HTTP → Socket.IO → namespace handler（实时路径）】
browserId UUID → 客户端自报 handshake.query.userId → 无校验加入 room
或：createSocketConnection 创建时 closure 绑定 userId → 所有后来连接共享该 userId
信任度：★★☆☆☆  closure 绑定的方向正确，但连接本身零认证
```

---

## 9. 队列运行处理的权限绕开与数据访问

队列运行处理（Queued Run Processing）是系统中最容易 **绕开用户认证层** 的场景之一：它由定时轮询和 graphile-worker 异步作业驱动，完全不经过 HTTP 中间件链，所有用户上下文都必须 **主动携带**（carried context）而非被动注入（passive injected）。

### 9.1 流程总览

```
用户请求入队 ──▶ Run.status='queued', runByUserId=userId
                         │
        ┌────────────────┴───────────────────┐
        │                                    │
  轮询定时器（每5s）                     graphile-worker
  processQueuedRuns()                   processRunExecution(data)
  【零认证，全表扫描】                   【data.userId 是唯一凭证】
```

### 9.2 processQueuedRuns — 全局调度器的无认证行为

**[storage.ts#L1493-L1566](server/src/routes/storage.ts#L1493-L1566)**

该函数由 `setInterval` 每 5 秒调用一次（`5000ms`），触发入口是 [server.ts#L151-L158](server/src/server.ts#L151-L158)。

**访问控制分析：**

| 步骤 | 代码行为 | 是否依赖用户过滤 | 风险点 |
|------|----------|-----------------|--------|
| ① 取待运行项 | `Run.findOne({ where: { status: 'queued' }, order: [...] })` | ❌ **完全无过滤**，跨所有用户竞争 | FIFO 顺序，用户 A 的 run 可能被排在 B 之后 |
| ② 取 userId | `const userId = queuedRun.runByUserId` | ⚠️ 依赖数据本身的正确性，非校验 | 若数据库中 runByUserId 被篡改，会导致错误用户的浏览器配额被消耗 |
| ③ 查 Robot | `Robot.findOne({ where: { 'recording_meta.id': queuedRun.robotMetaId } })` | ❌ **无 userId 过滤**，仅靠 meta.id | 若 robotMetaId 被伪造，可触发任意用户的 Robot 配置 + 凭据 |
| ④ 创建浏览器 | `createRemoteBrowserForRun(userId)` | ✅ 占用该用户的浏览器配额槽位 | ✅ 正确（userId 来自 runByUserId） |
| ⑤ 更新 Run | `queuedRun.update({ status, browserId, log })` | ⚠️ 通过 Run 实例方法，无二次校验 | Run 已通过 findOne 取出，更新默认锁定该行 |
| ⑥ 入执行队列 | `addJob(QUEUE_NAMES.EXECUTE_RUN, { userId, runId, browserId })` | ⚠️ userId 透传到下一跳 | ✅ 正确传递 |

> **关键发现**：步骤 ③ Robot 查询 **缺少 `userId: runByUserId` 联合条件**。虽然这是内部代码路径（非用户直接输入），但如果攻击者能在 Run 表中构造虚假的 `robotMetaId`，理论上可读取其他用户的 Robot 工作流（含 Google/Airtable 凭据字段）。

### 9.3 recoverOrphanedRuns — 启动时的孤儿恢复

**[storage.ts#L1572-L1643](server/src/routes/storage.ts#L1572-L1643)**

在 [server.ts#L171-L176](server/src/server.ts#L171-L176) 启动后立即执行一次。

**访问控制分析：**

- `Run.findAll({ where: { status: ['running', 'scheduled'] } })` — ❌ **全局扫描，无用户过滤**
- 对于每个孤儿 run，检查 `browserPool.getRemoteBrowser(browserId)` 以判断浏览器是否仍存活
- 若浏览器已不存在：根据 retryCount 将 run 重新标记为 `queued` 或 `failed`
- 重新入队后，run 会带着 **原始的 runByUserId** 再次经过 processQueuedRuns，流程同 §9.2

> **风险**：此函数不校验当前服务端是否"拥有"这些 Run（比如在多实例部署时，runByUserId 对应的用户资源可能在另一台实例上）。但由于 BrowserPool 是进程内单例，误标记为 orphan 最多触发重试，不会泄露跨用户数据。

### 9.4 graphile-worker 中的 EXECUTE_RUN 任务

**[task-runner.ts#L130-L419](server/src/task-runner.ts#L130-L419)** 中的 `processRunExecution(data: ExecuteRunData)`。

data 结构：
```typescript
{ userId: string; runId: string; browserId: string }
```

**数据范围控制：**

| 操作 | 查询条件 | 过滤情况 |
|------|----------|---------|
| 查 Run | `Run.findOne({ where: { runId } })` | ❌ 仅按 runId，无 userId |
| 查 Robot | `Robot.findOne({ where: { 'recording_meta.id': plainRun.robotMetaId } })` | ❌ 仅按 robotMetaId，无 userId |
| 查 Browser | `browserPool.getRemoteBrowser(browserId)` | ⚠️ 通过内存 Map 查找，理论上可能找到非 userId 归属的浏览器（但 browserId 是 UUID，猜测困难） |
| 查代理 | `getDecryptedProxyConfig(userId)` | ✅ 按任务 data.userId 取用户代理配置 |
| 写入 Run 更新 | `run.update({ ... })` 通过 Sequelize 实例 | ⚠️ 实例方法，无二次校验 |

> **双重绕过点**：Run 和 Robot 查询均无 userId 联合过滤。如果 data.userId 和 run.robotMetaId 不一致（被恶意构造），就会用 **用户 A 的代理配置** 去执行 **用户 B 的 Robot 工作流**，这是跨用户资源混用的严重问题。

### 9.5 调度器（Scheduler）中的定时运行

**[scheduler/index.ts](server/src/workflow-management/scheduler/index.ts)**

调度场景有两条路径：

**路径 A：`scheduled-workflow` 任务（graphile-worker）**

```typescript
// 任务 data: { robotMetaId, userId }
const recording = await Robot.findOne({
  where: { 'recording_meta.id': id }, raw: true
});
// ❌ 无 userId 联合过滤

const proxyConfig = await getDecryptedProxyConfig(userId);
// ✅ 代理按任务 data.userId 获取

await createWorkflowAndStoreMetadata(id, userId);
// ⚠️ createWorkflowAndStoreMetadata 中 Robot 查找同样无 userId 过滤
```

**路径 B：`handleRunRecording` 内部回连**

[handleRunRecording](server/src/workflow-management/scheduler/index.ts#L860-L909) 中 scheduler 进程作为 **Socket.IO 客户端** 主动连接自身 HTTP 服务：

```typescript
socket = io(`http://localhost:5000/${browserId}`, {
  transports: ['websocket'],
  rejectUnauthorized: false,
  timeout: 30000,
});
```

- 这条连接 **完全绕过 HTTP 认证**（不走 Cookie，不带 x-api-key）
- 依赖的安全性完全建立在：`browserId` 是 UUID 难以猜测 + namespace 是动态创建且仅短时间存在

### 9.6 队列处理的权限风险汇总

| # | 场景 | 问题 | 严重度 |
|---|------|------|--------|
| Q1 | `processQueuedRuns` 取 Robot | 无 `runByUserId` 联合条件 | 中 |
| Q2 | `processRunExecution` 查 Run/Robot | 无 userId 联合条件 | 中 |
| Q3 | 任务 data.userId 与 run.robotMetaId 不一致 | 代理配置和 Robot 工作流可跨用户混用 | 高 |
| Q4 | Scheduler 回连 Socket | 不经过任何认证，靠 browserId UUID 保密性 | 中 |
| Q5 | `recoverOrphanedRuns` 全表扫描 | 可能错误重试跨实例 run（多部署场景） | 低 |

---

## 10. Webhook 回调的权限模型

Webhook 系统分为两部分：**Webhook 配置的 CRUD**（受 `requireSignIn` 保护）和 **Webhook 的触发发送**（后台异步，完全绕开认证）。

webhook 路由在 [server.ts#L119](server/src/server.ts#L119) 中挂载于 `/webhook` 前缀，即所有端点以 `/webhook` 开头。

### 10.1 Webhook 配置管理（受权限保护）

**[webhook.ts](server/src/routes/webhook.ts)**

| 端点（完整路径） | HTTP 方法 | 中间件 | Robot 归属校验 |
|-----------------|----------|--------|---------------|
| `/webhook/add` | `POST` | `requireSignIn` | ✅ `where: { 'recording_meta.id': robotId, userId }` |
| `/webhook/update` | `POST` | `requireSignIn` | ✅ 同上 |
| `/webhook/remove` | `POST` | `requireSignIn` | ✅ 同上 |
| `/webhook/list/:robotId` | `GET` | `requireSignIn` | ✅ 同上 |
| `/webhook/test` | `POST` | `requireSignIn` | ✅ 同上 |
| `/webhook/clear/:robotId` | `DELETE` | `requireSignIn` | ✅ 同上 |

所有 CRUD 操作都正确做了 userId 过滤。

### 10.2 sendWebhook — 异步发送路径（绕开认证）

**[webhook.ts#L404-L434](server/src/routes/webhook.ts#L404-L434)**

```typescript
export const sendWebhook = async (robotId: string, eventType: string, data: any): Promise<void> => {
    const robot = await Robot.findOne({ where: { 'recording_meta.id': robotId } });
    // ❌ 关键：无 userId 过滤！仅按 recording_meta.id 查

    if (!robot || !robot.webhooks) return;

    const activeWebhooks = robot.webhooks.filter(
        w => w.active && w.events.includes(eventType)
    );

    const webhookPromises = activeWebhooks.map(async (webhook: WebhookConfig) => {
        const payload = {
            event_type: eventType,
            timestamp: new Date().toISOString(),
            webhook_id: webhook.id,
            data: data
        };
        return sendWebhookWithRetry(robotId, webhook, payload);
    });

    await Promise.allSettled(webhookPromises);
};
```

**调用链（13 处触发点）：**

| 调用位置 | 调用者身份 | 传入的 robotMetaId 来源 |
|----------|-----------|----------------------|
| [scheduler/index.ts#L492](server/src/workflow-management/scheduler/index.ts#L492) | scheduled-workflow 任务（doc robot 成功） | `plainRun.robotMetaId` |
| [scheduler/index.ts#L749](server/src/workflow-management/scheduler/index.ts#L749) | scheduled-workflow 任务（scrape 成功） | `plainRun.robotMetaId` |
| [scheduler/index.ts#L799](server/src/workflow-management/scheduler/index.ts#L799) | scheduled-workflow 任务（失败） | `run.robotMetaId` |
| [task-runner.ts#L358](server/src/task-runner.ts#L358) | execute-run 任务（scrape 成功） | `plainRun.robotMetaId` |
| [task-runner.ts#L493](server/src/task-runner.ts#L493) | execute-run 任务（scrape 成功） | `plainRun.robotMetaId` |
| [task-runner.ts#L543](server/src/task-runner.ts#L543) | execute-run 任务（scrape 失败） | `plainRun.robotMetaId` |
| [task-runner.ts#L567](server/src/task-runner.ts#L567) | execute-run 任务（最终失败） | `run.robotMetaId` |
| [api/record.ts#L1017](server/src/api/record.ts#L1017) | API/SDK 运行（scrape 成功） | `plainRun.robotMetaId` |
| [api/record.ts#L1080](server/src/api/record.ts#L1080) | API/SDK 运行（scrape 失败） | `plainRun.robotMetaId` |
| [api/record.ts#L1302](server/src/api/record.ts#L1302) | API/SDK 运行（工作流成功） | `plainRun.robotMetaId` |
| [api/record.ts#L1377](server/src/api/record.ts#L1377) | API/SDK 运行（工作流失败） | `run.robotMetaId` |
| [executeDocumentParseRun.ts#L53](server/src/utils/document/executeDocumentParseRun.ts#L53) | doc-parse 类型运行 | `robotMeta.id` |
| [executeDocumentRun.ts#L67](server/src/utils/document/executeDocumentRun.ts#L67) | doc-extract 类型运行 | `robotMeta.id` |

### 10.3 Webhook 测试 Payload

**[webhook.ts#L272-L354](server/src/routes/webhook.ts#L272-L354)** 中 `/webhook/test` 端点发送的测试 payload 是一个完整的模拟数据结构：

```typescript
const testPayload = {
    event_type: "webhook_test",
    timestamp: new Date().toISOString(),
    webhook_id: webhook.id,
    data: {
        robot_id: robotId,
        run_id: "110c4dae-c39b-4b30-a932-eff1022e4bb0",
        robot_name: robot.recording_meta?.name || "E-commerce Product Scraper",
        status: "test",
        started_at: new Date(Date.now() - 45000).toISOString(),
        finished_at: new Date().toISOString(),
        extracted_data: {
            captured_texts: [
                {
                    "Product Name": "MacBook Pro 16-inch M3 Max",
                    "Price": "$3,999.00",
                    "Rating": "4.8/5 stars",
                    "Availability": "In Stock - Ships within 2-3 business days",
                    "SKU": "MBPM3-16-1TB-SLV",
                    "Description": "The most powerful MacBook Pro ever..."
                }
            ],
            captured_lists: {
                "list_1": [
                    { "Rank": "1", "Product": "MacBook Air M2", "Category": "Laptops", "Units Sold": "2,847", "Revenue": "$2,847,000" },
                    { "Rank": "2", "Product": "iPhone 15", "Category": "Smartphones", "Units Sold": "1,923", "Revenue": "$1,923,000" },
                    // ... 更多模拟数据
                ],
                // ...
            },
            total_rows: 11,
            captured_texts_count: 5,
            captured_lists_count: 6,
            screenshots_count: 5
        },
        metadata: {
            test_mode: true,
            browser_id: "d27ace57-75cb-441c-8589-8ba34e52f7d1",
            user_id: 108,
        }
    }
};
```

- 测试 payload 包含完整的 `event_type`、`timestamp`、`webhook_id` 信封结构
- `data.extracted_data` 中填充了模拟的电商商品抓取结果
- `metadata.test_mode: true` 标记这是测试调用
- 注意：`metadata.user_id: 108` 是硬编码的示例值，**不反映当前登录用户的真实 ID**

### 10.4 Webhook 数据泄露风险分析

**sendWebhook 查询 Robot 时缺少 userId 过滤**，这意味着：

1. **理论风险**：若攻击者能控制某条代码路径的 `robotMetaId` 参数传入 `sendWebhook`，就可触发向 **任意用户配置的 webhook URL** 发送 HTTP 请求（SSRF 风险），或读取任意 Robot 的 webhook 配置列表。

2. **实际触发场景**：目前所有 13 处调用都来自 Run 表中已存储的 `robotMetaId`，或来自 Robot 查询结果本身。由于 Run 和 Robot 的关联是内部创建的，攻击者如果无法直接写 Run 表，则此路径不易被利用。

3. **Payload 泄露**：webhook payload 中包含完整的抓取结果（markdown/html/links 等）。如果 webhook URL 本身被恶意配置（比如某用户将 URL 设为攻击者服务器），**只会泄露该用户自己的 Robot 运行结果**，不会跨用户泄露（因为 robotMetaId 是从属于用户的 Run 中取出的）。

4. **无签名/Secret 校验**：`sendWebhookWithRetry` 仅发送 `payload`，不计算 HMAC 签名。接收方无法验证 webhook 的真实性和完整性。

5. **SSRF 风险**：`/webhook/test` 允许用户配置任意 URL 并发送上述大型 payload。如果服务端网络环境允许内网访问，可被用于 SSRF 探测内网。

### 10.5 Webhook 权限问题汇总

| # | 问题 | 详情 | 严重度 |
|---|------|------|--------|
| W1 | sendWebhook 查 Robot 无 userId 过滤 | 缺少 `userId` 联合条件，内部防御深度不足 | 低 |
| W2 | 无 HMAC 签名 | 接收方无法验证消息来源和完整性 | 低 |
| W3 | test-webhook 接口可触发 SSRF | 用户可配置任意 URL，发送完整测试 payload | 中（payload 包含大量模拟数据） |
| W4 | 测试 payload 硬编码 user_id: 108 | 测试调用的 metadata.user_id 不反映真实用户 | 低 |

---

## 11. 本机/浏览器链接（Socket + BrowserPool）的身份绑定

Maxun 的浏览器交互链路高度依赖 **用户 ID ↔ 浏览器 ID ↔ Socket Namespace** 三者的绑定。这是一条 **完全绕过 HTTP 认证中间件** 的通道。

### 11.1 三层绑定关系

```
User.id (number)
    │  owns (1:N)
    ▼
Browser ID (UUID)  ←── 记录在 BrowserPool.userToBrowserMap
    │  namespace (1:1)
    ▼
Socket.io Namespace `/${browserId}`
    │  on('connection')
    ▼
registerInputHandlers(socket, userId)
    │  每个 handler 闭包中都绑定了 userId
    ▼
browserPool.getActiveBrowserId(userId, "recording")
    │  按 userId + state 查找
    ▼
操作对应的 RemoteBrowser
```

### 11.2 Recording 路径：userId 绑定的完整链路

**[controller.ts#L26-L104](server/src/browser-management/controller.ts#L26-L104)** — `initializeRemoteBrowserForRecording(userId)`

```
HTTP POST /recordings/start (requireSignIn)
    │  req.user.id
    ▼
initializeRemoteBrowserForRecording(userId.toString(), mode)
    │
    ├─▶ createSocketConnection(io.of(browserId), userId, callback)
    │       │
    │       └─▶ onConnection → registerInputHandlers(socket, userId)
    │               │  所有 14 个 input handler 都带 userId 闭包
    │               ▼
    │               handleWrapper(handleFunc, userId, data)
    │                   │
    │                   └─▶ browserPool.getActiveBrowserId(userId, "recording")
    │                          // ✅ 只在 userId 自己的浏览器池中查找
    │
    ├─▶ new RemoteBrowser(socket, userId, id, true)
    │       │
    │       └─▶ browserSession.initialize(userId)
    │               │
    │               └─▶ getDecryptedProxyConfig(userId)  // ✅ 取自己的代理配置
    │
    └─▶ browserPool.addRemoteBrowser(id, browserSession, userId, false, "recording")
            │
            └─▶ userToBrowserMap.set(userId, [..., id])  // ✅ 正确加入映射
```

**关键安全机制**：

1. **闭包绑定 userId**：[registerInputHandlers](server/src/browser-management/inputHandlers.ts#L871-L887) 中每个 socket 事件 handler 都是通过闭包绑定了创建时的 userId，不从 socket 消息中读取 userId。这意味着即使客户端能连接到 namespace，发送的所有操作仍然指向创建者 userId 的浏览器。

2. **BrowserPool 按 userId 查询**：`getActiveBrowserId(userId, state)` 始终先查 `userToBrowserMap.get(userId)`，再过滤 state。客户端无法通过 socket 消息指定操作其他用户的浏览器。

3. **浏览器槽位限制**：`addRemoteBrowser` 中每个 userId 最多 2 个浏览器（1 recording + 1 run），防资源耗尽。

### 11.3 Run 路径：浏览器创建与权限

**[controller.ts#L114-L137](server/src/browser-management/controller.ts#L114-L137)** — `createRemoteBrowserForRun(userId)`

```typescript
export const createRemoteBrowserForRun = (userId: string): string => {
  if (!userId) throw new Error('userId is required');   // ✅ 防御性检查
  const id = uuid();
  const slotReserved = browserPool.reserveBrowserSlotAtomic(id, userId, "run");
  if (!slotReserved) throw new Error('User has reached maximum browser limit');
  initializeBrowserAsync(id, userId)
    .catch(error => browserPool.failBrowserSlot(id));
  return id;
};
```

Run 路径不注册 input handler（因为不需要用户交互），但 browserId 仍加入 `userToBrowserMap`，代理配置同样按 userId 获取。

### 11.4 Socket 连接的身份验证缺口

虽然 handler 内部用闭包绑定 userId，但 **Socket.IO 连接建立本身不做任何身份验证**：

#### Recording Namespace（`/${browserId}`）

- [createSocketConnection](server/src/socket-connection/connection.ts#L12-L29)：`io.on('connection', onConnection)` 没有任何 `socket.handshake.auth` 或 token 校验。
- 客户端连接时仅需知道 `browserId` 即可。但 browserId 是 UUID v4，2^122 空间，暴力猜测不现实。
- **实际风险**：如果 browserId 通过某种方式泄露（如日志、错误消息、URL 参数），攻击者可连接到该 namespace：
  - 接收 DOM 流（可看到目标页面内容）
  - 由于 `createSocketConnection(io.of(id), userId, callback)` 会在 `io.on('connection')` 时为 **所有后来连接的 socket** 都注册 **同一个 userId** 的 handler，**任何能连接到该 namespace 的客户端发送的所有 input 操作都会被当作原始 userId 的操作执行**。这是一个 **严重的越权风险**。

#### Run Namespace（`/${browserId}`）

- [createSocketConnectionForRun](server/src/socket-connection/connection.ts#L38-L49)：同样无认证。
- 但 run 路径没有 input handler，仅使用 namespace 推送运行状态（run-started、run-completed 等）。泄露后果：攻击者可看到他人的运行进度和结果数据。

#### Queued-Run Namespace（`/queued-run`）

- [server.ts#L179-L201](server/src/server.ts#L179-L201)：仅根据 handshake.query.userId 加入 room，**不验证该 userId 的真实性**。
- 恶意客户端可构造 `?userId=<targetUserId>` 加入任意用户的 room，接收该用户所有 run-scheduled / run-started / run-completed 通知。

### 11.5 调度器的内部回连（Intra-process Socket Loopback）

[handleRunRecording](server/src/workflow-management/scheduler/index.ts#L882-L886) 中 scheduler 作为 Socket.IO 客户端连接自身：

```typescript
socket = io(`${process.env.BACKEND_URL ? process.env.BACKEND_URL : 'http://localhost:5000'}/${browserId}`, {
  transports: ['websocket'],
  rejectUnauthorized: false,
  timeout: 30000,
});
```

- 这是合法的内部通信，但它 **完全依靠 browserId 的保密性作为安全边界**。
- 如果 namespace socket 没有 auth 中间件，这条内部通道和外部攻击者可利用的通道在安全等级上相同。

### 11.6 BrowserPool 身份查询 API

**[BrowserPool.ts](server/src/browser-management/classes/BrowserPool.ts)**

| 方法 | 是否校验 userId | 作用 |
|------|----------------|------|
| `addRemoteBrowser(id, session, userId, ...)` | ✅ 创建 userToBrowserMap 条目 | 登记新浏览器 |
| `getRemoteBrowser(id)` | ❌ 仅按 id 查 | 返回浏览器实例 |
| `getActiveBrowserId(userId, state)` | ✅ 按 userId + state 过滤 | 查找激活的浏览器 |
| `hasAvailableBrowserSlots(userId, state)` | ✅ 按 userId 计数 | 检查是否可创建 |
| `reserveBrowserSlotAtomic(id, userId, state)` | ✅ 按 userId 去重 | 原子预留槽位 |
| `deleteRemoteBrowser(id)` | ⚠️ 内部通过 pool[id].userId 清理 userToBrowserMap | 释放浏览器 |
| `removeAllRemoteBrowsers(userId)` | ✅ 仅清理 userId 所属 | 用户退出清理 |
| `getUserForBrowser(id)` | ❌ 反向查询，可用于获取 browserId → userId | 辅助方法 |

> **关注点**：`getRemoteBrowser(id)` 无 userId 校验。如果内部代码路径获得了一个 browserId（比如从 Run 表中读取），可以直接拿到 RemoteBrowser 实例。内部代码需要自行保证 browserId 来源的可信性。

### 11.7 本机链接权限问题汇总

| # | 问题 | 详情 | 严重度 |
|---|------|------|--------|
| N1 | Recording Namespace 连接无认证 + handler 绑定创建者 userId | 任何知道 browserId 的人都能 **操作目标用户的浏览器**（点击、输入、导航）| 高 |
| N2 | Run Namespace 连接无认证 | 任何知道 browserId 的人能看到他人的运行结果 | 中 |
| N3 | `/queued-run` room 身份无验证 | 伪造 userId 可接收他人运行通知 | 中 |
| N4 | `getRemoteBrowser(id)` 无 userId 校验 | 内部调用者获得 browserId 即可跨用户操作浏览器 | 中（仅内部路径）|
| N5 | Scheduler 回连无 token 保护 | 内部通道与外部通道同安全等级 | 低 |
| N6 | browserId 可能在日志中泄露 | logger 多处输出 `browserId: ${id}` | 低 |

---

## 12. 总结

Maxun 的权限模型在同步 HTTP 路径上是清晰的 **用户级隔离**，但在异步和实时路径上存在大量 **"靠可信度而非强制校验"** 的软边界。

### 12.1 所有路径的权限执行强度对比

| 路径 / 边界 | 是否经过中间件 | 用户身份来源 | 数据查询是否带 userId | 整体信任级别 |
|------|---------------|-------------|----------------------|------------|
| **HTTP 路由**（Web UI / API）| ✅ `requireSignIn` / `requireAPIKey` | `req.user`（经 JWT 或 DB 验证）| ✅ 绝大多数带 `where: { userId }` | 强 ★★★★★ |
| **B1 边界** `processQueuedRuns`（setInterval 5s）| ❌ 绕过 | `Run.runByUserId` 从 DB 回读 | ⚠️ Robot 查询无联合条件 | 弱 ★★★☆☆ |
| **B2 边界** graphile-worker 7 种任务 | ❌ 绕过 | 任务 JSON payload 的 `data.userId` | ❌ Run/Robot 查询大多无联合条件 | 弱 ★★☆☆☆ |
| **B3 边界** `schedule-worker`（setInterval 30s）+ `SCHEDULED_WORKFLOW` | ❌ 绕过 | 第 1 层：`Robot.userId` DB 列（可信）→ 第 2、3 层：payload 的 `data.userId` | ❌ createWorkflowAndStoreMetadata 的 Robot 查询无联合条件 | 中 ★★★☆☆ |
| **B4 边界** 辅助链（gsheet / airtable）| ❌ 绕过 | **完全丢弃 userId**，任务对象无该字段 | ❌ `update*Table()` 的 Run / Robot 查询全无 userId | 极弱 ★☆☆☆☆ |
| **文档处理链**（doc-extract / doc-parse）| ❌ 绕过 | `executeDocument*Run(userId, ...)` 参数透传 | ⚠️ 仅 Socket emit 用 userId，DB 查询无联合 | 弱 ★★☆☆☆ |
| **实时通道**（Socket.IO namespace）| ❌ 绕过 | ① 创建时闭包绑定的 userId ② 客户端 handshake.query 自报 | ⚠️ BrowserPool 内存查询按 userId，但 socket 连接本身无认证 | 中 ★★☆☆☆ |

### 12.2 协同关系总结

1. **HTTP 中间件层** 负责颁发身份（设置 `req.user`），这是唯一的强认证点。
2. **路由 handler 层** 用 `req.user.id` 做数据过滤，这是主数据边界。
3. **同步→异步边界**（§8 B1–B4）是身份最容易丢失的断层：B1/B3 使用 `setInterval` 从 DB 回读身份，B2 把 `userId` 序列化进 JSON payload，B4 **直接丢弃 userId**（信任链 D ★☆☆☆☆）。
4. **异步任务层** 从 payload 反序列化 `data.userId` 并一路透传，**不再回头校验 `Robot.userId === data.userId`**——这是最大的信任链薄弱点（Q3 高风险）：可导致"用户 A 的代理配置执行用户 B 的 Robot 工作流"的归属错位。
5. **辅助链（Google Sheets / Airtable 同步）** 是全系统信任度最低的路径：任务对象无 userId 字段、DB 查询无 userId 联合条件、外部凭据仅按 robotId 取。一旦 runId 来源不可信，会触发跨用户的数据外泄。
6. **Socket 层** 把 `userId` 绑定为闭包变量，保证操作方向正确，但 **不验证连接者身份**——任何知道 browserId 的第三方都能以合法用户的身份发送输入（N1 高危）。
7. **Webhook 发送**、**孤儿恢复**、**DB 调度轮询**、**集成后处理** 等后台自动进程在数据库层面做 **全表扫描**，不区分用户。它们的数据范围由入参（robotMetaId / runId / browserId）的来源可信度间接保证。

### 12.3 改进优先级建议

| 优先级 | 措施 | 覆盖风险 |
|--------|------|---------|
| P0 | 在所有 `process*` 和 `execute*` 异步函数中，对 Run/Robot 查询增加 `userId` 联合条件，或在任务开始时校验 `Robot.userId === data.userId` | Q1, Q2, Q3 |
| P0 | **修复 B4 边界**：`add*UpdateTask()` 和 `update*Table()` 增加 `userId` 参数，在 `Robot.findOne()` / `Run.findOne()` 中加入联合过滤；给 `triggerIntegrationUpdates()` 增加 `userId` 实参 | 辅助链断层（信任链 D）|
| P0 | Socket.IO namespace 增加 auth 中间件（JWT token 或一次性 session token），验证连接者身份与 namespace 的 userId 一致 | N1, N2, N3, Q4 |
| P1 | `finalizeSchedule()` / `releaseScheduleClaim()` 在 `Robot.findOne({'recording_meta.id':...})` 查询中附加 `Robot.userId` 校验（从 payload 的 userId 反查），或使用 `Robot.id` 而非 `recording_meta.id` 定位 | 调度 worker 一致性 |
| P1 | `getRemoteBrowser(id)` 增加可选 `expectedUserId` 参数，内部校验归属一致性 | N4 |
| P1 | `sendWebhook` 中 Robot 查询增加可选 `expectedUserId` 校验（从 Run → Robot.userId 反查），或在调用前校验一致性 | W1 |
| P1 | `ABORT_RUN` 任务增加 `Run.runByUserId === data.userId` 校验，防止伪造 runId 跨用户中止 | ABORT_RUN 缺口 |
| P2 | `/queued-run` room 接入 JWT 验证，从 token 中解析 userId 而非从 query 取 | N3 |
| P2 | Webhook payload 增加 HMAC-SHA256 签名，使用 webhook 配置中的 secret | W2 |
| P2 | graphile-worker 任务 payload 增加签名或加密，防止被第三方通过 PG 注入伪造任务 | B2 边界 |
| P3 | JWT 增加 `expiresIn`，并增加 refresh token 机制 | 原风险 #3 |

原模型适合单人使用场景的结论不变，但在多用户部署时，上述异步和实时路径的软边界需要被强制加固为硬校验。
