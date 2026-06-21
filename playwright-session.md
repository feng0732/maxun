# Playwright 驱动会话流转分析

## 架构总览

Maxun 的浏览器会话管理采用**三层架构**：

```
┌──────────────────────────────────────────────────────────────┐
│  browser/server.ts     — 浏览器服务进程（独立进程/容器）       │
│  chromium.launchServer() 暴露 WebSocket 端点                  │
└─────────────────────┬────────────────────────────────────────┘
                      │ WS 连接
                      ▼
┌──────────────────────────────────────────────────────────────┐
│  browserConnection.ts  — 连接层                              │
│  connectToRemoteBrowser()  通过 WS 连接到远程浏览器            │
└─────────────────────┬────────────────────────────────────────┘
                      │ Browser 实例
                      ▼
┌──────────────────────────────────────────────────────────────┐
│  RemoteBrowser.ts      — 会话实体                             │
│  Browser → BrowserContext → Page → CDPSession                │
└─────────────────────┬────────────────────────────────────────┘
                      │ 注册到池中
                      ▼
┌──────────────────────────────────────────────────────────────┐
│  BrowserPool.ts        — 会话池管理                           │
│  "1 User - 2 Browser" 策略，槽位预留/升级/清理                  │
└─────────────────────┬────────────────────────────────────────┘
                      │ 编排调用
                      ▼
┌──────────────────────────────────────────────────────────────┐
│  controller.ts         — 控制器（入口/出口）                    │
│  录制会话 / 运行会话 / 校验会话 / 销毁会话                       │
└──────────────────────────────────────────────────────────────┘
```

---

## 一、会话创建

### 1.1 浏览器服务启动

文件：[server.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/browser/server.ts)

浏览器服务是**独立进程**，通过 `playwright-extra` + stealth 插件启动 headless Chromium 服务器：

```ts
browserServer = await chromium.launchServer({
    headless: true,
    args: ['--disable-blink-features=AutomationControlled', ...],
    port: BROWSER_WS_PORT,   // 默认 3001
});
```

- 暴露 WebSocket 端点（如 `ws://localhost:3001/...`）
- 另起 HTTP 健康检查服务（端口 3002），`/health` 端点返回 `wsEndpoint`
- 进程收到 SIGTERM/SIGINT 时调用 `browserServer.close()` 优雅关闭

### 1.2 连接到远程浏览器

文件：[browserConnection.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/browserConnection.ts)

`connectToRemoteBrowser()` 是获取 Playwright `Browser` 实例的唯一入口：

```
1. getBrowserServiceEndpoint()
   → HTTP GET http://localhost:3002/health
   → 返回 { wsEndpoint: "ws://..." }

2. chromium.connect(wsEndpoint, { timeout: 30000 })
   → 最多重试 3 次，间隔 2s

3. 如果远程连接全部失败
   → 降级到 launchLocalBrowser()
   → chromium.launch({ headless: true, args: [...] })
```

**关键设计**：每个 `RemoteBrowser` 实例都通过 `connectToRemoteBrowser()` 获取**独立的 Browser 连接**，并非共享同一个 Browser 对象。

### 1.3 RemoteBrowser 初始化

文件：[RemoteBrowser.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L460-L612)

`initialize()` 方法完成 Playwright 对象链的完整创建，流程如下：

```
connectToRemoteBrowser()                    → Browser (this.browser)
    │
    ├─ browser.newContext(contextOptions)    → BrowserContext (this.context)
    │      │
    │      ├─ applyEnhancedFingerprinting()  → 注入指纹伪装
    │      ├─ context.addInitScript(...)     → 隐藏 navigator.webdriver
    │      │
    │      └─ context.newPage()             → Page (this.currentPage)
    │             │
    │             ├─ setupPageEventListeners()   → 注册 framenavigated / load 事件
    │             ├─ initializeRRWebRecording()  → 仅录制模式：注入 rrweb
    │             └─ context.newCDPSession()     → CDPSession (this.client)
    │
    └─ 失败重试：最多 3 次，每次先 browser.close() 再重连
```

**Context 配置要点**：
- 代理：从用户配置读取 `proxy_url/username/password`
- 反检测：随机 User-Agent、FingerprintSuite 注入、webdriver 隐藏
- 广告拦截：PlaywrightBlocker（easylist 规则集）
- 超时保护：整体初始化 120s 超时，Context 创建 15s 超时

**rrweb 录制模式**（`isRecordingMode = true` 时）：
1. 注入 `rrweb.min.js` 到页面
2. `page.exposeFunction('emitEventToBackend')` 暴露后端回调
3. `window.rrweb.record({ emit })` 启动录制
4. 页面导航时自动重新初始化 rrweb

---

## 二、会话复用

### 2.1 录制会话复用

文件：[controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/controller.ts#L26-L104)

`initializeRemoteBrowserForRecording()` 的复用逻辑：

```
1. getActiveBrowserIdByState(userId, "recording")
   → 如果用户已有 recording 状态的浏览器

2. 若已有 → remoteBrowser.updateSocket(socket)
   → 不创建新浏览器，仅替换 Socket 连接
   → 重新注册编辑器事件和滚动监听

3. 若没有 → new RemoteBrowser(socket, userId, id, true)
   → 全新创建，初始化后加入池
```

**核心复用机制**：当用户刷新页面或重连时，前端会重新建立 Socket，后端通过 `updateSocket()` 将新 Socket 绑定到已存在的 `RemoteBrowser`，避免重新启动浏览器。

### 2.2 updateSocket 流程

文件：[RemoteBrowser.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L848-L857)

```ts
updateSocket(socket: Socket): void {
    this.socket = socket;                    // 替换 Socket
    this.registerEditorEvents();             // 重新注册编辑器事件
    this.generator?.updateSocket(socket);    // 更新 Generator 的 Socket
    this.interpreter?.updateSocket(socket);  // 更新 Interpreter 的 Socket
    if (this.isDOMStreamingActive) {
        this.setupScrollEventListener();     // 恢复滚动监听
    }
}
```

### 2.3 BrowserPool 的槽位管理

文件：[BrowserPool.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/classes/BrowserPool.ts)

**策略："1 User - 2 Browser"** — 每个用户最多拥有 2 个浏览器实例，其中最多 1 个 recording 状态。

| 方法 | 作用 |
|------|------|
| `addRemoteBrowser()` | 直接添加浏览器到池，检查用户槽位上限。**不设置 `status` 字段** |
| `reserveBrowserSlotAtomic()` | 原子预留槽位（防止竞态），设置 `status: "reserved"`、`browser: null` |
| `upgradeBrowserSlot()` | 将 reserved 槽位升级为 `status: "ready"`，绑定实际 RemoteBrowser 实例 |
| `failBrowserSlot()` | 尝试 `switchOff()` 后调用 `deleteRemoteBrowser()`，从池中删除 |
| `getRemoteBrowser()` | 获取浏览器实例；`status === "reserved"` 或 `"failed"` 时返回 `undefined` |

#### 核心纠错：三种会话走不同的池入路径，`status` 字段值完全不同

虽然 `BrowserPoolInfo` 接口定义了 `status?: "reserved" | "initializing" | "ready" | "failed"` 四种状态值，但**代码中 `"initializing"` 从未被任何路径赋值**，它是一个预留但未实现的状态。实际存在的状态流转取决于会话类型：

**① Recording 会话 — 无预留，直接入库，`status` 为 `undefined`**

调用链：[controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/controller.ts#L26-L104) `initializeRemoteBrowserForRecording()`

```
new RemoteBrowser(socket, userId, id, true)
    ↓
await browserSession.initialize(userId)          // 同步等待初始化完成
    ↓
browserPool.addRemoteBrowser(id, browser, userId, false, "recording")
    ↓                                             // addRemoteBrowser 不设置 status
pool[id] = { browser, active: false, userId, state: "recording" }
    → status: undefined（未设置）
    → createdAt: undefined（未设置）
    → browser: RemoteBrowser 实例（非 null）
```

- 没有调用 `reserveBrowserSlotAtomic()`，没有 `reserved` 状态
- 没有调用 `upgradeBrowserSlot()`，没有 `ready` 状态
- 初始化**先于**入库完成，池中看到时就已经是完整的浏览器实例

**② Run 会话 — 两阶段提交（预留 → 升级），`status` 从 `"reserved"` 到 `"ready"`**

调用链：[controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/controller.ts#L114-L137) `createRemoteBrowserForRun()` → [initializeBrowserAsync](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/controller.ts#L333-L480)

```
阶段一：预留（同步，立即返回 browserId 给调用方）
browserPool.reserveBrowserSlotAtomic(id, userId, "run")
    ↓
pool[id] = { browser: null, active: false, userId, state: "run",
             status: "reserved", createdAt: now, lastAccessed: now }

阶段二：异步初始化（initializeBrowserAsync）
new RemoteBrowser(socket/dummy, userId, id)
    ↓
await browserSession.initialize(userId)         // 可能成功或失败
    ↓
如果成功：
browserPool.upgradeBrowserSlot(id, browserSession)
    ↓
pool[id].browser = browserSession
pool[id].status = "ready"                       // reserved → ready

如果失败（任何异常）：
browserPool.failBrowserSlot(id)
    ↓
browser.switchOff?.()                           // 尝试清理（browser 可能为 null）
deleteRemoteBrowser(id)                         // 从池中删除
```

- **唯一经过 `reserved → ready` 完整流转**的路径
- 预留是同步的：调用方立即拿到 `browserId`，但此时浏览器尚未初始化
- `getRemoteBrowser()` 在 `status === "reserved"` 时返回 `undefined`，确保外部不会拿到未就绪的浏览器
- 失败路径统一走 `failBrowserSlot()`，而非 `addRemoteBrowser`

**③ Validation 会话 — 无预留，直接入库，`status` 为 `undefined`**

调用链：[controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/controller.ts#L489-L543) `createRemoteBrowserForValidation()`

```
new RemoteBrowser(dummySocket, userId, id)
    ↓
await browserSession.initialize(userId)         // 同步等待
    ↓
browserPool.addRemoteBrowser(id, browser, userId, true, "run")
    ↓                                             // addRemoteBrowser 不设置 status
pool[id] = { browser, active: true, userId, state: "run" }
    → status: undefined（未设置）
    → createdAt: undefined（未设置）
    → browser: RemoteBrowser 实例（非 null）
```

- 与 Recording 会话一样，不走预留/升级流程
- `active: true`（Recording 是 `false`）

#### 实际槽位状态流转图

```
┌─ Recording 会话 ─────────────────────────────────────┐
│  (不存在) ──addRemoteBrowser()──→ status=undefined    │
│  复用路径：updateSocket()，池条目不变                    │
└──────────────────────────────────────────────────────┘

┌─ Run 会话 ───────────────────────────────────────────┐
│  (不存在)                                     │
│      ↓ reserveBrowserSlotAtomic()                    │
│  status="reserved", browser=null                     │
│      ↓ upgradeBrowserSlot()                          │
│  status="ready", browser=RemoteBrowser               │
│                                                       │
│  失败分支：                                           │
│  status="reserved" ──failBrowserSlot()──→ (删除)      │
└──────────────────────────────────────────────────────┘

┌─ Validation 会话 ────────────────────────────────────┐
│  (不存在) ──addRemoteBrowser()──→ status=undefined    │
└──────────────────────────────────────────────────────┘

注："initializing" 在接口中定义但代码从未赋值，不存在此状态。
注："failed" 仅在 failBrowserSlot 内部作为日志标记，执行后立即删除，不在池中停留。
```

#### 复用时的槽位变化

[controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/controller.ts#L26-L36) 中，Recording 会话复用路径：

```
getActiveBrowserIdByState(userId, "recording") 找到 activeId
    ↓
browserPool.getRemoteBrowser(activeId)  // status=undefined，不会返回 undefined
    ↓
remoteBrowser.updateSocket(socket)      // 仅替换 Socket，池条目不变
```

- 池中的 `BrowserPoolInfo` 条目**完全不变**（`status`、`browser`、`active` 等字段均不变）
- `addRemoteBrowser()` 的"同 ID 同用户"更新分支（[BrowserPool.ts#L101-L111](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/classes/BrowserPool.ts#L101-L111)）在复用路径中**不会触发**，因为复用时根本不调用 `addRemoteBrowser()`

#### 过时清理的实际作用范围

[cleanupStaleBrowserSlots](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/classes/BrowserPool.ts#L702-L729) 的清理条件：

```ts
const isStale = info.status === "reserved" || info.status === "initializing";
const age = now - (info.createdAt || 0);
if (isStale && info.browser === null && age > staleThreshold) { ... }
```

三个条件**同时满足**才会清理：
1. `status` 为 `"reserved"` 或 `"initializing"` — **Recording/Validation 会话的 `status` 是 `undefined`，不会被清理**
2. `browser === null` — **Recording/Validation 入库时 browser 已经是实例，也不满足**
3. `createdAt` 存在且超过 5 分钟 — **Recording/Validation 未设置 `createdAt`，默认为 0，`now - 0` 远超阈值，但被前两个条件挡住**

因此 `cleanupStaleBrowserSlots()` **仅对 Run 会话的预留槽位有效**，用于清理 `initializeBrowserAsync` 中途崩溃导致一直卡在 `reserved` 状态的槽位。

---

## 三、失败清理路径深度分析

### 3.1 核心纠错：`failed` 状态从未写入池中

`BrowserPoolInfo` 接口虽然定义了 `status?: "reserved" | "initializing" | "ready" | "failed"`，但：

1. **`"failed"` 从未被赋值** — [failBrowserSlot()](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/classes/BrowserPool.ts#L677-L696) 的实现是**直接删除池条目**，而非先设置 `status = "failed"` 再保留
2. **`getRemoteBrowser()` 有检查但永远命中不到** — [getRemoteBrowser()](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/classes/BrowserPool.ts#L238-L242) 中 `if (poolInfo.status === "failed")` 的分支在当前代码逻辑下永远不会执行
3. **只有日志意义** — `logger.log('info', \`Marking browser slot ${id} as failed\`)` 只是日志，不对应实际的池状态变更

```ts
// failBrowserSlot 的真实行为（非：先设 failed 再删）
public failBrowserSlot = (id: string): void => {
    if (this.pool[id]) {
        logger.log('info', `Marking browser slot ${id} as failed`);  // 仅日志
        if (browserInfo.browser) {
            browserInfo.browser.switchOff?.().catch(...);  // 尝试关浏览器
        }
        this.deleteRemoteBrowser(id);  // 直接删除，不设 status = "failed"
    }
};
```

**结论**：`"failed"` 是接口中定义、`getRemoteBrowser` 中检查、但实际运行时池里永远不会出现的"幽灵状态"。它可能是设计预留但未实现的功能。

---

### 3.2 五种失败场景的清理路径对比

| 失败场景 | 触发位置 | 清理方法 | 是否调用 `failBrowserSlot` | 是否清理 Socket 命名空间 | 池是否会残留条目 |
|---------|---------|---------|--------------------------|------------------------|----------------|
| 录制会话初始化失败 | [controller.ts#L72-L99](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/controller.ts#L72-L99) | 直接 `browserSession.switchOff()` | ❌ 否 | ❌ **否（命名空间泄漏）** | ❌ 无残留（从未入池） |
| 录制会话入池失败 | [controller.ts#L48-L56](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/controller.ts#L48-L56) | 直接 `browserSession.switchOff()` | ❌ 否 | ❌ **否（命名空间泄漏）** | ❌ 无残留（入池被拒） |
| Run 会话初始化失败 | [controller.ts#L429-L438](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/controller.ts#L429-L438) | `browserPool.failBrowserSlot(id)` | ✅ 是 | ❌ **否（命名空间泄漏）** | ❌ 无残留（fail 即删） |
| 校验会话入池失败 | [controller.ts#L519-L523](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/controller.ts#L519-L523) | 直接 `browserSession.switchOff()` | ❌ 否 | ❌ 否（校验用 dummy，无真实命名空间） | ❌ 无残留（入池被拒） |
| 校验会话拿不到 Page | [controller.ts#L525-L529](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/controller.ts#L525-L529) | `destroyRemoteBrowser(id, userId)` | ❌ 否 | ✅ 是（完整销毁） | ❌ 无残留（完整销毁） |

---

### 3.3 逐场景详细分析

#### 场景 1：录制会话初始化失败

**触发条件**：`browserSession.initialize(userId)` 抛出异常（如连不上浏览器服务、Context 创建超时等）。

**清理流程**：

```
try {
    await browserSession.initialize(userId);  // ← 这里抛异常
    await browserSession.registerEditorEvents();
    browserPool.addRemoteBrowser(...);         // ← 不会执行到
} catch (initError) {
    socket.emit('dom-mode-error', ...);        // 1. 通知前端
    socket.emit('error', { message, details }); // 2. 发送错误详情
    await new Promise(r => setTimeout(r, 100)); // 3. 等 100ms
    await browserSession.switchOff();          // 4. 关闭浏览器
    return id;                                 // 5. 直接返回
}
```

**关键观察**：
- 浏览器**从未入池** — 因为 `addRemoteBrowser` 在 try 块的后部，初始化失败不会执行到那里
- **Socket 命名空间泄漏** — `createSocketConnection(io.of(id), ...)` 已经创建了命名空间（`io.of(id)` 是惰性创建，调用即存在），但失败时没有清理它。命名空间会一直保留在 Socket.IO 的 `_nsps` Map 中
- 没有调用 `failBrowserSlot` — 因为 Recording 会话不走预留/升级流程，失败时直接手动清理

---

#### 场景 2：录制会话入池失败

**触发条件**：`browserPool.addRemoteBrowser()` 返回 `false`（用户已有 recording 浏览器，或已达 2 个浏览器上限）。

**清理流程**：

```
const added = browserPool.addRemoteBrowser(id, browserSession, userId, false, "recording");
if (!added) {
    socket.emit('dom-mode-error', ...);       // 1. 通知前端
    await browserSession.switchOff();         // 2. 关闭浏览器
    return id;                                // 3. 直接返回
}
```

**关键观察**：
- 浏览器已经初始化完成，但入池被拒（并发/配额问题）
- 同样**没有清理 Socket 命名空间**
- 同样**不走 `failBrowserSlot`**
- 与"初始化失败"的区别：浏览器已经完整启动了，只是因为配额原因无法入池

---

#### 场景 3：Run 会话初始化失败

**触发条件**：`initializeBrowserAsync()` 中任何异常（命名空间错误、Socket 连接失败、浏览器初始化失败、升级失败等）。

**清理流程**（共 3 处异常出口，全部走 `failBrowserSlot`）：

```
① 命名空间 error 事件 → browserPool.failBrowserSlot(id)
② 浏览器初始化失败 catch → browserPool.failBrowserSlot(id)
③ 外层 try/catch 兜底 → browserPool.failBrowserSlot(id)
```

`failBrowserSlot` 内部做的事：
1. 打日志 `"Marking browser slot ${id} as failed"`
2. 如果 `browserInfo.browser` 存在，调用 `switchOff()`（**异步 fire-and-forget**，不等待）
3. `deleteRemoteBrowser(id)` — 从池中删除条目

**关键观察**：
- Run 会话**唯一**通过 `failBrowserSlot` 清理的路径
- `switchOff()` 是 `.catch(...)` 的 fire-and-forget 调用，**不等待关闭完成**就删除池条目
- **Socket 命名空间同样泄漏** — `failBrowserSlot` 只处理浏览器和池，不管命名空间
- 失败后池里**没有 failed 状态的条目** — 直接删了

---

#### 场景 4：校验会话入池失败

**触发条件**：`browserPool.addRemoteBrowser()` 返回 `false`（用户已达 2 个浏览器上限）。

**清理流程**：

```
const added = browserPool.addRemoteBrowser(id, browserSession, userId, true, 'run');
if (!added) {
    await browserSession.switchOff();        // 1. 关闭浏览器
    throw new Error('Failed to add validation browser to pool'); // 2. 抛错
}
```

**关键观察**：
- 与录制入池失败类似，直接 `switchOff()`，不走 `failBrowserSlot`
- 校验会话用 dummy socket，没有真实的前端命名空间，不存在命名空间泄漏问题

---

#### 场景 5：校验会话拿不到 Page

**触发条件**：浏览器成功入池，但 `browserSession.getCurrentPage()` 返回 null/undefined。

**清理流程**：

```
const page = browserSession.getCurrentPage();
if (!page) {
    await destroyRemoteBrowser(id, userId);  // 走完整销毁流程
    throw new Error('Failed to get page from validation browser');
}
```

**关键观察**：
- **唯一走 `destroyRemoteBrowser()` 的失败路径** — 因为浏览器已经成功入池了，需要完整清理
- 会清理 Socket 命名空间（虽然是 dummy 的）
- 会从池中删除条目

---

### 3.4 失败清理的共性问题

#### ① 命名空间泄漏（Recording / Run 场景）

Recording 和 Run 会话在初始化失败时，Socket.IO 的动态命名空间（`io.of(id)`）已经创建但**从未被清理**。命名空间会一直存在于 `io._nsps` Map 中，虽然空的命名空间资源消耗不大，但属于资源泄漏。

只有正常销毁路径（`destroyRemoteBrowser`）才会清理命名空间。

#### ② `failBrowserSlot` 与直接 `switchOff` 的差异

| 维度 | `failBrowserSlot(id)` | 直接 `browserSession.switchOff()` |
|------|----------------------|----------------------------------|
| 调用者 | Run 会话初始化失败 | Recording 初始化/入池失败、Validation 入池失败 |
| 浏览器关闭方式 | `switchOff?.().catch(...)` — fire-and-forget，不等待 | `await switchOff()` — 同步等待 |
| 是否删池条目 | ✅ 是 | ❌ 否（因为从未入池） |
| 是否清理命名空间 | ❌ 否 | ❌ 否 |
| 适用场景 | 浏览器已在池中（reserved 状态），需要清池 | 浏览器还没入池，只需要关浏览器 |

#### ③ `failed` 状态的设计与实现不一致

- **接口层**：定义了 `"failed"` 状态值
- **读取层**：`getRemoteBrowser()` 检查了 `"failed"` 并返回 undefined
- **写入层**：没有任何代码将 `status` 设为 `"failed"`
- **清理层**：`failBrowserSlot()` 直接删除，不留存失败状态

这意味着 `"failed"` 状态是一个**设计了但未落地**的状态，可能是为未来扩展（如失败重试、失败统计）预留的。

---

## 四、会话释放

### 4.1 switchOff — 单个浏览器关闭

文件：[RemoteBrowser.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L736-L839)

`switchOff()` 按**反向层次**逐步释放资源，每步均有 5s 超时保护：

```
1. isDOMStreamingActive = false
2. removeAllSocketListeners()              → 清除 Socket 事件监听
3. currentPage.removeAllListeners()        → 清除 Page 事件监听
4. generator.cleanup()                     → 清理工作流生成器
5. interpreter.stopInterpretation()        → 停止工作流解释器
6. client.detach()                         → 分离 CDP 会话
7. currentPage.close()                     → 关闭 Page
8. context.close()                         → 关闭 BrowserContext
9. browser.close()                         → 关闭 Browser 连接
```

每步均在 `try/catch/finally` 中执行，即使某步失败也会继续清理后续资源，最终将所有引用置为 `null`。

### 4.2 destroyRemoteBrowser — 完整销毁流程

文件：[controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/controller.ts#L155-L233)

```
1. clearRecordingTimeout(id)          → 取消录制超时定时器
2. browserSession.switchOff()         → 关闭浏览器（见上节）
3. namespace.fetchSockets()           → 断开所有 Socket 连接
4. namespace.removeAllListeners()     → 清除命名空间监听
5. io._nsps.delete(`/${id}`)         → 从 SocketIO 内部 Map 中删除命名空间
6. browserPool.deleteRemoteBrowser(id) → 从池中删除记录
```

整体 30s 超时保护；超时后强制从池中删除。

### 4.3 Recording 超时自动清理

文件：[controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/controller.ts#L58-L71)

录制会话设置 10 分钟（`RECORDING_TIMEOUT_MS`）超时：

```ts
const timeoutHandle = setTimeout(async () => {
    io.of(id).emit('recording-timeout');     // 通知前端
    await new Promise(resolve => setTimeout(resolve, 1000)); // 等待前端处理
    await destroyRemoteBrowser(id, userId);  // 销毁浏览器
}, RECORDING_TIMEOUT_MS);
```

### 4.4 BrowserPool 清理方法

| 方法 | 场景 |
|------|------|
| `closeAndDeleteBrowser()` | 从池中移除并删除映射（不关闭浏览器本身） |
| `deleteRemoteBrowser()` | 仅从池中移除，不尝试关闭 |
| `failBrowserSlot()` | 尝试 `switchOff()` → 删除（不写入 failed 状态） |
| `cleanupStaleBrowserSlots()` | 清理超过 5 分钟的 reserved 槽位（仅 Run 会话有效） |

---

## 五、三种会话类型对比

| 维度 | Recording 会话 | Run 会话 | Validation 会话 |
|------|---------------|----------|-----------------|
| **入口** | `initializeRemoteBrowserForRecording()` | `createRemoteBrowserForRun()` | `createRemoteBrowserForValidation()` |
| **isRecordingMode** | `true` | `false` | `false` |
| **rrweb** | ✅ 启用 | ❌ 跳过 | ❌ 跳过 |
| **Socket** | 真实 Socket（前端连接） | 真实或 Dummy Socket | Dummy Socket |
| **池入路径** | `addRemoteBrowser()`（status=undefined） | `reserveBrowserSlotAtomic()` → `upgradeBrowserSlot()`（reserved→ready） | `addRemoteBrowser()`（status=undefined） |
| **复用** | ✅ `updateSocket()`，池条目不变 | ❌ 不复用 | ❌ 不复用 |
| **超时** | 10 分钟自动销毁 | 无自动超时 | 无自动超时 |
| **典型用途** | 用户录制操作流程 | 机器人执行工作流 | SDK 校验任务 |

---

## 六、Socket 通信与会话的绑定

文件：[connection.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/socket-connection/connection.ts)

每个浏览器会话通过 **Socket.IO 动态命名空间** 隔离通信：

```
前端连接 → io.of(`/${browserId}`) → Namespace
                                       │
                    createSocketConnection(namespace, userId, callback)
                                       │
                    registerInputHandlers(socket, userId)
                    → dom:click, dom:keypress, input:keyup, input:url 等
```

- 录制会话：`createSocketConnection()` — 注册输入处理器 + 断开时清理
- 运行会话：`createSocketConnectionForRun()` — 仅处理连接/断开事件
- 校验会话：无 Socket 连接，使用 Dummy Socket

---

## 七、关键对象关系图

```
BrowserPool
  │
  ├── pool: { [id]: BrowserPoolInfo }
  │       │
  │       └── BrowserPoolInfo
  │             ├── browser: RemoteBrowser | null     ← Run预留时为null，其余为实例
  │             ├── active: boolean
  │             ├── userId: string
  │             ├── state: "recording" | "run"
  │             ├── status?: "reserved" | "ready"     ← 仅Run会话设置；Recording/Validation为undefined
  │             ├── createdAt?: number                 ← 仅Run会话设置（reserveBrowserSlotAtomic）
  │             └── lastAccessed?: number              ← 仅Run会话设置（reserveBrowserSlotAtomic）
  │
  └── userToBrowserMap: Map<userId, browserId[]>

RemoteBrowser
  ├── browser: Browser          ← connectToRemoteBrowser() 返回
  ├── context: BrowserContext   ← browser.newContext()
  ├── currentPage: Page         ← context.newPage()
  ├── client: CDPSession        ← context.newCDPSession(page)
  ├── socket: Socket            ← Socket.IO 连接或 Dummy
  ├── generator: WorkflowGenerator
  ├── interpreter: WorkflowInterpreter
  ├── isRecordingMode: boolean
  └── isDOMStreamingActive: boolean
```

---

## 八、资源释放保障机制

1. **逐步释放 + 超时**：`switchOff()` 每步 5s 超时，`destroyRemoteBrowser()` 整体 30s 超时
2. **异常不中断**：每步在 try/catch/finally 中执行，单步失败不影响后续清理
3. **引用置空**：finally 块中将 `client`/`currentPage`/`context`/`browser` 置为 null
4. **命名空间清理**：销毁时断开所有 Socket、移除命名空间监听、从 `_nsps` Map 中删除
5. **超时自动销毁**：录制会话 10 分钟超时自动触发 `destroyRemoteBrowser()`
6. **过时槽位清理**：`cleanupStaleBrowserSlots()` 仅对 Run 会话的 `reserved` 槽位有效（需 status 为 reserved/initializing 且 browser 为 null 且超过 5 分钟），Recording/Validation 的 `status` 为 `undefined` 且 `browser` 非 null，不受此清理影响
7. **预留锁超时**：`cleanupStaleReservationLocks()` 清理超过 1 分钟的预留锁
8. **降级兜底**：`connectToRemoteBrowser()` 远程连接失败时降级到本地启动浏览器
