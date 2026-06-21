# 调度器流程分析：Cron 任务 vs 一次性任务

## 一、整体架构概览

调度系统采用 **DB 轮询 + Graphile Worker 任务队列** 的混合架构，核心分层如下：

```
┌───────────────────────────────────────────────────────────────┐
│                        API 路由层                              │
│  storage.ts (PUT /runs/:id, PUT /schedule/:id/, POST /abort) │
└────────────────────┬──────────────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        ▼                         ▼
┌───────────────┐        ┌──────────────────┐
│ 一次性任务入口 │        │  Cron 配置入口    │
│ create Run +  │        │ scheduleWorkflow │
│ addJob()      │        │ 写 Robot.schedule │
└───────┬───────┘        └────────┬─────────┘
        │                         │
        ▼                         ▼
┌─────────────────────────────────────────────┐
│         Graphile Worker (PostgreSQL 队列)    │
│  ┌───────────────────────────────────────┐  │
│  │ QUEUE_NAMES                           │  │
│  │ · EXECUTE_RUN      ← 一次性/文档任务   │  │
│  │ · SCHEDULED_WORKFLOW ← Cron 触发任务   │  │
│  │ · ABORT_RUN        ← 中止任务         │  │
│  │ · INITIALIZE_BROWSER_RECORDING        │  │
│  │ · DESTROY_BROWSER                     │  │
│  │ · INTERPRET_WORKFLOW                  │  │
│  │ · STOP_INTERPRETATION                 │  │
│  └───────────────────────────────────────┘  │
└──────────────────┬──────────────────────────┘
                   │
        ┌──────────┴───────────┐
        ▼                      ▼
┌───────────────┐      ┌─────────────────────┐
│ processRunEx… │      │ handleRunRecording  │
│ (task-runner) │      │ (scheduler/index)   │
│ 执行工作流     │      │ 创建Run+浏览器+执行 │
└───────┬───────┘      └─────────┬───────────┘
        │                        │
        ▼                        ▼
┌─────────────────────────────────────────────┐
│              Run 状态机 + 通知               │
│  scheduled → queued → running → success/    │
│                        failed/aborted       │
│  + Socket.io 实时通知 + Webhook 回调         │
└─────────────────────────────────────────────┘
```

---

## 二、Cron 定时任务完整流程

### 2.1 Cron 配置写入

**入口**：[storage.ts#L1225-L1342](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/routes/storage.ts#L1225-L1342) `PUT /schedule/:id/`

用户通过 API 设置调度规则，流程如下：

1. 接收参数：`runEvery`、`runEveryUnit`（分钟/小时/天/周/月）、`startFrom`、`atTimeStart`、`atTimeEnd`、`timezone`
2. 构建 cron 表达式（使用 `node-cron.validate` 验证）：
   - 每分钟：`*/N * * * *`
   - 每小时：`M */N * * *`
   - 每天：`M H */N * *`
   - 每周：`M H * * DOW`
   - 每月：`M H DOM */N * [DOW]`
3. 调用 [storage/schedule.ts#L5-L28](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/storage/schedule.ts#L5-L28) `scheduleWorkflow()`：
   - 使用 [utils/schedule.ts#L4-L12](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/utils/schedule.ts#L4-L12) `computeNextRun(cronExpression, timezone)` 计算下次运行时间（基于 `cron-parser`）
   - 更新 `Robot.schedule` JSONB 字段：
     ```ts
     schedule: {
       cronExpression,
       timezone,
       nextRunAt,        // 下次触发时间（Date）
       schedulerClaimedAt: undefined,
       // ... 其他配置字段
     }
     ```

### 2.2 DB 轮询与认领

**入口**：[schedule-worker.ts#L156-L177](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/schedule-worker.ts#L156-L177) `startScheduleWorker()`

服务启动时启动定时器，每 `30000ms`（30秒）轮询一次：

```
startScheduleWorker()
  └─ setInterval(processDueSchedules, 30000)
       └─ claimDueDbSchedules()  // 核心认领逻辑
```

**认领逻辑** [schedule-worker.ts#L29-L85](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/schedule-worker.ts#L29-L85)：

采用 **PostgreSQL 咨询锁 + 行级锁（SKIP LOCKED）** 保证分布式安全：

1. 事务内获取咨询锁：`pg_try_advisory_xact_lock(43821742)`
2. 查询符合条件的 Robot：
   - `schedule.cronExpression IS NOT NULL`（配置了 cron）
   - `schedule.nextRunAt <= NOW()`（已到触发时间）
   - `schedule.schedulerClaimedAt IS NULL OR < 10分钟前`（未被认领或认领已超时）
3. `SELECT ... FOR UPDATE SKIP LOCKED` 跳过已被锁定的行
4. 对每个满足条件的 Robot，设置 `schedulerClaimedAt = NOW()` 标记为已认领
5. 按 `nextRunAt ASC` 排序，最多处理 `BATCH_SIZE = 10` 条

### 2.3 入队 SCHEDULED_WORKFLOW

**调度分发** [schedule-worker.ts#L122-L154](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/schedule-worker.ts#L122-L154) `processDueSchedules()`：

```ts
for each claimed robot:
  executedAt = NOW()
  dispatched = false
  try:
    addJob(QUEUE_NAMES.SCHEDULED_WORKFLOW, 
      { robotMetaId, userId }, 
      { maxAttempts: 6 }   // 最多重试6次
    )
    dispatched = true
  finally:
    if dispatched:
      finalizeSchedule(robotMetaId, executedAt)  // 计算下一次nextRunAt
    else:
      releaseScheduleClaim(robotMetaId)           // 释放认领标记
```

**finalizeSchedule** [schedule-worker.ts#L87-L108](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/schedule-worker.ts#L87-L108)：
- 调用 `computeNextRun()` 计算下一次运行时间
- 更新 Robot：`schedulerClaimedAt` 清空，`lastRunAt = executedAt`，`nextRunAt = 新时间`

### 2.4 Cron 任务执行

**任务处理器** [task-runner.ts#L670-L674](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/task-runner.ts#L670-L674)：

```ts
QUEUE_NAMES.SCHEDULED_WORKFLOW → handleRunRecording(robotMetaId, userId)
```

调用链 [workflow-management/scheduler/index.ts#L860-L909](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/workflow-management/scheduler/index.ts#L860-L909)：

```
handleRunRecording(id, userId)
  ├─ createWorkflowAndStoreMetadata(id, userId)
  │   ├─ 查询 Robot 记录
  │   ├─ 创建浏览器实例（doc-robot除外：直接用uuid()）
  │   ├─ Run.create({ status: 'scheduled', ... })
  │   ├─ 发送 Socket 'run-scheduled' 通知
  │   └─ if isDocRobot:
  │        └─ addJob(QUEUE_NAMES.EXECUTE_RUN, {...}, {maxAttempts: 1})
  │           → 直接交给通用执行器
  │
  └─ else (非文档机器人):
       ├─ socket.connect(`/${browserId}`)
       ├─ 监听 'ready-for-run' 事件
       └─ readyForRunHandler → executeRun(runId, userId)
```

**executeRun** [workflow-management/scheduler/index.ts#L186-L832](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/workflow-management/scheduler/index.ts#L186-L832) 是 Cron 任务的实际执行函数（一次性任务使用 `processRunExecution`），两者逻辑类似，详见下文。

---

## 三、一次性（手动）任务完整流程

### 3.1 手动触发入口

**入口**：[storage.ts#L997-L1131](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/routes/storage.ts#L997-L1131) `PUT /runs/:id`

核心是 **浏览器槽位检查 + 分支处理**：

```
PUT /runs/:id
  │
  ├─ hasAvailableBrowserSlots(userId, "run")
  │
  ├─ [有槽位] → 立即执行路径:
  │    ├─ createRemoteBrowserForRun(userId) → browserId
  │    ├─ Run.create({ status: 'running', browserId, ... })
  │    └─ addJob(QUEUE_NAMES.EXECUTE_RUN, 
  │         { userId, runId, browserId }, 
  │         { maxAttempts: 1 }
  │       )
  │
  └─ [无槽位] → 排队等待路径:
       ├─ browserId = uuid()  // 占位用
       ├─ Run.create({ status: 'queued', 
       │               log: 'Run queued - waiting for available browser slot',
       │               ... })
       └─ 由 processQueuedRuns() 定时器稍后处理
```

### 3.2 排队任务处理

**入口**：[storage.ts#L1493-L1566](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/routes/storage.ts#L1493-L1566) `processQueuedRuns()`

服务启动时通过 `setInterval` 定时调用（由 server.ts 注册）：

```
processQueuedRuns()
  ├─ 熔断器检查：连续3次错误 → 冷却30秒
  ├─ Run.findOne({ status: 'queued' }, ORDER BY startedAt ASC)
  │
  └─ 再次检查 hasAvailableBrowserSlots
       ├─ [有槽位]:
       │    ├─ createRemoteBrowserForRun(userId)
       │    ├─ Run.update({ status: 'running', browserId, ... })
       │    └─ addJob(QUEUE_NAMES.EXECUTE_RUN, {...})
       │
       └─ [仍无槽位]: 跳过，下次轮询再试
```

### 3.3 通用执行器 processRunExecution

**入口**：[task-runner.ts#L130-L582](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/task-runner.ts#L130-L582)，由 `QUEUE_NAMES.EXECUTE_RUN` 触发

这是 **一次性任务、文档机器人、Cron文档任务** 的共享执行路径。完整流程：

#### Step 1: 前置检查 (L136-L179)
```
- 查询 Run 记录
- 检查 status:
  · 'aborted' | 'aborting' → 跳过
  · 'queued' → 跳过（由恢复机制处理）
- 判断机器人类型:
  · doc-extract → import(executeDocumentRun) 执行文档抽取
  · doc-parse   → import(executeDocumentParseRun) 执行文档解析
```

#### Step 2: 浏览器等待 (L181-L213)
```
- browserPool.getRemoteBrowser(browserId)
- 轮询等待，最多 60秒 (BROWSER_INIT_TIMEOUT)
- 浏览器状态检查:
  · null → 浏览器槽位不存在，报错
  · 'failed' → 浏览器初始化失败，报错
```

#### Step 3: Page 等待 (L217-L237)
```
- browser.getCurrentPage()
- 轮询等待，最多 15秒 (BROWSER_PAGE_TIMEOUT)
```

#### Step 4: 按机器人类型分支执行

**分支 A: scrape 类型** (L239-L379)
```
- 从 interpreterSettings 或 Robot 配置中读取 formats
- Run.update({ status: 'running' })
- 发送 Socket 'run-started' 通知
- 按顺序执行 format 转换（每个带120秒超时）:
  · screenshot-visible / screenshot-fullpage
  · text / markdown / html / links
  · summary（基于markdown → LLM）
  · promptInstructions（BrowserAgent LLM智能查询）
- Run.update({ status: 'success/failed', serializableOutput, binaryOutput })
- 二进制文件上传 MinIO (BinaryOutputService)
- 发送 Socket 'run-completed' + Webhook
- destroyRemoteBrowser()
```

**分支 B: 工作流类型（extract/crawl/search）** (L381-L555)
```
- Run.update({ status: 'running' })
- 发送 Socket 'run-started' 通知
- browser.interpreter.setRunId(runId) → 实时数据持久化
- InterpretRecording() → 600秒超时 (10分钟)
- 期间检查 isRunAborted() → 如果被用户中止则直接返回
- crawl/search 类型后处理:
  · processRobotOutputFormats() → 格式转换 + LLM摘要
  · hasExpectedRobotOutput() → 输出验证
- Run.update({ status: 'success', serializableOutput + binaryOutput })
- 二进制上传 + Socket + Webhook + Integration
- destroyRemoteBrowser()
```

#### Step 5: 错误兜底 (L512-L581)
```
执行异常时:
  - 检查是否有部分数据已提取 → 触发 Integration 更新
  - Run.update({ status: 'failed', log })
  - 发送失败通知 (Socket + Webhook)
  - 捕获 analytics 事件
  - 清理浏览器
```

---

## 四、任务状态机与状态流转

### 4.1 Run 状态枚举

定义于 [models/Run.ts#L69-L72](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/models/Run.ts#L69-L72)，实际使用值：

| 状态 | 含义 | 设置位置 |
|------|------|----------|
| `scheduled` | Cron任务已创建Run记录，等待浏览器就绪 | scheduler/index.ts#L78 |
| `queued` | 浏览器槽位不足，排队等待 | storage.ts#L1104 |
| `running` | 浏览器就绪，工作流执行中 | processRunExecution L245/L389, scheduler L273 |
| `success` | 执行成功，结果已保存 | processRunExecution L335/L469, scheduler L436/L658 |
| `failed` | 执行失败（含超时） | 多处 catch 块 |
| `aborting` | 用户请求中止，正在清理 | storage.ts#L1438, task-runner L592 |
| `aborted` | 中止已完成 | task-runner.ts#L602/L607, storage.ts#L1443 |

### 4.2 完整状态流转图

```
Cron 任务路径:
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────┐
│  scheduled  │────▶│   running   │────▶│   success   │ or  │ failed  │
│ (Run.create)│     │ (execute)   │     │             │     │         │
└─────────────┘     └──────┬──────┘     └─────────────┘     └─────────┘
                           │
                           ▼
                      ┌──────────┐  ┌─────────┐
                      │ aborting │─▶│ aborted │
                      └──────────┘  └─────────┘

手动任务路径:
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────┐
│   running   │────▶│  (execute)  │────▶│   success   │ or  │ failed  │
│ (有浏览器)  │     │ processRun  │     │             │     │         │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────┘

排队任务路径:
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────┐
│   queued    │────▶│   running   │────▶│   success   │ or  │ failed  │
│ (无浏览器)  │     │ processRun  │     │             │     │         │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────┘
       │
       └──── processQueuedRuns() 轮询，每轮检查浏览器槽位
```

### 4.3 状态更新中的通知机制

每次状态变化触发三类通知：

1. **Socket.io 实时推送**（用户前端界面）
   - 命名空间：`/queued-run` → `user-${userId}` 房间
   - 命名空间：`/${browserId}`（浏览器专属通道）
   - 事件：`run-scheduled` / `run-started` / `run-completed` / `run-aborted`

2. **Webhook 回调**（用户配置的外部系统）
   - 事件类型：`run_completed` / `run_failed`
   - 载荷包含：run_id、status、extracted_data、error 等
   - 入口：[routes/webhook.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/routes/webhook.ts) `sendWebhook()`

3. **第三方集成更新**
   - Google Sheets：`processGoogleSheetUpdates()`
   - Airtable：`processAirtableUpdates()`
   - 每个有 65秒 超时保护

---

## 五、Graphile Worker 队列系统

### 5.1 队列定义

定义于 [task-runner.ts#L37-L45](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/task-runner.ts#L37-L45)：

```ts
const QUEUE_NAMES = {
  INITIALIZE_BROWSER_RECORDING: 'initialize-browser-recording',
  DESTROY_BROWSER:            'destroy-browser',
  INTERPRET_WORKFLOW:         'interpret-workflow',
  STOP_INTERPRETATION:        'stop-interpretation',
  EXECUTE_RUN:                'execute-run',        // ← 一次性任务+文档任务
  ABORT_RUN:                  'abort-run',
  SCHEDULED_WORKFLOW:         'scheduled-workflow', // ← Cron 定时任务
} as const;
```

### 5.2 addJob 封装

定义于 [storage/graphileWorker.ts#L61-L71](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/storage/graphileWorker.ts#L61-L71)：

```ts
export async function addJob(
  taskIdentifier: string,        // 对应 QUEUE_NAMES
  payload: Record<string, unknown>,
  options?: {
    maxAttempts?: number;        // 最大重试次数
    runAt?: Date;                // 延迟执行时间
    jobKey?: string;             // 去重键
  }
): Promise<string>  // 返回 job.id
```

### 5.3 Worker 启动配置

定义于 [task-runner.ts#L681-L716](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/task-runner.ts#L681-L716)：

```ts
runner = await run({
  pgPool,
  concurrency: TOTAL_CONCURRENCY,  // max(1, parseInt(env.WORKER_CONCURRENCY || '10'))
  noHandleSignals: true,
  pollInterval: 3600000,           // 1小时轮询（主要靠LISTEN/NOTIFY）
  taskList,                         // 任务处理函数映射表
});
```

### 5.4 Cron vs 一次性任务在队列中的区别

| 维度 | Cron 任务 | 一次性任务 |
|------|-----------|------------|
| 入队队列 | `SCHEDULED_WORKFLOW` | `EXECUTE_RUN` |
| maxAttempts | 6（重试6次） | 1（不重试） |
| 执行函数 | `handleRunRecording` → `executeRun` | `processRunExecution` |
| Run初始状态 | `scheduled` | `running`（有浏览器）/ `queued`（无浏览器） |
| 浏览器初始化 | handleRunRecording 内同步创建 | API入口创建 / queued轮询后创建 |
| 文档机器人 | 二次入队 EXECUTE_RUN | 直接执行 processRunExecution 分支 |

---

## 六、故障恢复与保护机制

### 6.1 服务器崩溃恢复

**入口**：[storage.ts#L1572-L1643](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/routes/storage.ts#L1572-L1643) `recoverOrphanedRuns()`

服务启动时调用，处理 `status IN ('running', 'scheduled')` 的孤立 Run：

```
for each orphaned run:
  if browserPool中已无对应browser:
    if retryCount < 3:
      Run.update({
        status: 'queued',
        retryCount++,
        browserId: undefined,
        log: '[RETRY N/3] Re-queuing due to server crash'
      })
    else:
      Run.update({ status: 'failed', log: 'Max retries exceeded...' })
  else:
    // 浏览器仍然活跃 → 不处理
```

### 6.2 Cron 认领超时

在 `claimDueDbSchedules()` 中：
```ts
claimExpiry = NOW() - 10 * 60 * 1000  // 10分钟
// 条件: schedulerClaimedAt IS NULL OR < claimExpiry
```
- 若调度器崩溃，认领标记10分钟后自动失效
- 下次轮询会重新认领该任务

### 6.3 熔断器（processQueuedRuns）

连续3次DB错误后，打开熔断器，30秒内不再处理排队任务：
```ts
consecutiveDbErrors >= 3 → circuitBreakerOpenUntil = NOW() + 30000
```

### 6.4 执行超时保护

- 浏览器初始化超时：60秒
- Page 就绪超时：15秒
- 工作流执行超时：600秒（10分钟）
- Scrape 格式转换超时：每个120秒
- Integration 更新超时：每个65秒

---

## 七、关键数据模型

### 7.1 Robot.schedule (JSONB)

定义于 [models/Robot.ts#L61-L73](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/models/Robot.ts#L61-L73)：

```ts
interface ScheduleConfig {
  runEvery: number;
  runEveryUnit: 'MINUTES' | 'HOURS' | 'DAYS' | 'WEEKS' | 'MONTHS';
  startFrom: 'SUNDAY'..'SATURDAY';
  atTimeStart?: string;     // 'HH:mm'
  atTimeEnd?: string;
  timezone: string;         // e.g. 'Asia/Shanghai'
  lastRunAt?: Date;
  nextRunAt?: Date;         // ← 轮询判断的核心字段
  dayOfMonth?: string;
  cronExpression?: string;  // ← 实际调度表达式
  schedulerClaimedAt?: Date; // ← 分布式锁标记
}
```

### 7.2 Run 核心字段

定义于 [models/Run.ts#L14-L35](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/models/Run.ts#L14-L35)：

| 字段 | 类型 | 说明 |
|------|------|------|
| `status` | string | 状态机核心字段 |
| `runId` | UUID | 对外暴露的Run ID（不同于主键id） |
| `robotMetaId` | UUID | 关联 Robot.recording_meta.id |
| `browserId` | UUID | 关联浏览器实例 |
| `startedAt` / `finishedAt` | string | 开始/结束时间（本地化字符串） |
| `interpreterSettings` | JSONB | maxConcurrency, formats, robotType 等 |
| `serializableOutput` | JSONB | 结构化结果（markdown, scrapeSchema 等） |
| `binaryOutput` | JSONB | MinIO URL映射 |
| `retryCount` | int | 崩溃重试计数（默认0，最多3次） |
| `runByUserId` | int | 触发用户 |
| `runByScheduleId` | UUID | Cron任务专用 |
| `runByAPI` / `SDK` / `MCP` / `CLI` | bool | 调用来源标记 |

---

## 八、分流总结：Cron vs 一次性任务

```
用户动作:
  "设置调度规则"          "点击立即运行"
       │                       │
       ▼                       ▼
  PUT /schedule/:id     PUT /runs/:id
       │                       │
       ▼                       ▼
  scheduleWorkflow()    ┌──────┴───────┐
  写 cronExpression     │ 检查浏览器槽  │
  + nextRunAt           │              │
       │            有槽▼              ▼无槽
       │          Run(status:running) Run(status:queued)
       │          add EXECUTE_RUN     │  由 processQueuedRuns
       │                              │  轮询后转 running
       │                              └─────┬──────┘
       ▼                                    ▼
  ┌──────────────────────┐        EXECUTE_RUN 队列
  │ schedule-worker 轮询 │        (Graphile Worker)
  │ 每30秒扫描 Robot表   │               │
  └──────────┬───────────┘               ▼
             │                   processRunExecution()
             ▼                           │
  claimDueDbSchedules()                  │
  (咨询锁+SKIP LOCKED)                   │
             │                           │
             ▼                           │
  SCHEDULED_WORKFLOW 队列                │
  (Graphile Worker, maxAttempts=6)       │
             │                           │
             ▼                           │
  handleRunRecording()                   │
             │                           │
    ┌────────┴────────┐                  │
    │ doc robot?      │                  │
    ├─Yes────add EXECUTE_RUN─────────────┤
    │                                     │
    └─No──create browser──socket─────────┘
             │
             ▼
        executeRun()
  (Cron任务专用执行函数，逻辑与
   processRunExecution 高度相似)
```

**核心分流点**：
1. **入口不同**：Cron 通过 `schedule-worker` DB轮询 → `SCHEDULED_WORKFLOW`；一次性任务通过 REST API → `EXECUTE_RUN`
2. **初始状态不同**：Cron创建的Run是 `scheduled`，一次性任务是 `running` 或 `queued`
3. **执行路径部分合并**：Cron文档机器人也会二次入队 `EXECUTE_RUN`，最终与一次性任务汇合
4. **重试策略不同**：Cron调度失败最多重试6次（`SCHEDULED_WORKFLOW`），一次性任务执行不重试（`maxAttempts: 1`）
5. **浏览器创建时机不同**：一次性任务在API入口先创建浏览器再入队；Cron任务在 `handleRunRecording` 内部创建浏览器
