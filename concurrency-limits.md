# 多任务并发与资源限额机制分析

## 概述

Maxun 项目采用**三层并发控制 + 三层排队机制**来管理任务执行和资源使用。

**三层并发控制：**
1. **全局任务队列层**：基于 Graphile Worker 的 PostgreSQL 任务队列，控制系统级并发
2. **浏览器资源池层**：BrowserPool 管理浏览器实例，实施"1 用户 - 2 浏览器"策略
3. **工作流内部并发层**：Concurrency 类管理单个工作流内的多任务并发执行

**三层排队机制（按时间顺序）：**
1. **定时调度层**：基于 Cron 表达式的数据库轮询调度（每 30 秒）
2. **应用层排队**：当浏览器槽位不足时，Run 记录标记为 `queued` 状态等待（每 5 秒轮询）
3. **Graphile Worker 执行队列**：真正的异步任务队列，由 Worker 池消费执行

⚠️ **关键发现**：应用层排队（`queued` 状态）**仅在 Web UI 手动运行路径中实现**，定时调度和 API/SDK 运行路径中没有应用层排队，浏览器槽位不足时直接失败。

---

## 一、任务排队机制

### 1.1 三层排队架构全景

```
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 1: 定时调度层 (schedule-worker.ts)                           │
│  - 每 30 秒轮询 Robot 表中 Cron 到期的任务                            │
│  - 使用 PostgreSQL 咨询锁保证多实例安全                                │
│  - 将任务加入 SCHEDULED_WORKFLOW 队列                                 │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 2: 应用层排队 (storage.ts)                                    │
│  ⚠️ 仅 Web UI 手动运行路径有此层！                                     │
│  - 检查 hasAvailableBrowserSlots()                                    │
│  - 有槽位 → 创建浏览器 + 创建 Run(running) + 入队 EXECUTE_RUN          │
│  - 无槽位 → 创建 Run(queued)，每 5 秒 processQueuedRuns 轮询重试       │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 3: Graphile Worker 执行队列                                   │
│  - Worker 并发数由 WORKER_CONCURRENCY 控制（默认 10）                  │
│  - 消费 EXECUTE_RUN、SCHEDULED_WORKFLOW、ABORT_RUN 等任务              │
│  - 执行 processRunExecution 等实际工作                                  │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 定时调度层（Layer 1）

**核心文件**：[schedule-worker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/schedule-worker.ts)

#### 1.2.1 调度流程

```typescript
// 调度频率
const DB_SCHEDULER_POLL_MS = 30000;          // 每 30 秒轮询一次
const DB_SCHEDULER_BATCH_SIZE = 10;           // 每次最多处理 10 个任务
const DB_SCHEDULER_CLAIM_TIMEOUT_MS = 10 * 60 * 1000;  // 声明超时 10 分钟
```

**执行步骤**：
1. 每 30 秒触发一次 `processDueSchedules()`
2. 通过 `pg_try_advisory_xact_lock` 获取 PostgreSQL 咨询锁（防止多实例重复处理）
3. 查询 Robot 表中 `schedule.nextRunAt <= now` 且未被声明的记录
4. 声明任务（更新 `schedulerClaimedAt = now`）
5. 将任务加入 `SCHEDULED_WORKFLOW` 队列
6. 成功后计算并更新 `nextRunAt`，失败则释放声明

#### 1.2.2 SCHEDULED_WORKFLOW 任务消费

**核心代码**：[task-runner.ts L670-L674](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts#L670-L674)

```typescript
[QUEUE_NAMES.SCHEDULED_WORKFLOW]: async (payload: unknown) => {
  const data = payload as ScheduledWorkflowData;
  logger.log('info', `Processing scheduled workflow for robot ${data.robotMetaId}`);
  await handleRunRecording(data.robotMetaId, data.userId);
},
```

调用的是 [scheduler/index.ts L860](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/workflow-management/scheduler/index.ts#L860) 中的 `handleRunRecording`。

#### 1.2.3 定时调度路径中没有应用层排队！

**关键发现**：定时调度路径在浏览器槽位不足时**直接失败**，不会进入 queued 状态。

**代码证据**：
- [scheduler/index.ts L74](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/workflow-management/scheduler/index.ts#L74)：`const browserId = isDocRobot ? uuid() : createRemoteBrowserForRun(userId);`
- [controller.ts L122-L126](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/controller.ts#L122-L126)：`createRemoteBrowserForRun` 在槽位不足时**抛出异常**：

```typescript
const slotReserved = browserPool.reserveBrowserSlotAtomic(id, userId, "run");
if (!slotReserved) {
  logger.log('warn', `Cannot create browser for user ${userId}: no available slots`);
  throw new Error('User has reached maximum browser limit');
}
```

#### 1.2.4 错误传播链与 maxAttempts 失效分析

**⚠️ 关键发现：`maxAttempts: 6` 在浏览器槽位不足场景下完全不会生效！**

**完整错误传播链（定时调度路径）：**

| 步骤 | 代码位置 | 动作 | 结果 |
|------|----------|------|------|
| 1 | [controller.ts L125](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/controller.ts#L125) | `createRemoteBrowserForRun` 槽位不足 | `throw new Error('User has reached maximum browser limit')` |
| 2 | [scheduler/index.ts L129-L137](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/workflow-management/scheduler/index.ts#L129-L137) | 被 `createWorkflowAndStoreMetadata` catch | 返回 `{ success: false, error: message }`，**不重新抛出** |
| 3 | [scheduler/index.ts L864-L865](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/workflow-management/scheduler/index.ts#L864-L865) | `handleRunRecording` 解构返回值 | `browserId = undefined`, `runId = undefined` |
| 4 | [scheduler/index.ts L867-L869](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/workflow-management/scheduler/index.ts#L867-L869) | 检查 `runId` 有效性 | `throw new Error('runId or userId is undefined')` |
| 5 | [scheduler/index.ts L903-L908](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/workflow-management/scheduler/index.ts#L903-L908) | 被 `handleRunRecording` 自己的 catch 捕获 | 仅日志，**不重新抛出** |
| 6 | [task-runner.ts L673](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts#L673) | Graphile Worker 收到返回值 | 函数返回 `undefined`，Worker 认为任务**成功完成** |

**核心问题**：两处 catch 都"吞掉"了错误，没有重新抛出给 Graphile Worker。

```
Graphile Worker
      │
      ▼ 调用
handleRunRecording()
      │
      ├─ try {
      │    createWorkflowAndStoreMetadata()
      │      │
      │      ├─ try {
      │      │    createRemoteBrowserForRun() → 抛错
      │      │  } catch (e) {
      │      │    return { success: false }  ← 第1次吞错
      │      │  }
      │      ↓
      │    runId = undefined → 抛错
      │  } catch (error) {
      │    logger.error(...)  ← 第2次吞错
      │    // 没有 throw error！
      │  }
      ↓
返回 undefined → Graphile Worker 标记为成功
```

**maxAttempts 何时才会生效？**

只有当 `handleRunRecording` 抛出**未被捕获**的异常时，Graphile Worker 才会触发重试。例如：
- `await handleRunRecording(...)` 抛出异常（但目前不会）
- Worker 进程崩溃（不是应用层错误）
- 数据库连接中断（不是业务逻辑错误）

**即使修复了重新抛出，重试仍无意义**：

即使我们在 `handleRunRecording` 的 catch 块末尾加上 `throw error`，让 Graphile Worker 触发重试，由于：
- Graphile Worker 默认重试间隔是指数退避（1s, 2s, 4s, 8s, 16s, 32s...）
- 浏览器槽位通常不会在几秒内释放（运行中的任务可能持续几分钟）
- 6 次重试会在约 1 分钟内全部失败

所以 `maxAttempts: 6` 本质上是一个**无效的重试机制**，无法解决浏览器槽位不足的排队问题。真正的解决方案应该是**将定时调度路径也纳入应用层排队（queued 状态）**。

### 1.3 应用层排队（Layer 2）

**仅 Web UI 手动运行路径实现了应用层排队**：[storage.ts L1018-L1124](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts#L1018-L1124)

#### 1.3.1 入队逻辑

```typescript
const canCreateBrowser = await browserPool.hasAvailableBrowserSlots(req.user.id, "run");

if (canCreateBrowser) {
  // 路径 A：有可用槽位 → 立即创建浏览器 + 入队 Worker
  const browserId = await createRemoteBrowserForRun(req.user.id);
  await Run.create({ status: 'running', ... });
  await addJob(QUEUE_NAMES.EXECUTE_RUN, { ... });
  return { queued: false };
} else {
  // 路径 B：无可用槽位 → 应用层排队
  await Run.create({ 
    status: 'queued', 
    log: 'Run queued - waiting for available browser slot',
    ... 
  });
  return { queued: true };
}
```

#### 1.3.2 排队恢复

**核心函数**：[processQueuedRuns](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts#L1493-L1566)

每 5 秒轮询一次，在 [server.ts L151-L158](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/server.ts#L151-L158) 中设置：

```typescript
const processQueuedRunsInterval = setInterval(async () => {
  await processQueuedRuns();
}, 5000);
```

**processQueuedRuns 执行流程**：
1. 查询最早的一条 `status: 'queued'` 运行记录
2. 检查该用户是否有可用浏览器槽位
3. 如有槽位：创建浏览器 → 更新 Run 为 `running` → 加入 `EXECUTE_RUN` 队列
4. 如无槽位：跳过，等待下一次轮询
5. 带熔断器保护：连续 3 次数据库错误后冷却 30 秒

#### 1.3.3 哪些路径没有应用层排队？

| 触发路径 | 入口函数 | 是否有应用层排队 |
|----------|----------|-----------------|
| Web UI 手动运行 | [storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts#L1018) | ✅ 有 |
| Web UI 重试运行 | [storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts#L1188) | ✅ 有（重试从 queued 开始） |
| 定时调度 | [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/workflow-management/scheduler/index.ts#L860) | ❌ 无，槽位不足直接失败 |
| API / SDK 运行 | [record.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/api/record.ts#L552) | ❌ 无，槽位不足直接失败 |

### 1.4 Graphile Worker 执行队列（Layer 3）

**核心文件**：[graphileWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/storage/graphileWorker.ts)

```typescript
export async function addJob(
  taskIdentifier: string,
  payload: Record<string, unknown>,
  options?: { maxAttempts?: number; runAt?: Date; jobKey?: string },
): Promise<string>
```

**所有 EXECUTE_RUN 入队入口**：

| 代码位置 | 触发场景 | maxAttempts |
|----------|----------|-------------|
| [storage.ts L1069](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts#L1069) | Web UI 手动运行 | 1 |
| [storage.ts L1188](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts#L1188) | Web UI 重试运行 | 1 |
| [storage.ts L1539](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts#L1539) | processQueuedRuns 恢复排队任务 | 1 |
| [storage.ts L2047](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts#L2047) | 验证 API 运行 | 1 |
| [storage.ts L2103](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts#L2103) | 另一个 API 运行入口 | 1 |
| [scheduler/index.ts L116](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/workflow-management/scheduler/index.ts#L116) | 定时调度 DocRobot | 1 |
| [record.ts L632](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/api/record.ts#L632) | API DocRobot 运行 | 1 |

### 1.5 服务启动崩溃恢复

**核心函数**：[recoverOrphanedRuns](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts#L1572-L1655)

服务启动时执行一次：
1. 查询所有 `status: 'running'` 或 `status: 'scheduled'` 的运行
2. 检查浏览器是否还在 BrowserPool 中
3. 如浏览器不存在且重试次数 < 3：将状态改为 `queued` 重新排队
4. 如重试次数 >= 3：标记为 `failed`

---

## 二、并发控制机制

### 2.1 全局 Worker 并发数

**核心文件**：[task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts#L80)

```typescript
const TOTAL_CONCURRENCY = Math.max(1, parseInt(process.env.WORKER_CONCURRENCY || '10', 10));
```

**Worker 启动配置**：
```typescript
runner = await run({
  pgPool: runnerPool,
  concurrency: TOTAL_CONCURRENCY,
  noHandleSignals: true,
  pollInterval: 3600000,  // 1小时轮询间隔（依赖 LISTEN/NOTIFY 实时通知）
  taskList,
});
```

### 2.2 浏览器资源池（BrowserPool）

**核心文件**：[BrowserPool.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts)

#### 2.2.1 资源限额策略

**"1 用户 - 2 浏览器"硬编码策略**：
- 每个用户最多拥有 2 个浏览器实例
- 每个用户最多只能有 1 个处于 "recording" 状态的浏览器
- "run" 状态的浏览器可以有 1-2 个（取决于是否还有 recording 浏览器）

**限额检查**：[hasAvailableBrowserSlots](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts#L550-L565)

#### 2.2.2 原子预约机制

**核心方法**：[reserveBrowserSlotAtomic](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts#L595-L646)

使用内存中的 `reservationLocks` Map 防止同一用户并发预约：

```typescript
public reserveBrowserSlotAtomic = (id: string, userId: string, state: BrowserState = "run"): boolean => {
  const lockKey = `${userId}-${state}`;
  if (this.reservationLocks.has(lockKey)) return false;
  
  try {
    this.reservationLocks.set(lockKey, Date.now());
    if (!this.hasAvailableBrowserSlots(userId, state)) return false;
    
    this.pool[id] = {
      browser: null,
      active: false,
      userId,
      state,
      status: "reserved",  // 直接设置为 reserved
      createdAt: now,
      lastAccessed: now,
    };
    // 更新用户映射
    return true;
  } finally {
    this.reservationLocks.delete(lockKey);
  }
};
```

#### 2.2.3 浏览器槽位状态（重要发现）

**类型定义**：[BrowserPool.ts L36-L38](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts#L36-L38)

```typescript
status?: "reserved" | "initializing" | "ready" | "failed",
```

**⚠️ `initializing` 状态从未被实际设置！**

代码核查结果：
- 全局搜索 `status = 'initializing'` 或 `status: 'initializing'` **无任何匹配**
- `reserveBrowserSlotAtomic` 设置状态为 `"reserved"`
- `upgradeBrowserSlot` **直接从 `"reserved"` 升级为 `"ready"`**，跳过 `initializing`
- 失败时调用 `failBrowserSlot` 设置为 `"failed"` 并移除

**实际状态流转**：
```
reserved → (异步初始化进行中，status 保持 reserved) → ready
                    ↓
                  failed
```

**代码缺陷影响**：
- `cleanupStaleBrowserSlots` 方法检查 `info.status === "initializing"`，但这个状态永远不会出现
- 实际只需要检查 `status === "reserved"` 即可覆盖所有未就绪情况

### 2.3 超时边界分析（两个不同的超时）

系统中存在两个独立但有关联的超时，用于不同阶段的保护：

#### 2.3.1 浏览器初始化超时（Browser Initialization Timeout）

**位置**：[controller.ts L420](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/controller.ts#L420)

```typescript
const BROWSER_INIT_TIMEOUT = 45000;  // 45 秒
```

**含义**：保护浏览器初始化过程本身的超时。

**生效阶段**：`initializeBrowserAsync` 函数内部，`browserSession.initialize(userId)` 调用时。

```typescript
const initPromise = browserSession.initialize(userId);
const timeoutPromise = new Promise<never>((_, reject) => {
  setTimeout(() => reject(new Error('Browser initialization timeout')), BROWSER_INIT_TIMEOUT);
});
await Promise.race([initPromise, timeoutPromise]);
```

**超时后的处理**：
1. 调用 `browserSession.switchOff()` 尝试清理
2. 抛出 `Browser initialization timeout` 错误
3. 外层 catch 调用 `browserPool.failBrowserSlot(id)` 标记失败并移除槽位

#### 2.3.2 Worker 等待就绪超时（Worker Wait-for-Ready Timeout）

**位置**：[task-runner.ts L131](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts#L131)

```typescript
const BROWSER_INIT_TIMEOUT = 60000;   // 60 秒（⚠️ 与上面同名但不同值！）
const BROWSER_PAGE_TIMEOUT = 15000;   // 15 秒
```

**含义**：保护 Worker 等待浏览器槽位就绪的超时。由于浏览器初始化是异步进行的，Worker 需要轮询等待浏览器从 `reserved` 变为 `ready`。

**生效阶段**：`processRunExecution` 函数中，Worker 从队列取出任务后。

```typescript
let browser = browserPool.getRemoteBrowser(browserId);
const browserWaitStart = Date.now();
const MAX_POLL_ATTEMPTS = Math.ceil(BROWSER_INIT_TIMEOUT / 2000);  // 60/2 = 30 次

while (!browser && (Date.now() - browserWaitStart) < BROWSER_INIT_TIMEOUT && pollAttempts < MAX_POLL_ATTEMPTS) {
  const browserStatus = browserPool.getBrowserStatus(browserId);
  if (browserStatus === null) throw new Error(`Browser slot ${browserId} does not exist in pool`);
  if (browserStatus === 'failed') throw new Error(`Browser ${browserId} initialization failed`);
  
  await new Promise(resolve => setTimeout(resolve, 2000));  // 每 2 秒轮询一次
  browser = browserPool.getRemoteBrowser(browserId);
}

if (!browser) {
  throw new Error(`Browser ${browserId} not found in pool after ${BROWSER_INIT_TIMEOUT / 1000}s timeout ...`);
}
```

**超时后的处理**：
1. 抛出错误，被外层 catch 捕获
2. 更新 Run 状态为 `failed`
3. 调用 `destroyRemoteBrowser(browserId, data.userId)` 清理资源

还有第三层超时：**页面就绪超时**（`BROWSER_PAGE_TIMEOUT = 15 秒`），用于等待浏览器的 currentPage 对象可用。

#### 2.3.3 超时关系对比

| 超时名称 | 位置 | 值 | 保护阶段 | 触发场景 |
|----------|------|-----|----------|----------|
| 浏览器初始化超时 | controller.ts | 45 秒 | `browserSession.initialize()` 调用 | Playwright 启动浏览器卡死、页面加载过慢 |
| Worker 等待就绪超时 | task-runner.ts | 60 秒 | Worker 轮询等待浏览器槽位从 reserved→ready | 初始化异步任务被阻塞、槽位升级失败 |
| 页面就绪超时 | task-runner.ts | 15 秒 | 等待 `browser.getCurrentPage()` 返回非空值 | 浏览器启动后页面未创建 |

**时间关系**：Worker 等待超时（60s）> 浏览器初始化超时（45s），确保初始化有足够时间完成后 Worker 再判断超时。

### 2.4 工作流内部并发（Concurrency 类）

**核心文件**：[concurrency.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/maxun-core/src/utils/concurrency.ts)

Concurrency 类用于在单个工作流执行中管理并行任务，是一个轻量级的并发控制器。

#### 2.4.1 核心属性

```typescript
export default class Concurrency {
  maxConcurrency: number = 1;        // 最大并发数
  activeWorkers: number = 0;         // 当前活动 Worker 数
  private jobQueue: Function[] = []; // 等待执行的任务队列
  private waiting: Array<{ resolve: () => void; reject: (error: Error) => void }> = []; // 等待完成的回调
}
```

#### 2.4.2 执行流程

```
添加任务 → 检查 activeWorkers < maxConcurrency
    ↓ 是                    ↓ 否
  立即执行                加入队列
    ↓
  任务完成
    ↓
  从队列取下一个任务
    ↓
  队列为空且 activeWorkers === 0
    ↓
  触发所有 waitForCompletion 回调
```

### 2.5 解释器中的并发使用

**核心文件**：[interpret.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/maxun-core/src/interpret.ts)

```typescript
// 默认并发数为 5
this.concurrency = new Concurrency(this.options.maxConcurrency);

// 弹窗/多页面处理
this.concurrency.addJob(() => this.runLoop(popup, workflowCopy));

// 主循环启动
this.concurrency.addJob(() => this.runLoop(page, this.initializedWorkflow!));
await this.concurrency.waitForCompletion();
```

---

## 三、资源释放机制

### 3.1 浏览器销毁流程

**核心函数**：[destroyRemoteBrowser](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/controller.ts#L155-L233)

#### 3.1.1 销毁步骤

1. **清除录制超时**：`clearRecordingTimeout(id)`
2. **关闭浏览器实例**：`browserSession.switchOff()`
3. **清理 Socket 连接**：断开所有 socket、移除事件监听器、删除命名空间
4. **从池移除**：`browserPool.deleteRemoteBrowser(id)`

#### 3.1.2 销毁超时保护

```typescript
const DESTROY_TIMEOUT = 30000; // 30秒超时
const destroyPromise = (async () => { /* 销毁逻辑 */ })();
const timeoutPromise = new Promise<boolean>((_, reject) =>
  setTimeout(() => reject(new Error(`Browser destruction timed out after ${DESTROY_TIMEOUT}ms`)), DESTROY_TIMEOUT)
);
return await Promise.race([destroyPromise, timeoutPromise]);
```

超时后强制从池中删除浏览器记录。

### 3.2 完整资源释放路径汇总

| 触发场景 | 代码位置 | 释放函数 |
|----------|----------|----------|
| **正常完成** | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts) | `processRunExecution` 末尾 → `destroyRemoteBrowser` |
| **执行失败** | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts) | catch 块 → `destroyRemoteBrowser` |
| **Worker 等待浏览器超时** | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts#L210-L213) | 超时错误 → catch 块 → `destroyRemoteBrowser` |
| **用户中止** | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts#L584-L629) | `abortRun` → `destroyRemoteBrowser` |
| **录制超时（10分钟）** | [controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/controller.ts#L14-L15) | `setTimeout` 回调 → `destroyRemoteBrowser` |
| **槽位过期清理（5分钟）** | [BrowserPool.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts#L702-L729) | `cleanupStaleBrowserSlots` → `failBrowserSlot` |
| **浏览器创建失败** | [controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/controller.ts#L466-L478) | `initializeBrowserAsync` catch → `failBrowserSlot` |
| **浏览器初始化超时（45秒）** | [controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/controller.ts#L429-L438) | catch initError → `failBrowserSlot` |
| **Run 创建数据库错误** | [storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts#L1056-L1063) | catch 块 → `destroyRemoteBrowser` |
| **入队失败** | [storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts#L1076-L1089) | catch 块 → `destroyRemoteBrowser` |
| **调度器成功完成** | [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/workflow-management/scheduler/index.ts#L655) | 成功路径 → `destroyRemoteBrowser` |
| **调度器执行失败** | [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/workflow-management/scheduler/index.ts#L763-L766) | catch 块 → `destroyRemoteBrowser` |
| **录制完成/失败** | [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/workflow-management/scheduler/index.ts#L842-L849) | `readyForRunHandler` → `destroyRemoteBrowser` |
| **服务启动清理** | [server.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/server.ts#L170-L171) | `browserPool.cleanupStaleBrowserSlots()` |
| **服务优雅关闭（SIGINT）** | [server.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/server.ts#L220-L260) | SIGINT 处理 → 遍历保存数据后关闭 |

### 3.3 定时清理机制

**服务启动时**：[server.ts L170-L171](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/server.ts#L170-L171)

```typescript
logger.log('info', 'Cleaning up stale browser slots...');
browserPool.cleanupStaleBrowserSlots();
```

**每分钟清理**：[server.ts L160-L163](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/server.ts#L160-L163)

```typescript
const browserPoolCleanupInterval = setInterval(() => {
  browserPool.cleanupStaleBrowserSlots();
}, 60000);  // 每分钟执行一次
```

### 3.4 过期槽位清理逻辑

**核心方法**：[cleanupStaleBrowserSlots](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts#L702-L729)

```typescript
public cleanupStaleBrowserSlots = (): void => {
  const staleThreshold = 5 * 60 * 1000; // 5分钟
  
  for (const [id, info] of Object.entries(this.pool)) {
    // ⚠️ 检查 "initializing" 状态，但实际这个状态从未被设置
    const isStale = info.status === "reserved" || info.status === "initializing";
    const age = now - (info.createdAt || 0);
    
    if (isStale && info.browser === null && age > staleThreshold) {
      this.failBrowserSlot(id, `Slot stale for ${age/1000}s`);
    }
  }
  
  // 同时清理过期的预约锁（超过1分钟）
  for (const [lockKey, timestamp] of this.reservationLocks.entries()) {
    if (now - timestamp > 60000) {
      this.reservationLocks.delete(lockKey);
    }
  }
};
```

### 3.5 服务优雅关闭时的资源释放

**核心代码**：[server.ts L213-L260](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/server.ts#L213-L260)

SIGINT 信号处理流程：
1. 等待 2 秒
2. 遍历所有运行中的浏览器
3. 如有采集数据，保存到对应的 Run 记录
4. 逐个调用 `destroyRemoteBrowser`
5. 依次停止：scheduleWorker → workers → graphileWorkerUtils
6. 关闭服务器

---

## 四、各层配合关系

### 4.1 完整任务执行链路（三种路径对比）

#### 路径 A：Web UI 手动运行（有应用层排队）

```
用户点击 "Run"
      ↓
POST /storage/robots/:id/run
      ↓
hasAvailableBrowserSlots?
    ├─ 是 → 预约槽位(reserved) → 异步初始化浏览器
    │              ↓
    │        创建 Run(status: 'running')
    │              ↓
    │        加入 EXECUTE_RUN 队列
    │              ↓
    │        Worker 消费 → 轮询等待浏览器就绪(60s超时)
    │              ↓
    │        浏览器就绪(ready) → 执行工作流 → 完成/失败 → 销毁浏览器
    │
    └─ 否 → 创建 Run(status: 'queued')
                 ↓
           [每 5 秒 processQueuedRuns 轮询]
                 ↓
           槽位释放后，重复 "是" 的路径
```

#### 路径 B：定时调度运行（无应用层排队）

```
Cron 时间到达
      ↓
schedule-worker 每 30 秒轮询
      ↓
声明任务 + 加入 SCHEDULED_WORKFLOW 队列 (maxAttempts: 6)
      ↓
Worker 消费 → 调用 handleRunRecording(scheduler版本)
      ↓
createWorkflowAndStoreMetadata
      ↓
createRemoteBrowserForRun(userId)
      ↓
reserveBrowserSlotAtomic 成功?
    ├─ 是 → 预约槽位 → 异步初始化 → 连接 Socket 等待 ready-for-run
    │              ↓
    │        readyForRunHandler → 执行工作流 → 完成/失败 → 销毁浏览器
    │
    └─ 否 → ① throw Error('User has reached maximum browser limit')
                 ↓
           ② 被 createWorkflowAndStoreMetadata catch → return {success: false}
                 ↓
           ③ handleRunRecording 解构失败对象 → runId = undefined
                 ↓
           ④ throw new Error('runId or userId is undefined')
                 ↓
           ⑤ 被 handleRunRecording 自己的 catch 捕获 → 仅日志，不重新抛出
                 ↓
           ⑥ 函数静默返回 undefined → Graphile Worker 认为任务成功
                 ↓
           ⚠️  maxAttempts: 6 完全不会触发！任务永久丢失！
```

#### 路径 C：API/SDK 运行（无应用层排队）

```
POST /api/robots/:id/runs
      ↓
handleRunRecording(API版本)
      ↓
createWorkflowAndStoreMetadata(API版本)
      ↓
createRemoteBrowserForRun(userId)
      ↓
reserveBrowserSlotAtomic 成功?
    ├─ 是 → 预约槽位 → 异步初始化 → 连接 Socket 等待 ready-for-run
    │              ↓
    │        readyForRunHandler → 执行工作流 → 完成/失败 → 销毁浏览器
    │
    └─ 否 → throw Error → catch 返回 {success: false}
                 ↓
           handleRunRecording 拿不到 runId → 再次 throw
                 ↓
           API 返回 500 错误（无重试，无排队）
```

### 4.2 排队状态流转全景

```
用户请求 / 定时调度
    ↓
┌─────────────────────────────────────────────────────────┐
│  检查浏览器槽位                                            │
│                                                           │
│  Web UI 路径: hasAvailableBrowserSlots()                   │
│  定时调度/API: createRemoteBrowserForRun() 内部检查         │
└───────────────────────┬───────────────────────────────────┘
                        │
          ┌─────────────┴─────────────┐
          ▼                           ▼
     有可用槽位                     无可用槽位
          │                           │
          ▼                           ▼
   reserved 状态              ┌─────────────────────┐
   (异步初始化中)              │  Web UI: queued 状态 │
          │                   │  定时/API: 直接失败  │
          ▼                   └─────────┬───────────┘
     ready 状态                        │
          │                     [每 5 秒轮询]
          ▼                           │
   Worker 执行                        │
   (60s 等待就绪超时)                  │
          │                           │
          ▼                           │
   success/failed                     │
          │                           │
          └─────────── 销毁浏览器 ←───┘
                          │
                          ▼
                   释放槽位 + Worker 空闲
```

### 4.3 并发层级汇总

| 层级 | 控制对象 | 限额配置 | 默认值 | 所在文件 |
|------|----------|----------|--------|----------|
| 全局任务 | Worker 并发数 | `WORKER_CONCURRENCY` | 10 | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts#L80) |
| 用户级 | 浏览器实例数 | 硬编码策略 | 2/用户 | [BrowserPool.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts) |
| 用户级 | 录制浏览器数 | 硬编码策略 | 1/用户 | [BrowserPool.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts) |
| 应用层 | 排队任务数 | 无限制（数据库存储） | 无限制 | - |
| 工作流内 | 内部任务并发 | `maxConcurrency` | 5 | [interpret.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/maxun-core/src/interpret.ts#L102) |
| 数据库连接 | Worker 池大小 | `TOTAL_CONCURRENCY + 2` | 12 | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts#L692) |
| 数据库连接 | Utils 池大小 | 硬编码 | 3 | [graphileWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/storage/graphileWorker.ts#L28) |

### 4.4 超时层级汇总

| 层级 | 超时名称 | 值 | 所在文件 | 保护目标 |
|------|----------|-----|----------|----------|
| 浏览器初始化 | 浏览器初始化超时 | 45 秒 | [controller.ts L420](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/controller.ts#L420) | Playwright 启动浏览器过程 |
| Worker 等待 | 浏览器就绪超时 | 60 秒 | [task-runner.ts L131](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts#L131) | Worker 等待浏览器从 reserved→ready |
| Worker 等待 | 页面就绪超时 | 15 秒 | [task-runner.ts L132](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts#L132) | 等待 currentPage 对象可用 |
| 资源清理 | 槽位过期阈值 | 5 分钟 | [BrowserPool.ts L704](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts#L704) | reserved 状态槽位清理 |
| 资源清理 | 预约锁过期 | 1 分钟 | [BrowserPool.ts L710](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts#L710) | reservationLocks 清理 |
| 录制保护 | 录制超时 | 10 分钟 | [controller.ts L14](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/controller.ts#L14) | 录制浏览器自动销毁 |
| 销毁保护 | 浏览器销毁超时 | 30 秒 | [controller.ts L159](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/controller.ts#L159) | destroyRemoteBrowser 强制完成 |
| 调度声明 | 调度声明超时 | 10 分钟 | [schedule-worker.ts L16](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/schedule-worker.ts#L16) | schedulerClaimedAt 可被重新声明 |
| 排队恢复 | 排队轮询间隔 | 5 秒 | [server.ts L157](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/server.ts#L157) | processQueuedRuns 频率 |
| 调度检查 | 调度轮询间隔 | 30 秒 | [schedule-worker.ts L15](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/schedule-worker.ts#L15) | processDueSchedules 频率 |

---

## 五、代码缺陷与改进建议

### 5.1 已发现的代码缺陷

**缺陷 1：`initializing` 状态从未被设置（死代码）**

- **位置**：[BrowserPool.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts)
- **问题**：类型定义包含 `"initializing"` 状态，`cleanupStaleBrowserSlots` 也检查该状态，但代码中从未实际设置
- **影响**：清理逻辑中 `status === "initializing"` 分支永远不会触发
- **建议**：要么在 `initializeBrowserAsync` 开始时设置 `status = "initializing"`，要么从类型定义和清理逻辑中移除该状态

**缺陷 2：三个运行路径行为不一致**

- **问题**：Web UI 手动运行有应用层排队（queued 状态），但定时调度和 API/SDK 运行没有
- **影响**：浏览器槽位不足时，定时任务直接失败（虽有 Graphile Worker 重试），API 请求直接返回 500
- **建议**：统一三个路径的行为，要么都实现应用层排队，要么明确文档说明差异

**缺陷 3：两处 BROWSER_INIT_TIMEOUT 同名不同值**

- **位置**：[controller.ts L420](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/controller.ts#L420) = 45s，[task-runner.ts L131](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts#L131) = 60s
- **问题**：同名变量值不同，容易混淆
- **建议**：重命名区分含义，如 `BROWSER_SESSION_INIT_TIMEOUT`（45s）和 `WORKER_BROWSER_READY_TIMEOUT`（60s）

**缺陷 4：定时调度失败后 maxAttempts 完全不生效（任务永久丢失）**

- **位置**：[scheduler/index.ts L903-L908](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/workflow-management/scheduler/index.ts#L903-L908)
- **问题**：`SCHEDULED_WORKFLOW` 任务设置 `maxAttempts: 6`，但两处 catch 都"吞掉"了错误（`createWorkflowAndStoreMetadata` catch 返回 `{success: false}` 不重新抛出，`handleRunRecording` catch 仅日志不重新抛出），函数最终返回 `undefined`，Graphile Worker 认为任务成功完成
- **影响**：浏览器槽位不足时，定时调度任务**永久丢失**，没有重试，没有排队，甚至连失败日志都不会被 Graphile Worker 记录
- **建议**：
  1. 在 `handleRunRecording` 的 catch 块末尾添加 `throw error`，让 Graphile Worker 能感知到失败
  2. 但即使修复了重新抛出，由于重试间隔太短（指数退避，约 1 分钟内 6 次重试全部失败），仍无法解决问题
  3. **真正的修复**：将定时调度路径也纳入应用层排队（queued 状态），与 Web UI 路径保持一致

### 5.2 潜在风险

1. **排队任务无超时**：`queued` 状态的任务没有超时机制，理论上可能永远排队
2. **重试次数硬编码**：崩溃恢复的重试次数（3次）、Graphile Worker 重试次数（6次）均硬编码
3. **用户级资源隔离不足**：一个用户占满 2 个浏览器槽位后，其他任务只能排队，没有优先级或公平性机制
