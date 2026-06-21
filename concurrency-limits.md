# 多任务并发与资源限额机制分析

## 概述

Maxun 项目采用**三层并发控制架构**来管理任务执行和资源使用：

1. **全局任务队列层**：基于 Graphile Worker 的 PostgreSQL 任务队列，控制系统级并发
2. **浏览器资源池层**：BrowserPool 管理浏览器实例，实施"1 用户 - 2 浏览器"策略
3. **工作流内部并发层**：Concurrency 类管理单个工作流内的多任务并发执行

---

## 一、任务排队机制

### 1.1 Graphile Worker 任务队列

系统使用 [Graphile Worker](https://github.com/graphile/worker) 作为基于 PostgreSQL 的任务队列，实现任务的持久化排队和调度。

**核心文件**：[graphileWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/storage/graphileWorker.ts)

```typescript
// 任务入队函数
export async function addJob(
  taskIdentifier: string,
  payload: Record<string, unknown>,
  options?: { maxAttempts?: number; runAt?: Date; jobKey?: string },
): Promise<string>
```

**关键特性**：
- 任务持久化存储在 PostgreSQL 中
- 支持任务重试（`maxAttempts` 参数）
- 支持定时执行（`runAt` 参数）
- 支持任务去重（`jobKey` 参数）

### 1.2 任务类型定义

**核心文件**：[task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts#L37-L45)

```typescript
export const QUEUE_NAMES = {
  INITIALIZE_BROWSER_RECORDING: 'initialize-browser-recording',
  DESTROY_BROWSER: 'destroy-browser',
  INTERPRET_WORKFLOW: 'interpret-workflow',
  STOP_INTERPRETATION: 'stop-interpretation',
  EXECUTE_RUN: 'execute-run',
  ABORT_RUN: 'abort-run',
  SCHEDULED_WORKFLOW: 'scheduled-workflow',
} as const;
```

### 1.3 任务入队入口

任务可以从多个路径入队：

| 入口 | 文件 | 说明 |
|------|------|------|
| 定时调度 | [schedule-worker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/schedule-worker.ts) | 轮询数据库找到期任务，加入 SCHEDULED_WORKFLOW 队列 |
| 工作流调度 | [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/workflow-management/scheduler/index.ts) | 调度运行时创建任务，加入 EXECUTE_RUN 队列 |
| API 触发 | [storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts) | REST API 调用直接入队 |
| SDK API | [record.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/api/record.ts) | SDK API 调用入队 |

### 1.4 定时调度流程

**核心文件**：[schedule-worker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/schedule-worker.ts)

调度流程：
1. 每 30 秒轮询一次（`DB_SCHEDULER_POLL_MS = 30000`）
2. 使用 PostgreSQL 咨询锁（`pg_try_advisory_xact_lock`）保证多实例安全
3. 批量获取到期任务（`DB_SCHEDULER_BATCH_SIZE = 10`）
4. 声明（claim）任务并更新 `schedulerClaimedAt` 时间戳
5. 将任务加入 Graphile Worker 队列
6. 成功后更新 `nextRunAt`，失败则释放声明

---

## 二、并发控制机制

### 2.1 全局 Worker 并发数

**核心文件**：[task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts#L80)

```typescript
const TOTAL_CONCURRENCY = Math.max(1, parseInt(process.env.WORKER_CONCURRENCY || '10', 10));
```

**配置说明**：
- 环境变量 `WORKER_CONCURRENCY` 控制总并发数
- 默认值为 10
- 最小值为 1
- 对应 Graphile Worker 的 `concurrency` 参数

**Worker 启动配置**：
```typescript
runner = await run({
  pgPool: runnerPool,
  concurrency: TOTAL_CONCURRENCY,
  noHandleSignals: true,
  pollInterval: 3600000,  // 1小时轮询间隔（依赖监听通知）
  taskList,
});
```

### 2.2 浏览器资源池（BrowserPool）

**核心文件**：[BrowserPool.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts)

#### 2.2.1 资源限额策略

**"1 用户 - 2 浏览器"策略**：
- 每个用户最多拥有 2 个浏览器实例
- 每个用户最多只能有 1 个处于 "recording" 状态的浏览器
- "run" 状态的浏览器可以有 1-2 个（取决于是否还有 recording 浏览器）

**限额检查**：[hasAvailableBrowserSlots](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts#L550-L565)

```typescript
public hasAvailableBrowserSlots = (userId: string, state?: BrowserState): boolean => {
  const userBrowserIds = this.userToBrowserMap.get(userId) || [];
  
  if (userBrowserIds.length >= 2) {
    return false;
  }
  
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

为防止并发请求导致的资源超限，BrowserPool 实现了**原子预约**机制：

**核心方法**：[reserveBrowserSlotAtomic](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts#L595-L646)

```typescript
public reserveBrowserSlotAtomic = (id: string, userId: string, state: BrowserState = "run"): boolean => {
  const lockKey = `${userId}-${state}`;
  
  if (this.reservationLocks.has(lockKey)) {
    return false;
  }
  
  try {
    this.reservationLocks.set(lockKey, Date.now());
    
    if (!this.hasAvailableBrowserSlots(userId, state)) {
      return false;
    }
    
    // 创建 reserved 状态的槽位
    this.pool[id] = {
      browser: null,
      active: false,
      userId,
      state,
      status: "reserved",
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

**浏览器槽位状态**：
| 状态 | 说明 |
|------|------|
| `reserved` | 已预约，浏览器实例尚未创建 |
| `initializing` | 初始化中 |
| `ready` | 就绪，可使用 |
| `failed` | 初始化失败 |

**状态流转**：
```
reserved → initializing → ready
                ↓
              failed
```

#### 2.2.3 槽位升级与失败处理

- **升级槽位**：[upgradeBrowserSlot](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts#L655-L670) - 将 `reserved` 状态升级为 `ready`
- **标记失败**：[failBrowserSlot](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts#L677-L696) - 失败时清理资源并移除槽位

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
// 用户触发的并发由 Concurrency 管理器完全控制
this.concurrency.addJob(() => this.runLoop(popup, workflowCopy));
```

#### 2.4.3 主循环启动

```typescript
// 添加主循环任务
this.concurrency.addJob(() => this.runLoop(page, this.initializedWorkflow!));

// 等待所有并发任务完成
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

### 3.2 异常时的资源释放

#### 3.2.1 执行失败时释放

**核心文件**：[task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts)

在 `processRunExecution` 函数中，无论成功或失败，最终都会调用 `destroyRemoteBrowser`：

```typescript
// 成功路径
await destroyRemoteBrowser(browserId, data.userId);

// 失败路径
try { if (browser && browser.interpreter) await browser.interpreter.clearState(); } catch (_) {}
await destroyRemoteBrowser(browserId, data.userId);
```

#### 3.2.2 中止运行时释放

**核心函数**：[abortRun](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts#L584-L629)

```typescript
await new Promise(resolve => setTimeout(resolve, 500));
await destroyRemoteBrowser(plainRun.browserId, userId);
```

### 3.3 录制超时自动释放

**核心文件**：[controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/controller.ts#L14-L15)

```typescript
const RECORDING_TIMEOUT_MS = 10 * 60 * 1000; // 10分钟
const recordingTimeouts = new Map<string, NodeJS.Timeout>();
```

录制浏览器在创建时设置 10 分钟超时，超时后自动销毁：

```typescript
const timeoutHandle = setTimeout(async () => {
  recordingTimeouts.delete(id);
  io.of(id).emit('recording-timeout');
  await new Promise(resolve => setTimeout(resolve, 1000)); // 等待前端接收事件
  await destroyRemoteBrowser(id, userId);
}, RECORDING_TIMEOUT_MS);
```

### 3.4 过期槽位清理

**核心方法**：[cleanupStaleBrowserSlots](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts#L702-L729)

防止浏览器初始化失败或卡住导致的资源泄漏：

- **清理阈值**：5 分钟（`staleThreshold = 5 * 60 * 1000`）
- **清理条件**：状态为 `reserved` 或 `initializing`，且 `browser === null`
- **同时清理**：过期的预约锁（超过 1 分钟）

### 3.5 解释器状态清理

当运行中止或失败时，解释器需要清理状态：

```typescript
await browser.interpreter.clearState();
```

---

## 四、各层配合关系

### 4.1 完整任务执行链路

```
用户请求 / 定时调度
      ↓
  任务入队 (Graphile Worker)
      ↓
  Worker 池消费 (TOTAL_CONCURRENCY)
      ↓
  创建浏览器槽位 (BrowserPool)
      ↓  [资源限额: 1用户-2浏览器]
  浏览器初始化 (异步)
      ↓
  工作流执行 (Interpreter)
      ↓  [内部并发: maxConcurrency]
  任务完成 / 失败
      ↓
  销毁浏览器，释放资源
      ↓
  Worker 释放，可消费下一个任务
```

### 4.2 并发层级汇总

| 层级 | 控制对象 | 限额配置 | 默认值 | 所在文件 |
|------|----------|----------|--------|----------|
| 全局任务 | Worker 并发数 | `WORKER_CONCURRENCY` | 10 | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts#L80) |
| 用户级 | 浏览器实例数 | 硬编码策略 | 2/用户 | [BrowserPool.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts) |
| 用户级 | 录制浏览器数 | 硬编码策略 | 1/用户 | [BrowserPool.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts) |
| 工作流内 | 内部任务并发 | `maxConcurrency` | 5 | [interpret.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/maxun-core/src/interpret.ts#L102) |
| 数据库连接 | Worker 池大小 | `TOTAL_CONCURRENCY + 2` | 12 | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts#L692) |
| 数据库连接 | Utils 池大小 | 硬编码 | 3 | [graphileWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/storage/graphileWorker.ts#L28) |

### 4.3 关键交互点

#### 4.3.1 任务队列与浏览器池的交互

- **创建运行时**：在 `createWorkflowAndStoreMetadata` 中调用 `createRemoteBrowserForRun` 预约浏览器槽位，然后将任务加入队列
- **任务执行时**：Worker 从队列取出任务，等待浏览器就绪（轮询检查槽位状态），然后执行工作流

#### 4.3.2 浏览器池与解释器的交互

- 每个浏览器实例（`RemoteBrowser`）包含一个解释器（`Interpreter`）
- 解释器内部的并发控制独立于全局并发控制
- 解释器的 `maxConcurrency` 控制单个工作流内的并行操作数

---

## 五、关键代码路径索引

### 5.1 任务排队路径

| 操作 | 入口文件 | 核心函数/方法 |
|------|----------|--------------|
| 定时任务入队 | [schedule-worker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/schedule-worker.ts) | `processDueSchedules` → `claimDueDbSchedules` → `addJob` |
| 手动运行入队 | [storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/routes/storage.ts) | API 端点 → `addJob(QUEUE_NAMES.EXECUTE_RUN)` |
| 调度器入队 | [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/workflow-management/scheduler/index.ts) | `createWorkflowAndStoreMetadata` → `addJob` |

### 5.2 并发控制路径

| 层级 | 文件 | 关键代码 |
|------|------|----------|
| 全局 Worker | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts) | `TOTAL_CONCURRENCY` 变量、`startWorkers` 函数 |
| 浏览器池 | [BrowserPool.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts) | `reserveBrowserSlotAtomic`、`hasAvailableBrowserSlots` |
| 工作流内部 | [concurrency.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/maxun-core/src/utils/concurrency.ts) | `addJob`、`runNextJob` |

### 5.3 资源释放路径

| 触发场景 | 文件 | 关键函数 |
|----------|------|----------|
| 正常完成 | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts) | `processRunExecution` 末尾 → `destroyRemoteBrowser` |
| 执行失败 | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts) | catch 块 → `destroyRemoteBrowser` |
| 用户中止 | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/task-runner.ts) | `abortRun` → `destroyRemoteBrowser` |
| 录制超时 | [controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/controller.ts) | `setTimeout` 回调 → `destroyRemoteBrowser` |
| 槽位过期 | [BrowserPool.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/classes/BrowserPool.ts) | `cleanupStaleBrowserSlots` → `failBrowserSlot` |
| 浏览器创建失败 | [controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/112-maxun/server/src/browser-management/controller.ts) | `initializeBrowserAsync` catch 块 → `browserPool.failBrowserSlot` |
