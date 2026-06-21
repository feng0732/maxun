# 代理和 IP 池切换代码链路分析

## 1. 架构概览

当前系统**没有实现真正的代理轮换（Proxy Rotation）或 IP 池功能**。每个用户只能配置一个固定代理，在浏览器会话初始化时应用，没有运行时的代理切换或失败自动切换机制。

```
用户配置代理 → 加密存储到User表 → 浏览器初始化时解密 → 应用到BrowserContext → 运行工作流
```

---

## 2. 代理配置选择链路

### 2.1 数据模型 - 代理存储

**文件**: [User.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/models/User.ts#L11-L13)

用户模型存储三个代理相关字段，均为加密后的值：
```typescript
interface UserAttributes {
    proxy_url?: string | null;       // 加密后的代理服务器URL
    proxy_username?: string | null;  // 加密后的代理用户名
    proxy_password?: string | null;  // 加密后的代理密码
}
```

### 2.2 代理配置 API

**文件**: [proxy.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/routes/proxy.ts)

| 端点 | 方法 | 功能 |
|------|------|------|
| `/api/proxy/config` | POST | 保存代理配置（加密存储） |
| `/api/proxy/config` | GET | 获取代理配置（脱敏返回） |
| `/api/proxy/config` | DELETE | 删除代理配置 |
| `/api/proxy/test` | GET | 测试代理连接 |

**关键代码 - 保存代理（加密）** [L13-L57](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/routes/proxy.ts#L13-L57):
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

**文件**: [proxy.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/routes/proxy.ts#L159-L177)

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
- [RemoteBrowser.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L483) - 浏览器初始化
- [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/workflow-management/scheduler/index.ts#L58) - 调度运行
- [api/record.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/api/record.ts#L572) - API运行

---

## 3. 浏览器会话应用链路

### 3.1 浏览器初始化时应用代理

**文件**: [RemoteBrowser.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L460-L612)

在 `initialize()` 方法中，代理配置在创建 BrowserContext 时应用：

```typescript
public initialize = async (userId: string): Promise<void> => {
    // ...
    // 第483行：获取解密的代理配置
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

    // 第496行：构建context选项
    const contextOptions: any = {
        reducedMotion: 'reduce',
        javaScriptEnabled: true,
        timeout: 50000,
        userAgent: this.getUserAgent(),
        // ...
    };

    // 第512-518行：将代理配置注入context
    if (proxyOptions.server) {
        contextOptions.proxy = {
            server: proxyOptions.server,
            username: proxyOptions.username ? proxyOptions.username : undefined,
            password: proxyOptions.password ? proxyOptions.password : undefined,
        };
    }

    // 第522行：创建带代理的context
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

1. **录制模式** - [initializeRemoteBrowserForRecording](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/browser-management/controller.ts#L26-L104)
   - 用户启动录制时创建浏览器
   - 调用 `browserSession.initialize(userId)` 应用代理

2. **运行模式** - [createRemoteBrowserForRun](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/browser-management/controller.ts#L114-L137)
   - 调度或API触发运行时创建浏览器
   - 调用 `initializeBrowserAsync` → `browserSession.initialize(userId)` 应用代理

---

## 4. 失败重试与切换逻辑

### 4.1 浏览器初始化重试（非代理切换）

**文件**: [RemoteBrowser.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L460-L612)

浏览器初始化有 **3次重试** 机制，但**不会切换代理**，始终使用同一个代理配置：

```typescript
public initialize = async (userId: string): Promise<void> => {
    const MAX_RETRIES = 3;
    let retryCount = 0;
    let success = false;

    while (!success && retryCount < MAX_RETRIES) {
        try {
            this.browser = await connectToRemoteBrowser();
            // 每次重试都获取同一个代理配置
            const proxyConfig = await getDecryptedProxyConfig(userId);
            // 使用相同的proxyOptions创建context
            // ...
            success = true;
        } catch (error: any) {
            retryCount++;
            // 失败后清理并重试，但代理配置不变
            if (this.browser) {
                await this.browser.close();
                this.browser = null;
            }
            await new Promise(resolve => setTimeout(resolve, 1000));
        }
    }
};
```

### 4.2 运行级别重试（非代理切换）

**文件**: [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/workflow-management/scheduler/index.ts#L216-L246)

工作流运行有 **3次重试** 限制，但同样**不会切换代理**：

```typescript
const retryCount = plainRun.retryCount || 0;
if (retryCount >= 3) {
    logger.log('warn', `Scheduled Run ${id} has exceeded max retries (${retryCount}/3), marking as failed`);
    // 标记为永久失败，不切换代理
    await run.update({
        status: 'failed',
        log: `Max retries exceeded (${retryCount}/3) - Run failed after multiple attempts.`
    });
    return { success: false, error: 'Max retries exceeded' };
}
```

### 4.3 连接重试（非代理切换）

**文件**: [controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/browser-management/controller.ts#L367-L389)

Socket连接有指数退避重试，但也不涉及代理切换：

```typescript
const connectWithRetry = async (maxRetries: number = 3): Promise<Socket | null> => {
    let retryCount = 0;
    while (retryCount < maxRetries) {
        try {
            const socket = await waitForConnection;
            if (socket || retryCount === maxRetries - 1) return socket;
        } catch (error: any) {
            logger.log('warn', `Connection attempt ${retryCount + 1} failed for browser ${id}: ${error.message}`);
        }
        retryCount++;
        if (retryCount < maxRetries) {
            const delay = Math.pow(2, retryCount) * 1000; // 指数退避
            await new Promise(resolve => setTimeout(resolve, delay));
        }
    }
    return null;
};
```

---

## 5. 代码调用链路总图

### 5.1 代理配置流

```
前端ProxyForm.tsx
    ↓ POST /api/proxy/config
server/routes/proxy.ts (encrypt)
    ↓
User表 (proxy_url, proxy_username, proxy_password - 加密存储)
    ↓
getDecryptedProxyConfig(userId)  (decrypt)
    ├─→ RemoteBrowser.initialize()       ← 录制时
    ├─→ scheduler/createWorkflowAndStoreMetadata()  ← 调度运行时
    └─→ api/record/createWorkflowAndStoreMetadata() ← API运行时
```

### 5.2 运行时代理应用流

```
handleRunRecording(robotId, userId)
    ↓
createWorkflowAndStoreMetadata()
    ├─ getDecryptedProxyConfig(userId)  // 获取代理（但仅用于log，实际应用在下一步）
    └─ createRemoteBrowserForRun(userId)
        ↓
initializeBrowserAsync(id, userId)
    ↓
new RemoteBrowser(socket, userId, id)
    ↓
browserSession.initialize(userId)
    ├─ connectToRemoteBrowser()  // 连接浏览器
    ├─ getDecryptedProxyConfig(userId)  // 再次获取代理
    ├─ 构建contextOptions.proxy
    └─ browser.newContext(contextOptions)  // 代理在这里生效
        ↓
工作流执行（期间代理不变）
```

---

## 6. 关键设计缺陷分析

### 6.1 无代理轮换 / IP池机制

- **问题**：每个用户只能配置一个代理，没有多代理池
- **影响**：无法实现IP轮换，容易被目标网站封禁
- **代码证据**：User模型只有单组代理字段，没有ProxyPool表

### 6.2 无运行时代理切换

- **问题**：代理仅在BrowserContext创建时应用，运行中无法切换
- **影响**：遇到代理失败时只能重试同一个代理，无法自动切换到备用代理
- **代码证据**：`RemoteBrowser` 没有提供 `changeProxy()` 方法

### 6.3 重试机制不切换代理

- **问题**：所有重试逻辑（浏览器初始化、运行重试、连接重试）都使用同一个代理
- **影响**：代理本身故障时，重试多少次都不会成功
- **代码证据**：`getDecryptedProxyConfig(userId)` 始终返回同一个配置

### 6.4 无代理健康检查

- **问题**：测试接口 `/api/proxy/test` [L59-L96](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/routes/proxy.ts#L59-L96) 实际上没有使用代理进行测试
- **代码证据**：
  ```typescript
  // 测试接口获取了proxyOptions，但没有实际使用
  const browser = await connectToRemoteBrowser();
  const page = await browser.newPage();  // 没有传入proxyOptions！
  await page.goto('https://example.com');
  await browser.close();
  ```

### 6.5 SDK更新代理但不立即生效

**文件**: [sdk.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/api/sdk.ts#L595-L603)

SDK允许更新用户代理配置，但如果有正在运行的浏览器，不会生效：
```typescript
if (updates.proxy_url !== undefined) updateData.proxy_url = updates.proxy_url;
if (updates.proxy_username !== undefined) updateData.proxy_username = updates.proxy_username;
if (updates.proxy_password !== undefined) updateData.proxy_password = updates.proxy_password;
```

---

## 7. 改进建议

### 7.1 实现真正的代理池

1. 新建 `Proxy` 模型表，支持多代理配置
2. 增加 `ProxyPool` 管理类，实现轮换策略（轮询、随机、最少使用）
3. 每次创建浏览器时从池中选择一个可用代理

### 7.2 增加运行时代理切换能力

在 `RemoteBrowser` 中增加方法：
```typescript
public async switchProxy(newProxyConfig: ProxyConfig): Promise<void> {
    // 关闭现有context
    await this.context?.close();
    // 使用新代理创建新context
    const contextOptions = { ...existingOptions, proxy: newProxyConfig };
    this.context = await this.browser!.newContext(contextOptions);
    this.currentPage = await this.context.newPage();
}
```

### 7.3 失败时自动切换代理

修改重试逻辑，代理失败时从池中选择下一个代理：
```typescript
// 伪代码
while (!success && retryCount < MAX_RETRIES) {
    const proxyConfig = proxyPool.getNextProxy(userId);  // 从池中取下一个
    try {
        // 使用新代理初始化
        success = true;
    } catch (error) {
        proxyPool.markProxyFailed(proxyConfig.id);  // 标记失败
        retryCount++;
    }
}
```

### 7.4 修复代理测试接口

```typescript
// 修复后的测试逻辑
const context = await browser.newContext({ proxy: proxyOptions });
const page = await context.newPage();
await page.goto('https://example.com');
```

---

## 8. 相关文件清单

| 文件 | 职责 |
|------|------|
| [User.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/models/User.ts) | 代理配置存储模型 |
| [proxy.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/routes/proxy.ts) | 代理配置API、加解密、测试 |
| [auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/utils/auth.ts) | encrypt/decrypt 工具函数 |
| [RemoteBrowser.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/browser-management/classes/RemoteBrowser.ts) | 浏览器初始化、代理应用到context |
| [controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/browser-management/controller.ts) | 浏览器创建入口、连接重试 |
| [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/workflow-management/scheduler/index.ts) | 调度运行、运行级别重试 |
| [api/record.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/api/record.ts) | API运行入口 |
| [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/task-runner.ts) | 任务执行器、工作流执行 |
| [api/sdk.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/server/src/api/sdk.ts) | SDK接口、允许更新代理配置 |
| [ProxyForm.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/src/components/proxy/ProxyForm.tsx) | 前端代理配置表单 |
| [proxy.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/114-maxun/src/api/proxy.ts) | 前端代理API调用 |
