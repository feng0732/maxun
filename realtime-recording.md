# 实时录制通道 Socket.IO 消息协作分析

## 一、整体架构

实时录制功能采用 **前端 + 后端 + Playwright 浏览器** 的三层架构，通过 Socket.IO 实现双向实时通信。核心设计是使用 **动态命名空间（Dynamic Namespaces）** 对不同浏览器实例的流量进行多路复用。

```
┌─────────────┐     Socket.IO      ┌─────────────┐     CDP/Playwright     ┌─────────────┐
│  前端 React  │ ◄────────────────► │  Node.js    │ ◄───────────────────► │  远程浏览器   │
│  (录制页面)  │   /{browserId}     │  (服务端)    │                        │ (Chromium)   │
└─────────────┘                    └─────────────┘                        └─────────────┘
```

## 二、核心文件索引

### 后端
- [server.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/server.ts) - Socket.IO 服务器初始化
- [connection.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/socket-connection/connection.ts) - Socket 连接建立
- [controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/browser-management/controller.ts) - 浏览器管理控制器
- [inputHandlers.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/browser-management/inputHandlers.ts) - 输入事件处理器
- [RemoteBrowser.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/browser-management/classes/RemoteBrowser.ts) - 远程浏览器类
- [Generator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/workflow-management/classes/Generator.ts) - 工作流生成器
- [record.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/routes/record.ts) - 录制 REST API 路由

### 前端
- [socket.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/src/context/socket.tsx) - Socket 上下文
- [browserSocket.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/src/utils/browserSocket.ts) - 浏览器 Socket 工具
- [RecordingPage.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/src/pages/RecordingPage.tsx) - 录制页面入口
- [DOMBrowserRenderer.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/src/components/recorder/DOMBrowserRenderer.tsx) - DOM 浏览器渲染器
- [browserSteps.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/src/context/browserSteps.tsx) - 浏览器步骤状态管理
- [recording.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/src/api/recording.ts) - 录制 API 调用

## 三、Socket.IO 命名空间机制

每个录制会话使用独立的动态命名空间，格式为 `/{browserId}`。

### 命名空间创建流程

1. **后端创建命名空间**：在 [controller.ts#L28](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/browser-management/controller.ts#L28-L29) 中
   ```typescript
   createSocketConnection(
       io.of(id),  // 动态创建以 browserId 命名的 namespace
       userId,
       callback
   );
   ```

2. **前端连接命名空间**：在 [socket.tsx#L38-L42](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/src/context/socket.tsx#L38-L42) 中
   ```typescript
   const socket = io(`${SERVER_ENDPOINT}/${id}`, {
       transports: ["websocket", "polling"],
       rejectUnauthorized: false
   });
   ```

3. **连接建立回调**：在 [connection.ts#L17-L26](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/socket-connection/connection.ts#L17-L26) 中
   - 注册输入处理器 `registerInputHandlers(socket, userId)`
   - 注册断开连接清理 `removeInputHandlers(socket)`
   - 调用回调函数执行浏览器初始化

### 命名空间的作用
- **隔离性**：每个浏览器实例独立的消息通道，互不干扰
- **广播能力**：通过 `socket.nsp.emit()` 可以向同一命名空间下的所有客户端广播
- **生命周期**：浏览器销毁时清理命名空间

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

**关键代码点**：
- 启动入口：[record.ts#L37-L49](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/routes/record.ts#L37-L49)
- 浏览器初始化：[controller.ts#L26-L103](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/browser-management/controller.ts#L26-L103)
- rrweb 录制初始化：[RemoteBrowser.ts#L321-L418](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L321-L418)

### 4.2 会话超时机制

录制会话默认超时时间为 **10 分钟**（`RECORDING_TIMEOUT_MS`）。

- 超时设置：[controller.ts#L14-L15](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/browser-management/controller.ts#L14-L15)
- 超时触发：[controller.ts#L58-L71](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/browser-management/controller.ts#L58-L71)
  - 向前端发送 `recording-timeout` 事件
  - 等待 1 秒让前端处理
  - 调用 `destroyRemoteBrowser()` 销毁浏览器

前端超时处理：[RecordingPage.tsx#L50-L70](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/src/pages/RecordingPage.tsx#L50-L70)

### 4.3 会话销毁流程

销毁入口：[controller.ts#L155-L233](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/browser-management/controller.ts#L155-L233)

1. 清除录制超时定时器
2. 关闭浏览器页面、上下文、浏览器实例
3. 断开所有 Socket 连接
4. 移除命名空间监听器
5. 从浏览器池中删除

## 五、消息分类与流向

### 5.1 前端 → 后端消息

#### 输入操作类

| 消息事件 | 触发场景 | 处理器 | 作用 |
|---------|---------|--------|------|
| `input:url` | 地址栏输入 URL | `onChangeUrl` → `handleChangeUrl` | 导航到新 URL 并生成 goto 动作 |
| `input:refresh` | 刷新按钮 | `onRefresh` → `handleRefresh` | 刷新页面 |
| `input:back` | 后退按钮 | `onGoBack` → `handleGoBack` | 页面后退并生成 goBack 动作 |
| `input:forward` | 前进按钮 | `onGoForward` → `handleGoForward` | 页面前进并生成 goForward 动作 |
| `input:keyup` | 键盘按键释放 | `onKeyup` → `handleKeyup` | 在远程浏览器释放按键（仅回放，不生成工作流） |
| `input:date` | 日期选择器 | `onDateSelection` → `handleDateSelection` | 选择日期并生成 fill 动作 |
| `input:dropdown` | 下拉选择 | `onDropdownSelection` → `handleDropdownSelection` | 选择下拉选项并生成 selectOption 动作 |
| `input:time` | 时间选择器 | `onTimeSelection` → `handleTimeSelection` | 选择时间并生成 fill 动作 |
| `input:datetime-local` | 日期时间选择器 | `onDateTimeLocalSelection` → `handleDateTimeLocalSelection` | 选择日期时间并生成 fill 动作 |

#### DOM 交互类

| 消息事件 | 触发场景 | 处理器 | 作用 |
|---------|---------|--------|------|
| `dom:click` | 点击页面元素 | `onDOMClickAction` → `handleClickAction` | 执行点击并生成 click 动作 |
| `dom:keypress` | 键盘按键按下 | `onDOMKeyboardAction` → `handleKeyboardAction` | 执行按键并生成 press 动作 |
| `dom:scroll` | 滚动页面 | `setupScrollEventListener` | 同步滚动到远程浏览器 |

#### 标签页管理类

| 消息事件 | 触发场景 | 处理器 | 作用 |
|---------|---------|--------|------|
| `changeTab` | 切换标签 | `registerEditorEvents` | 切换到指定索引的标签页 |
| `addTab` | 新建标签 | `registerEditorEvents` | 新建标签页 |
| `closeTab` | 关闭标签 | `registerEditorEvents` | 关闭指定标签页 |

#### 工作流操作类

| 消息事件 | 触发场景 | 处理器 | 作用 |
|---------|---------|--------|------|
| `action` | 添加自定义动作（抓取/截图等） | `onGenerateAction` → `handleGenerateAction` | 生成自定义动作（scrapeList/screenshot/scrapeSchema） |
| `removeAction` | 删除动作 | `onRemoveAction` → `handleRemoveAction` | 从工作流中移除指定 actionId 的动作 |
| `save` | 保存录制 | `registerEventHandlers` | 保存工作流到数据库 |
| `new-recording` | 新建录制 | `registerEventHandlers` | 清空工作流记录 |
| `updatePair` | 更新工作流对 | `registerEventHandlers` | 更新指定索引的工作流对 |
| `activeIndex` | 当前活动索引 | `registerEventHandlers` | 设置生成器的活动索引 |

#### 其他

| 消息事件 | 触发场景 | 处理器 | 作用 |
|---------|---------|--------|------|
| `setGetList` | 列表抓取模式切换 | `initializeSocketListeners` | 设置列表抓取模式 |
| `listSelector` | 列表选择器变更 | `initializeSocketListeners` | 设置列表选择器 |
| `setPaginationMode` | 分页模式切换 | `initializeSocketListeners` | 设置分页模式 |
| `testPaginationScroll` | 测试滚动分页 | `onTestPaginationScroll` | 测试滚动是否加载更多内容 |
| `captureDirectScreenshot` | 直接截图 | `registerEditorEvents` | 直接截取当前页面截图 |
| `request-refresh` | 请求刷新 | - | 请求重新发送 DOM 快照 |

### 5.2 后端 → 前端消息

#### DOM 流相关

| 消息事件 | 触发时机 | 发送方 | 作用 |
|---------|---------|--------|------|
| `rrweb-event` | 浏览器页面 DOM 变化 | `RemoteBrowser.initializeRRWebRecording` | 实时 DOM 事件流（rrweb 录制事件） |
| `domLoadingProgress` | DOM 加载进度 | `RemoteBrowser.emitLoadingProgress` | 报告 DOM 加载进度百分比 |
| `dom-snapshot-loading` | 开始加载 | `RemoteBrowser.initialize` | 通知开始加载 DOM 快照 |
| `dom-mode-error` | 浏览器启动失败 | `initializeRemoteBrowserForRecording` | 通知 DOM 模式错误 |

#### 工作流相关

| 消息事件 | 触发时机 | 发送方 | 作用 |
|---------|---------|--------|------|
| `workflow` | 工作流更新 | `Generator.addPairToWorkflowAndNotifyClient` | 推送最新的完整工作流 |
| `fileSaved` | 保存完成 | `Generator.saveNewWorkflow` | 通知保存结果（saved/error/nameExists/retrained） |

#### UI 交互相关

| 消息事件 | 触发时机 | 发送方 | 作用 |
|---------|---------|--------|------|
| `highlighter` | 元素高亮 | `Generator.generateDataForHighlighter` | 推送元素高亮数据 |
| `showDropdown` | 点击下拉框 | `Generator.onClick` | 显示下拉选择器 |
| `showDatePicker` | 点击日期输入框 | `Generator.onClick` | 显示日期选择器 |
| `showTimePicker` | 点击时间输入框 | `Generator.onClick` | 显示时间选择器 |
| `showDateTimePicker` | 点击日期时间输入框 | `Generator.onClick` | 显示日期时间选择器 |
| `decision` | 自定义动作决策 | `Generator.customAction` | 请求用户决策（是否添加选择器） |

#### 浏览器状态相关

| 消息事件 | 触发时机 | 发送方 | 作用 |
|---------|---------|--------|------|
| `urlChanged` | 页面 URL 变化 | `RemoteBrowser.setupPageEventListeners` | 通知 URL 变更 |
| `newTab` | 新标签页打开 | `Generator.notifyOnNewTab` | 通知新标签页 |
| `tabHasBeenClosed` | 标签页关闭 | `Generator.notifyOnNewTab` | 通知标签页关闭 |
| `loaded` | 浏览器加载完成 | `initializeRemoteBrowserForRecording` | 通知浏览器初始化完成 |
| `recording-timeout` | 录制超时 | `controller` | 通知录制会话超时 |

#### 截图相关

| 消息事件 | 触发时机 | 发送方 | 作用 |
|---------|---------|--------|------|
| `screenshotCaptureStarted` | 截图开始 | `RemoteBrowser.captureDirectScreenshot` | 通知截图开始 |
| `directScreenshotCaptured` | 截图完成 | `RemoteBrowser.captureDirectScreenshot` | 返回截图数据 |
| `screenshotError` | 截图失败 | `RemoteBrowser.captureDirectScreenshot` | 通知截图错误 |

## 六、核心数据流详解

### 6.1 DOM 实时流（rrweb）

**流向**：浏览器页面 → 后端 → 前端 replayer

```
Chromium 页面
    │
    │  rrweb.record() 录制 DOM 事件
    ▼
window.emitEventToBackend(event)  [页面内 JS]
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
rrweb Replayer 渲染 DOM
```

**关键代码**：
- 后端注入 rrweb：[RemoteBrowser.ts#L321-L418](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L321-L418)
- 后端暴露函数：[RemoteBrowser.ts#L356-L364](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L356-L364)
- 前端接收渲染：[DOMBrowserRenderer.tsx#L808-L889](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/src/components/recorder/DOMBrowserRenderer.tsx#L808-L889)

### 6.2 点击操作流

**流向**：前端 iframe 点击 → Socket 消息 → 后端 Playwright 点击 → 生成工作流 → 通知前端

```
前端 iframe
    │
    │  mousedown 事件
    │  生成 selector
    ▼
socket.emit('dom:click', { selector, elementInfo, coordinates, ... })
    │
    ▼
后端 inputHandlers.ts
    │
    │  handleWrapper → handleClickAction
    │
    ├─► page.click(selector)  // Playwright 执行点击
    │
    └─► generator.onDOMClickAction(page, data)
           │
           ▼
        addPairToWorkflowAndNotifyClient(pair, page)
           │
           ▼
        socket.emit('workflow', workflowRecord)
           │
           ▼
        前端更新工作流显示
```

**关键代码**：
- 前端点击处理：[DOMBrowserRenderer.tsx#L367-L588](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/src/components/recorder/DOMBrowserRenderer.tsx#L367-L588)
- 后端点击处理：[inputHandlers.ts#L445-L585](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/browser-management/inputHandlers.ts#L445-L585)
- 工作流生成：[Generator.ts#L426-L458](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/workflow-management/classes/Generator.ts#L426-L458)

### 6.3 自定义动作流（抓取/截图）

**流向**：前端按钮点击 → 本地状态更新 → Socket 发送动作 → 后端生成工作流 → 通知前端

以列表抓取（scrapeList）为例：

```
前端 RightSidePanel
    │
    │  用户点击"抓取列表"按钮
    ▼
browserSteps.addListStep()  // 本地状态更新
    │
    ▼
browserSteps.emitActionForStep(step)
    │
    ▼
socket.emit('action', { action: 'scrapeList', actionId, settings })
    │
    ▼
后端 inputHandlers.ts
    │
    │  onGenerateAction → handleGenerateAction
    ▼
generator.customAction(action, actionId, settings, page)
    │
    ▼
addPairToWorkflowAndNotifyClient(pair, page)
    │
    ▼
socket.emit('workflow', workflowRecord)
```

**关键代码**：
- 前端动作发射：[browserSteps.tsx#L152-L226](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/src/context/browserSteps.tsx#L152-L226)
- 后端动作处理：[inputHandlers.ts#L75-L103](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/browser-management/inputHandlers.ts#L75-L103)
- 生成器自定义动作：[Generator.ts#L743-L839](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/workflow-management/classes/Generator.ts#L743-L839)

### 6.4 键盘输入流

```
前端 iframe keydown 事件
    │
    │  生成 selector
    ▼
socket.emit('dom:keypress', { selector, key, inputType, ... })
    │
    ▼
后端 handleKeyboardAction
    │
    ├─► page.press(selector, key)  // Playwright 执行按键
    │
    └─► generator.onDOMKeyboardAction(page, data)
           │
           ▼
        生成 press 动作，加入工作流
```

**注意**：键盘输入有特殊的优化机制 —— [Generator.optimizeWorkflow](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/workflow-management/classes/Generator.ts#L1417-L1518)：
- 录制时每个按键都是独立的 `press` 动作
- 保存时会优化：将连续的按键合并为单个 `type` 动作
- 支持 Backspace、Delete 等特殊键的状态追踪

## 七、状态管理与同步

### 7.1 后端状态

**RemoteBrowser 类** 维护浏览器会话状态：
- `browser` / `context` / `currentPage`：Playwright 实例
- `generator`：工作流生成器（维护 `workflowRecord`）
- `interpreter`：工作流解释器
- `isDOMStreamingActive`：DOM 流是否激活
- `isRecordingMode`：是否为录制模式

**BrowserPool 类** 管理所有浏览器实例：
- 按用户 ID 分组
- 限制每个用户的浏览器数量
- 跟踪浏览器状态（recording / run）

### 7.2 前端状态

前端使用多个 Context 管理状态：

| Context | 状态内容 | 相关文件 |
|--------|---------|---------|
| `SocketStore` | Socket 连接、browserId、队列 Socket | [socket.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/src/context/socket.tsx) |
| `BrowserStepsStore` | 抓取步骤列表（文本/列表/截图） | [browserSteps.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/src/context/browserSteps.tsx) |
| `GlobalInfoStore` | 全局信息（录制 URL、名称、模式等） | [globalInfo.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/src/context/globalInfo.tsx) |
| `ActionContext` | 动作模式（抓取文本/列表/截图） | [browserActions.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/src/context/browserActions.tsx) |

### 7.3 工作流同步机制

工作流是录制状态的核心，采用 **后端为单一数据源** 的模式：

1. 所有工作流变更都在后端 `Generator` 中发生
2. 每次变更后通过 `socket.emit('workflow', workflowRecord)` 推送给前端
3. 前端接收后更新本地状态和 UI

**关键代码**：
- 后端推送：[Generator.ts#L342](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/workflow-management/classes/Generator.ts#L342)
  ```typescript
  this.socket.emit('workflow', this.workflowRecord);
  ```

## 八、关键设计模式

### 8.1 包装器模式（Wrapper Pattern）

输入处理器使用双层包装：
- 外层 `onXxx`：socket 事件回调，日志记录
- 中层 `handleWrapper`：获取活跃浏览器，检查解释器状态
- 内层 `handleXxx`：实际的业务逻辑

**代码示例**：[inputHandlers.ts#L28-L57](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/browser-management/inputHandlers.ts#L28-L57)

### 8.2 广播模式（Broadcast Pattern）

使用 `socket.nsp.emit()` 向命名空间内所有客户端广播，确保重连后新的 Socket 也能收到消息。

**代码**：[RemoteBrowser.ts#L206-L216](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L206-L216)

### 8.3 引用计数 Socket 缓存

前端 `browserSocket.ts` 使用引用计数管理 Socket 连接，多个组件共享同一连接。

**代码**：[browserSocket.ts#L4-L45](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/src/utils/browserSocket.ts#L4-L45)

### 8.4 乐观 UI 更新

前端执行操作时先乐观更新本地状态（如标签页、步骤列表），再通过 Socket 同步后端，提升用户体验。

## 九、错误与异常处理

### 9.1 浏览器启动失败
- 捕获初始化异常
- 发送 `dom-mode-error` 事件通知前端
- 清理浏览器会话资源

### 9.2 页面关闭检测
- 每个动作执行前检查 `page.isClosed()`
- 避免在已关闭页面上执行操作

### 9.3 解释运行中忽略输入
- `handleWrapper` 检查 `interpreter.interpretationInProgress()`
- 解释运行期间忽略用户输入，防止状态混乱

### 9.4 Socket 重连
- `RemoteBrowser.updateSocket()` 支持更新 Socket 实例
- 重连后重新注册所有事件监听器
- 通过命名空间广播确保消息可达

## 十、性能优化点

1. **rrweb 采样配置**：[RemoteBrowser.ts#L388-L396](file:///d:/fz/0601-2/solo-dogfeeding/code/108-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L388-L396)
   - mousemove: false（禁用鼠标移动采样）
   - scroll: 75ms（滚动采样间隔）
   - input: 'last'（输入只保留最后一个）

2. **工作流优化**：保存时合并连续按键操作为 type 动作

3. **前端节流**：鼠标移动、滚动事件都有节流处理

4. **机器人运行模式**：非录制模式跳过 rrweb 注入，提升性能
