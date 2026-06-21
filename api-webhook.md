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

## 5. Webhook 触发点全览（核准统计口径）

本系统的 webhook 调用需要从 **三个层次** 进行统计，不可混淆：

| 统计口径 | 含义 | 数量 | 核心函数 |
|----------|------|------|----------|
| **① 运行结束回调调用点** | 业务代码中调用 `sendWebhook()` 分发器的位置 | **13 处** | `sendWebhook(robotId, eventType, data)` |
| **② 测试接口直接发送** | 测试接口绕过分发器，直接 axios.post | **1 处** | `axios.post(webhook.url, testPayload)` |
| **③ 实际 HTTP 投递次数** | 每个匹配 webhook URL 的发送（含重试） | **动态** | `sendWebhookWithRetry()` |

> **核心概念**：`sendWebhook()` 是**分发器**，不是实际发送函数。它查询配置 → 过滤匹配的 webhook → 对每个 URL 调用 `sendWebhookWithRetry()`（实际 HTTP 发送）。因此「13 处运行结束回调」并不等于「13 次 HTTP 请求」，实际请求数取决于该 Robot 配置了多少个 active 的、订阅对应事件的 webhook。

---

### 5.1 口径①：运行结束回调调用点（13 处）

这些是业务流程结束时调用 `sendWebhook()` 分发器的位置，分布在 6 个文件中。

#### 5.1.1 成功事件 `run_completed`（8 处）

| # | 文件 | 行号 | 触发路径 | 场景 |
|---|------|------|----------|------|
| 1 | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/task-runner.ts#L358) | L358 | UI/Worker | scrape 类型成功（页面转 Markdown 等） |
| 2 | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/task-runner.ts#L493) | L493 | UI/Worker | extract/crawl/search 类型成功 |
| 3 | [api/record.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/api/record.ts#L1017) | L1017 | API/SDK/MCP | scrape 类型成功 |
| 4 | [api/record.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/api/record.ts#L1302) | L1302 | API/SDK/MCP | extract/crawl/search 类型成功 |
| 5 | [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/workflow-management/scheduler/index.ts#L492) | L492 | 定时调度 | scrape 类型成功 |
| 6 | [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/workflow-management/scheduler/index.ts#L749) | L749 | 定时调度 | extract/crawl/search 类型成功 |
| 7 | [executeDocumentRun.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/utils/document/executeDocumentRun.ts#L67) | L67 | API/定时调度 | doc-extract（PDF 数据抽取）成功 |
| 8 | [executeDocumentParseRun.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/utils/document/executeDocumentParseRun.ts#L53) | L53 | API/定时调度 | doc-parse（PDF 格式转换）成功 |

**成功事件小计：8 处**

#### 5.1.2 失败事件 `run_failed`（5 处）

| # | 文件 | 行号 | 触发路径 | 场景 |
|---|------|------|----------|------|
| 1 | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/task-runner.ts#L543) | L543 | UI/Worker | 执行失败（内层 catch：scrape 转换失败 / InterpretRecording 抛错） |
| 2 | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/task-runner.ts#L567) | L567 | UI/Worker | 执行失败（外层 catch：浏览器准备阶段异常 / 未预期异常） |
| 3 | [api/record.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/api/record.ts#L1080) | L1080 | API/SDK/MCP | scrape 类型失败 |
| 4 | [api/record.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/api/record.ts#L1377) | L1377 | API/SDK/MCP | extract/crawl/search 类型失败 |
| 5 | [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/workflow-management/scheduler/index.ts#L799) | L799 | 定时调度 | 执行失败（main catch：所有类型异常汇总） |

**失败事件小计：5 处**

> **总计**：成功 8 + 失败 5 = **13 处运行结束回调**

---

### 5.2 口径②：测试接口直接发送（1 处）

**位置**：[webhook.ts#L356-L361](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/routes/webhook.ts#L356-L361)

**注意**：测试接口 **不经过 `sendWebhook()` 分发器**，而是绕过它直接发送：

```typescript
// webhook.ts#L356-L361
await updateWebhookLastCalled(robotId, webhook.id);

const response = await axios.post(webhook.url, testPayload, {
    timeout: (webhook.timeout || 30) * 1000,
    validateStatus: (status) => status < 500
});
```

特点：
- 事件类型固定为 `event_type: "webhook_test"`（在 payload 中硬编码，不是 `sendWebhook()` 的 eventType 参数）
- **没有重试逻辑**（只发 1 次，失败立即返回给用户）
- 只发给用户指定的**单个** webhook URL（不是所有匹配配置）

---

### 5.3 口径③：实际 HTTP 投递（动态数量）

**函数**：`sendWebhookWithRetry(robotId, webhook, payload, attempt)` [webhook.ts#L437](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/routes/webhook.ts#L437)

每次业务调用 `sendWebhook()` 时的实际 HTTP 请求数：

```
实际 HTTP 请求数 = (匹配的 active webhook 数量) × (1 + 重试次数)
```

其中：
- **匹配数量**：`robot.webhooks` 中满足 `w.active === true && w.events.includes(eventType)` 的数量
- **重试次数**：0 ~ `webhook.retryAttempts - 1`（默认最多重试 2 次，合计发 3 次）
- **退避策略**：5s → 10s → 20s（指数退避）

---

### 5.4 调用层级关系图

```
业务代码（13 处运行结束回调）               + 测试接口（1 处直接发送）
       │                                            │
       ▼                                            │
sendWebhook(robotId, eventType, data)               │
  [webhook.ts#L404]  ───────────────────────┐       │
       │                                     │       │
       ├─ 1. Robot.findOne() → webhooks[]   │       │
       ├─ 2. 过滤 active + events 匹配       │       │
       ├─ 3. Promise.allSettled() 并发       │       │
       │   N 次（N = 匹配 webhook 数）       │       │
       ▼                                     ▼       ▼
sendWebhookWithRetry()               axios.post(testPayload)
  [webhook.ts#L437]                  [webhook.ts#L358]
       │
       ├─ axios.post(webhook.url)   ←── 这才是实际 HTTP 请求
       └─ 若失败: setTimeout → 递归调用自己（最多 3 次）
```

---

### 5.5 Webhook 发送时机与状态变化的关系

```
Run.status 变化                     触发 sendWebhook()?     事件类型
─────────────────                   ──────────────────     ──────────

创建阶段:
  'scheduled' (定时)                 ── 否 ──              —
  'queued'    (UI槽位不足)           ── 否 ──              —
  'running'                          ── 否 ──              —

执行结束:
  ┌─ 'success'                      ── 是（8 处）──       run_completed
  │                                    task-runner×2
  │                                    api/record×2
  │                                    scheduler×2
  │                                    executeDocument×2
  │
  └─ 'failed'                       ── 是（5 处）──       run_failed
                                       task-runner×2
                                       api/record×2
                                       scheduler×1

中止:
  'aborting' → 'aborted'             ── 否 ──              —
                                     （abortRun() 中无 sendWebhook
                                       见 task-runner.ts#L584-L628）

文档机器人注意事项:
  doc-extract/doc-parse → success    ── 是 ──             run_completed
  doc-extract/doc-parse → failed     ── 否 ──             （代码缺失，
                                                             仅有 Socket 通知）
```

### 5.6 成功/失败事件数量汇总

| 统计维度 | 数量 |
|----------|------|
| `run_completed`（成功）回调点 | **8 处** |
| `run_failed`（失败）回调点 | **5 处** |
| 运行结束回调合计 | **13 处** |
| 测试接口直接发送 | **1 处**（不计入运行结束回调） |
| 不含 webhook 的状态分支 | `aborted`、`doc-* failed`（2 处缺口） |

---

## 6. sendWebhook 分发器与 sendWebhookWithRetry 实际发送

### 6.1 sendWebhook() — 分发器（无实际 HTTP 调用）

**文件**：[webhook.ts#L404-L434](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/routes/webhook.ts#L404-L434)

`sendWebhook()` 本身不做 HTTP 请求，它是一个「查询 + 过滤 + 分发」的协调器：

```
sendWebhook(robotMetaId, eventType, data)
  │
  ├─ 1. Robot.findOne({ where: { 'recording_meta.id': robotMetaId } })
  │     → 取出 recording_meta.webhooks[] （JSONB 数组）
  │
  ├─ 2. 过滤 activeWebhooks:
  │     w.active === true
  │     AND eventType ∈ w.events
  │     → 无匹配则静默 return（不打日志）
  │
  ├─ 3. 对每个匹配 webhook 构建完整 payload:
  │     {
  │       event_type: eventType,      // 'run_completed' | 'run_failed'
  │       timestamp: ISO8601,
  │       webhook_id: webhook.id,
  │       data: data                 // 调用方传入（各路径不同）
  │     }
  │
  ├─ 4. Promise.allSettled() 并行调用 sendWebhookWithRetry()
  │     → 每个 active webhook URL 走独立的重试逻辑
  │
  └─ 5. 所有错误仅 console.error()，不抛出异常
       → webhook 失败不影响主流程（Run 状态已持久化）
```

### 6.2 sendWebhookWithRetry() — 实际 HTTP 投递

**文件**：[webhook.ts#L437-L465](file:///d:/fz/0601-2/solo-dogfeeding/code/117-maxun/server/src/routes/webhook.ts#L437-L465)

这是真正执行 `axios.post()` 的函数，带重试逻辑：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `timeout` | 30s | axios 超时（毫秒） |
| `retryAttempts` | 3 次 | 最多发送次数（含首次） |
| `retryDelay` | 5s | 基础退避时间 |
| 退避公式 | `retryDelay × 2^(attempt-1)` | 指数退避：5s → 10s → 20s |

**执行流程**：
```
sendWebhookWithRetry(robotId, webhook, payload, attempt=1)
  │
  ├─ 1. updateWebhookLastCalled()  ← 每次尝试都更新 lastCalledAt（即使失败）
  │
  ├─ 2. axios.post(webhook.url, payload, { timeout, ... })
  │
  ├─ 3. 成功 → return
  │
  └─ 4. 失败:
       ├─ attempt < retryAttempts → setTimeout(sendWebhookWithRetry(...), delay × 1000)
       │    （递归 setTimeout，不阻塞 sendWebhook 的 Promise.allSettled）
       └─ 已达最大重试 → console.error() 放弃
```

> **注意**：重试使用 `setTimeout` 异步调度，因此 `sendWebhook()` 返回的 Promise 只等待**首次发送**完成，后续重试在后台继续。HTTP 响应返回给 API 客户端时，重试可能还在进行。

---

## 7. REST API 请求-Webhook 回调时间轴

以 `POST /api/robots/:id/runs`（scrape 类型机器人，Robot 配置了 2 个 active 的 `run_completed` webhook）为例，精确时序区分三层调用：

```
时间轴 (ms)      事件                                   Run.status         层级
──────────      ──────────────────────────────────     ──────────         ───────
T=0             客户端发起请求
T≈20            requireAPIKey 校验通过
T≈80            createWorkflowAndStoreMetadata()
                 └─ Run.create(...)                  ──► 'running'
T≈100           Socket.IO 连接建立
T≈2000          浏览器启动完成 → 'ready-for-run'
T≈2010          executeRun() 开始执行
                 ├─ Run.update(status='running')       (已是 running)
                 └─ 开始页面格式转换...
T≈5000          Markdown/HTML/截图等转换完成
                 ├─ Run.update(status='success')    ──► 'success'
                 ├─ MinIO 上传截图
                 ├─ Socket.IO emit('run-completed')
                 └─ sendWebhook('run_completed')  ──── ① 分发器调用 (api/record.ts#L1017)
T≈5050           ├─ Robot.findOne() → 找到 2 个匹配 webhook
T≈5060           ├─ Promise.allSettled() 并发派发        │
T≈5061           ├─ sendWebhookWithRetry(webhook A)  ── ② 首次实际发送
T≈5061           └─ sendWebhookWithRetry(webhook B)  ── ② 首次实际发送
T≈5070                ├─ axios.post(webhookA.url)         ├──► 第三方 A
T≈5070                └─ axios.post(webhookB.url)         ├──► 第三方 B
T≈5100          waitForRunCompletion() 轮询 DB
                 → 检测到 status='success'                  (webhook 仍在飞行中)
T≈5105          返回 HTTP 200                              (sendWebhook() 此时尚未 resolve)
                                                              │
T≈5180          第三方 B 返回 200 OK ◄───────────────────────┘
T≈5200          第三方 A 超时 / 500 / ECONNREFUSED ◄───────┘
T≈5201           └─ setTimeout(sendWebhookWithRetry(A), 5s)   ③ 第1次重试排期
T≈10201         sendWebhookWithRetry(A, attempt=2)        ── ③ 第1次重试实际发送
T≈10201           └─ axios.post(webhookA.url)              ├──► 第三方 A
T≈10250          第三方 A 返回 200 OK ◄──────────────────────┘
                 (所有 webhook 最终完成)
```

**关键点**：
1. **三层区分**：① 业务代码调用 `sendWebhook()` 分发器 → ② `sendWebhookWithRetry()` 首次 HTTP 发送 → ③ 超时/失败后递归 setTimeout 重试
2. **HTTP 响应与 webhook 异步**：HTTP 200 在 T≈5105 返回时，首次 webhook 可能仍在飞行，第 2、3 次重试可能延后数秒甚至数十秒
3. **重试在后台继续**：失败重试用 `setTimeout` 异步调度，`Promise.allSettled` 只等待首次发送，不等待重试完成
4. **webhook 失败不回滚**：Run.status='success' 已在 T≈5000 持久化，后续 webhook 全失败也不改变
5. **第三方系统建议**：同时支持 **轮询 GET** 和 **webhook 回调**，避免依赖 webhook 的时效可靠性

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
