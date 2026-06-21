# 调度器完整链路分析：Cron / 手动 / API / SDK / CLI / MCP

> 所有代码引用使用仓库相对路径，格式 `server/src/文件路径:行号范围`，可直接在仓库中定位。

---

## 一、核心分流规则一览（先看这张表）

五路触发源最终分成 **三条完全独立** 的执行路径：

| 触发源 | 非文档机器人执行路径 | 文档机器人执行路径 | Graphile Worker 队列 | Run 初始状态 | Run 来源标记 |
|--------|-------------------|------------------|--------------------|-------------|------------|
| ① **Cron 定时** | `SCHEDULED_WORKFLOW` → `scheduler/handleRunRecording()` → socket → `scheduler/executeRun()` | `SCHEDULED_WORKFLOW` → socket → 二次入队 `EXECUTE_RUN` → `processRunExecution()` | ✅ 走队列（`SCHEDULED_WORKFLOW`，重试 6 次） | `scheduled` | `runByScheduleId` |
| ② **前端手动** | 直接入队 `EXECUTE_RUN` → `processRunExecution()` | 同左 | ✅ 走队列（`EXECUTE_RUN`，不重试） | `running` / `queued` | 无 |
| ③ **REST API** | **不走队列**，socket 直调 → `api/executeRun()` | 入队 `EXECUTE_RUN` → `processRunExecution()` | 非文档❌；文档✅（`EXECUTE_RUN`） | `running` | `runByAPI` |
| ④ **SDK / CLI** | **不走队列**，socket 直调 → `api/executeRun()` | 入队 `EXECUTE_RUN` → `processRunExecution()` | 非文档❌；文档✅（`EXECUTE_RUN`） | `running` | `runBySDK` / `runByCLI` |
| ⑤ **MCP Worker** | **不走队列**，socket 直调 → `api/executeRun()` | 入队 `EXECUTE_RUN` → `processRunExecution()` | 非文档❌；文档✅（`EXECUTE_RUN`） | `running` | `runByMCP` |

> **关键结论**：外部调用（API/SDK/CLI/MCP）的非文档任务与 Cron 定时队列 **完全没有交集**——它们是两条完全独立的路径，只是因为 `handleRunRecording` 函数同名（分属 `scheduler/index.ts` 和 `api/record.ts` 两个不同文件）才容易混淆。

---

## 二、统一链路全景图

```
                          ┌──────────────────────────────────────────────────────────────┐
                          │                        五路触发源                              │
                          └───────────────┬───────────────────┬──────────────────────────┘
                                          │                   │
                    ┌─────────────────────┘                   └─────────────────────┐
                    │                                                                 │
          ┌─────────▼──────────┐                                           ┌──────────▼──────────────────┐
          │  ① Cron 定时调度器  │                                           │  ②~⑤ 一次性触发（外部调用）   │
          │  schedule-worker   │                                           │  前端 / API / SDK / CLI / MCP │
          └─────────┬──────────┘                                           └──────────┬──────────────────┘
                    │                                                                  │
                    ▼                                                                  ▼
       ┌─────────────────────────────┐                            ┌─────────────────────┴─────────────────────┐
       │ addJob(SCHEDULED_WORKFLOW,  │                            │                                           │
       │        { maxAttempts: 6 })  │                     ┌──────▼──────┐                           ┌──────▼──────────┐
       └─────────────┬───────────────┘                     │ ②前端手动   │                           │ ③~⑤ API/SDK/   │
                     │                                     │ routes/     │                           │ CLI / MCP       │
                     ▼                                     │ storage.ts  │                           │ (api/*.ts)      │
       ┌────────────────────────────────────┐              │ PUT /runs/:id│                           └──────┬──────────┘
       │ task-runner.ts:660-674              │              └──────┬──────┘                                  │
       │ taskList[SCHEDULED_WORKFLOW]        │                     │                                         │
       │ → import from scheduler/index.ts:25 │                     ▼                                         ▼
       │   handleRunRecording(id, userId)    │         addJob(EXECUTE_RUN, {maxAttempts:1})    handleRunRecording(
       └─────────────┬──────────────────────┘                                                     id,userId,runSource)
                     │                                                                            server/src/api/record.ts:1403
                     ▼                                                                            └────────┬─────────┘
    ┌──────────────────────────────────────────────┐                                                │
    │ scheduler/index.ts:860-909                   │                            ┌───────────────────┴───────────────────┐
    │ handleRunRecording(id, userId)  ← Cron专用   │                            │                                       │
    │ ├─ createWorkflowAndStoreMetadata()          │                    ┌───────▼────────┐              ┌──────────▼──────────┐
    │ │  Run.status='scheduled'                    │                    │ ②前端(有/无槽)   │              │ ③~⑤ 外部调用          │
    │ │  runByScheduleId                           │                    │ processRun-      │              │ createWorkflowAnd-    │
    │ │  文档机器人: addJob(EXECUTE_RUN)            │                    │ Execution()      │              │ StoreMetadata()       │
    │ │                                             │                    │ task-runner.ts:130│             │ server/src/api/       │
    │ └─ 非文档机器人: socket →                     │                    └────────┬─────────┘              │ record.ts:552         │
    │      'ready-for-run' → executeRun()          │                             │                        │ Run.status='running'  │
    │      scheduler/index.ts:186-832              │                             │                        │ runByAPI/SDK/MCP/CLI  │
    └──────────────────────────┬───────────────────┘                             │                        │                        │
                               │                                                 │                        └───────────┬────────────┘
                               │                                                 │                                    │
                               │                                                 │                           ┌────────┴──────────┐
                               │                                                 │                           │                   │
                               │                                                 │                      ┌────▼────┐      ┌─────▼──────┐
                               │                                                 │                      │ 文档机器人│      │ 非文档机器人 │
                               │                                                 │                      │ 入队      │      │ socket 直调  │
                               │                                                 │                      │EXECUTE_RUN│      │ 不经过队列   │
                               │                                                 │                      └────┬─────┘      └──────┬───────┘
                               │                                                 │                           │                 │
                               └──────────────────────┬──────────────────────────┘                           │                 ▼
                                                      │                                                          │  ready-for-run
                                                      │                                                          │  → executeRun()
                                                      ▼                                                          │  api/record.ts:732
                                    ┌─────────────────────────────────────────────┐                              │
                                    │     Graphile Worker: EXECUTE_RUN            │                              │
                                    │     task-runner.ts:660-662                   │                              │
                                    │     → processRunExecution(payload)           │                              │
                                    └───────────────────┬─────────────────────────┘                              │
                                                        │                                                          │
                                                        ▼                                                          ▼
                                    ┌───────────────────────────────────────────────────────────────────────────────┐
                                    │                  执行函数（三路实现，逻辑高度相似）                              │
                                    │                                                                                │
                                    │  processRunExecution      executeRun(api版)       executeRun(scheduler版)      │
                                    │  task-runner.ts:130-582   api/record.ts:732-1401  workflow-management/         │
                                    │                                                   scheduler/index.ts:186-832   │
                                    │                                                                                │
                                    │  · 通用队列处理器         · API/SDK/CLI/MCP专用   · Cron专用                    │
                                    │  · 前端手动 + 所有文档    · 外部调用非文档机器人   · Cron非文档机器人            │
                                    └───────────────────────────────────────────────────────┬───────────────────────┘
                                                                            ▼
                                                          ┌──────────────────────────────────────────┐
                                                          │  状态更新 running → success / failed      │
                                                          │  通知: Socket.io + Webhook + Integration  │
                                                          └──────────────────────────────────────────┘
```

---

## 三、五路触发源详解（带精确定位）

### 3.1 触发源 ①：Cron 定时调度

**完整链路：DB轮询认领 → SCHEDULED_WORKFLOW队列 → scheduler/handleRunRecording → (socket或二次入队EXECUTE_RUN)**

#### Step 1：服务启动时注册轮询

`server/src/schedule-worker.ts:156-177`
```ts
export function startScheduleWorker() {
  setInterval(() => processDueSchedules(), 30_000);  // 每 30 秒
  processDueSchedules();  // 启动时立即执行一次
}
```

#### Step 2：DB 认领（分布式锁）

`server/src/schedule-worker.ts:29-85` — `claimDueDbSchedules()`
- 条件：`schedule.cronExpression IS NOT NULL` AND `schedule.nextRunAt <= NOW()` AND (`schedulerClaimedAt IS NULL` OR `< 10 分钟前`)
- 锁机制：`pg_try_advisory_xact_lock(43821742)` + `SELECT ... FOR UPDATE SKIP LOCKED`
- 标记：`schedule.schedulerClaimedAt = NOW()`
- 批量：`BATCH_SIZE = 10`

#### Step 3：入队 SCHEDULED_WORKFLOW

`server/src/schedule-worker.ts:122-154` — `processDueSchedules()`
```ts
addJob(QUEUE_NAMES.SCHEDULED_WORKFLOW,
  { robotMetaId, userId },
  { maxAttempts: 6 }   // Cron 最多重试 6 次
)
```
成功后调用 `finalizeSchedule()` (`server/src/schedule-worker.ts:87-108`) 计算下一次 `nextRunAt`，并清零 `schedulerClaimedAt`。

#### Step 4：SCHEDULED_WORKFLOW 队列处理器

`server/src/task-runner.ts:670-674`
```ts
[QUEUE_NAMES.SCHEDULED_WORKFLOW]: async (payload) => {
  const data = payload as ScheduledWorkflowData;
  // 重点：import 来自 server/src/task-runner.ts:25
  // import { handleRunRecording } from './workflow-management/scheduler';
  await handleRunRecording(data.robotMetaId, data.userId);
}
```
注意：这里调用的是 **`scheduler/index.ts` 版** 的 `handleRunRecording(id, userId)`（仅 2 个参数）。

#### Step 5：Cron 专用 handleRunRecording

`server/src/workflow-management/scheduler/index.ts:860-909`
```ts
export async function handleRunRecording(id: string, userId: string) {
  const result = await createWorkflowAndStoreMetadata(id, userId);
  const { browserId, runId, isDocRobot } = result;

  if (isDocRobot) return runId;  // 文档机器人: 已在下方 createWorkflow 内二次入队 EXECUTE_RUN

  // 非文档机器人: Socket 等待浏览器就绪
  socket = io(BACKEND_URL/${browserId})
  socket.on('ready-for-run', () =>
    readyForRunHandler(browserId, runId, userId, socket)
    // → scheduler/index.ts:841 → executeRun(runId, userId)
  )
}
```

#### Step 6：Cron 专用 createWorkflowAndStoreMetadata

`server/src/workflow-management/scheduler/index.ts:54-98`
```ts
async function createWorkflowAndStoreMetadata(id, userId) {
  browserId = isDocRobot ? uuid() : createRemoteBrowserForRun(userId)
  const run = await Run.create({
    status: 'scheduled',            // ← Cron 专用初始状态
    runByScheduleId: scheduleId,    // ← Cron 专用标记
    ...
  })
  serverIo.of('/queued-run').to('user-'+userId).emit('run-scheduled', ...)

  if (isDocRobot) {
    // 文档机器人二次入队 → 汇入 EXECUTE_RUN 通用处理
    await addJob(QUEUE_NAMES.EXECUTE_RUN,
      { userId, runId, browserId },
      { maxAttempts: 1 }
    )  // server/src/workflow-management/scheduler/index.ts:86-89
  }
  return { browserId, runId, isDocRobot }
}
```

---

### 3.2 触发源 ②：前端手动触发

**完整链路：PUT /runs/:id → (有浏览器槽:直接入队 EXECUTE_RUN) / (无槽: Run.status=queued + 轮询) → processRunExecution**

#### Step 1：入口 API

`server/src/routes/storage.ts:997-1131` — `PUT /runs/:id`

```ts
if (hasAvailableBrowserSlots(userId, "run")) {
  // 路径 A：有浏览器槽位 → 立即执行
  browserId = createRemoteBrowserForRun(userId)
  Run.create({ status: 'running', browserId, ... })
  addJob(QUEUE_NAMES.EXECUTE_RUN,            // server/src/routes/storage.ts:1069
    { userId, runId, browserId },
    { maxAttempts: 1 }
  )
} else {
  // 路径 B：无槽位 → 排队
  browserId = uuid()  // 占位 ID
  Run.create({ status: 'queued',               // server/src/routes/storage.ts:1104
               log: 'Run queued - waiting for available browser slot', ... })
  // 不入队，由 processQueuedRuns() 定时轮询处理
}
```

#### Step 2：排队任务后续处理

`server/src/routes/storage.ts:1493-1566` — `processQueuedRuns()`
- 服务启动时 `setInterval` 注册，定时轮询
- 熔断器：连续 3 次 DB 错误 → 冷却 30 秒 (`server/src/routes/storage.ts:1507-1515`)
- 处理逻辑：
  ```ts
  Run.findOne({ status: 'queued' }, ORDER BY startedAt ASC)
    → 再次检查 hasAvailableBrowserSlots
       → 有槽位: Run.update(status:'running') + addJob(EXECUTE_RUN)  (server/src/routes/storage.ts:1539)
       → 无槽位: 跳过，下次再试
  ```

---

### 3.3 触发源 ③④⑤：API / SDK / CLI / MCP

**完整链路：各入口 → 统一汇入 api/record.ts:1403 的 handleRunRecording(id, userId, runSource) → (文档机器人:入队 EXECUTE_RUN) / (非文档机器人:socket直调 api/executeRun, 完全不经过 Graphile Worker)**

#### 调用关系图

```
触发源 ⑤ MCP Worker
  server/src/mcp-worker.ts:113-182 — run_robot 工具
  │ HTTP POST /api/robots/:id/runs
  │ Header: x-run-source: 'mcp'
  ▼
触发源 ③ REST API
  server/src/api/record.ts:1589-1620 — POST /api/robots/:id/runs
  │ runSource = headers['x-run-source'] === 'mcp' ? 'mcp' : 'api'
  │
  └────────────────────────────────────────────────┐
                                                   │
触发源 ④ SDK / CLI                                 │
  server/src/api/sdk.ts:682-801 — POST /api/sdk/robots/:id/execute
  │ runSource = headers['x-run-source'] === 'cli' ? 'cli' : 'sdk'
  │
  ▼                                                  ▼
统一入口：api/record.ts:1403-1458  handleRunRecording(id, userId, runSource, ...)
  │  runSource ∈ {'api' | 'sdk' | 'mcp' | 'cli'}
  │
  └─→ createWorkflowAndStoreMetadata()  server/src/api/record.ts:552-654
        ├─ Run.status = 'running'
        ├─ runByAPI / runBySDK / runByMCP / runByCLI 标记
        │
        ├─ [文档机器人] addJob(EXECUTE_RUN, {maxAttempts:1})  server/src/api/record.ts:632-636
        │     → 汇入通用队列处理器 processRunExecution
        │
        └─ [非文档机器人] socket 等待 'ready-for-run'
              → readyForRunHandler()  server/src/api/record.ts:691-713
              → executeRun(runId, userId)  server/src/api/record.ts:732-1401
              【完全不经过 Graphile Worker 队列！】
```

#### 统一入口：handleRunRecording（api 版，4 参数）

`server/src/api/record.ts:1403-1458`
```ts
export async function handleRunRecording(
  id: string, userId: string,
  runSource: 'api' | 'sdk' | 'mcp' | 'cli' = 'api',
  requestedFormats?, promptInstructions?
) {
  const { browserId, runId, isDocRobot } = await createWorkflowAndStoreMetadata(
    id, userId, runSource, requestedFormats, promptInstructions
  )

  if (isDocRobot) return runId;  // 文档机器人: 已在 createWorkflow 内入队 EXECUTE_RUN

  // 非文档机器人: Socket 等待浏览器就绪，然后直接调用 executeRun
  // 【重点：完全不经过 Graphile Worker 队列】
  socket = io(BACKEND_URL/${browserId})
  socket.on('ready-for-run', () =>
    readyForRunHandler(browserId, runId, userId, socket)
  )
}
```

#### Run 创建：createWorkflowAndStoreMetadata（api 版）

`server/src/api/record.ts:552-654`
```ts
async function createWorkflowAndStoreMetadata(
  id, userId,
  runSource: 'api' | 'sdk' | 'mcp' | 'cli',
  requestedFormats?, promptInstructions?
) {
  browserId = isDocRobot ? uuid() : createRemoteBrowserForRun(userId)
  const run = await Run.create({
    status: 'running',                     // ← 直接 running
    interpreterSettings: { formats, promptInstructions, ... },
    runByAPI: runSource === 'api',         // ← 四路来源标记
    runBySDK: runSource === 'sdk',
    runByMCP: runSource === 'mcp',
    runByCLI: runSource === 'cli',
    ...
  })
  serverIo.of('/queued-run').to('user-'+userId).emit('run-started', ...)

  if (isDocRobot) {
    // 文档机器人入队 EXECUTE_RUN → 与前端手动触发汇合
    await addJob(QUEUE_NAMES.EXECUTE_RUN,
      { userId, runId, browserId },
      { maxAttempts: 1 }
    )  // server/src/api/record.ts:632-636
  }
  return { browserId, runId, isDocRobot }
}
```

#### readyForRunHandler → executeRun（api 版，socket 直调）

`server/src/api/record.ts:691-713`
```ts
async function readyForRunHandler(browserId, id, userId, socket) {
  const result = await executeRun(id, userId)
  // ↑ 直接调用同文件内的 executeRun，不经过任何队列
  // 成功: resetRecordingState + 清理 socket
  // 失败: destroyRemoteBrowser + 清理 socket
}
```

---

## 四、执行层：三条路径的实现与状态更新归属

### 4.1 三条执行路径对照

| 路径 | 执行函数 | 文件位置 | 适用场景 | Run 初始状态 | 是否经过队列 |
|------|---------|---------|---------|-------------|------------|
| **A** | `processRunExecution()` | `server/src/task-runner.ts:130-582` | 前端手动（有槽/排队）+ 所有文档机器人（Cron/API/SDK/CLI/MCP） | `running` / `queued` | ✅ `EXECUTE_RUN` |
| **B** | `executeRun()` (api版) | `server/src/api/record.ts:732-1401` | API/SDK/CLI/MCP 的非文档机器人 | `running` | ❌ 完全不经过队列，socket 直调 |
| **C** | `executeRun()` (scheduler版) | `server/src/workflow-management/scheduler/index.ts:186-832` | Cron 触发的非文档机器人 | `scheduled` | 先经 `SCHEDULED_WORKFLOW` 队列，后 socket 直调 |

> **重要**：三条路径的函数内部逻辑（scrape 格式转换、工作流 Interpret、LLM 摘要、后处理、通知发送）**高度相似**，但属于三次独立实现。

### 4.2 路径 A：processRunExecution（EXECUTE_RUN 队列处理器）

`server/src/task-runner.ts:130-582`

```
processRunExecution(job)
  ├─ 前置检查 [:136-179]
  │   ├─ Run 状态: aborted/aborting → skip; queued → skip(由崩溃恢复处理)
  │   └─ 机器人类型: doc-extract → executeDocumentRun; doc-parse → executeDocumentParseRun
  │
  ├─ 浏览器等待 [:181-213]
  │   ├─ browserPool.getRemoteBrowser(browserId)  60秒超时
  │   └─ getCurrentPage()  15秒超时
  │
  ├─ 分支 scrape 类型 [:239-379]
  │   ├─ Run.update(status:'running') + Socket 'run-started'  [:245, :277-285]
  │   ├─ 格式转换 (每个 120 秒超时):
  │   │   screenshot-visible/fullpage → text → markdown → summary(LLM)
  │   │   → html → links → promptInstructions(BrowserAgent)
  │   ├─ Run.update(status:'success', serializableOutput, binaryOutput)  [:335]
  │   ├─ BinaryOutputService 上传 MinIO
  │   ├─ Socket 'run-completed' + Webhook 'run_completed'  [:362-373]
  │   └─ destroyRemoteBrowser()
  │
  ├─ 分支 extract/crawl/search 工作流类型 [:381-555]
  │   ├─ Run.update(status:'running') + Socket 'run-started'  [:389, :428-433]
  │   ├─ interpreter.setRunId(runId)  实时持久化
  │   ├─ InterpretRecording()  600 秒超时(10分钟)  [:405]
  │   ├─ 期间轮询 isRunAborted() → 若中止则提前返回
  │   ├─ crawl/search 后处理: processRobotOutputFormats()
  │   ├─ Run.update(status:'success', ...)  [:469]
  │   ├─ Socket 'run-completed' + Webhook + triggerIntegrationUpdates()  [:459-498]
  │   └─ destroyRemoteBrowser()
  │
  └─ 异常兜底 [:512-581]
      ├─ 有部分提取数据 → triggerIntegrationUpdates()
      ├─ Run.update(status:'failed', log)
      ├─ Socket 'run-completed'(失败) + Webhook 'run_failed'
      └─ 清理浏览器
```

### 4.3 路径 B/C：executeRun（Socket 回调路径，不经过队列）

以 api 版为例（`server/src/api/record.ts:732-1401`），scheduler 版（`server/src/workflow-management/scheduler/index.ts:186-832`）逻辑几乎一致：

```
executeRun(id, userId)
  ├─ 前置检查 [:736-785]
  │   ├─ Run 状态: aborted/aborting → skip; queued → skip; retryCount>=3 → 永久失败
  │   ├─ browserPool.getRemoteBrowser() + getCurrentPage()
  │
  ├─ 分支 scrape 类型 [:789-1110]
  │   ├─ Run.update(status:'running')  [:801-804] / scheduler版 [:273]
  │   └─ (格式转换逻辑与 processRunExecution 完全一致)
  │   └─ Run.update(status:'success')  [:961] / scheduler版 [:436]
  │
  ├─ 分支 extract/crawl/search 工作流类型 [:1112-1313]
  │   ├─ AddGeneratedFlags() 在 workflow 步骤前插入 'generated' flag
  │   ├─ interpreter.setRunId()
  │   ├─ InterpretRecording()  600 秒超时  [:1118] / scheduler版 [:566]
  │   ├─ crawl/search 后处理: processRobotOutputFormats()
  │   ├─ Run.update(status:'success', ...)  [:1182] / scheduler版 [:658]
  │   └─ Socket + Webhook + triggerIntegrationUpdates()  [:1188-1308]
  │
  └─ 异常兜底 [:1315-1401]
      ├─ Run.update(status:'failed', log + stack)  [:1330-1334]
      ├─ Socket 'run-completed'(失败) + Webhook 'run_failed' + analytics
      └─ 清理浏览器
```

---

## 五、Run 状态机与状态更新归属

### 5.1 状态枚举与每条状态的设置位置（精确到文件和行）

状态定义：`server/src/models/Run.ts:69-72`

| 状态 | 含义 | 触发源/路径 | 精确设置位置 |
|------|------|------------|-------------|
| `scheduled` | Cron 创建 Run，等待浏览器就绪 | Cron 路径 C | `server/src/workflow-management/scheduler/index.ts:78` |
| `queued` | 浏览器槽位不足，排队等待 | 前端手动 ② | `server/src/routes/storage.ts:1104` |
| `running` | 浏览器就绪，执行中 | 前端手动(有槽) ② | `server/src/routes/storage.ts:1033` |
| | | API/SDK/CLI/MCP ③~⑤ | `server/src/api/record.ts:590` |
| | | 路径 A processRunExecution | `server/src/task-runner.ts:245` (scrape), `server/src/task-runner.ts:389` (工作流) |
| | | 路径 B api/executeRun | `server/src/api/record.ts:801-804` (scrape) |
| | | 路径 C scheduler/executeRun | `server/src/workflow-management/scheduler/index.ts:273` (scrape), `:566` (工作流) |
| `success` | 执行成功 | 路径 A | `server/src/task-runner.ts:335` (scrape), `:469` (工作流) |
| | | 路径 B | `server/src/api/record.ts:961` (scrape), `:1182` (工作流) |
| | | 路径 C | `server/src/workflow-management/scheduler/index.ts:436` (scrape), `:658` (工作流) |
| `failed` | 执行失败（含超时） | 所有路径 | 各 catch 块 (例: `server/src/task-runner.ts:541-550`, `server/src/api/record.ts:1330-1334`) |
| `aborting` | 用户请求中止，正在清理 | 前端手动 / 队列 | `server/src/routes/storage.ts:1438`, `server/src/task-runner.ts:592` |
| `aborted` | 中止已完成 | 所有路径 | `server/src/task-runner.ts:602-607`, `server/src/routes/storage.ts:1443` |

### 5.2 五路触发对应的初始状态流转

```
① Cron 非文档:  scheduled ──socket──▶ running ──┐
① Cron 文档:    scheduled ──EXECUTE_RUN──▶ running ──┐
                                                    │
② 前端手动(有槽): running ──EXECUTE_RUN──────────────┤
② 前端手动(无槽): queued ──processQueuedRuns()──▶ running ──┐
                                                            │
③~⑤ 外部调用 非文档: running ──socket 直调──▶ running ──┐   │
③~⑤ 外部调用 文档:   running ──EXECUTE_RUN──────▶ running ──┐
                                                            │
                                                            ▼
                                          ┌─────────────────────────────┐
                                          │      running (执行中)       │
                                          └──┬──────────┬───────────┬──┘
                                             ▼          ▼           ▼
                                        success     failed    aborting→aborted
```

### 5.3 每次状态更新触发的三类通知

| 通知类型 | 设置位置示例 | 事件/载荷 |
|---------|------------|----------|
| **Socket.io** | `server/src/api/record.ts:625`, `server/src/task-runner.ts:428-433` | 命名空间 `/queued-run` → 房间 `user-${userId}` <br>事件: `run-scheduled` / `run-started` / `run-completed` / `run-aborted` |
| **Webhook** | `server/src/task-runner.ts:459-465`, `server/src/api/record.ts:1016-1027` | 事件: `run_completed` / `run_failed` <br>载荷: run_id, status, extracted_data, metadata 等 |
| **Integration** | `server/src/task-runner.ts:493-498`, `server/src/api/record.ts:665-689` | Google Sheets: `processGoogleSheetUpdates()` (65秒超时) <br>Airtable: `processAirtableUpdates()` (65秒超时) |

---

## 六、Graphile Worker 队列层：分流的核心枢纽

### 6.1 队列定义

`server/src/task-runner.ts:37-45`
```ts
export const QUEUE_NAMES = {
  INITIALIZE_BROWSER_RECORDING: 'initialize-browser-recording',
  DESTROY_BROWSER:            'destroy-browser',
  INTERPRET_WORKFLOW:         'interpret-workflow',
  STOP_INTERPRETATION:        'stop-interpretation',
  EXECUTE_RUN:                'execute-run',          // 前端手动 + 所有文档机器人
  ABORT_RUN:                  'abort-run',
  SCHEDULED_WORKFLOW:         'scheduled-workflow',   // Cron 专用（非文档机器人走socket，文档机器人二次入队EXECUTE_RUN）
} as const;
```

### 6.2 addJob 封装

`server/src/storage/graphileWorker.ts:61-71`
```ts
export async function addJob(
  taskIdentifier: string,                    // 对应 QUEUE_NAMES
  payload: Record<string, unknown>,
  options?: {
    maxAttempts?: number;                    // Cron用6，一次性用1
    runAt?: Date;
    jobKey?: string;
  }
): Promise<string>
```

### 6.3 五路触发 → 入队对照表（最终版，已核对）

| 触发源 | 机器人类型 | 是否入队 | 队列名 | maxAttempts | 精确入队位置 |
|--------|----------|---------|--------|-------------|------------|
| ① Cron | 非文档 | ✅ | `SCHEDULED_WORKFLOW` | **6** | `server/src/schedule-worker.ts:143` |
| ① Cron | 文档 | ✅ 两次: 先 SCHEDULED_WORKFLOW, 二次 EXECUTE_RUN | `SCHEDULED_WORKFLOW` → `EXECUTE_RUN` | 6 → 1 | 二次入队: `server/src/workflow-management/scheduler/index.ts:86-89` |
| ② 前端手动(有槽) | 任意 | ✅ | `EXECUTE_RUN` | **1** | `server/src/routes/storage.ts:1069` |
| ② 前端手动(排队后) | 任意 | ✅ | `EXECUTE_RUN` | **1** | `server/src/routes/storage.ts:1539` |
| ③ API | 文档 | ✅ | `EXECUTE_RUN` | **1** | `server/src/api/record.ts:632-636` |
| ③ API | 非文档 | ❌ **不走队列，socket直调** | - | - | - |
| ④ SDK / CLI | 文档 | ✅ | `EXECUTE_RUN` | **1** | 同 API (共用 handleRunRecording) |
| ④ SDK / CLI | 非文档 | ❌ **不走队列，socket直调** | - | - | - |
| ⑤ MCP | 文档 | ✅ | `EXECUTE_RUN` | **1** | 同 API (共用 handleRunRecording) |
| ⑤ MCP | 非文档 | ❌ **不走队列，socket直调** | - | - | - |

### 6.4 队列 → 处理器映射

`server/src/task-runner.ts:660-674`
```ts
const taskList = {
  // ... 其他队列
  [QUEUE_NAMES.EXECUTE_RUN]: async (payload) => {
    await processRunExecution(payload as ExecuteRunData);  // 路径 A
  },
  [QUEUE_NAMES.SCHEDULED_WORKFLOW]: async (payload) => {
    const data = payload as ScheduledWorkflowData;
    // import 来自 scheduler/index.ts（仅2个参数，无 runSource）
    await handleRunRecording(data.robotMetaId, data.userId);  // Cron 路径 C
  },
}
```

Worker 启动：`server/src/task-runner.ts:681-716`
- concurrency = `max(1, parseInt(env.WORKER_CONCURRENCY || '10'))`
- pollInterval = `3_600_000`（1小时，主要依赖 LISTEN/NOTIFY 实时通知）

---

## 七、故障恢复与保护机制

### 7.1 崩溃恢复：recoverOrphanedRuns

`server/src/routes/storage.ts:1572-1643`

服务启动时调用，处理 `status IN ('running', 'scheduled')` 的孤立 Run：
```
条件: status IN ('running','scheduled') AND browserPool 中无对应 browser
  ├─ retryCount < 3:
  │   Run.update({ status: 'queued', retryCount++, browserId: undefined,
  │                log: '[RETRY N/3] Re-queuing due to server crash' })
  └─ retryCount >= 3:
      Run.update({ status: 'failed', log: 'Max retries exceeded...' })
```

### 7.2 Cron 认领超时

`server/src/schedule-worker.ts:62-65`
```ts
claimExpiry = NOW() - 10 * 60 * 1000   // 10 分钟
WHERE schedulerClaimedAt IS NULL OR schedulerClaimedAt < claimExpiry
```
> 调度器崩溃 10 分钟后，认领标记自动失效，下次轮询重新认领。

### 7.3 排队任务熔断器

`server/src/routes/storage.ts:1507-1515`
```ts
consecutiveDbErrors >= 3
  → circuitBreakerOpenUntil = NOW() + 30_000   // 冷却 30 秒
```

### 7.4 多层超时保护

| 超时项 | 位置 | 值 |
|-------|------|-----|
| 浏览器初始化 | `server/src/task-runner.ts:183` | 60 秒 |
| Page 就绪 | `server/src/task-runner.ts:219` | 15 秒 |
| 工作流 Interpret | `server/src/task-runner.ts:405` | 600 秒（10分钟） |
| Scrape 格式转换 | `server/src/api/record.ts:818` | 每个 120 秒 |
| Integration 更新 | `server/src/api/record.ts:681-685` | 每个 65 秒 |
| Socket 连接 | `server/src/api/record.ts:1423` | 30 秒 |

---

## 八、关键数据模型

### 8.1 Robot.schedule（JSONB）

定义：`server/src/models/Robot.ts:61-73`

| 字段 | 写入位置 | 读取位置 |
|------|---------|---------|
| `cronExpression` | `server/src/routes/storage.ts:1225-1342` (PUT /schedule/:id/) <br> `server/src/api/sdk.ts:528-554` (SDK 设置调度) | `server/src/schedule-worker.ts:60` |
| `nextRunAt` | `server/src/storage/schedule.ts:15` (scheduleWorkflow) <br> `server/src/schedule-worker.ts:101` (finalizeSchedule) | `server/src/schedule-worker.ts:61` |
| `schedulerClaimedAt` | 认领时 `server/src/schedule-worker.ts:79` <br> 清零时 `server/src/schedule-worker.ts:105` | `server/src/schedule-worker.ts:62-65` |
| `timezone` | 同 cronExpression 写入 | `server/src/utils/schedule.ts:8` (computeNextRun) |

### 8.2 Run 核心字段

定义：`server/src/models/Run.ts:14-35`

| 字段 | 设置位置（五路触发差异） |
|------|----------------------|
| `status` | Cron→`scheduled`; 前端(有槽)→`running`; 前端(无槽)→`queued`; API/SDK/CLI/MCP→`running` |
| `runByScheduleId` | 仅 Cron: `server/src/workflow-management/scheduler/index.ts:83` |
| `runByAPI` | API 时为 true, MCP 时为 false: `server/src/api/record.ts:601` |
| `runBySDK` | SDK 时为 true: `server/src/api/record.ts:602` |
| `runByMCP` | MCP 时为 true: `server/src/api/record.ts:603` |
| `runByCLI` | CLI 时为 true: `server/src/api/record.ts:604` |
| `runByAPI` (SDK/CLI/MCP 时) | 均为 false |
| `retryCount` | 崩溃恢复递增: `server/src/routes/storage.ts:1601` |

---

## 九、分流与汇合的四个关键节点总结

```
节点 1 [入队前最大分叉] ───────────────────────────────────────────┐
                                                                   │
  五路触发源分成三种模式：                                          │
  · Cron 非文档          → SCHEDULED_WORKFLOW 队列 (重试 6 次)      │
  · 前端手动 + 所有文档机器人 → EXECUTE_RUN 队列 (不重试)            │
  · API/SDK/CLI/MCP 非文档 → 完全不经过队列，socket 直调 executeRun  │
                                                                   │
节点 2 [文档机器人汇合点] ───────────────────────────────────────────┤
                                                                   │
  无论来自哪路触发（Cron / 前端 / API / SDK / CLI / MCP），            │
  文档机器人最终都入队 EXECUTE_RUN，                                 │
  统一由 processRunExecution() 的 doc-extract/doc-parse 分支处理。    │
                                                                   │
节点 3 [执行函数分叉] ──────────────────────────────────────────────┤
                                                                   │
  虽然执行逻辑高度相似，但有三份独立实现：                            │
  · processRunExecution  task-runner.ts:130-582   队列路径          │
  · executeRun (api版)  api/record.ts:732-1401   外部调用非文档      │
  · executeRun (scheduler版) scheduler/index.ts:186-832 Cron非文档  │
                                                                   │
节点 4 [状态更新统一出口] ──────────────────────────────────────────┘

  无论走哪条路径，成功/失败状态更新时都触发相同的三类通知：
  Socket.io → 前端实时推送
  Webhook → 外部系统回调
  Integration → GoogleSheet / Airtable 更新
```
