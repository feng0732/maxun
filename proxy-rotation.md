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

## 8. 关键设计缺陷分析

### 8.1 无代理轮换 / IP 池机制

- **问题**：每个用户只能配置一个代理，没有多代理池
- **影响**：无法实现 IP 轮换，容易被目标网站封禁
- **代码证据**：User 模型只有单组代理字段，没有 Proxy 或 ProxyPool 表

### 8.2 无运行时代理切换

- **问题**：代理仅在 BrowserContext 创建时应用，运行中无法切换
- **影响**：遇到代理失败时只能重试同一个代理，无法自动切换到备用代理
- **代码证据**：`RemoteBrowser` 没有提供 `changeProxy()` 方法

### 8.3 所有重试都不切换代理

- **问题**：四级重试机制（连接重试、初始化重试、运行重试、任务队列重试）都使用同一个代理
- **影响**：代理本身故障时，重试多少次都不会成功
- **代码证据**：`getDecryptedProxyConfig(userId)` 始终返回同一个配置

### 8.4 本地回退后代理失效问题

- **问题**：远程浏览器连接失败回退到本地浏览器时，代理配置依然会应用
- **但**：如果是代理本身导致的问题，本地浏览器也会同样失败
- **注意**：本地浏览器启动参数里没有 `--proxy-server` 参数，代理是通过 Playwright 的 `context.proxy` 配置的

### 8.5 代理测试接口名不副实

- **问题**：测试接口 `/api/proxy/test` 实际上没有使用代理进行测试
- **代码证据**：
  ```typescript
  // 获取了 proxyOptions，但创建页面时没有传入代理配置
  const browser = await connectToRemoteBrowser();
  const page = await browser.newPage();  // 没有传 proxyOptions！
  await page.goto('https://example.com');
  await browser.close();
  ```

### 8.6 SDK 更新代理但不立即生效

**文件**: `server/src/api/sdk.ts`

SDK 允许更新用户代理配置，但如果有正在运行的浏览器，不会立即生效。
新创建的浏览器才会使用新的代理配置。

---

## 9. 改进建议

### 9.1 实现真正的代理池

1. 新建 `Proxy` 模型表，支持多代理配置
2. 增加 `ProxyPool` 管理类，实现轮换策略（轮询、随机、最少使用）
3. 每次创建浏览器时从池中选择一个可用代理

### 9.2 增加运行时代理切换能力

在 `RemoteBrowser` 中增加方法：

```typescript
public async switchProxy(newProxyConfig: ProxyConfig): Promise<void> {
    await this.context?.close();
    const contextOptions = { ...existingOptions, proxy: newProxyConfig };
    this.context = await this.browser!.newContext(contextOptions);
    this.currentPage = await this.context.newPage();
}
```

### 9.3 失败时自动切换代理

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

### 9.4 修复代理测试接口

```typescript
// 修复后的测试逻辑
const context = await browser.newContext({ proxy: proxyOptions });
const page = await context.newPage();
await page.goto('https://example.com');
```

### 9.5 本地回退时考虑代理因素

如果连接失败时，判断是否是代理导致的问题，如果是可以尝试下一个代理而不是直接回退到本地。

---

## 10. 相关文件清单

| 文件路径 | 职责 |
|----------|------|
| `server/src/models/User.ts` | 代理配置存储模型 |
| `server/src/routes/proxy.ts` | 代理配置 API、加解密、测试 |
| `server/src/utils/auth.ts` | encrypt/decrypt 工具函数 |
| `server/src/browser-management/classes/RemoteBrowser.ts` | 浏览器初始化、代理应用到 context |
| `server/src/browser-management/browserConnection.ts` | 远程浏览器连接、重试、本地回退 |
| `server/src/browser-management/classes/BrowserPool.ts` | 浏览器池管理、状态机、失效清理 |
| `server/src/browser-management/controller.ts` | 浏览器创建入口、连接重试 |
| `server/src/workflow-management/scheduler/index.ts` | 调度运行、运行级别重试 |
| `server/src/api/record.ts` | API 运行入口 |
| `server/src/task-runner.ts` | 任务执行器、工作流执行、浏览器等待 |
| `server/src/storage/graphileWorker.ts` | Graphile Worker 任务队列 |
| `server/src/schedule-worker.ts` | 调度工作者、调度任务入队 |
| `server/src/routes/storage.ts` | 运行存储、孤儿运行恢复、排队运行处理 |
| `server/src/server.ts` | 服务启动、恢复流程、定时清理 |
| `server/src/api/sdk.ts` | SDK 接口、允许更新代理配置 |
| `src/components/proxy/ProxyForm.tsx` | 前端代理配置表单 |
| `src/api/proxy.ts` | 前端代理 API 调用 |
