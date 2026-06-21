# REST API 与 Webhook 回调协作流程分析

## 1. 整体架构概览

Maxun 是一个浏览器自动化/RPA 平台，核心围绕 **Robot（录制的工作流）** 和 **Run（一次执行实例）** 两个实体展开。外部请求通过多条路径进入，最终都汇聚到任务执行逻辑，任务结束时触发 webhook 回调。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           外部请求来源                                    │
├──────────┬──────────┬──────────┬──────────┬────────────────────────────┤
│  UI 手动  │ REST API │   SDK    │  MCP/CLI │     定时调度 (Schedule)     │
└────┬─────┴─────┬────┴────┬─────┴────┬─────┴─────────────┬──────────────┘
     │           │          │          │                   │
     ▼           ▼          ▼          ▼                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           任务创建层                                      │
│  ┌──────────────────┐   ┌─────────────────────────────────────────┐    │
│  │  storage.ts       │   │       api/record.ts                     │    │
│  │  (UI 手动触发)    │   │  handleRunRecording() 统一入口           │    │
│  └────────┬─────────┘   └───────────────┬─────────────────────────┘    │
│           │                             │                              │
│           ▼                             ▼                              │
│   Run(status=running/queued)      Run(status=running)                  │
└───────────┬─────────────────────────────┬──────────────────────────────┘
            │                             │
            ▼                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         任务执行层 (Worker)                               │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    Graphile Worker 任务队列                        │   │
│  │  QUEUE_NAMES.EXECUTE_RUN     → processRunExecution()             │   │
│  │  QUEUE_NAMES.SCHEDULED_WORKFLOW → handleRunRecording()           │   │
│  │  QUEUE_NAMES.ABORT_RUN       → abortRun()                        │   │
│  └──────────────────────────────┬───────────────────────────────────┘   │
│                                 │                                       │
│                                 ▼                                       │
│                        状态流转 + 数据采集 + 集成更新                    │
└─────────────────────────────────┬───────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        通知回调层                                         │
│    Socket.IO (前端实时通知)   +    Webhook (外部系统回调)                 │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. REST API 入口与认证

### 2.1 路由注册

所有路由在 [server.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/server.ts#L119-L135) 中注册：

| 路径前缀       | 路由文件               | 用途                     | 认证方式       |
|----------------|------------------------|--------------------------|----------------|
| `/api/*`       | `server/src/api/*.ts`  | 自动扫描加载             | API Key        |
| `/webhook`     | `routes/webhook.ts`    | Webhook CRUD + 发送      | Session        |
| `/storage`     | `routes/storage.ts`    | UI 用的 Robot/Run 管理   | Session        |
| `/record`      | `routes/record.ts`     | 浏览器录制               | Session        |
| `/workflow`    | `routes/workflow.ts`   | 工作流编辑               | Session        |

### 2.2 两种认证方式

**API Key 认证** — [middlewares/api.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/middlewares/api.ts)
- Header: `x-api-key: <your_key>`
- 适用于 `/api/sdk/*` 和 `/api/robots/*` 等外部调用接口

**Session 认证** — [middlewares/auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/middlewares/auth.ts)
- 基于 express-session + PostgreSQL 存储
- 适用于 Web UI 操作的所有接口

### 2.3 核心 API 端点（触发 Run 的入口）

| 方法 | 路径 | 触发方式 | 说明 |
|------|------|----------|------|
| PUT | `/storage/runs/:id` | UI 手动 | [storage.ts#L997](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/routes/storage.ts#L997) |
| POST | `/api/robots/:id/runs` | REST API | [api/record.ts#L1589](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/api/record.ts#L1589) |
| POST | `/api/sdk/robots/:id/execute` | SDK | [api/sdk.ts#L682](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/api/sdk.ts#L682) |
| (定时) | `schedule-worker` 轮询 | Schedule | [schedule-worker.ts#L122](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/schedule-worker.ts#L122) |

---

## 3. Run 状态模型与流转

### 3.1 Run 数据模型

定义在 [models/Run.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/models/Run.ts)，关键字段：

```typescript
interface RunAttributes {
  id: string;                    // 数据库主键 (UUID)
  runId: string;                 // 业务 Run ID (UUID)
  status: string;                // 状态：queued/running/success/failed/aborting/aborted
  name: string;
  robotId: string;               // 关联 Robot 表主键
  robotMetaId: string;           // Robot 的 recording_meta.id
  startedAt: string;
  finishedAt: string;
  browserId: string;             // 关联的浏览器实例 ID
  interpreterSettings: JSON;     // 解释器配置
  log: string;
  serializableOutput: JSON;      // 抓取的结构化数据
  binaryOutput: JSON;            // 截图等二进制数据
  // 来源标记
  runByUserId?: string;
  runByAPI?: boolean;
  runBySDK?: boolean;
  runByMCP?: boolean;
  runByCLI?: boolean;
  runByScheduleId?: string;
}
```

### 3.2 状态机

```
                    ┌───────────────────────┐
                    │        queued         │  浏览器槽位不足时排队
                    └──────────┬────────────┘
                               │ processQueuedRuns() 每 5s 轮询
                               ▼
┌──────────────┐    ┌───────────────────────┐    ┌──────────────────┐
│   aborting   │◄───│       running         │───►│     success      │
└──────┬───────┘    └──────────┬────────────┘    └──────────────────┘
       │                       │
       ▼                       ▼
┌──────────────┐    ┌───────────────────────┐
│   aborted    │    │       failed          │
└──────────────┘    └───────────────────────┘
```

**状态说明**：
- `queued`：用户浏览器槽位已满，等待 `processQueuedRuns()` 调度
- `running`：浏览器已分配，正在执行
- `success`：执行成功，数据已持久化
- `failed`：执行失败（含超时、异常）
- `aborting`：收到中止请求，正在清理
- `aborted`：已成功中止

---

## 4. 任务执行完整链路

系统存在**三条执行路径**，最终在数据采集和 webhook 发送处汇合。

### 4.1 路径 A：UI 手动触发（走 Graphile Worker）

**入口**：[storage.ts#L997](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/routes/storage.ts#L997) `PUT /storage/runs/:id`

```
1. 检查浏览器池可用槽位 browserPool.hasAvailableBrowserSlots()
   ├─ 有槽位 → 创建浏览器 → 创建 Run(status='running') → addJob(EXECUTE_RUN)
   └─ 无槽位 → 创建 Run(status='queued')

2. 排队任务由 processQueuedRuns() 每 5 秒轮询一次（storage.ts#L1493）
   → 找到 status='queued' 的 Run
   → 创建浏览器 → Run.status='running' → addJob(EXECUTE_RUN)

3. Graphile Worker 消费 EXECUTE_RUN 队列 → processRunExecution()
   [task-runner.ts#L130](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/task-runner.ts#L130)
```

### 4.2 路径 B：API/SDK/MCP/CLI 触发（同步执行 + Socket）

**入口**：[api/record.ts#L1403](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/api/record.ts#L1403) `handleRunRecording()`

```
1. handleRunRecording(robotId, userId, runSource)
   ├─ createWorkflowAndStoreMetadata() → 创建 Run(status='running')
   ├─ 建立 Socket.IO 连接到 /{browserId} 命名空间
   └─ 监听 'ready-for-run' 事件

2. 浏览器就绪后触发 'ready-for-run' → readyForRunHandler()
   → executeRun() [api/record.ts#L732](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/api/record.ts#L732)
   → 同步执行整个工作流（不走 Worker 队列）
```

> **特殊情况**：文档类型 Robot (`doc-extract` / `doc-parse`) 不需要浏览器，会通过 `addJob(EXECUTE_RUN)` 走 Worker 路径。

### 4.3 路径 C：定时调度触发

**调度器**：[schedule-worker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/schedule-worker.ts)

```
1. startScheduleWorker() 启动，每 30 秒执行 processDueSchedules()

2. claimDueDbSchedules() 事务性地认领到期任务：
   - 使用 PostgreSQL 咨询锁 (pg_try_advisory_xact_lock) 防止多实例重复调度
   - 查询 schedule.nextRunAt <= now 且未被认领的 Robot
   - 更新 schedulerClaimedAt 标记为已认领

3. addJob(QUEUE_NAMES.SCHEDULED_WORKFLOW, { robotMetaId, userId })

4. Worker 消费 SCHEDULED_WORKFLOW → handleRunRecording()
   → 后续流程与路径 B 完全相同
```

### 4.4 核心执行逻辑：processRunExecution()

位于 [task-runner.ts#L130](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/task-runner.ts#L130)，这是 Worker 路径的核心：

```
1. 加载 Run 记录，检查状态（已 abort/queued 则跳过）
2. 等待浏览器就绪（轮询 browserPool，超时 60s）
3. 等待当前页面就绪（超时 15s）
4. 根据 Robot 类型分支：
   ├─ type='scrape' → 直接页面转 Markdown/HTML/Text/截图 等
   └─ 其他类型    → browser.interpreter.InterpretRecording() 执行工作流
5. 处理输出：格式转换 → 上传截图到 MinIO → 更新 Run.status='success'
6. 触发通知：Socket.IO 广播 + sendWebhook() + 集成更新 (GS/Airtable)
7. destroyRemoteBrowser() 释放浏览器
```

---

## 5. Webhook 回调机制

### 5.1 Webhook 配置存储

Webhook 配置以 **JSONB** 形式存储在 Robot 表的 `webhooks` 字段中，定义在 [models/Robot.ts#L28-L58](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/models/Robot.ts#L28-L58)：

```typescript
interface WebhookConfig {
    id: string;                  // UUID
    url: string;                 // 回调地址
    events: string[];            // 订阅的事件：['run_completed', 'run_failed']
    active: boolean;             // 是否启用
    createdAt: string;
    updatedAt: string;
    lastCalledAt?: string | null;// 最近一次调用时间
    retryAttempts?: number;      // 最大重试次数，默认 3
    retryDelay?: number;         // 基础重试间隔秒数，默认 5
    timeout?: number;            // 请求超时秒数，默认 30
}
```

### 5.2 Webhook 管理 API

位于 [routes/webhook.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/routes/webhook.ts)：

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/webhook/add` | 添加 webhook，自动去重 URL |
| POST | `/webhook/update` | 更新 webhook 配置 |
| POST | `/webhook/remove` | 删除指定 webhook |
| GET | `/webhook/list/:robotId` | 获取 robot 的所有 webhook |
| POST | `/webhook/test` | 发送测试 payload（`event_type: webhook_test`） |
| DELETE | `/webhook/clear/:robotId` | 清空 robot 的所有 webhook |

此外，SDK API `PUT /api/sdk/robots/:id` 也支持通过 `webhooks` 字段批量更新。

### 5.3 回调触发时机

系统在**所有执行路径的结束点**都会调用 `sendWebhook()`，共 **5 处触发点**：

#### 触发点 1：Worker 路径 - scrape 类型成功
[task-runner.ts#L353-L361](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/task-runner.ts#L353-L361)
```typescript
await sendWebhook(plainRun.robotMetaId, 'run_completed', {
  runId, robotId, robotName, status: 'success', finishedAt,
  markdown, html, links, summary
});
```

#### 触发点 2：Worker 路径 - 普通类型成功
[task-runner.ts#L492-L506](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/task-runner.ts#L492-L506)
```typescript
await sendWebhook(plainRun.robotMetaId, 'run_completed', {
  robot_id, run_id, robot_name, status: 'success',
  started_at, finished_at,
  extracted_data: { captured_texts, captured_lists, crawl_data, search_data, ... },
  metadata: { browser_id, user_id }
});
```

#### 触发点 3：Worker 路径 - 执行失败（两层 catch）
[task-runner.ts#L542-L549](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/task-runner.ts#L542-L549) 和 [task-runner.ts#L566-L572](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/task-runner.ts#L566-L572)
```typescript
await sendWebhook(run.robotMetaId, 'run_failed', {
  robot_id, run_id, robot_name, status: 'failed',
  started_at, finished_at,
  error: { message, stack, type },
  partial_data_extracted, metadata
});
```

#### 触发点 4：API 同步路径 - scrape 类型成功/失败
[api/record.ts#L997-L1027](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/api/record.ts#L997-L1027)（成功）和 [api/record.ts#L1079-L1093](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/api/record.ts#L1079-L1093)（失败）

#### 触发点 5：API 同步路径 - 普通类型成功/失败
[api/record.ts#L1279-L1306](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/api/record.ts#L1279-L1306)（成功）和 [api/record.ts#L1358-L1381](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/api/record.ts#L1358-L1381)（失败）

### 5.4 sendWebhook 分发逻辑

核心函数在 [routes/webhook.ts#L404-L465](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/routes/webhook.ts#L404-L465)：

```
sendWebhook(robotId, eventType, data)
│
├─ 1. 查询 Robot，获取其 webhooks[] 配置
├─ 2. 过滤条件：w.active === true AND eventType in w.events
├─ 3. 对每个匹配的 webhook 构建 payload：
│     {
│       event_type: eventType,      // 'run_completed' | 'run_failed'
│       timestamp: ISO8601,
│       webhook_id: webhook.id,
│       data: data                 // 各触发点传入的数据
│     }
├─ 4. Promise.allSettled 并行调用 sendWebhookWithRetry()
└─ 5. 出错仅打日志，不影响主流程
```

### 5.5 重试与超时机制

`sendWebhookWithRetry(robotId, webhook, payload, attempt=1)`:

- **超时**：`webhook.timeout || 30` 秒（axios timeout）
- **最大重试**：`webhook.retryAttempts || 3` 次
- **退避策略**：指数退避 `retryDelay * 2^(attempt-1)` 秒
  - 第 1 次失败 → 等 5s → 第 2 次
  - 第 2 次失败 → 等 10s → 第 3 次
  - 第 3 次失败 → 放弃，打 error 日志
- **状态记录**：每次尝试前都会更新 `lastCalledAt`（即使失败也记录）

### 5.6 支持的事件类型

| 事件类型 | 触发条件 | Payload 关键字段 |
|----------|----------|------------------|
| `run_completed` | Run 状态变为 success | `robot_id`, `run_id`, `robot_name`, `status`, `started_at`, `finished_at`, `extracted_data`, `metadata` |
| `run_failed` | Run 状态变为 failed | 同上 + `error: { message, stack, type }`, `partial_data_extracted` |
| `webhook_test` | 用户点击测试按钮 | 模拟数据，包含示例 extracted_data |

### 5.7 run_completed Payload 结构（完整版）

```typescript
{
  event_type: "run_completed",
  timestamp: "2025-06-22T10:30:00.000Z",
  webhook_id: "wh-xxx",
  data: {
    robot_id: "robot-meta-id",
    run_id: "run-uuid",
    robot_name: "E-commerce Scraper",
    status: "success",
    started_at: "6/22/2025, 6:29:00 PM",
    finished_at: "6/22/2025, 6:30:00 PM",

    // scrape 类型特有
    markdown?: "# Page Title\n...",
    html?: "<html>...</html>",
    summary?: "AI 摘要...",
    screenshot_visible?: { /* MinIO URL */ },
    screenshot_fullpage?: { /* MinIO URL */ },

    // extract/crawl/search 类型特有
    extracted_data: {
      captured_texts: { "Product Name": ["MacBook Pro"], ... },
      captured_lists: { list_0: [...], ... },
      crawl_data: [...],
      search_data: {...},
      captured_texts_count: 5,
      captured_lists_count: 20,
      screenshots_count: 3
    },

    metadata: {
      browser_id: "browser-uuid",
      user_id: 108,
      // scrape 类型还可能包含 test_mode 等
    }
  }
}
```

---

## 6. 完整链路示例

以 **POST /api/sdk/robots/:id/execute** 为例，从请求到 webhook 的完整时序：

```
外部客户端                Maxun Server                         第三方 Webhook
     │                         │                                     │
     │ POST /api/sdk/robots/   │                                     │
     │   xxx/execute           │                                     │
     │────────────────────────►│                                     │
     │                         │ 1. requireAPIKey 校验 x-api-key     │
     │                         │ 2. handleRunRecording()             │
     │                         │    ├─ createWorkflowAndStoreMetadata│
     │                         │    │   └─ Run.create(status=running) │
     │                         │    └─ Socket 连接浏览器              │
     │                         │                                     │
     │                         │ 3. executeRun()                     │
     │                         │    ├─ 等待浏览器 ready              │
     │                         │    ├─ interpreter.InterpretRecording│
     │                         │    ├─ 采集数据 & 上传截图            │
     │                         │    ├─ Run.update(status=success)    │
     │                         │    ├─ Socket.IO 通知前端             │
     │                         │    └─ sendWebhook('run_completed')  │
     │                         │                                     │
     │                         │ 4. Promise.allSettled(webhooks)     │
     │                         │    ├─ 过滤 active + 匹配 events     │
     │                         │    ├─ 构建 payload                  │
     │                         │    └─ axios.post(webhook.url, ...)  │
     │                         │────────────────────────────────────►│
     │                         │                                     │ 2xx OK
     │                         │◄────────────────────────────────────│
     │                         │                                     │
     │  200 { data, status }   │                                     │
     │◄────────────────────────│                                     │
```

---

## 7. 关键文件索引

| 文件 | 作用 |
|------|------|
| [server.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/server.ts) | 服务器入口，路由注册，Worker 启动 |
| [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/task-runner.ts) | Graphile Worker 任务处理器，`processRunExecution()` |
| [schedule-worker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/schedule-worker.ts) | 定时调度轮询 |
| [routes/storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/routes/storage.ts) | UI 触发 Run 入口 + `processQueuedRuns()` |
| [routes/webhook.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/routes/webhook.ts) | Webhook CRUD + `sendWebhook()` + 重试逻辑 |
| [api/record.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/api/record.ts) | REST API 核心：`handleRunRecording()`, `executeRun()` |
| [api/sdk.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/api/sdk.ts) | SDK 专用 API |
| [models/Run.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/models/Run.ts) | Run 数据模型 |
| [models/Robot.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/models/Robot.ts) | Robot 数据模型（含 webhooks 字段） |
| [middlewares/api.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/middlewares/api.ts) | API Key 认证中间件 |
