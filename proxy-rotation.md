# 代理和 IP 池切换代码链路分析

## 1. 架构概览

当前系统**没有实现真正的代理轮换（Proxy Rotation）或 IP 池功能**。每个用户只能配置一个固定代理，在浏览器会话初始化时应用，没有运行时的代理切换或失败自动切换机制。

```
用户配置代理 → 加密存储到User表 → 浏览器初始化时解密 → 应用到BrowserContext → 运行工作流
```

系统有多层级的重试和失败恢复机制，但**所有重试都使用同一个代理配置，不会切换代理**。

---

## 2. 代理配置选择链路

### 2.1 数据模型 - 代理存储

**文件**: `server/src/models/User.ts`

用户模型存储三个代理相关字段，均为加密后的值：

```typescript
interface UserAttributes {
    proxy_url?: string | null;       // 加密后的代理服务器URL
    proxy_username?: string | null;  // 加密后的代理用户名
    proxy_password?: string | null;  // 加密后的代理密码
}
```

### 2.2 代理配置 API

**文件**: `server/src/routes/proxy.ts`

| 端点 | 方法 | 功能 |
|------|------|------|
| `/api/proxy/config` | POST | 保存代理配置（加密存储） |
| `/api/proxy/config` | GET | 获取代理配置（脱敏返回） |
| `/api/proxy/config` | DELETE | 删除代理配置 |
| `/api/proxy/test` | GET | 测试代理连接 |

**关键代码 - 保存代理（加密）**:

```typescript
// 加密后存储
const encryptedProxyUrl = encrypt(server_url);
let encryptedProxyUsername: string | null = null;
let encryptedProxyPassword: string | null = null;

if (username && password) {
    encryptedProxyUsername = encrypt(username);
    encryptedProxyPassword = encrypt(password);
}

await user.update({
    proxy_url: encryptedProxyUrl,
    proxy_username: encryptedProxyUsername,
    proxy_password: encryptedProxyPassword,
});
```

### 2.3 获取解密后的代理配置

**文件**: `server/src/routes/proxy.ts`

核心函数 `getDecryptedProxyConfig` 被多个模块调用：

```typescript
export const getDecryptedProxyConfig = async (userId: string) => {
    const user = await User.findByPk(userId, { raw: true });
    if (!user) throw new Error('User not found');

    return {
        proxy_url: user.proxy_url ? decrypt(user.proxy_url) : null,
        proxy_username: user.proxy_username ? decrypt(user.proxy_username) : null,
        proxy_password: user.proxy_password ? decrypt(user.proxy_password) : null,
    };
};
```

**调用方分布**:
- `server/src/browser-management/classes/RemoteBrowser.ts` — 浏览器初始化时调用
- `server/src/workflow-management/scheduler/index.ts` — 调度运行时调用（仅用于log）
- `server/src/api/record.ts` — API运行时调用（仅用于log）

---

## 3. 浏览器会话应用链路

### 3.1 浏览器初始化时应用代理

**文件**: `server/src/browser-management/classes/RemoteBrowser.ts`

在 `initialize()` 方法中，代理配置在创建 BrowserContext 时应用：

```typescript
public initialize = async (userId: string): Promise<void> => {
    // ...
    // 获取解密的代理配置
    const proxyConfig = await getDecryptedProxyConfig(userId);
    let proxyOptions: { server: string, username?: string, password?: string } = { server: '' };

    if (proxyConfig.proxy_url) {
        proxyOptions = {
            server: proxyConfig.proxy_url,
            ...(proxyConfig.proxy_username && proxyConfig.proxy_password && {
                username: proxyConfig.proxy_username,
                password: proxyConfig.proxy_password,
            }),
        };
    }

    // 构建context选项
    const contextOptions: any = {
        reducedMotion: 'reduce',
        javaScriptEnabled: true,
        timeout: 50000,
        userAgent: this.getUserAgent(),
        // ...
    };

    // 将代理配置注入context
    if (proxyOptions.server) {
        contextOptions.proxy = {
            server: proxyOptions.server,
            username: proxyOptions.username ? proxyOptions.username : undefined,
            password: proxyOptions.password ? proxyOptions.password : undefined,
        };
    }

    // 创建带代理的context
    const contextPromise = this.browser.newContext(contextOptions);
    this.context = await Promise.race([
        contextPromise,
        new Promise<never>((_, reject) => {
            setTimeout(() => reject(new Error('Context creation timed out after 15s')), 15000);
        })
    ]) as BrowserContext;
    // ...
};
```

### 3.2 代理应用时机

**关键发现**：代理仅在以下时机应用，且**一旦应用就不会在运行时更改**：

1. **录制模式** — `server/src/browser-management/controller.ts` 中的 `initializeRemoteBrowserForRecording`
   - 用户启动录制时创建浏览器
   - 调用 `browserSession.initialize(userId)` 应用代理

2. **运行模式** — `server/src/browser-management/controller.ts` 中的 `createRemoteBrowserForRun`
   - 调度或API触发运行时创建浏览器
   - 调用 `initializeBrowserAsync` → `browserSession.initialize(userId)` 应用代理

---

## 4. 失败重试与切换逻辑（全链路详解）

系统有**四级重试 + 多级失败恢复机制**，但**没有任何一级会切换代理**。
所有重试都使用同一个用户代理配置。

### 4.1 第一层：远程浏览器连接重试 + 本地浏览器回退

**文件**: `server/src/browser-management/browserConnection.ts`

这是最底层的连接重试机制，有 3 次重试 + 本地浏览器回退。

```
远程浏览器连接流程:
  ├─ 健康检查获取wsEndpoint
  ├─ 第1次连接 → 失败 → 等待2秒
  ├─ 第2次连接 → 失败 → 等待2秒
  └─ 第3次连接 → 失败
         ↓
    本地浏览器回退（launchLocalBrowser）
         ↓
    成功 / 失败
```

**关键代码 — `connectToRemoteBrowser()`**:

```typescript
const CONNECTION_CONFIG = {
    maxRetries: 3,
    retryDelay: 2000,
    connectionTimeout: 30000,
};

export async function connectToRemoteBrowser(retries?: number): Promise<Browser> {
    const maxRetries = retries ?? CONNECTION_CONFIG.maxRetries;

    try {
        const wsEndpoint = await getBrowserServiceEndpoint();

        // 循环重试连接
        for (let attempt = 1; attempt <= maxRetries; attempt++) {
            try {
                const browser = await chromium.connect(wsEndpoint, {
                    timeout: CONNECTION_CONFIG.connectionTimeout,
                });
                return browser; // 成功
            } catch (error: any) {
                if (attempt === maxRetries) {
                    throw new Error(`Remote connection failed: ${error.message}`);
                }
                await new Promise(resolve => setTimeout(resolve, CONNECTION_CONFIG.retryDelay));
            }
        }

        throw new Error('Failed to connect to browser service');
    } catch (error: any) {
        // 远程连接全部失败，回退到本地浏览器
        return await launchLocalBrowser();
    }
}
```

**本地浏览器回退 — `launchLocalBrowser()`**:

```typescript
async function launchLocalBrowser(): Promise<Browser> {
    const browser = await chromium.launch({
        headless: true,
        args: [
            '--disable-blink-features=AutomationControlled',
            '--disable-web-security',
            '--disable-features=IsolateOrigins,site-per-process',
            '--disable-site-isolation-trials',
            '--disable-extensions',
            '--no-sandbox',
            '--disable-dev-shm-usage',
            '--disable-gpu',
            '--force-color-profile=srgb',
            '--force-device-scale-factor=2',
            '--ignore-certificate-errors',
            '--mute-audio'
        ],
    });
    return browser;
}
```

**代理影响分析**:
- 远程浏览器连接失败时，**不一定是代理的问题**，可能是浏览器服务本身宕机
- 如果是代理本身的问题，回退到本地浏览器后，代理依然会被应用（通过 `context.proxy`）
- 如果是远程浏览器服务宕机，回退到本地浏览器可能绕过故障让运行继续
- 但如果是代理本身故障，回退到本地浏览器也同样会失败（因为代理配置还是同一个）

### 4.2 第二层：浏览器初始化重试

**文件**: `server/src/browser-management/classes/RemoteBrowser.ts`

浏览器初始化（包括 context 创建）有 **3 次重试**，每次重试时重新连接浏览器 + 重新应用同一个代理。

```
初始化流程:
  ├─ 第1次: connectToRemoteBrowser() → newContext() → 失败
  ├─ 第2次: connectToRemoteBrowser() → newContext() → 失败
  └─ 第3次: connectToRemoteBrowser() → newContext() → 失败 → 抛异常
```

**关键代码**:

```typescript
public initialize = async (userId: string): Promise<void> => {
    const MAX_RETRIES = 3;
    let retryCount = 0;
    let success = false;

    while (!success && retryCount < MAX_RETRIES) {
        try {
            // 重新连接浏览器（内部也有3次重试+本地回退）
            this.browser = await connectToRemoteBrowser();
            
            // 每次重试都获取同一个代理配置
            const proxyConfig = await getDecryptedProxyConfig(userId);
            // 构建相同的 proxyOptions
            // ...
            
            // 创建 context
            this.context = await this.browser.newContext(contextOptions);
            // ...
            success = true;
        } catch (error: any) {
            retryCount++;
            if (this.browser) {
                await this.browser.close();
                this.browser = null;
            }
            await new Promise(resolve => setTimeout(resolve, 1000));
        }
    }

    if (!success) {
        throw new Error(`Failed to initialize browser after ${MAX_RETRIES} attempts`);
    }
};
```

**代理影响分析**:
- 3 次重试都使用**同一个代理配置**
- 如果代理本身有问题，3 次重试都会失败
- 每次重试内部的 `connectToRemoteBrowser()` 还有自己的 3 次连接重试 + 本地回退
- 最坏情况下：3 × 3 = 9 次远程连接尝试 + 3 次本地浏览器尝试

### 4.3 第三层：运行级别重试

**文件**: `server/src/workflow-management/scheduler/index.ts`

工作流运行有 **3 次重试** 限制，调度器级别重试会重新创建浏览器，用同样的代理。

```
运行重试流程:
  ├─ 第1次运行 → 失败 → retryCount=1
  ├─ 第2次运行 → 失败 → retryCount=2
  └─ 第3次运行 → 失败 → retryCount=3 → 永久失败
```

**关键代码**:

```typescript
const retryCount = plainRun.retryCount || 0;
if (retryCount >= 3) {
    logger.log('warn', `Scheduled Run ${id} has exceeded max retries (${retryCount}/3), marking as failed`);
    await run.update({
        status: 'failed',
        log: `Max retries exceeded (${retryCount}/3) - Run failed after multiple attempts.`
    });
    return { success: false, error: 'Max retries exceeded' };
}

// 重试时重新创建浏览器（同样的代理）
await run.update({
    status: 'queued',
    retryCount: retryCount + 1,
    // ...
});
```

**代理影响分析**:
- 每次重试都会重新创建浏览器实例
- 但**代理配置不变**，还是用户的同一个代理
- 失败后状态设为 `queued`，由 `processQueuedRuns` 轮询处理

### 4.4 第四层：Graphile Worker 任务队列重试

**文件**: `server/src/schedule-worker.ts` + `server/src/storage/graphileWorker.ts`

调度任务通过 Graphile Worker 入队，有 **6 次重试**（调度任务）或 **1-3 次**（其他任务）。

```
任务重试次数配置:
  - 调度工作流任务: maxAttempts: 6
  - 普通运行任务:   maxAttempts: 1
  - 部分任务:       maxAttempts: 3
```

**关键代码 — 调度任务入队**:

```typescript
// schedule-worker.ts
await addJob(QUEUE_NAMES.SCHEDULED_WORKFLOW, 
    { robotMetaId: robot.robotMetaId, userId: robot.userId }, 
    { maxAttempts: 6 }  // 最多6次重试
);
```

**关键代码 — addJob 函数**:

```typescript
// graphileWorker.ts
export async function addJob(
  taskIdentifier: string,
  payload: Record<string, unknown>,
  options?: { maxAttempts?: number; runAt?: Date; jobKey?: string },
): Promise<string> {
  const job = await workerUtils.addJob(taskIdentifier, payload, options);
  return String(job.id);
}
```

**代理影响分析**:
- Graphile Worker 的重试是**任务级别的重试**
- 每次重试会重新执行整个工作流执行流程
- 但代理配置还是同一个
- 6 次重试后任务永久失败

---

## 5. 服务崩溃后的运行重排队与恢复

### 5.1 服务启动时的孤儿运行恢复

**文件**: `server/src/routes/storage.ts` 中的 `recoverOrphanedRuns()`

服务崩溃后重启时，会扫描处于 `running` 和 `scheduled` 状态的运行，检查浏览器是否已失效，对失效的运行重新入队。

**触发时机**: `server/src/server.ts` 服务启动时调用

```
服务启动恢复流程:
  1. 查找所有 status in ['running', 'scheduled'] 的运行
  2. 对每个运行:
     ├─ 检查浏览器是否还在池中（browserPool.getRemoteBrowser()）
     ├─ 浏览器还在 → 视为有效，跳过
     └─ 浏览器不在 → 视为孤儿，重新入队（最多3次重试）
         ├─ retryCount < 3 → 状态设为 queued，重新排队
         └─ retryCount >= 3 → 标记为 failed，永久失败
```

**关键代码**:

```typescript
export async function recoverOrphanedRuns() {
    const orphanedRuns = await Run.findAll({
        where: { 
            status: ['running', 'scheduled'] 
        },
        order: [['startedAt', 'ASC']]
    });

    for (const run of orphanedRuns) {
        const runData = run.toJSON();
        const browser = browserPool.getRemoteBrowser(runData.browserId);
        
        if (!browser) {
            const retryCount = runData.retryCount || 0;
            
            if (retryCount < 3) {
                await run.update({
                    status: 'queued',
                    retryCount: retryCount + 1,
                    serializableOutput: {},
                    binaryOutput: {},
                    browserId: undefined,
                    log: `[RETRY ${retryCount + 1}/3] Re-queuing due to server crash`
                });
            } else {
                await run.update({
                    status: 'failed',
                    finishedAt: new Date().toLocaleString(),
                    log: `Max retries exceeded (3/3) - Run failed after multiple server crashes.`
                });
            }
        }
    }
}
```

**代理影响分析**:
- 崩溃恢复时重新入队的运行，会重新创建浏览器
- 代理配置不变，还是用户的同一个代理
- 最多 3 次崩溃恢复重试

### 5.2 排队运行的轮询处理

**文件**: `server/src/routes/storage.ts` 中的 `processQueuedRuns()`

每 5 秒轮询一次 `queued` 状态的运行，逐个处理。

**触发时机**: `server/src/server.ts` 中 5 秒 interval

```typescript
const processQueuedRunsInterval = setInterval(async () => {
    await processQueuedRuns();
}, 5000);
```

还有断路器机制：连续 5 次 DB 错误后冷却 30 秒。

### 5.3 浏览器槽位失效清理

**文件**: `server/src/browser-management/classes/BrowserPool.ts`

每分钟清理一次处于 `reserved` 或 `initializing` 状态超过 5 分钟的浏览器槽位。

```
清理触发:
  - 服务启动时执行 1 次
  - 之后每分钟 1 次（setInterval 60秒）
```

**关键代码**:

```typescript
public cleanupStaleBrowserSlots = (): void => {
    const now = Date.now();
    const staleThreshold = 5 * 60 * 1000; // 5 分钟
    
    const staleSlots: string[] = [];
    
    for (const [id, info] of Object.entries(this.pool)) {
        const isStale = info.status === "reserved" || info.status === "initializing";
        const createdAt = info.createdAt || 0;
        const age = now - createdAt;
        
        if (isStale && info.browser === null && age > staleThreshold) {
            staleSlots.push(id);
        }
    }
    
    staleSlots.forEach(id => {
        this.failBrowserSlot(id);
    });
    
    this.cleanupStaleReservationLocks(); // 清理超过 1 分钟的预留锁
};
```

---

## 6. 浏览器池状态管理

**文件**: `server/src/browser-management/classes/BrowserPool.ts`

浏览器池管理每个用户最多 2 个浏览器实例，有完整的状态机。

### 6.1 浏览器状态机

```
reserved → initializing → ready
    ↓          ↓
  failed     failed
```

| 状态 | 说明 |
|------|------|
| `reserved` | 槽位已预留，浏览器即将初始化 |
| `initializing` | 浏览器正在初始化（连接中） |
| `ready` | 浏览器就绪可用 |
| `failed` | 浏览器初始化失败 |

### 6.2 原子预留机制

为了防止并发创建导致的超发，有原子预留锁：

```typescript
public reserveBrowserSlotAtomic = (id: string, userId: string, state: BrowserState): boolean => {
    const lockKey = `${userId}-${state}`;
    
    if (this.reservationLocks.has(lockKey)) {
        return false; // 已有预留进行中
    }
    
    try {
        this.reservationLocks.set(lockKey, Date.now());
        
        if (!this.hasAvailableBrowserSlots(userId, state)) {
            return false;
        }

        // 预留槽位
        this.pool[id] = {
            browser: null,
            active: false,
            userId,
            state,
            status: "reserved",
            createdAt: now,
            lastAccessed: now,
        };
        // ...
        return true;
    } finally {
        this.reservationLocks.delete(lockKey);
    }
};
```

### 6.3 任务执行时等待浏览器就绪

**文件**: `server/src/task-runner.ts`

任务执行时如果浏览器还没就绪，会轮询等待最多 60 秒。

```typescript
const BROWSER_INIT_TIMEOUT = 60000;
const MAX_POLL_ATTEMPTS = Math.ceil(BROWSER_INIT_TIMEOUT / 2000); // 30次

while (!browser && (Date.now() - browserWaitStart) < BROWSER_INIT_TIMEOUT && pollAttempts < MAX_POLL_ATTEMPTS) {
    const browserStatus = browserPool.getBrowserStatus(browserId);
    if (browserStatus === null) throw new Error(`Browser slot ${browserId} does not exist in pool`);
    if (browserStatus === 'failed') throw new Error(`Browser ${browserId} initialization failed`);
    
    await new Promise(resolve => setTimeout(resolve, 2000));
    browser = browserPool.getRemoteBrowser(browserId);
}
```

---

## 7. 完整失败恢复总览图

### 7.1 全链路失败重试层级关系

```
用户触发运行
    ↓
┌─ Graphile Worker 任务队列 (maxAttempts: 1~6)
│    ↓ 任务执行
│    ├─ 运行级别重试 (retryCount: 0~3)
│    │    ↓ 重新创建浏览器
│    │    ├─ RemoteBrowser.initialize() (3次重试)
│    │    │    ↓ 每次重试内部调用
│    │    │    └─ connectToRemoteBrowser() (3次重试 + 本地回退)
│    │    │         ├─ 远程浏览器连接
│    │    │         │    ├─ 健康检查获取wsEndpoint
│    │    │         │    ├─ 第1次连接
│    │    │         │    ├─ 第2次连接
│    │    │         │    └─ 第3次连接
│    │    │         │         ↓ 全部失败
│    │    │         └─ 本地浏览器回退
│    │    │               ↓ 成功/失败
│    │    └─ Context创建（应用同一个代理配置）
│    └─ 工作流执行
└─ 成功/失败
```

### 7.2 服务崩溃恢复流程

```
服务崩溃
    ↓
服务重启
    ├─ cleanupStaleBrowserSlots()  ← 清理失效浏览器槽位
    ├─ recoverOrphanedRuns()       ← 恢复孤儿运行
    │    ├─ 查找 running/scheduled 状态的运行
    │    ├─ 浏览器还在 → 跳过
    │    └─ 浏览器不在 → 重新入队 (queued)
    │         └─ retryCount >= 3 → 标记为 failed
    └─ processQueuedRuns() 轮询处理 queued 运行
         ↓
    重新创建浏览器（同一个代理）
```

---

## 8. 深度专项分析

### 8.1 SDK 更新代理字段的真实落点分析

**核心发现：SDK 的代理更新功能是死代码，完全不生效。**

**文件**: `server/src/api/sdk.ts`

SDK 的 `PUT /api/sdk/robots/:id` 路由中，接收 `proxy_url`、`proxy_username`、`proxy_password` 参数并赋值给 `updateData`：

```typescript
if (updates.proxy_url !== undefined) {
    updateData.proxy_url = updates.proxy_url;
}
if (updates.proxy_username !== undefined) {
    updateData.proxy_username = updates.proxy_username;
}
if (updates.proxy_password !== undefined) {
    updateData.proxy_password = updates.proxy_password;
}

await robot.update(updateData);
```

**但 Robot 模型中根本不存在这三个字段**：

```typescript
// server/src/models/Robot.ts — RobotAttributes 接口完整定义：
interface RobotAttributes {
  id: string;
  userId?: number;
  recording_meta: RobotMeta;
  recording: RobotWorkflow;
  google_sheet_email?: string | null;
  google_sheet_name?: string | null;
  google_sheet_id?: string | null;
  google_access_token?: string | null;
  google_refresh_token?: string | null;
  airtable_base_id?: string | null;
  airtable_base_name?: string | null;
  airtable_table_name?: string | null;
  airtable_access_token?: string | null;
  airtable_refresh_token?: string | null;
  schedule?: ScheduleConfig | null;
  airtable_table_id?: string | null;
  webhooks?: WebhookConfig[] | null;
  // ❌ 没有 proxy_url / proxy_username / proxy_password
}
```

**后果**：
1. Sequelize 调用 `robot.update(updateData)` 时，三个代理字段会被**静默丢弃**
2. 数据库的 Robot 表中不存在这些列，不会存储任何值
3. 后续运行时，代理配置仍然从 **User 表** 读取，和 Robot 表完全无关
4. 用户通过 SDK 以为自己给某个机器人配置了特定代理，但实际上**全局代理没变**

**代码调用链验证**：
- 所有代理读取都通过 `getDecryptedProxyConfig(userId)` — 传入的是 `userId`
- 该函数从 `User.findByPk(userId)` 查询，**不使用任何 robotId**
- 不存在 `getDecryptedProxyConfigForRobot(robotId)` 这类函数

---

### 8.2 文档类任务的代理链路：分路径结论

**统一结论：文档类机器人 (doc-extract / doc-parse) 是否创建浏览器、是否应用代理，取决于执行路径，两条路径结论完全不同，不可一概而论。**

| 执行路径 | 是否创建浏览器 | 是否应用代理 | 代理生效与否 |
|---------|---------------|------------|------------|
| **正常执行路径**（调度、API、专用文档端点） | ❌ 否（browserId = uuid() 假 ID） | ❌ 否 | 代理与任务无关 |
| **崩溃恢复排队执行路径**（processQueuedRuns） | ✅ **是**（不必要但真实创建） | ✅ **是**（代理注入 context.proxy） | 代理生效但无人使用 |

---

#### 8.2.1 正常执行路径：完全跳过浏览器与代理

**文件**: `server/src/task-runner.ts` L130-L179

在任务执行入口 `processRunExecution()` 中，文档类型会提前 return，不走到浏览器逻辑：

```typescript
// L155-L165: doc-extract 类型直接 return
if ((plainRun.interpreterSettings as any)?.robotType === 'doc-extract') {
    logger.log('info', `Run ${data.runId} is a document robot — skipping browser, running document extraction`);
    const recording = await Robot.findOne({ where: { 'recording_meta.id': plainRun.robotMetaId }, raw: true });
    // ...
    const { executeDocumentRun } = await import('./utils/document/executeDocumentRun');
    await executeDocumentRun(recording, run, data.userId, serverIo);
    return;  // ✅ 提前返回，不走到浏览器逻辑
}

// L168-L178: doc-parse 类型直接 return
if ((plainRun.interpreterSettings as any)?.robotType === 'doc-parse') {
    logger.log('info', `Run ${data.runId} is a document-parse robot — skipping browser, running document parsing`);
    // ...
    const { executeDocumentParseRun } = await import('./utils/document/executeDocumentParseRun');
    await executeDocumentParseRun(recording, run, data.userId, serverIo);
    return;  // ✅ 提前返回，不走到浏览器逻辑
}
```

**正常路径完整执行流程**：

```
创建 Run 阶段 (browserId = uuid() 假 ID):
  ├─ 调度: scheduler/index.ts L74: browserId = isDocRobot ? uuid() : createRemoteBrowserForRun(userId)
  ├─ API: record.ts L585: browserId = isDocRobot ? uuid() : createRemoteBrowserForRun(userId)
  └─ 专用文档端点: storage.ts L2038 / L2094: browserId = uuid()

       ↓

addJob(EXECUTE_RUN)
       ↓

processRunExecution(data)
    ├─ doc-extract → executeDocumentRun() → return ❌ 无浏览器无代理
    ├─ doc-parse → executeDocumentParseRun() → return ❌ 无浏览器无代理
    └─ 其他类型 → browserPool.getRemoteBrowser() → ✅ 有浏览器有代理
```

**注意事项**：
- 正常路径下，即使调度的 `createWorkflowAndStoreMetadata` 调用了 `getDecryptedProxyConfig(userId)`（scheduler L58 / record L572），返回结果也只是赋值给本地变量 `proxyOptions`，**没有注入到任何 BrowserContext**
- 文档机器人仍然会被 `QUEUE_NAMES.EXECUTE_RUN` 任务队列处理，`maxAttempts: 1`
- 如果文档机器人处理需要联网的文档（例如通过 URL 拉取 PDF），**这些网络请求走宿主机默认网络，不受代理控制**

---

#### 8.2.2 崩溃恢复排队执行路径：错误创建浏览器并初始化代理

**文件**: `server/src/routes/storage.ts` L1493-L1566

崩溃恢复流程中，`processQueuedRuns` 不区分文档任务和浏览器任务，一律创建真实浏览器：

```typescript
// processQueuedRuns() L1493-L1555
// ❌ 没有任何 isDocRobot / robotType 判断

async function processQueuedRuns() {
  // ...
  const queuedRun = await Run.findOne({ where: { status: 'queued' }, ... });
  const userId = queuedRun.runByUserId;  // ⚠️ 调度路径可能为 null

  const canCreateBrowser = await browserPool.hasAvailableBrowserSlots(userId, "run");
  // ❌ 只检查浏览器槽位，不检查是否为文档任务

  if (canCreateBrowser) {
    try {
      const newBrowserId = await createRemoteBrowserForRun(userId);  // ✅ 创建真实浏览器
      // createRemoteBrowserForRun → initializeBrowserAsync → RemoteBrowser.initialize():
      //   → connectToRemoteBrowser()              // L1: 连接浏览器
      //   → getDecryptedProxyConfig(userId)        // L2: 读取并解密代理
      //   → browser.newContext({ proxy })          // L3: 代理注入到 BrowserContext

      await queuedRun.update({ status: 'running', browserId: newBrowserId });
      await addJob(QUEUE_NAMES.EXECUTE_RUN, { userId, runId, browserId: newBrowserId });
    } catch (browserError) { ... }
  }
}
```

**然后 `processRunExecution` 才提前 return，浏览器已被创建但无人使用**：

```
processRunExecution(data)
    └─ robotType === 'doc-extract' → executeDocumentRun() → return
        ❌ 浏览器已在 processQueuedRuns 中创建完毕，代理已注入到 context.proxy
        ❌ 但这个浏览器在整个执行过程中从未被访问，也从未被清理
```

---

#### 8.2.3 两条路径对恢复流程图和重试边界的影响

**对崩溃恢复流程图的影响**：

```
服务崩溃 → 重启 → recoverOrphanedRuns()
    ├─ browserId 是 uuid() 假 ID → 池中不存在 → 视为孤儿
    └─ status = 'queued'
         ↓
    processQueuedRuns()
        ├─ [正常任务] → createRemoteBrowserForRun() → 正常（浏览器+代理）
        └─ [文档任务] → createRemoteBrowserForRun() → ⚠️ 不必要但真实创建了浏览器+代理
             ↓
        processRunExecution()
            ├─ [正常任务] → 使用 browserPool.getRemoteBrowser(browserId) → 正确
            └─ [文档任务] → 提前 return → ❌ 浏览器泄漏 + 代理无人使用
```

**对重试边界的影响**：

| 重试层级 | 正常路径文档任务 | 恢复路径文档任务 |
|---------|----------------|----------------|
| **L1 浏览器连接重试** (connectToRemoteBrowser, 3次) | ❌ 不触发 | ✅ 触发（真实连接浏览器） |
| **L2 代理层** (newContext({ proxy })) | ❌ 不触发 | ✅ 触发（代理注入 context） |
| **L3 RemoteBrowser.initialize() 重试** (3次) | ❌ 不触发 | ✅ 触发（浏览器初始化含代理） |
| **L4 运行级别 retryCount** (上限3) | ✅ 触发（重新入队 queued → 再次触发 processQueuedRuns → 再次创建浏览器！） | ✅ 触发（同上） |
| **Graphile Worker maxAttempts** (EXECUTE_RUN=1) | ✅ 触发（仅重试 executeDocumentRun） | ✅ 触发（重试整个 processRunExecution，但浏览器已创建） |

**关键风险**：文档任务的运行级别重试每次都会触发 `processQueuedRuns`，每次都会创建一个新的带代理的浏览器实例，但永远不会被使用。**3 次重试 = 泄漏 3 个浏览器槽位 + 3 个代理连接名额**。

---

### 8.3 任务队列重试与运行重试的边界分析

系统存在两套独立的重试机制，分别位于不同层级，用途不同，边界清晰：

#### 8.3.1 层级一：Graphile Worker 任务队列重试

**位置**: `server/src/storage/graphileWorker.ts` + 各 `addJob()` 调用点

**作用域**: 整个任务执行函数，用于基础设施级故障（DB 宕机、进程崩溃）

**配置方式**: 调用 `addJob()` 时传入 `{ maxAttempts: N }`

| 任务类型 | 队列名 | maxAttempts | 说明 |
|---------|--------|-------------|------|
| 调度触发工作流 | `SCHEDULED_WORKFLOW` | 6 | schedule-worker.ts 中入队 |
| 直接执行运行 | `EXECUTE_RUN` | 1 | 所有直接运行场景 |
| 文档运行(直接触发) | `EXECUTE_RUN` | 1 | scheduler/index.ts, storage.ts, record.ts |

**EXECUTE_RUN 的 maxAttempts 统一为 1** 的证据：

```typescript
// scheduler/index.ts L116: 文档机器人
await addJob(QUEUE_NAMES.EXECUTE_RUN, { userId, runId, browserId }, { maxAttempts: 1 });

// storage.ts L1069: 手动运行
const jobId = await addJob(QUEUE_NAMES.EXECUTE_RUN, { userId, runId, browserId }, { maxAttempts: 1 });

// storage.ts L1539: processQueuedRuns 重排队
const jobId = await addJob(QUEUE_NAMES.EXECUTE_RUN, { userId, runId, browserId }, { maxAttempts: 1 });

// record.ts L632: API 运行
await addJob(QUEUE_NAMES.EXECUTE_RUN, { userId, runId, browserId }, { maxAttempts: 1 });
```

**触发条件**:
- 任务执行函数抛出未捕获异常
- Worker 进程在任务执行中崩溃
- Worker 与 PostgreSQL 连接中断后恢复

#### 8.3.2 层级二：运行级别重试 (run.retryCount)

**位置**: `server/src/workflow-management/scheduler/index.ts` L216-L246

**作用域**: 单个 Run 记录，用于业务级故障（浏览器初始化失败、目标网站访问失败）

**配置方式**: 硬编码上限为 3，写在调度逻辑里

**触发时机 — 在 `createWorkflowAndStoreMetadata()` 中调用执行前检查**：

```typescript
const retryCount = plainRun.retryCount || 0;
if (retryCount >= 3) {
    // 直接标记为永久失败
    await run.update({ status: 'failed', log: 'Max retries exceeded (3/3)' });
    return { success: false, error: 'Max retries exceeded' };
}
```

**状态流转路径**：

```
第一次执行:
  创建 Run → status='scheduled' → retryCount=0
    → createWorkflowAndStoreMetadata()
      → 成功: completed
      → 失败: 检查 retryCount < 3 → status='queued', retryCount=1

processQueuedRuns() 每5秒轮询:
  找到 queued Run → createRemoteBrowserForRun() → status='running'
    → addJob(EXECUTE_RUN)
      → 成功: completed
      → 失败: 被 catch 住 → status='queued', retryCount=2

再次轮询:
  同上，retryCount++ 直到 =3
    → 下次进入时直接标记 failed
```

#### 8.3.3 边界总结

| 维度 | 任务队列重试 (maxAttempts) | 运行级别重试 (retryCount) |
|------|---------------------------|--------------------------|
| **计数器存储** | Graphile Worker jobs 表 (PostgreSQL) | 业务 Run 表 |
| **重试对象** | 任务处理函数整体 | 单个 Run 记录 |
| **上限配置** | addJob 参数指定 (1~6) | 硬编码 3 |
| **触发动作** | Worker 重跑同一个 payload | Run 状态改为 queued，重新走创建浏览器流程 |
| **是否新建浏览器** | 否，用 payload 里的 browserId 找现有浏览器 | 是，调用 createRemoteBrowserForRun() 重新创建 |
| **是否重新应用代理** | 否，浏览器 context 已创建 | 是，新建浏览器时重新从 User 表取代理 |
| **失败来源** | 进程崩溃、DB 故障等基础设施故障 | 浏览器崩溃、目标站点访问失败等业务故障 |
| **代理变更机会** | ❌ 无 | ⚠️ 有，但始终取同一个 User 的代理 |

---

### 8.4 浏览器服务连接重试与目标代理应用的层级区别

这是两个**完全独立**的层级，不要混淆：

#### 8.4.1 层级 A：浏览器服务连接（与代理无关）

**文件**: `server/src/browser-management/browserConnection.ts`

**职责**: 建立 Node.js 进程到 Playwright 浏览器引擎（远程 WebSocket 或本地进程）的**控制连接**

```
Node.js (服务端)
   │
   │  L1 连接：chromium.connect(wsEndpoint)
   │         或 chromium.launch() 本地启动
   ▼
Playwright Browser 对象
   │
   │  L2 应用：browser.newContext({ proxy: ... })
   ▼
BrowserContext + 代理配置
   │
   ▼
访问目标网站（通过代理）
```

**L1 连接重试参数**（`CONNECTION_CONFIG`）：

```typescript
{
    maxRetries: 3,           // 最多 3 次连接尝试
    retryDelay: 2000,        // 每次重试间隔 2 秒
    connectionTimeout: 30000 // 单次连接超时 30 秒
}
```

**L1 连接失败的典型原因**：
- 远程浏览器 Docker 容器未启动（健康检查 `http://localhost:3002/health` 不通）
- WebSocket 端口被防火墙拦截
- 浏览器服务资源耗尽，无法接受新连接
- Playwright 版本不兼容导致 `chromium.connect()` 握手失败

**L1 回退路径**: 3 次远程连接失败后，自动调用 `launchLocalBrowser()` 本地启动 Chromium

#### 8.4.2 层级 B：目标代理应用（L1 成功之后才会执行）

**文件**: `server/src/browser-management/classes/RemoteBrowser.ts` L483-L518

**职责**: 告诉已经连接好的浏览器引擎："**当你访问外部网站时，请通过这个代理服务器**"

**触发顺序**:

```
connectToRemoteBrowser()    ← L1 成功，拿到 Browser 对象
    ↓
getDecryptedProxyConfig()   ← 从 User 表解密代理配置
    ↓
构建 contextOptions.proxy   ← 组装代理参数
    ↓
browser.newContext({ proxy: { server, username, password } })  ← 真正生效
```

**L2 代理失败的典型原因**：
- 代理服务器 IP / 端口错误
- 代理用户名 / 密码认证失败
- 代理协议不匹配（要求 SOCKS5 却给了 HTTP）
- 代理服务器本身宕机或带宽耗尽
- 目标网站封禁了该代理的出口 IP

**L2 重试机制**:
- RemoteBrowser.initialize() 内 3 次重试
- 但每一次重试都调用 `getDecryptedProxyConfig(userId)` — 同一个用户，同一个代理
- 3 次全部失败后抛异常，外层运行重试可能再次创建浏览器（但代理依然相同）

#### 8.4.3 层级影响矩阵

| 故障场景 | L1 连接层 | L2 代理层 | 是否回退本地浏览器 | 运行重试是否可能成功 |
|---------|----------|----------|-------------------|--------------------|
| 远程浏览器服务宕机 | ❌ 失败 | 不执行 | ✅ 自动回退本地 | ⚠️ 取决于本地能否启动 Chromium |
| 代理服务器宕机 | ✅ 正常 | ❌ 失败 | ❌ 本地浏览器也用同一个代理 | ❌ 所有重试都用同一个坏代理 |
| 代理认证失败 | ✅ 正常 | ❌ 失败 | ❌ 同上 | ❌ 同上 |
| 目标网站封禁代理IP | ✅ 正常 | ✅ 上下文创建成功 | ❌ 上下文已创建 | ❌ 同上，出口 IP 没变 |
| Docker 网络中断 | ❌ 失败 | 不执行 | ✅ 回退本地（本地网络可能通） | ⚠️ 本地回退成功就可能成功 |
| Playwright 版本冲突 | ❌ 失败 | 不执行 | ✅ 回退本地（同版本二进制） | ⚠️ 取决于本地二进制 |

---

## 9. 文档类任务深度分析：资源泄漏与用户标识风险

> 本节是 8.2 节的补充，聚焦于代理链路结论之外的深度问题：资源泄漏细节、用户标识来源风险，以及两条路径的完整流程对比。

### 9.1 正常 vs 恢复路径差异对比（与 8.2 对应，不含重复结论）

| 维度 | 正常执行 | 崩溃恢复排队执行 |
|------|---------|----------------|
| **browserId 来源** | `uuid()` 假 ID | `createRemoteBrowserForRun()` 创建真实浏览器 |
| **代理读取** | `getDecryptedProxyConfig()` 被调用但结果仅赋值给局部变量 | `getDecryptedProxyConfig()` 被调用，代理注入到 `context.proxy` |
| **浏览器清理** | 不需要 | ❌ **不会清理**，槽位泄漏（详见 9.2） |
| **浏览器槽位占用** | 不占用 | ⚠️ **占用一个槽位**（每个用户上限2个） |
| **`runByUserId`** | API 路径有值；**调度路径缺失** | 依赖 `queuedRun.runByUserId`，调度路径为 null |
| **网络请求** | MinIO + LLM，走宿主机默认网络 | 同左，代理与文档处理无关 |

### 9.2 文档分支提前返回后的资源泄漏细节

文档分支 `return` 后，以下资源**无人管理**：

**问题一：崩溃恢复创建的浏览器实例无法被清理**

```
processQueuedRuns() 创建浏览器 → 浏览器池中 reserved → initializing → ready
  ↓
processRunExecution() 文档分支 return
  ↓
浏览器实例在池中永远处于 ready 状态，无人使用，无人销毁
  ↓
占用用户的浏览器槽位（上限2个），可能导致后续运行无法创建新浏览器
  ↓
只能等待 cleanupStaleBrowserSlots() 在 5 分钟后清理
  但 cleanupStaleBrowserSlots 只清理 status=reserved/initializing 且 browser=null 的槽位
  文档任务创建的浏览器已经 ready 且 browser≠null → ❌ 不会被清理！
```

**代码证据** — `server/src/browser-management/classes/BrowserPool.ts`：

```typescript
public cleanupStaleBrowserSlots = (): void => {
    const staleThreshold = 5 * 60 * 1000; // 5 分钟
    for (const [id, info] of Object.entries(this.pool)) {
        const isStale = info.status === "reserved" || info.status === "initializing";
        const age = now - (info.createdAt || 0);
        // ❗ 只清理 reserved/initializing 且 browser === null 的情况
        // ❗ ready 状态 + browser !== null 的文档浏览器永远不会被清理
        if (isStale && info.browser === null && age > staleThreshold) {
            this.failBrowserSlot(id);
        }
    }
};
```

**问题二：代理配置已应用但无人使用**

崩溃恢复时，代理被初始化到 BrowserContext。但文档任务不会访问任何网页，代理连接处于空闲状态。如果代理服务器有连接数限制，这会**白白占用一个代理连接名额**。

**问题三：文档执行函数内的资源清理盲区**

`executeDocumentRun()` 和 `executeDocumentParseRun()` 自身的资源管理是正确的：
- 成功时：更新 Run 状态 → emit socket → send webhook
- 失败时：更新 Run 状态为 failed → emit socket（失败事件）
- 没有需要清理的浏览器资源（正常路径本来就没创建浏览器）

但在崩溃恢复路径中，这些函数**不知道外部已经创建了浏览器**，所以不会清理。

### 9.3 用户标识来源风险：`runByUserId` 在调度路径中缺失

**核心风险**：调度器创建的运行记录**没有设置 `runByUserId`**，但 `processQueuedRuns` 依赖这个字段。

**各入口 `runByUserId` 赋值情况**：

| 入口 | 文件 | `runByUserId` 赋值 |
|------|------|-------------------|
| 调度触发 | `scheduler/index.ts` L77-L92 | ❌ **未设置**（`Run.create` 无此字段） |
| 手动运行(有槽位) | `storage.ts` L1038-L1052 | ✅ `req.user.id` |
| 手动运行(排队) | `storage.ts` L1103-L1117 | ✅ `req.user.id` |
| API 运行 | `record.ts` L589-L608 | ✅ `userId`（从 `req.user.id` 传入） |
| 文档提取端点 | `storage.ts` L2031-L2045 | ✅ `req.user.id` |
| 文档解析端点 | `storage.ts` L2087-L2101 | ✅ `req.user.id` |

**风险链路**：

```
调度触发 → Run.create (无 runByUserId) → 服务崩溃 → recoverOrphanedRuns()
  → status='queued' → processQueuedRuns()
  → const userId = queuedRun.runByUserId     ← undefined / null
  → browserPool.hasAvailableBrowserSlots(undefined, "run")
      ↓
  如果 hasAvailableBrowserSlots 不校验 userId：
    → createRemoteBrowserForRun(undefined)   ← L115: throw Error('userId is required')
    → 浏览器创建失败 → 运行被标记为 failed
  如果 hasAvailableBrowserSlots 校验 userId 为空返回 false：
    → 永远无法获得浏览器槽位
    → 文档任务永远停在 queued 状态
    → 但文档任务根本不需要浏览器！
```

**两个问题叠加的最终后果**：调度触发的文档任务崩溃恢复后，**要么浏览器创建失败（userId=undefined），要么创建成功但浏览器槽位永久泄漏（ready状态无法被清理）**。

### 9.4 文档类任务两条路径完整流程对比图

```
═══════════════════════════════════════════════════════════════
路径 A：正常执行（调度 / API / 专用文档端点）
═══════════════════════════════════════════════════════════════

创建 Run 阶段:
  ├─ browserId = uuid()                        ← 假 ID，不创建真实浏览器
  ├─ Run.create({ browserId, interpreterSettings: { robotType } })
  └─ addJob(EXECUTE_RUN, { browserId })

任务执行阶段:
  processRunExecution(data)
    ├─ doc-extract → executeDocumentRun()
    │     ├─ MinIO 获取 PDF
    │     ├─ LLM 提取数据
    │     ├─ run.update({ status: 'success/failed' })
    │     └─ return  ✅ 完成
    └─ doc-parse → executeDocumentParseRun()
          ├─ MinIO 获取 PDF
          ├─ 解析数据
          ├─ run.update({ status: 'success/failed' })
          └─ return  ✅ 完成

结果统计:
  浏览器: 0 个    |  代理应用: 0 次    |  槽位占用: 0 个
  代理影响: 完全不涉及代理


═══════════════════════════════════════════════════════════════
路径 B：崩溃恢复排队执行（服务崩溃 → 重启）
═══════════════════════════════════════════════════════════════

服务恢复阶段:
  recoverOrphanedRuns()
    ├─ browserId 是 uuid() 假 ID → 池中不存在 → 视为孤儿
    └─ status = 'queued', browserId = undefined, retryCount++
         ↓
  processQueuedRuns() 每5秒轮询
    ├─ userId = queuedRun.runByUserId     ← ⚠️ 调度路径可能为 null
    ├─ hasAvailableBrowserSlots(userId)
    └─ createRemoteBrowserForRun(userId)  ← 真实创建浏览器（不必要！）
          ├─ connectToRemoteBrowser()     ← L1: 3次连接重试 + 本地回退
          ├─ getDecryptedProxyConfig(userId) ← L2: 读取并解密代理
          └─ newContext({ proxy })        ← L3: 代理注入 BrowserContext
          ↓
    addJob(EXECUTE_RUN, { browserId: newBrowserId })
         ↓
任务执行阶段:
  processRunExecution(data)
    └─ robotType === 'doc-extract' / 'doc-parse'
        └─ executeDocumentRun() / executeDocumentParseRun()
              ├─ MinIO 获取 PDF
              ├─ 解析 / LLM
              ├─ run.update({ status: 'success/failed' })
              └─ return
                  ❌ 浏览器已创建但从未使用
                  ❌ 浏览器实例未被清理（ready 状态无法被 stale 清理）
                  ❌ 代理连接白白占用

结果统计:
  浏览器: 1 个（泄漏）  |  代理应用: 1 次（无用）  |  槽位占用: 1 个（浪费）
  代理影响: 代理已注入 BrowserContext，但文档任务从不访问外网，代理空闲

  如果文档任务触发运行级别重试（retryCount 0→1→2→3）:
    → 每次重试都会再次调用 processQueuedRuns()
    → 每次都会创建新的浏览器+代理
    → 3 次重试 = 泄漏 3 个浏览器槽位 + 3 个代理连接名额
```

---

## 10. 关键设计缺陷分析

### 10.1 无代理轮换 / IP 池机制

- **问题**：每个用户只能配置一个代理，没有多代理池
- **影响**：无法实现 IP 轮换，容易被目标网站封禁
- **代码证据**：User 模型只有单组代理字段，没有 Proxy 或 ProxyPool 表

### 10.2 无运行时代理切换

- **问题**：代理仅在 BrowserContext 创建时应用，运行中无法切换
- **影响**：遇到代理失败时只能重试同一个代理，无法自动切换到备用代理
- **代码证据**：`RemoteBrowser` 没有提供 `changeProxy()` 方法

### 10.3 所有重试都不切换代理

- **问题**：四级重试机制（连接重试、初始化重试、运行重试、任务队列重试）都使用同一个代理
- **影响**：代理本身故障时，重试多少次都不会成功
- **代码证据**：`getDecryptedProxyConfig(userId)` 始终返回同一个配置

### 10.4 本地回退后代理失效问题

- **问题**：远程浏览器连接失败回退到本地浏览器时，代理配置依然会应用
- **但**：如果是代理本身导致的问题，本地浏览器也会同样失败
- **注意**：本地浏览器启动参数里没有 `--proxy-server` 参数，代理是通过 Playwright 的 `context.proxy` 配置的

### 10.5 代理测试接口名不副实

- **问题**：测试接口 `/api/proxy/test` 实际上没有使用代理进行测试
- **代码证据**：
  ```typescript
  // 获取了 proxyOptions，但创建页面时没有传入代理配置
  const browser = await connectToRemoteBrowser();
  const page = await browser.newPage();  // 没有传 proxyOptions！
  await page.goto('https://example.com');
  await browser.close();
  ```

### 10.6 SDK 代理更新是死代码（详细原理见 8.1 节）

- **问题**：SDK 的 `PUT /api/sdk/robots/:id` 虽然接收 `proxy_url` 等参数并赋值给 `updateData`，但 Robot 模型根本没有这些字段
- **影响**：Sequelize 静默丢弃这些值，数据库不存储，运行时也不读取。调用方以为给某个机器人设置了专属代理，实际全局代理没变
- **代码证据**：`RobotAttributes` 接口无代理字段，`getDecryptedProxyConfig()` 只查 User 表

### 10.7 SDK 用户级代理更新不立即生效

**文件**: `server/src/api/sdk.ts`

即使代理字段是正确写到 User 表（修复 SDK bug 之后），但正在运行的浏览器已经初始化完毕，代理配置已固化在 BrowserContext 里。只有**下次重新创建浏览器**时才会应用新的代理配置。

### 10.8 崩溃恢复为文档任务错误创建浏览器（详细原理见 8.2.2 / 9.2 节）

- **问题**：`processQueuedRuns` 不区分文档任务和浏览器任务，一律创建浏览器
- **影响**：文档任务崩溃恢复后，浏览器实例被创建但从未使用，槽位泄漏；代理被初始化到 BrowserContext 但无人消费
- **代码证据**：`processQueuedRuns` 无 `robotType` 检查

### 10.9 调度路径 `runByUserId` 缺失（详细原理见 9.3 节）

- **问题**：`scheduler/index.ts` 中 `Run.create` 不设置 `runByUserId`
- **影响**：崩溃恢复后 `processQueuedRuns` 取到 null，浏览器创建可能失败或行为异常
- **代码证据**：对比 `storage.ts` L1049 有 `runByUserId: req.user.id`，scheduler L77-L92 无此字段

---

## 11. 改进建议

### 11.1 实现真正的代理池

1. 新建 `Proxy` 模型表，支持多代理配置（每个用户可配置 N 个代理）
2. 增加 `ProxyPool` 管理类，实现轮换策略（轮询、随机、最少使用、健康过滤）
3. 每次创建浏览器时从池中选择一个**健康可用**的代理

### 11.2 增加运行时代理切换能力

在 `RemoteBrowser` 中增加方法：

```typescript
public async switchProxy(newProxyConfig: ProxyConfig): Promise<void> {
    await this.context?.close();
    const contextOptions = { ...existingOptions, proxy: newProxyConfig };
    this.context = await this.browser!.newContext(contextOptions);
    this.currentPage = await this.context.newPage();
}
```

### 11.3 失败时自动切换代理

修改重试逻辑，代理失败时从池中选择下一个代理：

```typescript
// 伪代码
while (!success && retryCount < MAX_RETRIES) {
    const proxyConfig = proxyPool.getNextProxy(userId);
    try {
        // 使用新代理初始化
        success = true;
    } catch (error) {
        proxyPool.markProxyFailed(proxyConfig.id);
        retryCount++;
    }
}
```

### 11.4 修复代理测试接口

```typescript
// 修复后的测试逻辑
const context = await browser.newContext({ proxy: proxyOptions });
const page = await context.newPage();
await page.goto('https://example.com');
```

### 11.5 本地回退时考虑代理因素

如果 L2 代理层失败，先尝试下一个代理而不是直接回退到本地浏览器；如果是 L1 连接层失败，再考虑本地回退。

### 11.6 修复 `processQueuedRuns` 对文档任务的处理

`processQueuedRuns` 应在创建浏览器前检查 `robotType`，文档任务直接入队执行，无需浏览器：

```typescript
// 伪代码
const interpreterSettings = queuedRun.interpreterSettings;
const robotType = interpreterSettings?.robotType;
const isDocRobot = robotType === 'doc-extract' || robotType === 'doc-parse';

if (isDocRobot) {
    // 文档任务直接入队执行，不创建浏览器
    await addJob(QUEUE_NAMES.EXECUTE_RUN, {
        userId,
        runId: queuedRun.runId,
        browserId: queuedRun.browserId || uuid(),
    }, { maxAttempts: 1 });
} else {
    // 浏览器任务才创建浏览器
    const newBrowserId = await createRemoteBrowserForRun(userId);
    // ...
}
```

### 11.7 修复调度路径 `runByUserId` 缺失

在 `scheduler/index.ts` 的 `Run.create` 中补充 `runByUserId`：

```typescript
const run = await Run.create({
    // ...
    runByUserId: userId,    // ← 当前缺失，必须补上
    // ...
});
```

---

## 12. 相关文件清单

| 文件路径 | 职责 |
|----------|------|
| `server/src/models/User.ts` | 代理配置存储模型 |
| `server/src/models/Run.ts` | 运行记录模型（含 runByUserId、interpreterSettings） |
| `server/src/models/Robot.ts` | 机器人模型（无代理字段，SDK 代理更新为死代码） |
| `server/src/routes/proxy.ts` | 代理配置 API、加解密、测试 |
| `server/src/utils/auth.ts` | encrypt/decrypt 工具函数 |
| `server/src/browser-management/classes/RemoteBrowser.ts` | 浏览器初始化、代理应用到 context |
| `server/src/browser-management/browserConnection.ts` | 远程浏览器连接、重试、本地回退 |
| `server/src/browser-management/classes/BrowserPool.ts` | 浏览器池管理、状态机、失效清理 |
| `server/src/browser-management/controller.ts` | 浏览器创建入口、连接重试 |
| `server/src/workflow-management/scheduler/index.ts` | 调度运行、运行级别重试（runByUserId 缺失） |
| `server/src/api/record.ts` | API 运行入口 |
| `server/src/task-runner.ts` | 任务执行器、文档分支提前返回 |
| `server/src/storage/graphileWorker.ts` | Graphile Worker 任务队列 |
| `server/src/schedule-worker.ts` | 调度工作者、调度任务入队 |
| `server/src/routes/storage.ts` | 运行存储、孤儿运行恢复、排队运行处理（不区分文档任务） |
| `server/src/server.ts` | 服务启动、恢复流程、定时清理 |
| `server/src/api/sdk.ts` | SDK 接口、代理更新死代码 |
| `server/src/utils/document/executeDocumentRun.ts` | 文档提取执行（MinIO + LLM，无浏览器无代理） |
| `server/src/utils/document/executeDocumentParseRun.ts` | 文档解析执行（MinIO + 解析，无浏览器无代理） |
| `src/components/proxy/ProxyForm.tsx` | 前端代理配置表单 |
| `src/api/proxy.ts` | 前端代理 API 调用 |
