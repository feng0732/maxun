# 调度器完整链路分析：Cron / 手动 / SDK / MCP / CLI 五路分流

---

## 一、统一链路全景图

```
                               ┌──────────────────────────────────────────────────────────┐
                               │                      触发源 (5 路)                         │
                               └────────────┬───────────────────────┬───────────────────────┘
                                            │                       │
                ┌───────────────────────────┘                       └──────────────────┐
                │                                                                        │
     ┌──────────▼──────────┐                                             ┌──────────────▼──────────────┐
     │ ① Cron 定时调度器    │                                             │ ②~⑤ 外部调用（一次性任务）    │
     │ schedule-worker.ts   │                                             │ API / SDK / CLI / MCP / 前端  │
     └──────────┬──────────┘                                             └──────────────┬──────────────┘
                │                                                                       │
                ▼                                                                       ▼
  addJob(SCHEDULED_WORKFLOW,                                                   ┌────────┴─────────┐
         {maxAttempts:6})                                                      │                  │
                │                                                    ┌───────▼──────┐    ┌──────▼──────────┐
                ▼                                                    │ 前端手动触发  │    │ API / SDK /    │
 ┌──────────────────────────────┐                                     │ routes/       │    │ CLI / MCP       │
 │ task-runner.ts               │                                     │ storage.ts     │    │ (api/*.ts)       │
 │ SCHEDULED_WORKFLOW 处理器    │                                     │ PUT /runs/:id │    └──────┬──────────┘
 │ → scheduler/index.ts         │                                     └───────┬──────┘           │
 │   handleRunRecording()       │                                             │                  │
 └──────────────┬───────────────┘                                             │                  │
                │                                                             ▼                  ▼
                │                                                addJob(EXECUTE_RUN,       handleRunRecording()
                │                                                {maxAttempts:1})        api/record.ts#L1403
                │                                                             │                  │
                │                                                             │   ┌──────────────┘
                │                                                             │   │
                │                                                             ▼   ▼
                │                                              ┌──────────────────────────────┐
                │                                              │  Graphile Worker 队列层       │
                │                                              │  · QUEUE: EXECUTE_RUN         │
                │                                              │  · QUEUE: SCHEDULED_WORKFLOW  │
                │                                              └──────────────┬───────────────┘
                │                                                             │
                │                                                             ▼
                │                                              ┌──────────────────────────────┐
                │                                              │ task-runner.ts                │
                │                                              │ EXECUTE_RUN 处理器            │
                │                                              │ → processRunExecution()      │
                │                                              └──────────────┬───────────────┘
                │                                                             │
                └──────────────────────┬──────────────────────────────────────┘
                                       ▼
                       ┌──────────────────────────────────────┐
                       │           执行 + 状态更新              │
                       │  scheduled/queued → running →        │
                       │  success / failed / aborting /       │
                       │  aborted                              │
                       └──────────────┬───────────────────────┘
                                      ▼
                       ┌──────────────────────────────────────┐
                       │        通知 & 后处理                   │
                       │  Socket.io → 前端                      │
                       │  Webhook → 外部系统                    │
                       │  Integration → GoogleSheet / Airtable │
                       └──────────────────────────────────────┘
```

---

## 二、五路触发源详解（含代码定位）

### 2.1 触发源总览表

| # | 触发方式 | 入口文件与函数 | Run 标记字段 | 入队队列 | 初始状态 |
|---|---------|--------------|-------------|---------|---------|
| ① | **Cron 定时** | [schedule-worker.ts#L156-L177](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/schedule-worker.ts#L156-L177) `processDueSchedules()` | `runByScheduleId` | `SCHEDULED_WORKFLOW` | `scheduled` |
| ② | **前端手动** | [routes/storage.ts#L997-L1131](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/routes/storage.ts#L997-L1131) `PUT /runs/:id` | - | `EXECUTE_RUN` | `running` / `queued` |
| ③ | **REST API** | [api/record.ts#L1589-L1620](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/record.ts#L1589-L1620) `POST /api/robots/:id/runs` | `runByAPI: true` | **不直接入队** | `running` |
| ④ | **SDK / CLI** | [api/sdk.ts#L682-L801](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/sdk.ts#L682-L801) `POST /api/sdk/robots/:id/execute` | `runBySDK: true` / `runByCLI: true` | **不直接入队** | `running` |
| ⑤ | **MCP Worker** | [mcp-worker.ts#L113-L182](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/mcp-worker.ts#L113-L182) `run_robot` 工具 → 内部调用 `POST /api/robots/:id/runs` 带 `x-run-source: mcp` | `runByMCP: true` | **不直接入队** | `running` |

> **关键分流点**：③~⑤ 虽然是不同入口，但最终都汇入 [api/record.ts#L1403-L1458](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/record.ts#L1403-L1458) 的 `handleRunRecording()`；而 ① Cron 走的是 [scheduler/index.ts#L860-L909](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/workflow-management/scheduler/index.ts#L860-L909) 的 **同名但不同文件** 的 `handleRunRecording()`；② 前端手动不走任何 `handleRunRecording`，直接入队 `EXECUTE_RUN`。

---

### 2.2 触发源 ①：Cron 定时调度（schedule-worker）

**步骤 1：服务启动时注册轮询** [schedule-worker.ts#L156-L177](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/schedule-worker.ts#L156-L177)

```ts
export function startScheduleWorker() {
  setInterval(() => processDueSchedules(), 30_000);  // 每 30 秒轮询一次
  processDueSchedules();  // 启动时立即执行一次
}
```

**步骤 2：DB 认领（分布式锁）** [schedule-worker.ts#L29-L85](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/schedule-worker.ts#L29-L85) `claimDueDbSchedules()`

```ts
// 条件：cronExpression 存在 + nextRunAt <= NOW + 认领未超时（10分钟）
// 锁：pg_try_advisory_xact_lock(43821742) + SELECT ... FOR UPDATE SKIP LOCKED
// 标记：schedule.schedulerClaimedAt = NOW()
// 批量：最多 BATCH_SIZE = 10
```

**步骤 3：入队 SCHEDULED_WORKFLOW** [schedule-worker.ts#L122-L154](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/schedule-worker.ts#L122-L154) `processDueSchedules()`

```ts
addJob(QUEUE_NAMES.SCHEDULED_WORKFLOW,
  { robotMetaId, userId },
  { maxAttempts: 6 }   // Cron 失败最多重试 6 次
)
```

**步骤 4：finalizeSchedule 计算下次运行** [schedule-worker.ts#L87-L108](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/schedule-worker.ts#L87-L108)

```ts
computeNextRun(cronExpression, timezone)  // 基于 cron-parser
→ 更新 Robot: nextRunAt, lastRunAt, schedulerClaimedAt=NULL
```

**步骤 5：SCHEDULED_WORKFLOW 任务处理器** [task-runner.ts#L670-L674](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/task-runner.ts#L670-L674)

```ts
[QUEUE_NAMES.SCHEDULED_WORKFLOW]: (job) =>
  handleRunRecording(job.payload.robotMetaId, String(job.payload.userId))
// 调用 scheduler/index.ts 的 handleRunRecording
```

**步骤 6：Cron 专用 handleRunRecording** [scheduler/index.ts#L860-L909](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/workflow-management/scheduler/index.ts#L860-L909)

```ts
handleRunRecording(id, userId)
  ├─ createWorkflowAndStoreMetadata(id, userId)
  │   ├─ Robot.findOne()  查询配置
  │   ├─ browserId = isDocRobot ? uuid() : createRemoteBrowserForRun(userId)
  │   ├─ Run.create({ status: 'scheduled', ..., runByScheduleId: scheduleId })
  │   ├─ Socket 'run-scheduled' → /queued-run user-${userId}
  │   └─ 文档机器人二次入队:
  │        addJob(EXECUTE_RUN, {...}, {maxAttempts:1})
  │
  └─ 非文档机器人:
       socket.connect(/${browserId})
       on('ready-for-run') → executeRun(runId, userId)
```

---

### 2.3 触发源 ②：前端手动触发（routes/storage.ts）

**入口** [routes/storage.ts#L997-L1131](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/routes/storage.ts#L997-L1131) `PUT /runs/:id`

核心是 **浏览器槽位检查 + 分支处理**：

```ts
if (hasAvailableBrowserSlots(userId, "run")) {
  // 路径 A：有浏览器槽位 → 立即执行
  browserId = createRemoteBrowserForRun(userId)
  Run.create({ status: 'running', browserId, ... })
  addJob(QUEUE_NAMES.EXECUTE_RUN,             // 直接入队 EXECUTE_RUN
    { userId, runId, browserId },
    { maxAttempts: 1 }                         // 不重试
  )
} else {
  // 路径 B：无槽位 → 排队
  browserId = uuid()  // 占位 ID
  Run.create({ status: 'queued',
               log: 'Run queued - waiting for available browser slot', ... })
  // 不入队，由 processQueuedRuns() 定时轮询处理
}
```

**排队任务后续处理** [routes/storage.ts#L1493-L1566](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/routes/storage.ts#L1493-L1566) `processQueuedRuns()`

```ts
// 服务启动时 setInterval 注册，定时轮询
// 熔断器：连续 3 次 DB 错误 → 冷却 30 秒
Run.findOne({ status: 'queued' }, ORDER BY startedAt ASC)
  → 再次检查 hasAvailableBrowserSlots
     → 有槽位: Run.update(status:'running') + addJob(EXECUTE_RUN)
     → 无槽位: 跳过，下次再试
```

---

### 2.4 触发源 ③④⑤：API / SDK / CLI / MCP（统一汇入 handleRunRecording）

#### 2.4.1 各入口的调用关系

```
触发源 ⑤ MCP Worker (mcp-worker.ts#L113-L182 run_robot 工具)
  │  HTTP POST /api/robots/:id/runs  带 header: x-run-source: 'mcp'
  ▼
触发源 ③ REST API (api/record.ts#L1589-L1620 POST /api/robots/:id/runs)
  │  runSource = headers['x-run-source']==='mcp' ? 'mcp' : 'api'
  │
  └──────────────────────┐
                         ▼
触发源 ④ SDK/CLI (api/sdk.ts#L682-L801 POST /api/sdk/robots/:id/execute)
  │  runSource = headers['x-run-source']==='cli' ? 'cli' : 'sdk'
  │
  ▼
统一入口：api/record.ts#L1403-L1458  handleRunRecording(id, userId, runSource)
  │
  ├─ runSource 值: 'api' | 'sdk' | 'mcp' | 'cli'
  │
  └─ → createWorkflowAndStoreMetadata()  写入对应 runBy* 标记
```

#### 2.4.2 统一入口：handleRunRecording（api/record.ts 版）

[api/record.ts#L1403-L1458](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/record.ts#L1403-L1458)

```ts
export async function handleRunRecording(
  id: string, userId: string,
  runSource: 'api' | 'sdk' | 'mcp' | 'cli' = 'api',
  requestedFormats?, promptInstructions?
) {
  // Step 1: 创建 Run 记录 + 浏览器
  const { browserId, runId, isDocRobot } = await createWorkflowAndStoreMetadata(
    id, userId, runSource, requestedFormats, promptInstructions
  )

  if (isDocRobot) return runId;  // 文档机器人: 已在 createWorkflow... 内入队 EXECUTE_RUN

  // Step 2: 非文档机器人 - 用 Socket 等待浏览器就绪
  socket = io(BACKEND_URL/${browserId})
  socket.on('ready-for-run', () =>
    readyForRunHandler(browserId, runId, userId, socket)
    // → 内部调用 executeRun(runId, userId)
  )
}
```

#### 2.4.3 Run 创建：createWorkflowAndStoreMetadata（api/record.ts 版）

[api/record.ts#L552-L654](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/record.ts#L552-L654)

```ts
async function createWorkflowAndStoreMetadata(
  id, userId,
  runSource: 'api' | 'sdk' | 'mcp' | 'cli',    // ← 标记来源
  requestedFormats?, promptInstructions?
) {
  const browserId = isDocRobot ? uuid() : createRemoteBrowserForRun(userId)
  const run = await Run.create({
    status: 'running',                           // ← 初始状态直接 running
    interpreterSettings: { formats, promptInstructions, ... },
    runByAPI: runSource === 'api',              // ← 四路标记
    runBySDK: runSource === 'sdk',
    runByMCP: runSource === 'mcp',
    runByCLI: runSource === 'cli',
    ...
  })
  serverIo.of('/queued-run').to('user-'+userId).emit('run-started', ...)

  if (isDocRobot) {
    // 文档机器人也入队 EXECUTE_RUN，与前端手动触发合并
    addJob(QUEUE_NAMES.EXECUTE_RUN, { userId, runId, browserId }, { maxAttempts: 1 })
  }
  return { browserId, runId, isDocRobot }
}
```

#### 2.4.4 readyForRunHandler → executeRun

[api/record.ts#L691-L713](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/record.ts#L691-L713)

```ts
async function readyForRunHandler(browserId, id, userId, socket) {
  const result = await executeRun(id, userId)  // ← api/record.ts 内部的 executeRun
  // 成功后: resetRecordingState + 清理 socket
  // 失败后: destroyRemoteBrowser + 清理 socket
}
```

---

## 三、执行层：两条执行路径最终合一

### 3.1 执行路径对照

| 路径 | 触发队列 | 执行函数 | 适用场景 |
|------|---------|---------|---------|
| **路径 A** | `EXECUTE_RUN` | [task-runner.ts#L130-L582](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/task-runner.ts#L130-L582) `processRunExecution()` | 前端手动触发 + API/SDK/CLI/MCP 的文档机器人 + Cron 的文档机器人 |
| **路径 B** | `SCHEDULED_WORKFLOW`（后走Socket） | [api/record.ts#L732-L1401](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/record.ts#L732-L1401) `executeRun()`（api版） | API/SDK/CLI/MCP 的非文档机器人 |
| **路径 C** | `SCHEDULED_WORKFLOW`（后走Socket） | [scheduler/index.ts#L186-L832](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/workflow-management/scheduler/index.ts#L186-L832) `executeRun()`（scheduler版） | Cron 触发的非文档机器人 |

> **注意**：路径 B 和路径 C 的两个 `executeRun()` 函数 **内容高度相似**（scrape 处理、工作流执行、格式转换、通知发送），但它们分属不同文件。

### 3.2 路径 A：processRunExecution（EXECUTE_RUN 队列处理器）

[task-runner.ts#L130-L582](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/task-runner.ts#L130-L582)

```
processRunExecution(job)
  ├─ 前置检查 [L136-L179]
  │   ├─ Run 状态: aborted/aborting → skip; queued → skip(由恢复处理)
  │   └─ 机器人类型: doc-extract → executeDocumentRun; doc-parse → executeDocumentParseRun
  │
  ├─ 浏览器等待 [L181-L213]
  │   ├─ browserPool.getRemoteBrowser(browserId)  最多60秒
  │   └─ getCurrentPage()  最多15秒
  │
  ├─ 分支: scrape 类型 [L239-L379]
  │   ├─ Run.update(status:'running') + Socket 'run-started'
  │   ├─ 按顺序转换格式 (每个120秒超时):
  │   │    screenshot-visible/fullpage → text → markdown → summary(LLM) →
  │   │    html → links → promptInstructions(BrowserAgent LLM)
  │   ├─ Run.update(status:'success', serializableOutput, binaryOutput)
  │   ├─ BinaryOutputService 上传 MinIO
  │   ├─ Socket 'run-completed' + Webhook
  │   └─ destroyRemoteBrowser()
  │
  ├─ 分支: extract/crawl/search 工作流类型 [L381-L555]
  │   ├─ Run.update(status:'running') + Socket 'run-started'
  │   ├─ interpreter.setRunId(runId)  实时持久化
  │   ├─ InterpretRecording()  600秒超时(10分钟)
  │   ├─ 期间轮询 isRunAborted() → 若中止则提前返回
  │   ├─ crawl/search 后处理: processRobotOutputFormats()
  │   ├─ Run.update(status:'success', serializableOutput + binaryOutput)
  │   ├─ 二进制上传 + Socket + Webhook + triggerIntegrationUpdates()
  │   └─ destroyRemoteBrowser()
  │
  └─ 异常兜底 [L512-L581]
      ├─ 有部分数据 → triggerIntegrationUpdates()
      ├─ Run.update(status:'failed', log)
      ├─ Socket 'run-completed'(失败) + Webhook run_failed
      └─ 清理浏览器
```

### 3.3 路径 B/C：executeRun（Socket 回调路径）

以 [api/record.ts#L732-L1401](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/record.ts#L732-L1401) 为例，scheduler/index.ts 版逻辑几乎一致：

```
executeRun(id, userId)
  ├─ 前置检查 [L736-L785]
  │   ├─ Run 状态: aborted/aborting → skip; queued → skip; retryCount>=3 → 永久失败
  │   ├─ browserPool.getRemoteBrowser() + getCurrentPage()
  │
  ├─ 分支 scrape 类型 [L789-L1110]
  │   └─ (与 processRunExecution 的 scrape 分支完全一致: 格式转换 + 存储 + 通知)
  │
  ├─ 分支 extract/crawl/search 工作流类型 [L1112-L1313]
  │   ├─ AddGeneratedFlags() 在每个 workflow 步骤前插入 'generated' flag
  │   ├─ interpreter.setRunId()
  │   ├─ InterpretRecording()  600秒超时
  │   ├─ crawl/search 后处理: processRobotOutputFormats()
  │   ├─ BinaryOutputService 上传
  │   ├─ Run.update(status:'success', ...)
  │   └─ Socket + Webhook + triggerIntegrationUpdates()
  │
  └─ 异常兜底 [L1315-L1401]
      ├─ Run.update(status:'failed', log + stack)
      ├─ Socket 'run-completed'(失败) + Webhook run_failed + analytics
      └─ 清理浏览器
```

---

## 四、Run 状态机与状态更新（含代码定位）

### 4.1 状态枚举与触发位置

状态定义：[models/Run.ts#L69-L72](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/models/Run.ts#L69-L72)（实际使用值见下表）

| 状态 | 含义 | 设置位置（文件#行号） |
|------|------|---------------------|
| `scheduled` | Cron 创建 Run，等待浏览器就绪 | [scheduler/index.ts#L78](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/workflow-management/scheduler/index.ts#L78) |
| `queued` | 浏览器槽位不足，排队 | [routes/storage.ts#L1104](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/routes/storage.ts#L1104) |
| `running` | 浏览器就绪，执行中 | [routes/storage.ts#L1033](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/routes/storage.ts#L1033) / [api/record.ts#L590](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/record.ts#L590) / [task-runner.ts#L245](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/task-runner.ts#L245) |
| `success` | 执行成功 | 多处: [task-runner.ts#L335](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/task-runner.ts#L335), [api/record.ts#L961](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/record.ts#L961), [scheduler/index.ts#L436](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/workflow-management/scheduler/index.ts#L436) |
| `failed` | 执行失败（含超时） | 多处 catch 块 |
| `aborting` | 用户请求中止，正在清理 | [routes/storage.ts#L1438](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/routes/storage.ts#L1438) / [task-runner.ts#L592](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/task-runner.ts#L592) |
| `aborted` | 中止已完成 | [task-runner.ts#L602-L607](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/task-runner.ts#L602-L607) / [routes/storage.ts#L1443](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/routes/storage.ts#L1443) |

### 4.2 五路触发对应的初始状态流转

```
① Cron 非文档: scheduled ─────┐
① Cron 文档:   scheduled ──EXECUTE_RUN──▶ running ──┐
                                                 │
② 前端手动(有槽): running ──EXECUTE_RUN───────────┤
② 前端手动(无槽): queued ──processQueuedRuns()──▶ running ──┐
                                                           │
③ API / ④ SDK / ⑤ MCP 非文档: running ──socket ready──▶ running ──┐
③ API / ④ SDK / ⑤ MCP 文档:     running ──EXECUTE_RUN────▶ running ──┐
                                                                    │
                                                                    ▼
                                          ┌─────────────────────────────┐
                                          │      running (执行中)       │
                                          └──┬──────────┬───────────┬──┘
                                             ▼          ▼           ▼
                                        success     failed    aborting→aborted
```

### 4.3 每次状态更新时触发的通知

每次状态更新都会同步触发三类通知（可在对应代码行复核）：

| 通知类型 | 代码行示例 | 事件/载荷 |
|---------|----------|----------|
| **Socket.io** | [api/record.ts#L625](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/record.ts#L625), [task-runner.ts#L428-L433](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/task-runner.ts#L428-L433) | 命名空间 `/queued-run` → 房间 `user-${userId}` <br>事件: `run-scheduled` / `run-started` / `run-completed` / `run-aborted` |
| **Webhook** | [task-runner.ts#L459-L465](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/task-runner.ts#L459-L465), [api/record.ts#L1016-L1027](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/record.ts#L1016-L1027) | 事件: `run_completed` / `run_failed` <br>载荷: run_id, status, extracted_data, metadata 等 |
| **Integration** | [task-runner.ts#L493-L498](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/task-runner.ts#L493-L498), [api/record.ts#L665-L689](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/record.ts#L665-L689) | Google Sheets: `processGoogleSheetUpdates()` (65秒超时) <br>Airtable: `processAirtableUpdates()` (65秒超时) |

---

## 五、Graphile Worker 队列层（分流的核心枢纽）

### 5.1 队列定义

[task-runner.ts#L37-L45](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/task-runner.ts#L37-L45)

```ts
const QUEUE_NAMES = {
  INITIALIZE_BROWSER_RECORDING: 'initialize-browser-recording',
  DESTROY_BROWSER:            'destroy-browser',
  INTERPRET_WORKFLOW:         'interpret-workflow',
  STOP_INTERPRETATION:        'stop-interpretation',
  EXECUTE_RUN:                'execute-run',          // ← 前端手动 + 文档机器人（所有触发源）
  ABORT_RUN:                  'abort-run',
  SCHEDULED_WORKFLOW:         'scheduled-workflow',   // ← Cron 定时专用
} as const;
```

### 5.2 addJob 封装

[storage/graphileWorker.ts#L61-L71](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/storage/graphileWorker.ts#L61-L71)

```ts
export async function addJob(
  taskIdentifier: string,                    // 对应 QUEUE_NAMES
  payload: Record<string, unknown>,
  options?: {
    maxAttempts?: number;                    // Cron 用 6，一次性用 1
    runAt?: Date;                            // 延迟执行
    jobKey?: string;                         // 去重键
  }
): Promise<string>  // 返回 job.id
```

### 5.3 五路触发 → 入队对照

| 触发源 | 最终入队的 QUEUE | maxAttempts | 入队代码行 |
|--------|-----------------|-------------|----------|
| ① Cron（非文档） | `SCHEDULED_WORKFLOW` | **6** | [schedule-worker.ts#L143](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/schedule-worker.ts#L143) |
| ① Cron（文档二次入队） | `EXECUTE_RUN` | **1** | [scheduler/index.ts#L86-L89](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/workflow-management/scheduler/index.ts#L86-L89) |
| ② 前端手动（有槽） | `EXECUTE_RUN` | **1** | [routes/storage.ts#L1041-L1045](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/routes/storage.ts#L1041-L1045) |
| ② 前端手动（排队后） | `EXECUTE_RUN` | **1** | [routes/storage.ts#L1544-L1548](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/routes/storage.ts#L1544-L1548) |
| ③ API（非文档） | **不入队，socket 直接调用 executeRun** | - | [api/record.ts#L1414-L1433](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/record.ts#L1414-L1433) |
| ③ API（文档） | `EXECUTE_RUN` | **1** | [api/record.ts#L632-L636](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/record.ts#L632-L636) |
| ④ SDK/CLI | 同 API（共用 handleRunRecording） | - | 同上 |
| ⑤ MCP | 同 API（共用 handleRunRecording，runSource='mcp'） | - | 同上 |

### 5.4 Worker 启动配置

[task-runner.ts#L681-L716](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/task-runner.ts#L681-L716)

```ts
runner = await run({
  concurrency: TOTAL_CONCURRENCY,   // max(1, parseInt(env.WORKER_CONCURRENCY || '10'))
  pollInterval: 3_600_000,          // 1 小时（主要依赖 LISTEN/NOTIFY 实时通知）
  taskList,                         // 任务处理函数映射表: queueName → handler
});
```

---

## 六、故障恢复与保护机制（含代码定位）

### 6.1 崩溃恢复：recoverOrphanedRuns

[routes/storage.ts#L1572-L1643](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/routes/storage.ts#L1572-L1643)

服务启动时调用，处理 `status IN ('running', 'scheduled')` 的孤立 Run：

```
条件: status in ('running','scheduled') AND browserPool中无对应browser
  ├─ retryCount < 3:
  │   Run.update({
  │     status: 'queued',
  │     retryCount++,
  │     browserId: undefined,
  │     log: '[RETRY N/3] Re-queuing due to server crash'
  │   })
  └─ retryCount >= 3:
      Run.update({ status: 'failed', log: 'Max retries exceeded...' })
```

### 6.2 Cron 认领超时

[schedule-worker.ts#L62-L65](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/schedule-worker.ts#L62-L65)

```ts
claimExpiry = NOW() - 10 * 60 * 1000   // 10 分钟
WHERE: schedulerClaimedAt IS NULL OR schedulerClaimedAt < claimExpiry
```

> 调度器崩溃 10 分钟后，认领标记自动失效，下次轮询重新认领。

### 6.3 排队任务熔断器

[routes/storage.ts#L1507-L1515](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/routes/storage.ts#L1507-L1515)

```ts
consecutiveDbErrors >= 3
  → circuitBreakerOpenUntil = NOW() + 30_000   // 冷却 30 秒
```

### 6.4 多层超时保护

| 超时项 | 代码行 | 值 |
|-------|-------|-----|
| 浏览器初始化 | [task-runner.ts#L183](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/task-runner.ts#L183) | 60 秒 |
| Page 就绪 | [task-runner.ts#L219](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/task-runner.ts#L219) | 15 秒 |
| 工作流 Interpret | [task-runner.ts#L405](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/task-runner.ts#L405) | 600 秒（10分钟） |
| Scrape 格式转换 | [api/record.ts#L818](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/record.ts#L818) | 每个 120 秒 |
| Integration 更新 | [api/record.ts#L681-L685](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/record.ts#L681-L685) | 每个 65 秒 |
| Socket 连接 | [api/record.ts#L1423](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/record.ts#L1423) | 30 秒 |

---

## 七、关键数据模型字段对照（含代码定位）

### 7.1 Robot.schedule（JSONB）

定义: [models/Robot.ts#L61-L73](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/models/Robot.ts#L61-L73)

| 字段 | 类型 | 写入位置 | 读取位置 |
|------|------|---------|---------|
| `cronExpression` | string | [routes/storage.ts#L1225-L1342](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/routes/storage.ts#L1225-L1342) `PUT /schedule/:id/` / [api/sdk.ts#L528-L554](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/sdk.ts#L528-L554) | [schedule-worker.ts#L60](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/schedule-worker.ts#L60) |
| `nextRunAt` | Date | [storage/schedule.ts#L15](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/storage/schedule.ts#L15) `scheduleWorkflow()` / [schedule-worker.ts#L101](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/schedule-worker.ts#L101) `finalizeSchedule()` | [schedule-worker.ts#L61](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/schedule-worker.ts#L61) |
| `schedulerClaimedAt` | Date | [schedule-worker.ts#L79](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/schedule-worker.ts#L79) 认领时 / [schedule-worker.ts#L105](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/schedule-worker.ts#L105) finalize时清零 | [schedule-worker.ts#L62-L65](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/schedule-worker.ts#L62-L65) |
| `timezone` | string | 同上 cronExpression 写入 | [utils/schedule.ts#L8](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/utils/schedule.ts#L8) `computeNextRun()` |

### 7.2 Run 核心字段

定义: [models/Run.ts#L14-L35](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/models/Run.ts#L14-L35)

| 字段 | 设置位置（五路触发各有不同） |
|------|---------------------------|
| `status` | Cron → `scheduled`; 前端(有槽) → `running`; 前端(无槽) → `queued`; API/SDK/CLI/MCP → `running` |
| `runByScheduleId` | 仅 Cron: [scheduler/index.ts#L83](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/workflow-management/scheduler/index.ts#L83) |
| `runByAPI` | API/MCP: [api/record.ts#L601](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/record.ts#L601)（MCP 时为 false） |
| `runBySDK` | SDK: [api/record.ts#L602](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/record.ts#L602) |
| `runByMCP` | MCP: [api/record.ts#L603](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/record.ts#L603) |
| `runByCLI` | CLI: [api/record.ts#L604](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/api/record.ts#L604) |
| `retryCount` | 崩溃恢复时递增: [routes/storage.ts#L1601](file:///d:/fz/0601-2/solo-dogfeeding/code/111-maxun/server/src/routes/storage.ts#L1601) |

---

## 八、分流总览：从触发到入队到执行的完整路径

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          触发源 (5 路进入)                                    │
├──────────┬──────────┬──────────┬──────────┬─────────────────────────────────┤
│ ① Cron   │ ② 前端  │ ③ API    │ ④ SDK    │ ⑤ MCP (mcp-worker.ts)          │
│ schedule │ 手动    │ /api/    │ /api/sdk │ HTTP: POST /api/robots/:id/runs │
│ -worker  │ PUT /   │ robots/  │ /robots/ │ x-run-source: 'mcp'             │
│          │ runs/:id│ :id/runs │ :id/exec │                                 │
└────┬─────┴────┬────┴────┬────┴────┬────┴────────────┬────────────────────┘
     │          │         │         │                 │
     ▼          ▼         ▼         ▼                 ▼
 SCHEDULED_  直接入队    handleRunRecording (api/record.ts#L1403)
 WORKFLOW    EXECUTE_RUN createWorkflowAndStoreMetadata (api/record.ts#L552)
 (max=6)     (max=1)     Run.status='running', 标记 runByAPI/SDK/MCP/CLI
     │          │         │                       │
     │          │    非文档│socket等待             │文档
     │          │         ▼                       ▼
     │          │    ready-for-run            addJob(EXECUTE_RUN, max=1)
     ▼          │    executeRun()                   │
 scheduler/     │    (api/record.ts#L732)           │
 index.ts       │                                    │
 handleRunRecording (scheduler版)                    │
     │          │                                    │
     │    ┌─────┴────────────────────────────────────┘
     │    ▼
     │  ┌──────────────────────────────────────────────────────────┐
     │  │         Graphile Worker: EXECUTE_RUN                     │
     │  │         task-runner.ts#L130 processRunExecution()        │
     │  │         处理: 前端手动(有槽/排队) + 文档类(Cron/API/SDK..) │
     │  └──────────────────────────┬───────────────────────────────┘
     │                             │
     ▼                             ▼
 ┌──────────────────────────────────────────────────────────────┐
 │                    执行函数（内容高度相似）                       │
 │  processRunExecution  │  executeRun (api版) │ executeRun(s版) │
 │  task-runner.ts       │  api/record.ts      │ scheduler/      │
 │  #L130-L582           │  #L732-L1401        │ index.ts #L186   │
 └──────────────────────┬──────────────────────┴──────────────────┘
                        ▼
          ┌─────────────────────────────────────────┐
          │ 状态更新: running → success/failed       │
          │ 通知: Socket.io + Webhook + Integration │
          └─────────────────────────────────────────┘
```

**分流与汇合的关键节点总结**：

1. **入队前的最大分叉**：5 路触发源分成 **Cron 专用队列（SCHEDULED_WORKFLOW）**、**前端专用队列（EXECUTE_RUN）**、**API/SDK/CLI/MCP 的 socket 直调路径（不入队）** 三种模式。
2. **文档机器人的汇合点**：无论来自哪路触发，文档机器人最终都入队 `EXECUTE_RUN`，统一由 `processRunExecution()` 的 doc-extract/doc-parse 分支处理。
3. **执行函数的分叉**：虽然有 3 个不同的执行函数（processRunExecution / 两个 executeRun），但内部逻辑（scrape处理、工作流执行、格式后处理、通知发送）高度一致，本质是同一逻辑的重复实现。
4. **状态更新的统一出口**：无论走哪条路径，成功/失败状态更新时都触发相同的 Socket/Webhook/Integration 三类通知。
