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
| `addRemoteBrowser()` | 直接添加浏览器到池，检查用户槽位上限 |
| `reserveBrowserSlotAtomic()` | 原子预留槽位（防止竞态），状态为 "reserved" |
| `upgradeBrowserSlot()` | 将 reserved 槽位升级为 ready（绑定实际 Browser 实例） |
| `failBrowserSlot()` | 标记槽位失败，清理后删除 |
| `getRemoteBrowser()` | 获取浏览器实例，reserved/failed 状态返回 undefined |

**槽位状态机**：

```
reserved  →  initializing  →  ready
                │
                └──→  failed  →  (删除)
```

**过时清理**：`cleanupStaleBrowserSlots()` 定期清理 reserved/initializing 状态超过 5 分钟的槽位，防止资源泄露。

---

## 三、会话释放

### 3.1 switchOff — 单个浏览器关闭

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

### 3.2 destroyRemoteBrowser — 完整销毁流程

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

### 3.3 Recording 超时自动清理

文件：[controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/107-maxun/server/src/browser-management/controller.ts#L58-L71)

录制会话设置 10 分钟（`RECORDING_TIMEOUT_MS`）超时：

```ts
const timeoutHandle = setTimeout(async () => {
    io.of(id).emit('recording-timeout');     // 通知前端
    await new Promise(resolve => setTimeout(resolve, 1000)); // 等待前端处理
    await destroyRemoteBrowser(id, userId);  // 销毁浏览器
}, RECORDING_TIMEOUT_MS);
```

### 3.4 BrowserPool 清理方法

| 方法 | 场景 |
|------|------|
| `closeAndDeleteBrowser()` | 从池中移除并删除映射（不关闭浏览器本身） |
| `deleteRemoteBrowser()` | 仅从池中移除，不尝试关闭 |
| `failBrowserSlot()` | 标记失败 → 调用 `switchOff()` → 删除 |
| `cleanupStaleBrowserSlots()` | 清理超过 5 分钟的 reserved/initializing 槽位 |

---

## 四、三种会话类型对比

| 维度 | Recording 会话 | Run 会话 | Validation 会话 |
|------|---------------|----------|-----------------|
| **入口** | `initializeRemoteBrowserForRecording()` | `createRemoteBrowserForRun()` | `createRemoteBrowserForValidation()` |
| **isRecordingMode** | `true` | `false` | `false` |
| **rrweb** | ✅ 启用 | ❌ 跳过 | ❌ 跳过 |
| **Socket** | 真实 Socket（前端连接） | 真实或 Dummy Socket | Dummy Socket |
| **槽位策略** | 每用户最多 1 个 recording | 每用户最多 2 个（含 recording） | 通过 addRemoteBrowser 添加 |
| **复用** | ✅ `updateSocket()` | ❌ 不复用 | ❌ 不复用 |
| **超时** | 10 分钟自动销毁 | 无自动超时 | 无自动超时 |
| **状态流转** | reserved → ready | reserved → ready | 直接添加 |
| **典型用途** | 用户录制操作流程 | 机器人执行工作流 | SDK 校验任务 |

---

## 五、Socket 通信与会话的绑定

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

## 六、关键对象关系图

```
BrowserPool
  │
  ├── pool: { [id]: BrowserPoolInfo }
  │       │
  │       └── BrowserPoolInfo
  │             ├── browser: RemoteBrowser | null
  │             ├── active: boolean
  │             ├── userId: string
  │             ├── state: "recording" | "run"
  │             ├── status: "reserved" | "initializing" | "ready" | "failed"
  │             ├── createdAt: number
  │             └── lastAccessed: number
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

## 七、资源释放保障机制

1. **逐步释放 + 超时**：`switchOff()` 每步 5s 超时，`destroyRemoteBrowser()` 整体 30s 超时
2. **异常不中断**：每步在 try/catch/finally 中执行，单步失败不影响后续清理
3. **引用置空**：finally 块中将 `client`/`currentPage`/`context`/`browser` 置为 null
4. **命名空间清理**：销毁时断开所有 Socket、移除命名空间监听、从 `_nsps` Map 中删除
5. **超时自动销毁**：录制会话 10 分钟超时自动触发 `destroyRemoteBrowser()`
6. **过时槽位清理**：`cleanupStaleBrowserSlots()` 定期清理卡在 reserved/initializing 的槽位
7. **预留锁超时**：`cleanupStaleReservationLocks()` 清理超过 1 分钟的预留锁
8. **降级兜底**：`connectToRemoteBrowser()` 远程连接失败时降级到本地启动浏览器
