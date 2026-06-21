# 多任务并发与资源限额机制分析

## 概述

Maxun 项目采用**三层并发控制 + 两层排队机制**来管理任务执行和资源使用：

**三层并发控制：**
1. **全局任务队列层**：基于 Graphile Worker 的 PostgreSQL 任务队列，控制系统级并发
2. **浏览器资源池层**：BrowserPool 管理浏览器实例，实施"1 用户 - 2 浏览器"策略
3. **工作流内部并发层**：Concurrency 类管理单个工作流内的多任务并发执行

**两层排队机制：**
1. **应用层排队**：当浏览器槽位不足时，Run 记录标记为 `queued` 状态等待
2. **Graphile Worker 队列**：真正的异步任务队列，由 Worker 池消费

---

## 一、任务排队机制

### 1.1 两层排队架构

系统存在两个独立但关联的排队层级：

#### 第一层：应用层排队（Application-Level Queue）

当用户浏览器槽位已满时，任务不直接进入 Worker 队列，而是在应用层排队：

**核心逻辑**：[storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts#L1018-L1124)

```typescript
const canCreateBrowser = await browserPool.hasAvailableBrowserSlots(req.user.id, "run");

if (canCreateBrowser) {
  // 路径A：有可用槽位 → 立即创建浏览器 + 入队 Worker
  const browserId = await createRemoteBrowserForRun(req.user.id);
  await Run.create({ status: 'running', ... });
  await addJob(QUEUE_NAMES.EXECUTE_RUN, { ... });
  return { queued: false };
} else {
  // 路径B：无可用槽位 → 应用层排队，返回 queued: true
  await Run.create({ 
    status: 'queued', 
    log: 'Run queued - waiting for available browser slot',
    ... 
  });
  return { queued: true };
}
```

**排队任务触发**：[processQueuedRuns](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts#L1493-L1566)

每 5 秒轮询一次（在 [server.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/server.ts#L151-L158) 中设置）：

```typescript
const processQueuedRunsInterval = setInterval(async () => {
  await processQueuedRuns();
}, 5000);
```

**processQueuedRuns 执行流程**：
1. 查询最早的 `status: 'queued'` 运行记录
2. 检查该用户是否有可用浏览器槽位
3. 如有槽位：创建浏览器 → 更新 Run 为 `running` → 加入 Worker 队列
4. 如无槽位：跳过，等待下一次轮询

#### 第二层：Graphile Worker 任务队列

**核心文件**：[graphileWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/storage/graphileWorker.ts)

```typescript
export async function addJob(
  taskIdentifier: string,
  payload: Record<string, unknown>,
  options?: { maxAttempts?: number; runAt?: Date; jobKey?: string },
): Promise<string>
```

### 1.2 所有 EXECUTE_RUN 入队入口

| 入口位置 | 代码行 | 触发场景 |
|----------|--------|----------|
| [storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts#L1069) | L1069 | 用户手动运行机器人（有浏览器槽位时） |
| [storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts#L1188) | L1188 | 用户重试运行 |
| [storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts#L1539) | L1539 | processQueuedRuns 处理排队任务 |
| [storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts#L2047) | L2047 | 验证 API 触发运行 |
| [storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts#L2103) | L2103 | 另一个 API 运行入口 |
| [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/workflow-management/scheduler/index.ts#L116) | L116 | 定时调度器创建 DocRobot 运行 |
| [record.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/api/record.ts#L632) | L632 | SDK API 触发运行 |

### 1.3 定时调度流程

**核心文件**：[schedule-worker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/schedule-worker.ts)

调度流程：
1. 每 30 秒轮询一次（`DB_SCHEDULER_POLL_MS = 30000`）
2. 使用 PostgreSQL 咨询锁（`pg_try_advisory_xact_lock`）保证多实例安全
3. 批量获取到期任务（`DB_SCHEDULER_BATCH_SIZE = 10`）
4. 声明（claim）任务并更新 `schedulerClaimedAt` 时间戳
5. 调用 `createWorkflowAndStoreMetadata` 创建运行（内部检查浏览器槽位）
6. 成功后更新 `nextRunAt`，失败则释放声明

**关键点**：定时调度不直接入队，而是调用调度器创建运行，调度器内部会走应用层排队逻辑。

### 1.4 服务启动恢复机制

**核心函数**：[recoverOrphanedRuns](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts#L1572-L1655)

服务启动时执行一次：
1. 查询所有 `status: 'running'` 或 `status: 'scheduled'` 的运行
2. 检查浏览器是否还在池中
3. 如浏览器不存在且重试次数 < 3：将状态改为 `queued` 重新排队
4. 如重试次数 >= 3：标记为 `failed`

---

## 二、并发控制机制

### 2.1 全局 Worker 并发数

**核心文件**：[task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts#L80)

```typescript
const TOTAL_CONCURRENCY = Math.max(1, parseInt(process.env.WORKER_CONCURRENCY || '10', 10));
```

**配置说明**：
- 环境变量 `WORKER_CONCURRENCY` 控制总并发数
- 默认值为 10，最小值为 1
- 对应 Graphile Worker 的 `concurrency` 参数

**Worker 启动配置**：
```typescript
runner = await run({
  pgPool: runnerPool,
  concurrency: TOTAL_CONCURRENCY,
  noHandleSignals: true,
  pollInterval: 3600000,  // 1小时轮询间隔（依赖 LISTEN/NOTIFY）
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

```typescript
public hasAvailableBrowserSlots = (userId: string, state?: BrowserState): boolean => {
  const userBrowserIds = this.userToBrowserMap.get(userId) || [];
  if (userBrowserIds.length >= 2) return false;
  if (state === "recording") {
    const hasBrowserInState = userBrowserIds.some(browserId => 
      this.pool[browserId] && this.pool[browserId].state === "recording"
    );
    return !hasBrowserInState;
  }
  return true;
};
```

#### 2.2.2 原子预约机制

**核心方法**：[reserveBrowserSlotAtomic](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts#L595-L646)

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
      status: "reserved",  // 注意：设置为 reserved
      createdAt: now,
      lastAccessed: now,
    };
    
    // 更新用户映射
    const userBrowserIds = this.userToBrowserMap.get(userId) || [];
    if (!userBrowserIds.includes(id)) {
      userBrowserIds.push(id);
      this.userToBrowserMap.set(userId, userBrowserIds);
    }
    
    return true;
  } finally {
    this.reservationLocks.delete(lockKey);
  }
};
```

#### 2.2.3 浏览器槽位状态（重要发现）

**类型定义**中的状态：[BrowserPool.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts#L36-L38)

```typescript
status?: "reserved" | "initializing" | "ready" | "failed",
```

**⚠️ 关键发现：`initializing` 状态从未被实际设置！**

代码核查结果：
- 全局搜索 `status = 'initializing'` 或 `status: 'initializing'` 无任何匹配
- `reserveBrowserSlotAtomic` 设置状态为 `"reserved"`
- `upgradeBrowserSlot` 直接从 `"reserved"` 升级为 `"ready"`
- 失败时调用 `failBrowserSlot` 设置为 `"failed"`

**实际状态流转**：
```
reserved → (异步初始化进行中，status 保持 reserved) → ready
                    ↓
                  failed
```

**代码缺陷说明**：
- `cleanupStaleBrowserSlots` 方法检查 `info.status === "initializing"`，但这个状态永远不会出现
- 实际只需要检查 `status === "reserved"` 即可覆盖所有未就绪情况
- 浏览器初始化过程在 `initializeBrowserAsync` 中异步进行，但期间状态一直是 `reserved`

**Worker 等待浏览器就绪**：[task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts#L187-L213)

```typescript
while (!browser && (Date.now() - browserWaitStart) < BROWSER_INIT_TIMEOUT && pollAttempts < MAX_POLL_ATTEMPTS) {
  const browserStatus = browserPool.getBrowserStatus(browserId);
  if (browserStatus === null) throw new Error(`Browser slot ${browserId} does not exist in pool`);
  if (browserStatus === 'failed') throw new Error(`Browser ${browserId} initialization failed`);
  
  await new Promise(resolve => setTimeout(resolve, 2000));
  browser = browserPool.getRemoteBrowser(browserId);
}
```

#### 2.2.4 槽位升级与失败处理

- **升级槽位**：[upgradeBrowserSlot](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts#L655-L670) - 检查当前状态为 `reserved`，然后设置 `status = "ready"` 并关联 browser 对象
- **标记失败**：[failBrowserSlot](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts#L677-L696) - 调用 `deleteRemoteBrowser` 从池中移除

### 2.3 工作流内部并发（Concurrency 类）

**核心文件**：[concurrency.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/maxun-core/src/utils/concurrency.ts)

Concurrency 类用于在单个工作流执行中管理并行任务，是一个轻量级的并发控制器。

#### 2.3.1 核心属性

```typescript
export default class Concurrency {
  maxConcurrency: number = 1;        // 最大并发数
  activeWorkers: number = 0;         // 当前活动 Worker 数
  private jobQueue: Function[] = []; // 等待执行的任务队列
  private waiting: Array<{ resolve: () => void; reject: (error: Error) => void }> = []; // 等待完成的回调
}
```

#### 2.3.2 核心方法

| 方法 | 说明 |
|------|------|
| `addJob(job)` | 添加任务到队列，有空闲则立即执行 |
| `runNextJob()` | （私有）从队列取出下一个任务执行 |
| `waitForCompletion()` | 返回 Promise，所有任务完成后 resolve |

#### 2.3.3 执行流程

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

### 2.4 解释器中的并发使用

**核心文件**：[interpret.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/maxun-core/src/interpret.ts)

解释器（Interpreter）在以下场景使用 Concurrency 类：

#### 2.4.1 初始化

```typescript
// 默认并发数为 5
this.concurrency = new Concurrency(this.options.maxConcurrency);
```

#### 2.4.2 弹窗/多页面处理

在处理弹窗或多页面时，使用并发控制管理多个页面的工作流执行：

```typescript
this.concurrency.addJob(() => this.runLoop(popup, workflowCopy));
```

#### 2.4.3 主循环启动

```typescript
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
3. **清理 Socket 连接**：
   - 断开所有 socket 连接
   - 移除所有事件监听器
   - 删除命名空间
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
| **用户中止** | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts#L584-L629) | `abortRun` → `destroyRemoteBrowser` |
| **录制超时** | [controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/controller.ts#L14-L15) | `setTimeout` 回调 → `destroyRemoteBrowser` |
| **槽位过期清理** | [BrowserPool.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts#L702-L729) | `cleanupStaleBrowserSlots` → `failBrowserSlot` |
| **浏览器创建失败** | [controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/controller.ts#L466-L478) | `initializeBrowserAsync` catch → `failBrowserSlot` |
| **Run 创建数据库错误** | [storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts#L1056-L1063) | catch 块 → `destroyRemoteBrowser` |
| **入队失败** | [storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts#L1076-L1089) | catch 块 → `destroyRemoteBrowser` |
| **调度器成功完成** | [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/workflow-management/scheduler/index.ts#L655) | 成功路径 → `destroyRemoteBrowser` |
| **调度器执行失败** | [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/workflow-management/scheduler/index.ts#L763-L766) | catch 块 → `destroyRemoteBrowser` |
| **录制完成/失败** | [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/workflow-management/scheduler/index.ts#L842-L849) | `readyForRunHandler` → `destroyRemoteBrowser` |
| **服务启动清理** | [server.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/server.ts#L170-L171) | `browserPool.cleanupStaleBrowserSlots()` |
| **服务优雅关闭** | [server.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/server.ts#L220-L260) | SIGINT 处理 → 遍历所有浏览器保存数据后关闭 |

### 3.3 定时清理机制

**服务启动时**：[server.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/server.ts#L170-L171)

```typescript
logger.log('info', 'Cleaning up stale browser slots...');
browserPool.cleanupStaleBrowserSlots();
```

**每分钟清理**：[server.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/server.ts#L160-L163)

```typescript
const browserPoolCleanupInterval = setInterval(() => {
  browserPool.cleanupStaleBrowserSlots();
}, 60000);  // 每分钟执行一次
```

### 3.4 过期槽位清理逻辑

**核心方法**：[cleanupStaleBrowserSlots](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts#L702-L729)

```typescript
public cleanupStaleBrowserSlots = (): void => {
  const now = Date.now();
  const staleThreshold = 5 * 60 * 1000; // 5分钟
  
  for (const [id, info] of Object.entries(this.pool)) {
    // 注意：检查 "initializing" 状态，但实际这个状态从未被设置
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

### 3.5 录制超时自动释放

**核心文件**：[controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/controller.ts#L14-L15)

```typescript
const RECORDING_TIMEOUT_MS = 10 * 60 * 1000; // 10分钟
const recordingTimeouts = new Map<string, NodeJS.Timeout>();
```

录制浏览器在创建时设置 10 分钟超时：

```typescript
const timeoutHandle = setTimeout(async () => {
  recordingTimeouts.delete(id);
  io.of(id).emit('recording-timeout');
  await new Promise(resolve => setTimeout(resolve, 1000)); // 等待前端接收事件
  await destroyRemoteBrowser(id, userId);
}, RECORDING_TIMEOUT_MS);
```

### 3.6 服务优雅关闭时的资源释放

**核心代码**：[server.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/server.ts#L213-L260)

SIGINT 信号处理流程：
1. 遍历所有运行中的浏览器
2. 如有采集数据，保存到对应的 Run 记录
3. 逐个调用 `destroyRemoteBrowser`
4. 依次停止：scheduleWorker → workers → graphileWorkerUtils
5. 关闭服务器

### 3.7 解释器状态清理

当运行中止或失败时，解释器需要清理状态：

```typescript
try { 
  if (browser && browser.interpreter) {
    await browser.interpreter.clearState(); 
  } 
} catch (_) {}
await destroyRemoteBrowser(browserId, data.userId);
```

---

## 四、各层配合关系

### 4.1 完整任务执行链路（两种路径）

#### 路径 A：有可用浏览器槽位

```
用户请求 / 定时调度
      ↓
  检查浏览器槽位 → 可用
      ↓
  预约槽位 (reserved)
      ↓
  异步启动浏览器初始化
      ↓
  创建 Run 记录 (status: 'running')
      ↓
  加入 Graphile Worker 队列 (EXECUTE_RUN)
      ↓
  Worker 池消费 (TOTAL_CONCURRENCY 限制)
      ↓
  Worker 轮询等待浏览器就绪 (最多 45s)
      ↓
  浏览器初始化完成 (status: 'ready')
      ↓
  工作流执行 (Interpreter)
      ↓  [内部并发: maxConcurrency = 5]
  任务完成 / 失败
      ↓
  销毁浏览器，释放资源
      ↓
  Worker 释放，可消费下一个任务
```

#### 路径 B：无可用浏览器槽位

```
用户请求 / 定时调度
      ↓
  检查浏览器槽位 → 不可用
      ↓
  创建 Run 记录 (status: 'queued')
      ↓
  返回 queued: true 给用户
      ↓
  [每 5 秒 processQueuedRuns 轮询]
      ↓
  槽位释放后，重复路径 A
```

### 4.2 排队状态流转

```
用户请求
    ↓
hasAvailableBrowserSlots?
    ├─ 是 → reserved → running → (Worker 执行) → success/failed
    └─ 否 → queued → [等待 5s 轮询] → [有槽位] → running → ...
                ↓
              用户中止 → aborting → aborted
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

### 4.4 关键交互点

#### 4.4.1 应用层排队与 Worker 队列的交互

- **用户请求时**：先检查浏览器槽位，决定走直接入队还是应用层排队
- **排队恢复时**：`processQueuedRuns` 每 5 秒检查一次，有槽位则创建浏览器并入队
- **崩溃恢复时**：`recoverOrphanedRuns` 将无浏览器的运行重新标记为 `queued`

#### 4.4.2 任务队列与浏览器池的交互

- **创建运行时**：调用 `createRemoteBrowserForRun` 预约浏览器槽位，然后将任务加入队列
- **任务执行时**：Worker 从队列取出任务，轮询等待浏览器就绪（检查 status 和 browser 对象）
- **超时时**：浏览器初始化超过 45 秒未就绪，抛出错误并释放资源

#### 4.4.3 浏览器池与解释器的交互

- 每个浏览器实例（`RemoteBrowser`）包含一个解释器（`Interpreter`）
- 解释器内部的并发控制独立于全局并发控制
- 解释器的 `maxConcurrency` 控制单个工作流内的并行操作数

---

## 五、代码缺陷与改进建议

### 5.1 已发现的代码缺陷

**缺陷 1：`initializing` 状态从未被设置**

- **位置**：[BrowserPool.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts)
- **问题**：类型定义包含 `"initializing"` 状态，但代码中从未实际设置
- **影响**：`cleanupStaleBrowserSlots` 中检查 `status === "initializing"` 的逻辑永远不会触发
- **建议**：要么在 `initializeBrowserAsync` 开始时设置 `status = "initializing"`，要么从类型定义和清理逻辑中移除该状态

**缺陷 2：浏览器初始化异步进行，但状态不更新**

- **问题**：`reserveBrowserSlotAtomic` 设置 `status = "reserved"` 后，调用 `initializeBrowserAsync` 异步初始化
- **影响**：在初始化过程中，状态一直是 `reserved`，无法区分"已预约未开始初始化"和"正在初始化中"
- **建议**：在 `initializeBrowserAsync` 开始时将状态更新为 `"initializing"`

### 5.2 潜在风险

1. **排队任务无超时**：`queued` 状态的任务没有超时机制，可能永远排队
2. **重试次数硬编码**：崩溃恢复的重试次数（3次）硬编码在代码中
3. **用户级资源隔离不足**：一个用户占满 2 个浏览器槽位后，其他任务只能排队，没有优先级机制
