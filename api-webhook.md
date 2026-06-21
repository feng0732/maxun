# REST API 与 Webhook 回调协作流程分析

## 1. 整体架构概览

Maxun 是一个浏览器自动化/RPA 平台，核心围绕 **Robot（录制的工作流）** 和 **Run（一次执行实例）** 两个实体展开。外部请求通过 5 条路径进入，最终都汇聚到任务执行逻辑，任务结束时触发 webhook 回调。

```
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    外部请求来源                                                │
├────────────┬────────────┬────────────┬────────────┬─────────────────────────────────────────┤
│  UI 手动    │  REST API  │    SDK     │  MCP/CLI   │          定时调度 (Schedule)             │
└─────┬──────┴──────┬─────┴─────┬──────┴─────┬──────┴────────────────┬────────────────────────┘
      │             │           │            │                       │
      ▼             ▼           ▼            ▼                       ▼
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                                     任务创建层                                                   │
│                                                                                                │
│  ┌──────────────────────┐     ┌───────────────────────────────────────────────────────────┐   │
│  │ storage.ts (UI)      │     │                api/record.ts / scheduler/index.ts         │   │
│  │ createWorkflowAnd... │     │        createWorkflowAndStoreMetadata() 统一入口            │   │
│  └──────────┬───────────┘     └───────────────────────────┬───────────────────────────────┘   │
│             │                                               │                                   │
│             ▼                                               ▼                                   │
│   Run(status=running/queued)              Run(status=running)  ─┐                                │
│                                                                │ 文档机器人?                      │
│                                                                ▼                                 │
│                                                   addJob(EXECUTE_RUN)                           │
│                                                                │                                 │
└────────────────────────────────────────────────────────────────┼────────────────────────────────┘
                                                                 │
                                                                 ▼
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                              Graphile Worker 任务队列                                          │
│                                                                                                │
│  QUEUE_NAMES.EXECUTE_RUN         → processRunExecution()    [task-runner.ts]                   │
│  QUEUE_NAMES.SCHEDULED_WORKFLOW  → handleRunRecording()     [scheduler/index.ts]               │
│  QUEUE_NAMES.ABORT_RUN           → abortRun()                [task-runner.ts]                   │
│                                                                                                │
│                              processRunExecution() 分支:                                        │
│                              ├─ doc-extract → executeDocumentRun()                              │
│                              ├─ doc-parse   → executeDocumentParseRun()                         │
│                              ├─ scrape      → 页面转 Markdown/HTML/...                          │
│                              └─ extract/... → InterpretRecording()                              │
└────────────────────────────────────────────────────┬─────────────────────────────────────────┘
                                                     │
                                                     ▼
                          ┌──────────────────────────────────────────────┐
                          │    Socket.IO (前端)  +  sendWebhook()         │
                          │     + 集成更新 (GS/Airtable)                  │
                          └──────────────────────────────────────────────┘
```

---

## 2. 定时调度完整链路

定时调度是最复杂的链路，横跨 3 个模块：轮询器 → Worker 队列 → 调度执行器。

### 2.1 第一阶段：schedule-worker 轮询认领

**文件**：[schedule-worker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/schedule-worker.ts)

```
startScheduleWorker()
  │
  └─ 每 30 秒执行 processDueSchedules()
      │
      └─ claimDueDbSchedules()   [事务 + PostgreSQL 咨询锁]
          ├─ pg_try_advisory_xact_lock('robot_hash')  ← 防多实例重复
          ├─ WHERE schedule.nextRunAt <= now AND schedulerClaimedAt IS NULL
          ├─ UPDATE SET schedulerClaimedAt = now()     ← 标记已认领
          └─ 返回 robotMetaId, userId, scheduleInterval
```

**调用点**：[schedule-worker.ts#L122](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/schedule-worker.ts#L122)

### 2.2 第二阶段：投递到 Worker 队列

每个认领的 Robot 都会投递 `SCHEDULED_WORKFLOW` 队列：

```typescript
// schedule-worker.ts#L127-L132
await addJob(
  QUEUE_NAMES.SCHEDULED_WORKFLOW,
  { robotMetaId: due.robotMetaId, userId: String(due.userId) },
  { jobKey: `scheduled:${due.robotMetaId}:${Date.now()}`, maxAttempts: 3 }
);
```

### 2.3 第三阶段：Worker 消费 → scheduler/index.ts 的 handleRunRecording

**队列处理器**：[task-runner.ts#L670-L674](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/task-runner.ts#L670-L674)

```typescript
[QUEUE_NAMES.SCHEDULED_WORKFLOW]: async (payload) => {
  const data = payload as ScheduledWorkflowData;
  await handleRunRecording(data.robotMetaId, data.userId);  // ← scheduler 版本
};
```

**调度执行器**：[scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/workflow-management/scheduler/index.ts)

```
handleRunRecording(robotMetaId, userId)   [scheduler/index.ts#L860]
  │
  ├─ createWorkflowAndStoreMetadata()     [scheduler/index.ts#L42]
  │   ├─ Run.create(status='scheduled')   ← 注意：定时调度初始状态是 'scheduled'！
  │   ├─ 发送 Socket: run-scheduled       ← 通知前端已排期
  │   │
  │   └─ 如果是文档机器人 (doc-extract / doc-parse):
  │       └─ addJob(EXECUTE_RUN, {...})  → 交给 Worker 执行
  │
  └─ 如果是浏览器机器人:
      ├─ 建立 Socket.IO 连接 /{browserId}
      └─ 监听 'ready-for-run' → readyForRunHandler()
          └─ executeRun()  [scheduler/index.ts#L186]
              ├─ 若 Run.status='scheduled' → 升级为 'running'
              ├─ scrape: 页面转换 → status='success/failed'
              └─ extract: InterpretRecording → status='success/failed'
                  └─ 无论成功/失败 → sendWebhook()
```

> **关键差异**：定时调度的 Run 初始状态是 `'scheduled'`，而 API/SDK 直接是 `'running'`。

---

## 3. 文档机器人执行路径（doc-extract / doc-parse）

文档机器人（PDF 解析/数据抽取）不需要浏览器，走独立的执行路径，但 webhook 触发逻辑一致。

### 3.1 创建时即投递 Worker

无论是 API 还是定时调度，文档机器人在 `createWorkflowAndStoreMetadata()` 阶段就会投递 `EXECUTE_RUN`：

**API 路径**：[api/record.ts#L631-L637](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/api/record.ts#L631-L637)
```typescript
if (isDocRobot) {
  await addJob(QUEUE_NAMES.EXECUTE_RUN, {
    userId, runId: plainRun.runId, browserId,
  }, { maxAttempts: 1 });
}
```

**定时调度路径**：[scheduler/index.ts#L115-L121](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/workflow-management/scheduler/index.ts#L115-L121)
```typescript
if (isDocRobot) {
  await addJob(QUEUE_NAMES.EXECUTE_RUN, {
    userId, runId: plainRun.runId, browserId,
  }, { maxAttempts: 1 });
}
```

### 3.2 processRunExecution() 中的分支判断

**入口**：[task-runner.ts#L155-L178](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/task-runner.ts#L155-L178)

```typescript
if ((plainRun.interpreterSettings as any)?.robotType === 'doc-extract') {
  const recording = await Robot.findOne({...});
  const { executeDocumentRun } = await import('./utils/document/executeDocumentRun');
  await executeDocumentRun(recording, run, data.userId, serverIo);
  return;  // ← 提前 return，不走浏览器逻辑
}

if ((plainRun.interpreterSettings as any)?.robotType === 'doc-parse') {
  const recording = await Robot.findOne({...});
  const { executeDocumentParseRun } = await import('./utils/document/executeDocumentParseRun');
  await executeDocumentParseRun(recording, run, data.userId, serverIo);
  return;
}
```

### 3.3 executeDocumentRun() — 数据抽取型文档机器人

**文件**：[executeDocumentRun.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/utils/document/executeDocumentRun.ts)

```
executeDocumentRun(recording, run, userId, serverIo)
  │
  ├─ 1. getDocumentFromMinio(robotRecording.documentKey)   ← 下载 PDF
  │
  ├─ 2. DocumentInterpreter.extractData(pdfBuffer, prompt, schema, llmConfig)
  │     └─ LLM 解析 PDF，输出结构化数据
  │
  ├─ 3. Run.update(status='success', finishedAt, serializableOutput={ scrapeDoc: {...} })
  │
  ├─ 4. Socket.IO → /queued-run: run-completed (success)
  │
  └─ 5. sendWebhook(robotMeta.id, 'run_completed', {
         robot_id, run_id, robot_name, status, finished_at
       })   ← 文档机器人 payload 比较精简
```

**失败分支**：try/catch 捕获异常 → Run.status='failed' → Socket 通知失败 → **注意：此处没有 sendWebhook！**（文档机器人失败未发送 webhook，见代码 [executeDocumentRun.ts#L77-L101](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/utils/document/executeDocumentRun.ts#L77-L101)）

### 3.4 executeDocumentParseRun() — 格式转换型文档机器人

**文件**：[executeDocumentParseRun.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/utils/document/executeDocumentParseRun.ts)

流程与 executeDocumentRun 几乎一致，区别是：
- 调用 `DocumentInterpreter.parse(pdfBuffer, outputFormats)` 而非 extractData
- 输出格式：markdown / html / links
- 成功时同样 sendWebhook，失败时**同样没有 sendWebhook**

---

## 4. 各路径精确状态变化时序

### 4.1 路径对比总览

| 触发源 | 初始状态 | 中间状态 | 终态 | 执行位置 |
|--------|----------|----------|------|----------|
| UI 手动 | `queued` 或 `running` | — | `success/failed` | processRunExecution() |
| REST API | `running` | — | `success/failed` | api/record.ts executeRun() |
| SDK | `running` | — | `success/failed` | api/record.ts executeRun() |
| MCP/CLI | `running` | — | `success/failed` | api/record.ts executeRun() |
| 定时调度 | `scheduled` | `running` | `success/failed` | scheduler/index.ts executeRun() |
| 文档机器人 (API) | `running` | — | `success/failed` | executeDocumentRun/Parse() |
| 文档机器人 (定时) | `scheduled` | — | `success/failed` | executeDocumentRun/Parse() |

### 4.2 REST API 外部请求完整时序

以 `POST /api/robots/:id/runs` 为例，**精确到行号**：

```
外部客户端                    Maxun Server
     │                             │
     │ POST /api/robots/:id/runs   │
     │ x-api-key: <KEY>            │
     │────────────────────────────►│
     │                             │
     │                             │ ① requireAPIKey 校验
     │                             │    [api/record.ts#L1589, middlewares/api.ts]
     │                             │
     │                             │ ② handleRunRecording()
     │                             │    [api/record.ts#L1403]
     │                             │    │
     │                             │    └─ createWorkflowAndStoreMetadata()
     │                             │       [api/record.ts#L552]
     │                             │       │
     │                             │       ├─ Run.create({ status: 'running', ... })
     │                             │       │  [api/record.ts#L589]
     │                             │       │
     │                             │       ├─ Socket → run-started 通知前端
     │                             │       │  [api/record.ts#L625]
     │                             │       │
     │                             │       ├─ [文档机器人] addJob(EXECUTE_RUN)
     │                             │       │  [api/record.ts#L632]
     │                             │       │
     │                             │       └─ return { browserId, runId, isDocRobot }
     │                             │
     │                             │ ③ 建立 Socket.IO 连接 /{browserId}
     │                             │    等待 'ready-for-run' 事件
     │                             │    [api/record.ts#L1425-L1433]
     │                             │
     │  (浏览器启动完成)            │
     │◄────────────────────────────│ Socket: ready-for-run
     │                             │
     │                             │ ④ readyForRunHandler() → executeRun()
     │                             │    [api/record.ts#L691, #L732]
     │                             │    │
     │                             │    ├─ 检查 Run.status 非 aborted/queued
     │                             │    │  [api/record.ts#L746-L754]
     │                             │    │
     │                             │    ├─ [scrape 类型]
     │                             │    │  ├─ Run.update(status='running')  [L801]
     │                             │    │  ├─ 页面格式转换 (Markdown/HTML/...)
     │                             │    │  ├─ Run.update(status='success')  [L960]
     │                             │    │  ├─ 上传截图到 MinIO              [L970]
     │                             │    │  ├─ Socket: run-completed         [L986]
     │                             │    │  └─ sendWebhook('run_completed')  [L1017] ✅
     │                             │    │
     │                             │    └─ [extract/crawl/search 类型]
     │                             │       ├─ browser.interpreter.InterpretRecording()
     │                             │       ├─ Run.update(status='success')  [L1244]
     │                             │       ├─ Socket: run-completed         [L1265]
     │                             │       └─ sendWebhook('run_completed')  [L1279] ✅
     │                             │
     │                             │ ⑤ waitForRunCompletion() 轮询数据库
     │                             │    [api/record.ts#L1603, #L1482]
     │                             │    → 等待 Run.status 变为 success/failed
     │                             │
     │  200 { run: { ... } }       │
     │◄────────────────────────────│
```

### 4.3 定时调度时序对比

```
schedule-worker (每 30s)        Graphile Worker               scheduler/index.ts
       │                              │                              │
       ├─ claimDueDbSchedules()       │                              │
       └─ addJob(SCHEDULED_WORKFLOW)  │                              │
       ──────────────────────────────►│                              │
                                      │                              │
                                      │ handleRunRecording()         │
                                      │ [task-runner.ts#L673]        │
                                      └─────────────────────────────►│
                                                                    │
                                                                    ├─ createWorkflowAndStoreMetadata()
                                                                    │  [scheduler/index.ts#L42]
                                                                    │  ├─ Run.create(status='scheduled')  ✦ 注意状态
                                                                    │  ├─ Socket: run-scheduled
                                                                    │  └─ [文档机器人] addJob(EXECUTE_RUN)
                                                                    │
                                                                    ├─ Socket 连接浏览器
                                                                    │
                                                                    └─ 浏览器 ready → executeRun()
                                                                       [scheduler/index.ts#L186]
                                                                       ├─ Run.update(status='running')  ✦ 状态升级
                                                                       ├─ ...执行工作流...
                                                                       ├─ Run.update(status='success')
                                                                       ├─ Socket: run-completed
                                                                       └─ sendWebhook('run_completed') ✅
```

---

## 5. Webhook 回调触发点全览

系统共 **10 处** `sendWebhook()` 调用，覆盖 4 个文件、所有执行路径、两种结果状态。

### 5.1 触发点位置索引

| # | 文件 | 行号 | 事件类型 | 触发路径 | 场景 |
|---|------|------|----------|----------|------|
| 1 | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/task-runner.ts) | L353-L361 | `run_completed` | UI/Worker | scrape 类型成功 |
| 2 | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/task-runner.ts) | L492-L506 | `run_completed` | UI/Worker | extract/crawl/search 成功 |
| 3 | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/task-runner.ts) | L542-L549 | `run_failed` | UI/Worker | 执行失败（内层 catch） |
| 4 | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/task-runner.ts) | L566-L572 | `run_failed` | UI/Worker | 执行失败（外层 catch） |
| 5 | [api/record.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/api/record.ts) | L1017 | `run_completed` | API/SDK/MCP | scrape 类型成功 |
| 6 | [api/record.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/api/record.ts) | L1279-L1306 | `run_completed` | API/SDK/MCP | extract/crawl/search 成功 |
| 7 | [api/record.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/api/record.ts) | L1079-L1093 | `run_failed` | API/SDK/MCP | scrape 类型失败 |
| 8 | [api/record.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/api/record.ts) | L1358-L1381 | `run_failed` | API/SDK/MCP | extract/crawl/search 失败 |
| 9 | [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/workflow-management/scheduler/index.ts) | L492 | `run_completed` | 定时调度 | scrape 类型成功 |
| 10 | [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/workflow-management/scheduler/index.ts) | L748-L753 | `run_completed` | 定时调度 | extract/crawl/search 成功 |
| 11 | [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/workflow-management/scheduler/index.ts) | L798-L803 | `run_failed` | 定时调度 | 执行失败（main catch） |
| 12 | [executeDocumentRun.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/utils/document/executeDocumentRun.ts) | L67-L73 | `run_completed` | API/定时调度 | doc-extract 成功 |
| 13 | [executeDocumentParseRun.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/utils/document/executeDocumentParseRun.ts) | L52-L62 | `run_completed` | API/定时调度 | doc-parse 成功 |
| 14 | [webhook.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/routes/webhook.ts) | L490 | `webhook_test` | UI 测试 | 手动测试调用 |

> **共 14 处**调用点（含 webhook 测试）。

### 5.2 Webhook 发送时机与状态变化的关系

```
Run.status 变化                sendWebhook() 调用点
─────────────────               ──────────────────────

创建阶段:
  'scheduled' (定时)            ── 不发送 webhook
  'queued'    (UI槽位不足)      ── 不发送 webhook
  'running'                     ── 不发送 webhook

执行结束:
  ┌─ 'success' ────────────────► sendWebhook('run_completed')  ← 12 处触发点
  │                              (task-runner, api/record, scheduler, doc*)
  │
  └─ 'failed'  ────────────────► sendWebhook('run_failed')     ← 5 处触发点
                                 (task-runner×2, api/record×2, scheduler)

中止:
  'aborting' → 'aborted'        ── 不发送 webhook  ← 注意：中止没有 webhook！
                                 (见 task-runner.ts#L414-L418 abortRun() 中无 sendWebhook)
```

### 5.3 文档机器人的特殊情况

| 场景 | 是否发送 webhook | 代码位置 |
|------|------------------|----------|
| doc-extract 成功 | ✅ 发送 | [executeDocumentRun.ts#L67-L73](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/utils/document/executeDocumentRun.ts#L67-L73) |
| doc-extract 失败 | ❌ 不发送 | [executeDocumentRun.ts#L77-L101](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/utils/document/executeDocumentRun.ts#L77-L101) （仅 Socket 通知，无 webhook） |
| doc-parse 成功 | ✅ 发送 | [executeDocumentParseRun.ts#L52-L62](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/utils/document/executeDocumentParseRun.ts#L52-L62) |
| doc-parse 失败 | ❌ 不发送 | [executeDocumentParseRun.ts#L63-L87](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/utils/document/executeDocumentParseRun.ts#L63-L87) （仅 Socket 通知，无 webhook） |

---

## 6. sendWebhook 分发与重试机制

### 6.1 sendWebhook 核心逻辑

**文件**：[routes/webhook.ts#L404-L465](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/routes/webhook.ts#L404-L465)

```
sendWebhook(robotMetaId, eventType, data)
  │
  ├─ 1. Robot.findOne({ where: { 'recording_meta.id': robotMetaId } })
  │     → 取出 recording_meta.webhooks[]
  │
  ├─ 2. 过滤 webhook:
  │     w.active === true
  │     AND eventType ∈ w.events
  │
  ├─ 3. 对每个匹配 webhook 构建完整 payload:
  │     {
  │       event_type: eventType,
  │       timestamp: new Date().toISOString(),
  │       webhook_id: webhook.id,
  │       data: data  ← 调用方传入的数据
  │     }
  │
  ├─ 4. Promise.allSettled() 并行调用 sendWebhookWithRetry()
  │
  └─ 5. 所有错误仅 logger.log('warn'/'error')，不抛出异常
       → webhook 失败不影响 Run 状态（已持久化 success/failed）
```

### 6.2 重试与超时

`sendWebhookWithRetry(robotMetaId, webhook, payload, attempt)`：

- **超时**：`webhook.timeout || 30` 秒（axios config.timeout）
- **最大重试**：`webhook.retryAttempts || 3` 次
- **退避策略**：指数退避 `retryDelay * 2^(attempt-1)` 秒
  - 第 1 次失败 → 等 `5s` → 第 2 次
  - 第 2 次失败 → 等 `10s` → 第 3 次
  - 第 3 次失败 → 放弃，打 error 日志
- **状态更新**：每次尝试前 `Robot.update()` 更新 `webhooks[].lastCalledAt = now()`

---

## 7. REST API 请求-Webhook 回调时间轴

以 `POST /api/robots/:id/runs`（scrape 类型机器人）为例，精确时序：

```
时间轴 (ms)      事件                                   Run.status         Webhook
──────────      ──────────────────────────────────     ──────────         ───────
T=0             客户端发起请求
T≈20            requireAPIKey 校验通过
T≈80            createWorkflowAndStoreMetadata()
                 └─ Run.create(...)                  ──► 'running'
T≈100           Socket.IO 连接建立
T≈2000          浏览器启动完成 → 'ready-for-run'
T≈2010          executeRun() 开始执行
                 ├─ Run.update(status='running')       (已是 running)
                 └─ 开始页面转换...
T≈5000          Markdown/HTML/截图等转换完成
                 ├─ Run.update(status='success')    ──► 'success'
                 ├─ MinIO 上传截图
                 ├─ Socket.IO emit('run-completed')
                 └─ sendWebhook('run_completed')  ────────┐
T≈5050                                                      │
                 sendWebhook 内部:                           │
                 ├─ 查询 Robot.webhooks[]                   │
                 ├─ 过滤 active + 匹配 events               │
                 └─ Promise.allSettled → axios.post()  ─────┼──► 第三方 Webhook URL
T≈5100          waitForRunCompletion() 轮询 DB               │
                 → 检测到 status='success'                  │
T≈5105          返回 HTTP 200                                │
                                                            │ (异步)
T≈5200          第三方收到 webhook 请求 ◄────────────────────┘
T≈5250          第三方返回 2xx OK
```

**关键点**：
1. HTTP 响应和 webhook 是**异步**的 —— HTTP 200 返回时 webhook 可能仍在重试中
2. webhook 发送失败**不回滚** Run 状态（success 已持久化）
3. 第三方系统应同时支持 **轮询 GET** 和 **webhook 回调** 两种方式

---

## 8. 关键文件索引

| 文件 | 作用 |
|------|------|
| [server.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/server.ts) | 服务器入口，路由注册，Worker 启动 |
| [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/task-runner.ts) | Graphile Worker 处理器：`processRunExecution()`, 队列定义 |
| [schedule-worker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/schedule-worker.ts) | 定时调度轮询器，`claimDueDbSchedules()` |
| [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/workflow-management/scheduler/index.ts) | 定时调度执行器：`handleRunRecording()`, `executeRun()` |
| [routes/storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/routes/storage.ts) | UI 触发 Run 入口 + `processQueuedRuns()` |
| [routes/webhook.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/routes/webhook.ts) | Webhook CRUD + `sendWebhook()` + 重试逻辑 |
| [api/record.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/api/record.ts) | REST API 核心：`handleRunRecording()`, `executeRun()` |
| [api/sdk.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/api/sdk.ts) | SDK 专用 API |
| [executeDocumentRun.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/utils/document/executeDocumentRun.ts) | doc-extract 文档机器人执行器 |
| [executeDocumentParseRun.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/utils/document/executeDocumentParseRun.ts) | doc-parse 文档机器人执行器 |
| [models/Run.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/models/Run.ts) | Run 数据模型 |
| [models/Robot.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/models/Robot.ts) | Robot 数据模型（含 webhooks 字段） |
| [middlewares/api.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/middlewares/api.ts) | API Key 认证中间件 |
