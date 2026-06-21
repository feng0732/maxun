# 实时录制通道 Socket.IO 消息协作分析

## 一、整体架构

实时录制功能采用 **前端 + 后端 + Playwright 浏览器** 三层架构，通过 Socket.IO 实现双向实时通信。核心设计是使用 **动态命名空间（Dynamic Namespaces）** 对不同浏览器实例的流量进行多路复用。

```
┌─────────────┐     Socket.IO      ┌─────────────┐     CDP/Playwright     ┌─────────────┐
│  前端 React  │ ◄────────────────► │  Node.js    │ ◄───────────────────► │  远程浏览器   │
│  (录制页面)  │   /{browserId}     │  (服务端)    │                        │ (Chromium)   │
└─────────────┘                    └─────────────┘                        └─────────────┘
```

> **说明**：本文档聚焦**实时录制通道**的 Socket.IO 消息。部分事件同时用于录制模式和运行/解释模式，已在文中注明。

---

## 二、核心文件索引

> 代码引用格式：`相对路径` + 行号（L 前缀），如 `server/src/foo.ts` L12-L15

### 后端

| 文件 | 职责 |
|------|------|
| `server/src/server.ts` | Socket.IO 服务器初始化 |
| `server/src/socket-connection/connection.ts` | Socket 连接建立与事件注册 |
| `server/src/browser-management/controller.ts` | 浏览器生命周期控制器（启动/销毁/超时） |
| `server/src/browser-management/inputHandlers.ts` | 前端输入事件的处理器分发 |
| `server/src/browser-management/classes/RemoteBrowser.ts` | 远程浏览器封装：Playwright 操作、rrweb 录制、截图 |
| `server/src/workflow-management/classes/Generator.ts` | 工作流生成器：录制状态管理、动作生成、保存到数据库 |
| `server/src/routes/record.ts` | 录制 REST API（`/record/start`、`/record/stop`） |

### 前端

| 文件 | 职责 |
|------|------|
| `src/context/socket.tsx` | Socket 连接上下文：建立命名空间连接 |
| `src/utils/browserSocket.ts` | Socket 引用计数缓存：多组件共享连接 |
| `src/pages/RecordingPage.tsx` | 录制页面入口：超时处理、录制生命周期 |
| `src/components/recorder/DOMBrowserRenderer.tsx` | DOM 浏览器渲染器：rrweb 事件接收、iframe 交互事件发送 |
| `src/components/recorder/RightSidePanel.tsx` | 右侧面板：工作流显示、抓取步骤管理、截图接收 |
| `src/components/recorder/SaveRecording.tsx` | 保存录制组件：save 发送、fileSaved 接收 |
| `src/components/browser/BrowserWindow.tsx` | 浏览器窗口壳：DOM 模式切换、高亮显示 |
| `src/context/browserSteps.tsx` | 浏览器步骤状态：抓取步骤的本地管理与 action 发射 |
| `src/context/browserActions.tsx` | 动作模式上下文：当前处于何种抓取模式（文本/列表/截图） |
| `src/api/recording.ts` | 录制 REST API 调用 |

---

## 三、Socket.IO 命名空间机制

每个录制会话使用独立的动态命名空间，格式为 `/{browserId}`。

### 命名空间创建流程

**1. 后端创建命名空间** —— `server/src/browser-management/controller.ts` L28-L29

```typescript
createSocketConnection(
    io.of(id),  // 动态创建以 browserId 命名的 namespace
    userId,
    callback
);
```

**2. 前端连接命名空间** —— `src/context/socket.tsx` L38-L42

```typescript
const socket = io(`${SERVER_ENDPOINT}/${id}`, {
    transports: ["websocket", "polling"],
    rejectUnauthorized: false
});
```

**3. 连接建立回调** —— `server/src/socket-connection/connection.ts` L17-L26

- 注册输入处理器 `registerInputHandlers(socket, userId)`
- 注册断开连接清理 `removeInputHandlers(socket)`
- 调用回调函数执行浏览器初始化

### 命名空间的作用

- **隔离性**：每个浏览器实例独立的消息通道，互不干扰
- **广播能力**：通过 `socket.nsp.emit()` 可以向同一命名空间下的所有客户端广播
- **生命周期**：浏览器销毁时清理命名空间

---

## 四、录制会话生命周期

### 4.1 会话启动流程

```
前端                                   后端                              Playwright
 │                                      │                                  │
 │  1. GET /record/start (HTTP)         │                                  │
 │─────────────────────────────────────►│                                  │
 │                                      │ 2. initializeRemoteBrowserForRecording()
 │                                      │  3. 生成 browserId (uuid)         │
 │                                      │  4. 创建动态命名空间               │
 │                                      │  5. 等待前端 Socket 连接          │
 │                                      │                                  │
 │  6. 连接 /{browserId} (Socket.IO)    │                                  │
 │─────────────────────────────────────►│                                  │
 │                                      │ 7. registerInputHandlers()       │
 │                                      │ 8. 创建 RemoteBrowser 实例        │
 │                                      │ 9. launch Chromium ───────────────►│
 │                                      │ 10. 注入 rrweb 脚本               │
 │                                      │ 11. 开始 DOM 录制                 │
 │                                      │                                  │
 │  12. 'loaded' 事件                   │                                  │
 │◄─────────────────────────────────────│                                  │
```

**关键代码**：
- 启动入口：`server/src/routes/record.ts` L37-L49
- 浏览器初始化：`server/src/browser-management/controller.ts` L26-L103
- rrweb 录制初始化：`server/src/browser-management/classes/RemoteBrowser.ts` L321-L418

### 4.2 会话超时机制

录制会话默认超时时间为 **10 分钟**（`RECORDING_TIMEOUT_MS`）。

- 超时设置：`server/src/browser-management/controller.ts` L14-L15
- 超时触发：`server/src/browser-management/controller.ts` L58-L71
  - 向前端发送 `recording-timeout` 事件
  - 等待 1 秒让前端处理
  - 调用 `destroyRemoteBrowser()` 销毁浏览器

前端超时处理：`src/pages/RecordingPage.tsx` L50-L70

### 4.3 会话销毁流程

销毁入口：`server/src/browser-management/controller.ts` L155-L233

1. 清除录制超时定时器
2. 关闭浏览器页面、上下文、浏览器实例
3. 断开所有 Socket 连接
4. 移除命名空间监听器
5. 从浏览器池中删除

---

## 五、消息流向三段式模型

所有 Socket.IO 消息都可以拆解为三个阶段来分析：

| 阶段 | 说明 | 位置 |
|------|------|------|
| **① 前端单向通知** | 前端 `socket.emit()` 发送事件，不期望即时返回 | 前端 React 组件中 |
| **② 后端监听处理** | 后端 `socket.on()` 接收事件，执行业务逻辑（Playwright 操作、状态变更） | 后端 `inputHandlers.ts` / `RemoteBrowser.ts` / `Generator.ts` |
| **③ 状态变更与推送** | 后端状态改变后，通过 `socket.emit()` 主动推送结果到前端 | 后端各类中 |

并非所有事件都有完整的三段。例如 `request-refresh` 只有阶段①，后端完全不监听；而 `workflow` 事件只有阶段②→③，由后端内部状态变更触发。

---

## 六、关键事件流向详解

### 6.1 `request-refresh` —— 孤立发送（前端发、后端不收）

**结论**：这是一个 **只有前端发出、后端完全不监听** 的"伪事件"。

#### ① 前端单向通知

触发位置：`src/components/recorder/DOMBrowserRenderer.tsx` L883-L884

```typescript
socket.on('rrweb-event', handleRRWebEvent);
socket.emit('request-refresh');   // ← 发出事件，但后端没有任何 .on('request-refresh')
```

**触发时机**：每次组件挂载、rrweb 事件监听器建立完成后立即调用。

**设计意图**：意图是"请求后端重新发送 DOM 全量快照"，但实际上后端并没有实现对应的监听器。rrweb 的 DOM 快照是通过页面内注入的录制脚本持续推送的，不需要额外触发。此事件目前是一个空操作。

#### ② 后端监听处理

**无**。全仓库后端代码搜索 `request-refresh` 无任何 `.on('request-refresh')` 匹配。

#### ③ 状态变更与推送

**无**。此事件不改变任何录制状态。

> **分类**：孤立发送事件（前端 emit，无后端监听）

---

### 6.2 截图流（`captureDirectScreenshot` → `directScreenshotCaptured`）

**完整三段式，但只有成功分支被前端接收**：前端通知 → 后端监听 → Playwright 截图 → 多阶段状态推送

#### ① 前端单向通知

触发位置：`src/components/recorder/RightSidePanel.tsx` L819-L824

```typescript
const screenshotSettings = {
  fullPage: fullPage,
  name: screenshotName,
  actionId: currentScreenshotActionId
};
socket?.emit('captureDirectScreenshot', screenshotSettings);
addScreenshotStep(fullPage, currentScreenshotActionId);  // 乐观更新本地状态
```

**同时做的本地状态变更**（乐观 UI 更新）：
- `addScreenshotStep()` 在 `src/context/browserSteps.tsx` L378-L382 本地添加一个截图步骤（此时还没有截图数据）

#### ② 后端监听处理

监听位置：`server/src/browser-management/classes/RemoteBrowser.ts` L699-L701

```typescript
this.socket.on("captureDirectScreenshot", async (settings) => {
  await this.captureDirectScreenshot(settings);
});
```

实际处理函数：`server/src/browser-management/classes/RemoteBrowser.ts` L619-L668

处理逻辑：
1. 检查 `this.currentPage` 是否存在
2. 调用 Playwright 的 `page.screenshot({ fullPage, type, ... })` 执行截图
3. 将 Buffer 转为 base64 DataURL

#### ③ 状态变更与推送（三分支，仅一分支被前端接收）

后端在处理过程中会发送 **3 种可能的事件**，但前端只监听了其中的成功分支：

| 事件 | 触发条件 | 推送位置 | 前端是否监听 |
|------|---------|---------|------------|
| `screenshotCaptureStarted` | 截图开始前 | `server/src/browser-management/classes/RemoteBrowser.ts` L637-L640 | ❌ 未监听 |
| `directScreenshotCaptured` | 截图成功 | `server/src/browser-management/classes/RemoteBrowser.ts` L655-L661 | ✅ RightSidePanel 监听 |
| `screenshotError` | 无页面或截图异常 | `server/src/browser-management/classes/RemoteBrowser.ts` L629-L632、L664-L667 | ❌ 未监听 |

**前端接收（成功分支）**：
- 监听 `directScreenshotCaptured`：`src/components/recorder/RightSidePanel.tsx` L241-L247
  - `updateScreenshotStepData(step.id, data.screenshot)` —— 将 base64 截图数据填入本地步骤
  - `emitActionForStep(step)` —— 发送 `action` 事件让后端生成 screenshot 工作流动作

> **UX 缺陷**：前端未监听 `screenshotCaptureStarted` 和 `screenshotError`，无法显示"截图中"loading 状态，也无法提示截图失败。
>
> **分类**：
> - `captureDirectScreenshot` —— 正常配对事件
> - `screenshotCaptureStarted` —— 孤立发送（后端发、前端不收）
> - `directScreenshotCaptured` —— 正常配对事件
> - `screenshotError` —— 孤立发送（后端发、前端不收）

```
 前端 RightSidePanel                    后端 RemoteBrowser                  Playwright
       │                                      │                                │
       │ ① socket.emit('captureDirectScreenshot', settings)                    │
       │─────────────────────────────────────►│                                │
       │  乐观 addScreenshotStep()             │                                │
       │                                      │ ② socket.on 接收               │
       │                                      │   captureDirectScreenshot()    │
       │                                      │                                │
       │                                      │ ③-a emit('screenshotCaptureStarted')
       │◄─────────────────────────────────────│                                │
       │  (前端未监听此事件)                    │                                │
       │                                      │                                │
       │                                      │   page.screenshot() ──────────►│
       │                                      │◄───────────────────────────────│
       │                                      │                                │
       │          ┌──────── 成功? ────────┐   │                                │
       │          │ 是                     │否 │                                │
       │          ▼                        ▼  │                                │
       │  ③-b emit('directScreenshotCaptured')│ ③-c emit('screenshotError')    │
       │◄─────────────────────────────────────┼────────────────────────────────│
       │                                      │                                │
       │  updateScreenshotStepData()          │                                │
       │  emitActionForStep() ──────────────► │  → 生成 screenshot 动作        │
       │                                      │   → emit('workflow')           │
       │◄─────────────────────────────────────│                                │
```

---

### 6.3 保存录制（`save` → `fileSaved`）

**完整三段式**：前端通知 → 后端监听 → 工作流优化与入库 → 结果推送

#### ① 前端单向通知

触发位置：`src/components/recorder/SaveRecording.tsx` L105-L127

```typescript
const payload = {
  fileName: (saveRecordingName || recordingName).trim(),
  userId: user.id,
  isLogin: isLogin,
  robotId: retrainRobotId,
};
socket?.emit('save', payload);
setWaitingForSave(true);   // 本地设置等待状态
```

**本地状态变更**：
- `setWaitingForSave(true)` —— 进入等待保存响应的状态

#### ② 后端监听处理

监听位置：`server/src/workflow-management/classes/Generator.ts` L221-L225

```typescript
socket.on('save', (data) => {
  const { fileName, userId, isLogin, robotId } = data;
  this.saveNewWorkflow(fileName, userId, isLogin, robotId);
});
```

实际处理函数 `saveNewWorkflow`：`server/src/workflow-management/classes/Generator.ts` L1059-L1128

**录制状态变更**（后端 Generator 内部）：
1. **文件名校验**：空文件名 → 直接返回错误
2. **重名检查**：查询 Robot 表，检查该用户是否已有同名机器人
3. **工作流优化**（核心状态变更）：调用 `this.optimizeWorkflow(this.workflowRecord)` —— `server/src/workflow-management/classes/Generator.ts` L1417-L1518
   - 将连续的 `press` 动作合并为 `type` 动作
   - 处理 Backspace、Delete 对按键缓冲的影响
4. **写入数据库**：创建或更新 Robot 记录（含 `recording_meta` 和 `workflow` JSON）
5. **设置 `recordingMeta`**：`this.recordingMeta.name = fileName`，`this.recordingMeta.id = robot.id`

#### ③ 状态变更与推送（四分支）

| 事件 `fileSaved` 的 actionType | 触发条件 | 推送位置 |
|--------------------------------|---------|---------|
| `'error'` | 文件名为空 | `server/src/workflow-management/classes/Generator.ts` L1077 |
| `'nameExists'` | 该用户已有同名机器人 | `server/src/workflow-management/classes/Generator.ts` L1087、L1120（数据库唯一约束冲突兜底） |
| `'saved'` | 新建机器人成功（无 retrainRobotId） | `server/src/workflow-management/classes/Generator.ts` L1127 |
| `'retrained'` | 重新训练已有机器人成功（有 retrainRobotId） | `server/src/workflow-management/classes/Generator.ts` L1127 |

**前端接收**：
- 监听位置：`src/components/recorder/SaveRecording.tsx` L138-L143
- 处理函数 `handleFileSaved`：`src/components/recorder/SaveRecording.tsx` L129-L136
  - `'nameExists'` → 显示错误提示，`setWaitingForSave(false)` 继续留在页面
  - 其他 → 调用 `exitRecording()`：调 `stopRecording` REST API 销毁浏览器、`setBrowserId(null)`、关闭窗口

> **分类**：正常配对事件

---

### 6.4 工作流更新（`workflow`）

**单向推送**：此事件 **不由前端任何操作直接触发**，而是后端 Generator 内部状态变化后主动推送。前端只做被动接收。

#### ③ 状态变更与推送（后端 → 前端）

**推送触发点**：`server/src/workflow-management/classes/Generator.ts` L342

```typescript
this.socket.emit('workflow', this.workflowRecord);
```

**推送数据结构**：`this.workflowRecord` —— 包含完整的 `{ workflow: WhereWhatPair[], recording_meta: {...} }`

**所有会触发 `workflow` 推送的后端操作**（通过 `addPairToWorkflowAndNotifyClient` 或直接 emit）：

| 触发动作 | 调用链 |
|---------|--------|
| URL 导航 | `onChangeUrl` → `generator.onGoto` → `addPairToWorkflowAndNotifyClient` |
| 页面点击 | `onDOMClickAction` → `generator.onDOMClickAction` → `addPairToWorkflowAndNotifyClient` |
| 键盘按键 | `onDOMKeyboardAction` → `generator.onDOMKeyboardAction` → `addPairToWorkflowAndNotifyClient` |
| 日期选择 | `onDateSelection` → `generator.onDateSelection` → `addPairToWorkflowAndNotifyClient` |
| 时间选择 | `onTimeSelection` → `generator.onTimeSelection` → `addPairToWorkflowAndNotifyClient` |
| 日期时间选择 | `onDateTimeLocalSelection` → `generator.onDateTimeLocalSelection` → `addPairToWorkflowAndNotifyClient` |
| 下拉选择 | `onDropdownSelection` → `generator.onDropdownSelection` → `addPairToWorkflowAndNotifyClient` |
| 页面后退 | `onGoBack` → `generator.onGoBack` → 直接 emit |
| 页面前进 | `onGoForward` → `generator.onGoForward` → 直接 emit |
| 自定义动作（scrapeList/screenshot/scrapeSchema） | `onGenerateAction` → `generator.customAction` → `addPairToWorkflowAndNotifyClient` |
| 删除动作 | `onRemoveAction` → `generator.removeAction` → 直接 emit |
| 直接 screenshot 步骤完成 | `RightSidePanel.emitActionForStep` → `socket.emit('action')` → 同上自定义动作 |

`addPairToWorkflowAndNotifyClient` 实现：`server/src/workflow-management/classes/Generator.ts` L270-L344

其内部的 **录制状态变更** 包括：
1. 根据 URL 和 selectors 判断是否与已有 workflow pair 合并（同页操作追加到同一个 what 数组）
2. 处理 shadow DOM 选择器遮蔽问题 `handleOverShadowing`
3. 自动追加 `waitForLoadState: networkidle` 动作
4. 按 `this.generatedData.lastIndex` 插入 workflow 数组
5. **最后** `this.socket.emit('workflow', this.workflowRecord)` 推送

#### 前端接收

监听位置：`src/components/recorder/RightSidePanel.tsx` L162-L178

```typescript
useEffect(() => {
  if (socket) {
    socket.on("workflow", workflowHandler);
  }
  // ... HTTP fallback: 每 15 分钟轮询 fetchWorkflow()
}, [id, socket, workflowHandler]);
```

处理函数 `workflowHandler`：`src/components/recorder/RightSidePanel.tsx` L75-L77

```typescript
const workflowHandler = useCallback((data: WorkflowFile) => {
  setWorkflow(data);
}, [setWorkflow]);
```

**本地状态变更**：
- `setWorkflow(data)` —— 更新 ActionContext 中的完整工作流状态
- 后续依赖 `workflow` 的 useEffect 会重新计算 `currentWorkflowActionsState`（显示哪些抓取按钮可用）

> **分类**：正常配对事件

```
  后端 Generator 内部状态变化              Socket                  前端 RightSidePanel
       │                                      │                            │
       │  addPairToWorkflowAndNotifyClient()  │                            │
       │    │ 修改 this.workflowRecord         │                            │
       │    ▼                                  │                            │
       │  this.socket.emit('workflow', record) │                            │
       │──────────────────────────────────────►│                            │
       │                                      │  socket.on("workflow",     │
       │                                      │    workflowHandler)         │
       │                                      │      │                     │
       │                                      │      ▼                     │
       │                                      │  setWorkflow(data)         │
       │                                      │    → 更新 ActionContext     │
       │                                      │    → 重新计算按钮可用性    │
```

---

### 6.5 元素高亮与选择器弹窗（`highlighter`、`showDropdown`、`showDatePicker` 等）

**孤立发送事件**：后端 Generator 在处理点击/操作时会 emit 这些事件，但前端录制页面**没有通过 Socket 监听**这些事件。

#### 后端发送位置

| 事件 | 发送位置 | 发送时机 |
|------|---------|---------|
| `highlighter` | `server/src/workflow-management/classes/Generator.ts` L1205、L1207、L1210 | `generateDataForHighlighter` 生成高亮数据后 |
| `showDropdown` | `server/src/workflow-management/classes/Generator.ts` L546 | 点击 `<select>` 下拉框时 |
| `showDatePicker` | `server/src/workflow-management/classes/Generator.ts` L557 | 点击 `type="date"` 输入框时 |
| `showTimePicker` | `server/src/workflow-management/classes/Generator.ts` L567 | 点击 `type="time"` 输入框时 |
| `showDateTimePicker` | `server/src/workflow-management/classes/Generator.ts` L577 | 点击 `type="datetime-local"` 输入框时 |

#### 前端监听情况

前端录制页面**无任何 `socket.on('highlighter')`、`socket.on('showDropdown')` 等监听**。

这些功能在前端的实际实现方式是：
- **元素高亮**：由 `DOMBrowserRenderer.tsx` 在本地通过 `clientSelectorGenerator.ts` 计算，不依赖后端 Socket 事件
- **日期/下拉选择器**：由 `DOMBrowserRenderer.tsx` 在 iframe 内直接捕获点击事件后本地弹出，不通过 Socket 从后端接收

#### 可能的用途推测

这些后端 emit 可能是：
1. **历史遗留代码**：早期架构中由后端驱动 UI，后来改为前端驱动后未清理
2. **运行/解释模式使用**：在机器人运行模式下由其他组件监听（但搜索结果显示运行模式也未监听这些事件）
3. **多端同步预留**：预留用于多客户端同步同一浏览器会话

> **分类**：孤立发送事件（后端 emit，前端无监听）

---

### 6.6 `listDataExtracted` —— 孤立监听（前端收、后端不发）

**结论**：前端有监听，但后端代码中**完全没有 emit 此事件**。属于孤立监听。

#### 前端监听

监听位置：`src/components/recorder/RightSidePanel.tsx` L212-L223

```typescript
socket.on('listDataExtracted', (response) => {
  if (!isDOMMode) {
    const { currentListId, data } = response;
    updateListStepData(currentListId, data);
  }
});
```

**注意**：监听条件是 `!isDOMMode`，即**非 DOM 模式**（也就是运行/解释模式）下才处理。录制模式下即使收到也会忽略。

#### 后端发送

**无**。全仓库后端代码搜索 `listDataExtracted` 无任何 `.emit('listDataExtracted')` 匹配。

#### 设计意图推测

- 可能是为**列表数据实时预览**预留的接口（运行时边抓取边显示数据）
- 目前功能未实现或已废弃，只留下了前端监听桩

> **分类**：孤立监听事件（前端 on，无后端 emit）

---

### 6.7 `dom-mode-error` —— 浏览器启动失败通知

**完整配对**：后端在浏览器启动失败时发送，前端 BrowserWindow 组件监听。

#### 后端发送

发送位置：`server/src/browser-management/controller.ts` L50-L53、L76-L79

有两个发送点：
1. **浏览器池添加失败**（L50）：`browserPool.addRemoteBrowser()` 返回 false 时
2. **浏览器初始化异常**（L76）：`remoteBrowser.initialize()` 抛出异常时

发送数据：
```typescript
socket.emit('dom-mode-error', {
  userId,
  error: 'Failed to start the browser, please try again in some time.'
});
```

#### 前端监听

监听位置：`src/components/browser/BrowserWindow.tsx` L940（不是 RecordingPage！）

处理函数 `domModeErrorHandler`：`src/components/browser/BrowserWindow.tsx` L196-L204

```typescript
const domModeErrorHandler = useCallback(
    (data: any) => {
        if (!data.userId || data.userId === user?.id) {
            updateDOMMode(false);
            // ... 显示错误提示
        }
    },
    [user?.id, updateDOMMode]
);
```

**状态变更**：调用 `updateDOMMode(false)` 关闭 DOM 模式。

> **修正说明**：原速查表中标注前端监听位置为 "RecordingPage"，实际为 `BrowserWindow.tsx`。已修正。
>
> **分类**：正常配对事件

---

### 6.8 `dom-mode-enabled` —— 方向容易搞反的事件

**注意**：这是一个 **前端发送、后端监听** 的事件，方向与 `dom-mode-error` 相反。

#### ① 前端发送

发送位置：`src/components/browser/BrowserWindow.tsx` L190

```typescript
const domModeHandler = useCallback(
    (data: any) => {
        if (!data.userId || data.userId === user?.id) {
            updateDOMMode(true);
            socket?.emit("dom-mode-enabled");  // ← 本地状态变更后，通知后端
        }
    },
    [user?.id, updateDOMMode, socket]
);
```

**触发时机**：前端收到其他来源的 `dom-mode-enabled` 事件后，更新本地状态并再次 emit 到后端（相当于一个确认/同步机制）。

#### ② 后端监听

监听位置：`server/src/workflow-management/classes/Generator.ts` L204-L207

```typescript
this.socket.on('dom-mode-enabled', () => {
  this.isDOMMode = true;
  logger.log('debug', 'Generator: DOM mode enabled');
});
```

**状态变更**：设置 `this.isDOMMode = true`，影响 Generator 的某些行为（如元素选择器生成策略）。

#### ③ 状态变更与推送

**无主动推送**。后端只更新内部状态，不主动 emit 回复。

> **分类**：正常配对事件（前端 → 后端方向）

---

## 七、其他核心数据流

### 7.1 DOM 实时流（rrweb）

**流向**：浏览器页面 → 后端桥接 → 前端 replayer

```
Chromium 页面
    │
    │  rrweb.record() 录制 DOM 事件
    ▼
window.emitEventToBackend(event)  [页面内注入的 JS]
    │
    │  page.exposeFunction 桥接
    ▼
Node.js (RemoteBrowser)
    │
    │  socket.emit('rrweb-event', event)
    ▼
前端 (DOMBrowserRenderer)
    │
    │  replayer.addEvent(event)
    ▼
rrweb Replayer 渲染 DOM 到 iframe
```

**关键代码**：
- 后端注入 rrweb：`server/src/browser-management/classes/RemoteBrowser.ts` L321-L418
- 后端暴露桥接函数：`server/src/browser-management/classes/RemoteBrowser.ts` L356-L364
- 前端接收渲染：`src/components/recorder/DOMBrowserRenderer.tsx` L808-L889

> **分类**：正常配对事件（后端 → 前端方向）

### 7.2 点击操作流

**三段式完整流程**：

```
前端 DOMBrowserRenderer iframe
    │  ① mousedown 事件 → 生成 selector
    ▼
socket.emit('dom:click', { selector, elementInfo, coordinates, ... })
    │
    ▼
后端 inputHandlers
    │  ② handleWrapper → handleClickAction
    │
    ├─► page.click(selector)  // Playwright 执行点击（页面状态变更）
    │
    └─► generator.onDOMClickAction(page, data)
           │
           ▼  ③ (工作流状态变更)
        addPairToWorkflowAndNotifyClient(pair, page)
           │
           ▼
        socket.emit('workflow', workflowRecord)
           │
           ▼
        前端 setWorkflow() 更新 UI
```

**关键代码**：
- 前端点击处理：`src/components/recorder/DOMBrowserRenderer.tsx` L367-L588
- 后端点击处理：`server/src/browser-management/inputHandlers.ts` L445-L585

### 7.3 自定义动作流（抓取/截图）

```
前端 RightSidePanel
    │  用户点击"抓取列表/截图"按钮
    ▼
browserSteps.addListStep() / addScreenshotStep()  // ① 乐观更新本地状态
    │
    ▼
browserSteps.emitActionForStep(step)
    │
    ▼
socket.emit('action', { action: 'scrapeList'|'screenshot'|'scrapeSchema', actionId, settings })
    │
    ▼
后端 inputHandlers
    │  ② onGenerateAction → handleGenerateAction
    ▼
generator.customAction(action, actionId, settings, page)
    │
    ▼  ③ addPairToWorkflowAndNotifyClient → socket.emit('workflow')
```

**关键代码**：
- 前端动作发射：`src/context/browserSteps.tsx` L152-L226
- 后端动作处理：`server/src/browser-management/inputHandlers.ts` L75-L103

### 7.4 键盘输入流

```
前端 iframe keydown
    │  ① 生成 selector
    ▼
socket.emit('dom:keypress', { selector, key, inputType, ... })
    │
    ▼
后端 handleKeyboardAction
    │
    ├─► page.press(selector, key)  // ② Playwright 执行按键
    │
    └─► generator.onDOMKeyboardAction(page, data)
           │
           ▼  ③ 生成 press 动作，addPairToWorkflowAndNotifyClient
```

**保存优化**：`Generator.optimizeWorkflow` 在保存时将连续 `press` 合并为 `type` 动作：`server/src/workflow-management/classes/Generator.ts` L1417-L1518

---

## 八、完整消息速查表

### 8.1 前端 → 后端（`socket.emit`）

> ✅ = 正常配对（后端有监听），❌ = 孤立发送（后端无监听）

| 事件名 | 前端发送位置 | 后端监听位置 | 配对状态 | 处理后是否 emit 回复 |
|--------|------------|------------|---------|-------------------|
| `input:url` | DOMBrowserRenderer | `onChangeUrl` → `handleChangeUrl` | ✅ | 是（`workflow`） |
| `input:refresh` | DOMBrowserRenderer | `onRefresh` → `handleRefresh` | ✅ | 否 |
| `input:back` | DOMBrowserRenderer | `onGoBack` → `handleGoBack` | ✅ | 是（`workflow`） |
| `input:forward` | DOMBrowserRenderer | `onGoForward` → `handleGoForward` | ✅ | 是（`workflow`） |
| `input:keyup` | DOMBrowserRenderer | `onKeyup` → `handleKeyup` | ✅ | 否（仅回放） |
| `input:date` | DOMBrowserRenderer | `onDateSelection` → `handleDateSelection` | ✅ | 是（`workflow`） |
| `input:time` | DOMBrowserRenderer | `onTimeSelection` → `handleTimeSelection` | ✅ | 是（`workflow`） |
| `input:datetime-local` | DOMBrowserRenderer | `onDateTimeLocalSelection` → `handleDateTimeLocalSelection` | ✅ | 是（`workflow`） |
| `input:dropdown` | DOMBrowserRenderer | `onDropdownSelection` → `handleDropdownSelection` | ✅ | 是（`workflow`） |
| `dom:click` | DOMBrowserRenderer | `onDOMClickAction` → `handleClickAction` | ✅ | 是（`workflow`） |
| `dom:keypress` | DOMBrowserRenderer | `onDOMKeyboardAction` → `handleKeyboardAction` | ✅ | 是（`workflow`） |
| `dom:scroll` | DOMBrowserRenderer | 页面滚动事件监听 | ✅ | 否（仅同步） |
| `action` | browserSteps.emitActionForStep | `onGenerateAction` → `handleGenerateAction` | ✅ | 是（`workflow`/`decision`） |
| `removeAction` | 前端编辑面板 | `onRemoveAction` → `handleRemoveAction` | ✅ | 是（`workflow`） |
| `save` | SaveRecording | Generator.registerEventHandlers | ✅ | 是（`fileSaved`） |
| `new-recording` | 前端 | Generator.registerEventHandlers | ✅ | 否 |
| `updatePair` | 前端编辑 | Generator.registerEventHandlers | ✅ | 否 |
| `activeIndex` | 前端编辑 | Generator.registerEventHandlers | ✅ | 否 |
| `changeTab` | 前端标签栏 | RemoteBrowser.registerEditorEvents | ✅ | 否 |
| `addTab` | 前端标签栏 | RemoteBrowser.registerEditorEvents | ✅ | 否 |
| `closeTab` | 前端标签栏 | RemoteBrowser.registerEditorEvents | ✅ | 否 |
| `captureDirectScreenshot` | RightSidePanel | RemoteBrowser.registerSocketEvents | ✅ | 是（`directScreenshotCaptured` 等） |
| `setGetList` | 前端 | RemoteBrowser.initializeSocketListeners | ✅ | 否 |
| `listSelector` | 前端 | RemoteBrowser.initializeSocketListeners | ✅ | 否 |
| `setPaginationMode` | 前端 | RemoteBrowser.initializeSocketListeners | ✅ | 否 |
| `testPaginationScroll` | 前端 | RemoteBrowser.onTestPaginationScroll | ✅ | 是（`paginationScrollTestResult`） |
| `dom-mode-enabled` | BrowserWindow | Generator.initializeDOMListeners | ✅ | 否 |
| `request-refresh` | DOMBrowserRenderer | **无** | ❌ 孤立发送 | 否 |

### 8.2 后端 → 前端（`socket.emit`）

> ✅ = 正常配对（前端有监听），❌ = 孤立发送（前端无监听），⚠️ = 仅运行模式监听

| 事件名 | 后端发送位置 | 前端监听位置 | 配对状态 | 作用 |
|--------|------------|------------|---------|------|
| `rrweb-event` | RemoteBrowser（桥接页面内事件） | DOMBrowserRenderer | ✅ | DOM 实时渲染 |
| `domLoadingProgress` | RemoteBrowser.emitLoadingProgress | DOMBrowserRenderer、BrowserWindow | ✅ | 加载进度条 |
| `dom-snapshot-loading` | RemoteBrowser.initialize | RunsTable（运行模式） | ⚠️ 仅运行模式 | 开始加载提示 |
| `dom-mode-error` | controller.initializeRemoteBrowserForRecording | BrowserWindow | ✅ | 启动失败提示 |
| `workflow` | Generator 多处 | RightSidePanel | ✅ | 工作流同步 |
| `fileSaved` | Generator.saveNewWorkflow | SaveRecording | ✅ | 保存结果通知 |
| `highlighter` | Generator.generateDataForHighlighter | **无** | ❌ 孤立发送 | 元素高亮 |
| `showDropdown` | Generator.onClick | **无** | ❌ 孤立发送 | 显示下拉选择器 |
| `showDatePicker` | Generator.onClick | **无** | ❌ 孤立发送 | 显示日期选择器 |
| `showTimePicker` | Generator.onClick | **无** | ❌ 孤立发送 | 显示时间选择器 |
| `showDateTimePicker` | Generator.onClick | **无** | ❌ 孤立发送 | 显示日期时间选择器 |
| `decision` | Generator.customAction | InterpretationButtons（运行模式） | ⚠️ 仅运行模式 | 用户决策对话框 |
| `urlChanged` | Generator、RemoteBrowser | BrowserNavBar、RunsTable | ✅ | URL 同步 |
| `newTab` | Generator.notifyOnNewTab | BrowserContent | ✅ | 新标签通知 |
| `tabHasBeenClosed` | Generator.notifyOnNewTab | BrowserContent | ✅ | 标签关闭通知 |
| `loaded` | controller.initializeRemoteBrowserForRecording | RecordingPage | ✅ | 浏览器就绪 |
| `recording-timeout` | controller 超时回调 | RecordingPage | ✅ | 超时销毁 |
| `screenshotCaptureStarted` | RemoteBrowser.captureDirectScreenshot | **无** | ❌ 孤立发送 | 截图开始 |
| `directScreenshotCaptured` | RemoteBrowser.captureDirectScreenshot | RightSidePanel | ✅ | 截图成功数据 |
| `screenshotError` | RemoteBrowser.captureDirectScreenshot | **无** | ❌ 孤立发送 | 截图失败 |
| `paginationScrollTestResult` | inputHandlers.handleTestPaginationScroll | RightSidePanel | ✅ | 滚动分页测试结果 |
| `listDataExtracted` | **无** | RightSidePanel（非 DOM 模式） | ❌ 孤立监听 | 列表数据预览 |

---

## 九、孤立事件汇总

### 9.1 孤立发送（后端 emit，前端无监听）

共 **7 个** 事件：

| 事件 | 后端发送位置 | 推测用途 |
|------|------------|---------|
| `highlighter` | `server/src/workflow-management/classes/Generator.ts` L1205、L1207、L1210 | 元素高亮数据推送（前端改为本地计算） |
| `showDropdown` | `server/src/workflow-management/classes/Generator.ts` L546 | 下拉选择器弹窗（前端改为本地弹出） |
| `showDatePicker` | `server/src/workflow-management/classes/Generator.ts` L557 | 日期选择器弹窗（前端改为本地弹出） |
| `showTimePicker` | `server/src/workflow-management/classes/Generator.ts` L567 | 时间选择器弹窗（前端改为本地弹出） |
| `showDateTimePicker` | `server/src/workflow-management/classes/Generator.ts` L577 | 日期时间选择器弹窗（前端改为本地弹出） |
| `screenshotCaptureStarted` | `server/src/browser-management/classes/RemoteBrowser.ts` L637-L640 | 截图开始通知（前端未实现 loading 状态） |
| `screenshotError` | `server/src/browser-management/classes/RemoteBrowser.ts` L629-L632、L664-L667 | 截图失败通知（前端未实现错误提示） |

### 9.2 孤立监听（前端 on，后端无 emit）

共 **1 个** 事件：

| 事件 | 前端监听位置 | 推测用途 |
|------|------------|---------|
| `listDataExtracted` | `src/components/recorder/RightSidePanel.tsx` L212-L223 | 列表数据实时预览（功能未实现或已废弃） |

### 9.3 方向容易混淆的事件

| 事件 | 方向 | 说明 |
|------|------|------|
| `dom-mode-enabled` | 前端 → 后端 | 前端通知后端"已进入 DOM 模式"，后端设置 `isDOMMode = true` |
| `dom-mode-error` | 后端 → 前端 | 后端通知前端"浏览器启动失败"，前端退出 DOM 模式 |

---

## 十、状态管理与同步

### 10.1 后端状态

**RemoteBrowser 类** 维护浏览器会话状态：
- `browser` / `context` / `currentPage`：Playwright 实例
- `generator`：工作流生成器（维护 `workflowRecord`）
- `interpreter`：工作流解释器
- `isDOMStreamingActive`：DOM 流是否激活
- `isRecordingMode`：是否为录制模式

**Generator 类** 维护录制核心状态：
- `workflowRecord`：当前完整工作流（`{ workflow: WhereWhatPair[], recording_meta }`）
- `generatedData.lastIndex`：当前活动插入位置
- `keyboardEventsBuffer`：按键缓冲（保存优化用）
- `recordingMeta`：录制元信息（name、id 等）
- `isDOMMode`：是否为 DOM 模式（影响选择器生成策略）

**BrowserPool 类** 管理所有浏览器实例：
- 按用户 ID 分组
- 限制每个用户的浏览器数量
- 跟踪浏览器状态（recording / run）

### 10.2 前端状态

前端使用多个 Context 管理状态：

| Context | 状态内容 | 文件 |
|--------|---------|------|
| `SocketStore` | Socket 连接、browserId、队列 Socket | `src/context/socket.tsx` |
| `BrowserStepsStore` | 抓取步骤列表（文本/列表/截图） | `src/context/browserSteps.tsx` |
| `GlobalInfoStore` | 全局信息（录制 URL、名称、模式等） | `src/context/globalInfo.tsx` |
| `ActionContext` | 工作流数据 + 当前动作模式 | `src/context/browserActions.tsx` |

### 10.3 工作流同步机制

工作流是录制状态的核心，采用 **后端单一数据源** 模式：

1. 所有工作流变更都在后端 `Generator` 中发生
2. 每次变更后通过 `socket.emit('workflow', this.workflowRecord)` 推送给前端
3. 前端接收后 `setWorkflow(data)` 更新本地状态和 UI
4. **兜底机制**：前端同时每 15 分钟通过 HTTP `fetchWorkflow` 轮询一次（Socket 丢消息时的备份）

---

## 十一、关键设计模式

### 11.1 包装器模式（Wrapper Pattern）

输入处理器使用三层包装：
- 外层 `onXxx`：socket 事件回调，日志记录
- 中层 `handleWrapper`：获取活跃浏览器，检查解释器状态
- 内层 `handleXxx`：实际的业务逻辑

代码示例：`server/src/browser-management/inputHandlers.ts` L28-L57

### 11.2 广播模式（Broadcast Pattern）

使用 `socket.nsp.emit()` 向命名空间内所有客户端广播，确保重连后新的 Socket 也能收到消息。

代码：`server/src/browser-management/classes/RemoteBrowser.ts` L206-L216

### 11.3 引用计数 Socket 缓存

前端 `browserSocket.ts` 使用引用计数管理 Socket 连接，多个组件共享同一连接。

代码：`src/utils/browserSocket.ts` L4-L45

### 11.4 乐观 UI 更新

前端执行操作时先乐观更新本地状态（如标签页、步骤列表），再通过 Socket 同步后端，提升用户体验。典型例子：
- 截图时先 `addScreenshotStep()` 添加空步骤，收到 `directScreenshotCaptured` 后再填入数据
- 抓取列表时先 `addListStep()` 添加空步骤

---

## 十二、错误与异常处理

| 场景 | 处理方式 | 代码位置 |
|------|---------|---------|
| 浏览器启动失败 | 捕获异常，发送 `dom-mode-error`，清理资源 | `server/src/browser-management/controller.ts` L49-L79 |
| 页面已关闭 | 动作执行前检查 `page.isClosed()` | 各 handleXxx 函数 |
| 解释运行中输入 | `handleWrapper` 检查 `interpreter.interpretationInProgress()`，忽略输入 | `server/src/browser-management/inputHandlers.ts` handleWrapper |
| Socket 重连 | `RemoteBrowser.updateSocket()` 支持更新 Socket 实例，重连后重新注册监听器 | RemoteBrowser.updateSocket |
| 保存时重名 | 前后端双重检查：前端检查 `recordings` 列表，后端查询 Robot 表 + 数据库唯一约束 | SaveRecording.saveRecording、Generator.saveNewWorkflow |
| 截图失败 | 发送 `screenshotError`（前端当前未监听） | `server/src/browser-management/classes/RemoteBrowser.ts` L629-L632 |

---

## 十三、性能优化点

1. **rrweb 采样配置**：`server/src/browser-management/classes/RemoteBrowser.ts` L388-L396
   - `mousemove: false`（禁用鼠标移动采样）
   - `scroll: 75ms`（滚动采样间隔）
   - `input: 'last'`（输入只保留最后一个）

2. **工作流优化**：保存时 `optimizeWorkflow()` 将连续按键操作为 `type` 动作，减少动作数量

3. **前端节流**：鼠标移动、滚动事件都有节流处理

4. **机器人运行模式**：非录制模式（`!isRecordingMode`）跳过 rrweb 注入，大幅提升性能
