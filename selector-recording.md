# 录制阶段选择器生成机制分析

## 一、核心概念：为什么容易混淆？

录制阶段选择器的抽象之所以绕，是因为项目中存在以下几组容易混淆的概念：

| 混淆点 | 说明 |
|--------|------|
| **两套选择器生成器** | 前端 `clientSelectorGenerator`（浏览器 iframe 内运行） vs 服务端 `selector.ts`（Playwright 控制的浏览器中运行） |
| **两种录制模式** | DOM 模式（当前主流，基于 rrweb 事件流） vs Screenshot 模式（已弃用，基于 Canvas 截图） |
| **三个 Socket 事件注册源** | `inputHandlers.ts`（用户交互事件）、`Generator.ts`（工作流元控制）、`RemoteBrowser.ts`（滚动/截图/标签页） |
| **两类 dom:* 事件** | 录制事件（`dom:click`、`dom:keypress`：含 selector） vs 纯同步事件（`dom:scroll`：只同步 Playwright 浏览器状态，**不进入 workflow**） |

> **关键理解**：`dom:click` / `dom:keypress` 携带 **前端生成好的 selector**，服务端直接消费；`dom:scroll` 只含 `{deltaX, deltaY}`，用于同步浏览器位置，**不产生 workflow 条目**。

---

## 二、三种 Socket 事件注册源

Socket 事件并不是集中在一个地方注册的，而是分散在 **三个独立的模块**中。这是之前理解混乱的根源。

### 2.1 注册源 1：inputHandlers.ts —— 用户交互事件

[inputHandlers.ts](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/inputHandlers.ts#L871-L912) `registerInputHandlers()` 注册所有**用户操作**相关的事件：

| Socket 事件 | 注册代码位置 | 处理函数 | 是否进入 workflow |
|------------|------------|---------|-----------------|
| `input:keyup` | L872 | `onKeyup` | — |
| `input:url` | L873 | `onChangeUrl` | ✓（生成 goto action） |
| `input:refresh` | L874 | `onRefresh` | ✓ |
| `input:back` | L875 | `onGoBack` | ✓ |
| `input:forward` | L876 | `onGoForward` | ✓ |
| `input:date` | L877 | `onDateSelection` | ✓（生成 type + Enter） |
| `input:dropdown` | L878 | `onDropdownSelection` | ✓（生成 selectOption） |
| `input:time` | L879 | `onTimeSelection` | ✓ |
| `input:datetime-local` | L880 | `onDateTimeLocalSelection` | ✓ |
| `action` | L881 | `onGenerateAction` | ✓ |
| `removeAction` | L882 | `onRemoveAction` | — |
| **`dom:click`** | **L884** | **`onDOMClickAction` → `generator.onDOMClickAction`** | **✓** |
| **`dom:keypress`** | **L885** | **`onDOMKeyboardAction` → `generator.onDOMKeyboardAction`** | **✓** |
| `testPaginationScroll` | L886 | `onTestPaginationScroll` | —（测试用） |

注册入口在 [connection.ts](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/socket-connection/connection.ts#L17-L25)：每次客户端连接时自动调用 `registerInputHandlers(socket, userId)`。

### 2.2 注册源 2：Generator.ts —— 工作流元控制

[Generator.ts](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L191-L257) 注册**工作流控制**相关的事件，包括模式切换、列表抓取配置、保存等：

| Socket 事件 | 注册位置 | 说明 |
|------------|---------|------|
| `setGetList` | L192 | 设置列表抓取模式 |
| `listSelector` | L195 | 指定列表容器选择器 |
| `setPaginationMode` | L198 | 开启分页检测模式 |
| `dom-mode-enabled` | L204 | **DOM 模式开启**（`isDOMMode = true`） |
| `screenshot-mode-enabled` | L209 | **Screenshot 模式开启**（`isDOMMode = false`，目前未使用） |
| `save` | L221 | 保存工作流到数据库 |
| `new-recording` | L226 | 重置工作流为空数组 |
| `activeIndex` | L231 | 更新当前选中步骤的索引 |
| `decision` | L232 | 用户决策（如取消 over-shadowing 合并） |
| `updatePair` | L254 | 用户手动编辑某个 pair |

### 2.3 注册源 3：RemoteBrowser.ts —— 浏览器状态同步

[RemoteBrowser.ts](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/classes/RemoteBrowser.ts) 注册**纯浏览器同步**事件，这些事件**不经过 Generator，也不产生 workflow 条目**。

注册分为两个阶段：

**阶段一：初始化时注册（滚动监听）**

| Socket 事件 | 注册位置 | 处理逻辑 | 是否进入 workflow |
|------------|---------|---------|-----------------|
| **`dom:scroll`** | **L228-L234** | **`page.mouse.wheel(deltaX, deltaY)`** 同步滚动位置 | **✗（只同步浏览器状态）** |

**阶段二：编辑器模式注册（`registerEditorEvents`，L694-L729）**

| Socket 事件 | 注册位置 | 处理逻辑 | 是否进入 workflow |
|------------|---------|---------|-----------------|
| `captureDirectScreenshot` | L699 | 执行直接截图并返回结果 | ✗ |
| `changeTab` | L703-L706 | 切换标签页 | ✗ |
| `addTab` | L708-L714 | 新建标签页并切换 | ✗ |
| `closeTab` | L716-L728 | 关闭指定标签页 | ✗ |

> **关键区别**：注册源 3 的事件全部是**浏览器状态控制**，只操作 Playwright 浏览器本身，不涉及工作流生成。与注册源 1（用户交互→工作流）和注册源 2（工作流元控制）形成互补。

---

## 三、dom:scroll：只同步状态，不进入工作流

这是最容易被误判为"录制事件"的事件。让我们完整梳理其链路。

### 3.1 发送端：前端 DOMBrowserRenderer.tsx wheelHandler

[DOMBrowserRenderer.tsx](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/src/components/recorder/DOMBrowserRenderer.tsx#L674-L728)

```
wheel 事件触发
      ↓
① 在 iframe 内本地滚动（不依赖服务端）:
   findScrollableAncestor(target) → scrollBy(deltaX, deltaY)
      ↓
② 节流（50ms）+ 累积增量
   pendingScrollDelta 累积多次 wheel
      ↓
③ Socket 发送（注意 payload 内容）:
   socket.emit("dom:scroll", {
     deltaX: accX,   // 累积增量 X
     deltaY: accY    // 累积增量 Y
   })
   // ⚠ 这里没有 selector，没有坐标，只有滚动增量
```

### 3.2 接收端：服务端 RemoteBrowser.setupScrollEventListener

[RemoteBrowser.ts](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L221-L235)

```typescript
this.socket.on(
  "dom:scroll",
  (data: { deltaX: number; deltaY: number }) => {
    if (!this.isDOMStreamingActive || !this.currentPage) return;
    // 只同步 Playwright 真实浏览器的滚动位置
    // 让 Playwright 浏览器的视口与前端 iframe 保持一致
    this.currentPage.mouse.wheel(data.deltaX, data.deltaY).catch(() => {});
  }
);
```

**关键结论**：`dom:scroll` **不经过 Generator，不产生任何 WhereWhatPair，不写入 workflow**。它的唯一作用是让 Playwright 控制的后端浏览器（后续选择器生成、截图的宿主）与前端 iframe 的滚动位置保持一致，确保 `elementsFromPoint(x, y)` 的坐标映射正确。

> 那滚动动作如何被录制？答：如果滚动导致页面 URL hash 变化 → 由 `input:url` 捕获；否则滚动**不作为独立动作录制**，回放时通过 `networkidle` 等等待机制自然处理。

---

## 四、DOM 模式 vs Screenshot 模式：历史沿革与现状

### 4.1 两种模式的来源

| 模式 | 代码位置 | 实现方式 | 现状 |
|------|---------|---------|------|
| **DOM 模式** | [DOMBrowserRenderer.tsx](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/src/components/recorder/DOMBrowserRenderer.tsx) | iframe + rrweb 事件流 + 前端直接监听 DOM 事件 | **当前默认启用** |
| **Screenshot 模式** | [legacy/src/Canvas.tsx](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/legacy/src/Canvas.tsx) | Canvas 渲染截图 + 坐标映射 + Playwright 生成选择器 | **已弃用，代码移至 legacy/** |

### 4.2 Screenshot 模式的工作原理（仅供参考，已不再使用）

**触发顺序**：
```
用户点击 Canvas (legacy/Canvas.tsx L140-L195)
      ↓
coordinateMapper.mapCanvasToBrowser() 换算坐标
      ↓
socket.emit('input:mousedown', browserCoordinates)
      ↓
（服务端注册缺失？→ 搜索全项目无 input:mousedown 监听）
```

**遗留的服务端接口**：`Generator.ts` 中仍然保留了对应的 [onClick(coordinates, page)](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L511) 方法，其核心逻辑：

```typescript
// Generator.ts L511 —— 仅在截图模式下理论可调用
public onClick = async (coordinates: Coordinates, page: Page) => {
  // ① 服务端通过坐标生成选择器
  const selector = await this.generateSelector(page, coordinates, ActionType.Click);
  // ② 获取元素信息（下拉/日期等特殊输入）
  const elementInfo = await getElementInformation(page, coordinates, '', false);
  // ③ 特殊输入 → 弹窗返回给前端
  if (isDropdown) { socket.emit('showDropdown', { coordinates, selector, options }); return; }
  if (isDateInput) { socket.emit('showDatePicker', { coordinates, selector }); return; }
  // ④ 输入框 → 计算光标位置，生成 click pair
  // ⑤ 普通元素 → 生成 click pair
  ...
};
```

**⚠ 现状核实**：
- 全项目搜索 `input:mousedown`、`input:keydown`、`input:wheel`：**仅存在于 legacy/Canvas.tsx**，服务端无监听
- `generator.onClick()` 方法：**无任何调用者**（全文搜索仅找到定义处，代码引用处仅文档）
- `screenshot-mode-enabled` 事件：前端无发送，仅服务端留有监听 stub
- `isDOMMode` 初始值：服务端 `false`，但 [BrowserWindow.tsx L186-L194](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/src/components/browser/BrowserWindow.tsx#L186-L194) 中前端收到 `dom-mode-enabled` 后立刻回发确认，实际运行中默认启用 DOM 模式
- [BrowserWindow.tsx L2002-L2135](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/src/components/browser/BrowserWindow.tsx#L2002-L2135)：`isDOMMode ? <DOMBrowserRenderer /> : <DOMLoadingIndicator />`，非 DOM 模式只显示加载指示器

**结论**：Screenshot 模式是**历史遗留的架构设计**，当前版本中：
- 前端已无 Screenshot 模式的 UI 入口
- 服务端的 `generator.onClick`、`onHover`、`onInput`、`onKeyUp` 等方法均无调用者
- `selector.ts` 中的服务端选择器生成函数，仅在 DOM 模式下的 `generateSelector` 内部间接使用（如需要服务端二次验证时），但主链路不经过它们

### 4.3 为什么还有两套选择器生成器？

| 生成器 | 实际使用场景 |
|--------|------------|
| **clientSelectorGenerator**（前端） | DOM 模式录制主链路，`mouseDownHandler` 中直接调用，生成 selector 后通过 `dom:click` 发送给服务端 |
| **selector.ts**（服务端） | ① SDK/API 模式下的工作流增强；② 某些列表检测场景下服务端二次验证；③ 历史遗留代码保留（Screenshot 模式） |

---

## 五、Selector 进入工作流的完整链路（DOM 模式）

### 5.1 总览图

```
前端浏览器（React 主窗口）
│
└── <iframe id="dom-browser-iframe">
      │
      ├── rrweb 实时回放（服务端 → 前端 rrweb-event）
      │   作用：让 iframe 显示和 Playwright 浏览器一样的页面
      │
      └── DOMBrowserRenderer 事件捕获
          │
          ├─ mousedown → mouseDownHandler
          │     │
          │     ├─ ① clientSelectorGenerator.generateSelector()
          │     │     运行环境：iframe.contentWindow
          │     │     算法：@medv/finder（内嵌实现）
          │     │     输入：{x, y} 坐标
          │     │     输出：string（最优选择器）
          │     │
          │     ├─ ② clientSelectorGenerator.getElementInformation()
          │     │     输出：ElementInfo（标签名、属性、文本等）
          │     │
          │     └─ ③ socket.emit("dom:click", {
          │              selector,        ← 核心！已生成好
          │              userId,
          │              elementInfo,    ← 元素详情
          │              coordinates,    ← 相对元素坐标
          │              isSPA
          │           })
          │
          ├─ keydown → keyDownHandler
          │     └─ socket.emit("dom:keypress", {
          │            selector,        ← 焦点元素的选择器
          │            key,
          │            inputType
          │        })
          │
          └─ wheel → wheelHandler
                └─ socket.emit("dom:scroll", { deltaX, deltaY })
                    （不含 selector，只同步浏览器，不进入 workflow）
```

```
服务端 (Node.js + Playwright)
│
├── Socket 层
│   │
│   ├── dom:click → onDOMClickAction（inputHandlers.ts）
│   │     │
│   │     ├─ ① 预处理：移除 a[target="_blank"] 防跳转
│   │     │
│   │     ├─ ② Playwright 执行真实点击（让浏览器状态同步）
│   │     │     ├─ input/textarea 且有坐标 → page.mouse.click(坐标)
│   │     │     └─ 普通元素 → page.click(selector)
│   │     │
│   │     └─ ③ generator.onDOMClickAction(page, data)
│   │           │
│   │           ├─ 直接消费 data.selector（不重新生成！）
│   │           │
│   │           └─ 构建 WhereWhatPair：
│   │                pair.where.url = page.url()
│   │                pair.where.selectors = [selector]   ← selector 进入 where
│   │                pair.what = [{
│   │                    action: 'click',
│   │                    args: [selector]                ← selector 进入 what
│   │                }]
│   │
│   ├── dom:keypress → onDOMKeyboardAction（inputHandlers.ts）
│   │     └─ generator.onDOMKeyboardAction(...)
│   │           └─ 构建 WhereWhatPair：
│   │                action: 'press'
│   │                args: [selector, key]
│   │              （后续 optimizeWorkflow 会将连续 press 合并为 type）
│   │
│   └── dom:scroll → RemoteBrowser.setupScrollEventListener
│         └─ page.mouse.wheel(deltaX, deltaY)
│             （仅同步浏览器滚动位置，不经过 Generator）
│
└── WorkflowGenerator 层
    │
    └── addPairToWorkflowAndNotifyClient(pair, page)
          │
          ├─ ① selectorAlreadyInWorkflow()
          │    同 where 选择器的 pair 已存在 → 追加 what 动作
          │
          ├─ ② handleOverShadowing()
          │    多规则同时可见 → 合并 to the same where
          │
          ├─ ③ 追加 waitForLoadState('networkidle') 到 what
          │
          ├─ ④ 写入 this.workflowRecord.workflow
          │
          └─ ⑤ socket.emit('workflow', this.workflowRecord)
               → 前端收到后更新 UI 面板显示
```

### 5.2 前端生成选择器：mouseDownHandler 详细流程

[DOMBrowserRenderer.tsx](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/src/components/recorder/DOMBrowserRenderer.tsx)

```
用户在 iframe 内 mousedown
      │
      ├─ [1] 坐标归一化
      │    相对于 iframe.contentDocument 的 viewport 坐标
      │
      ├─ [2] 模式分流
      │    ├─ getText = true → 触发 onElementSelect（文本抓取）
      │    ├─ getList = true → 触发 onElementSelect（列表抓取）
      │    └─ 普通录制模式 → 继续
      │
      ├─ [3] 链接特殊处理
      │    target.closest("a[href]") → 阻止默认跳转，标记 isSPA
      │
      ├─ [4] 生成选择器 ⭐
      │    selector = clientSelectorGenerator.generateSelector(
      │        iframeDoc,
      │        { x: iframeX, y: iframeY },
      │        ActionType.Click
      │    )
      │    // 内部：坐标 → 元素 → @medv/finder → 最优选择器
      │
      ├─ [5] 获取元素详情
      │    elementInfo = clientSelectorGenerator.getElementInformation(...)
      │    // 包含 tagName、attributes、innerHTML、textContent 等
      │
      ├─ [6] 特殊输入分流（弹 UI 不发 click）
      │    ├─ SELECT 标签 → onShowDropdown（前端弹出选择器）
      │    │                 选择后发 input:dropdown（含 value，不含 options）
      │    ├─ date input → onShowDatePicker
      │    ├─ time input → onShowTimePicker
      │    ├─ datetime-local → onShowDateTimePicker
      │    └─ 普通元素 → 继续
      │
      └─ [7] Socket 发送 dom:click ⭐⭐
           socket.emit("dom:click", {
             selector,        ← 前端已生成
             userId,
             elementInfo,     ← 供服务端判断类型
             coordinates,     ← 相对元素的坐标（input 用）
             isSPA            ← 是否 SPA 链接
           })
```

### 5.3 服务端消费选择器：onDOMClickAction 详细流程

**第一层：inputHandlers.ts handleClickAction**

[inputHandlers.ts](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/inputHandlers.ts)

```
onDOMClickAction(data, userId)
      │
      └─ handleWrapper(handleClickAction, userId, data)
            │
            ├─ [1] 活跃浏览器 + 非回放状态检查
            │
            ├─ [2] 移除 a[target="_blank"]（避免新标签页，Playwright 难追踪）
            │    page.evaluate(selector) → querySelector → 改 target
            │
            ├─ [3] Playwright 执行真实点击
            │    ├─ data.coordinates 存在（input）→ page.mouse.click(x, y)
            │    └─ 否则 → page.click(data.selector)
            │
            └─ [4] generator.onDOMClickAction(page, data)
```

**第二层：Generator.ts onDOMClickAction**

[Generator.ts](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L426-L458)

```typescript
// ⭐ 重点：此处不重新生成选择器！直接消费 data.selector
public onDOMClickAction = async (page, data) => {
  const { selector, url, elementInfo, coordinates } = data;

  // 构建 WhereWhatPair
  const pair: WhereWhatPair = {
    where: {
      url: this.getBestUrl(url ?? page.url()),
      selectors: [selector]     // ← 直接放入 where
    },
    what: [{
      action: 'click',
      args: [selector]          // ← 直接放入 what.args[0]
    }],
  };

  // input/textarea 附加坐标和 cursorIndex
  if (isInputElement(elementInfo) && coordinates) {
    pair.what[0].args.push(
      { position: coordinates },  // args[1]
      { cursorIndex: 0 }          // args[2]
    );
  }

  // 记录状态（用于后续 over-shadowing 判断）
  this.generatedData.lastUsedSelector = selector;
  this.generatedData.lastAction = 'click';

  // 加入工作流（含合并逻辑）
  await this.addPairToWorkflowAndNotifyClient(pair, page);
};
```

### 5.4 addPairToWorkflowAndNotifyClient：选择器的最终归宿

[Generator.ts](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L295-L344)

```
addPairToWorkflowAndNotifyClient(pair, page)
      │
      ├─ [1] 相同 where.selector 的 Pair 已存在？
      │    → selectorAlreadyInWorkflow(selector, workflow)
      │    → 存在：matched.what = matched.what.concat(pair.what)
      │      （同页面多步操作合并为 1 个 Pair）
      │
      ├─ [2] over-shadowing 检测
      │    → isRuleOvershadowing(pair, workflow, page)
      │    → 多个选择器在当前页面同时可见：合并到同一个 where
      │
      ├─ [3] 追加 waitForLoadState('networkidle')
      │    pair.what.push({ action: 'waitForLoadState', args: ['networkidle'] })
      │
      ├─ [4] 写入 this.workflowRecord.workflow（按 lastIndex 插入）
      │
      └─ [5] socket.emit('workflow', this.workflowRecord)
           → 前端 RightSidePanel 收到后刷新 workflow 列表
```

---

## 六、完整 Socket 事件清单（全项目汇总）

### 6.1 分类总览

| 类别 | 说明 | 进入 workflow |
|------|------|-------------|
| **录制交互事件** | `dom:click`、`dom:keypress`、`input:url` 等用户操作 | ✓（核心） |
| **浏览器同步事件** | `dom:scroll`、`changeTab`、`captureDirectScreenshot` 等纯浏览器控制 | ✗ |
| **浏览器状态通知** | `rrweb-event`、`urlChanged`、`domLoadingProgress` 等状态推送 | ✗ |
| **工作流元控制** | `save`、`new-recording`、`decision` 等工作流管理 | ✗（间接影响） |
| **列表抓取配置** | `setGetList`、`listSelector`、`setPaginationMode` | ✗ |

### 6.2 前端 → 服务端（录制交互类）

| 事件名 | 发送位置 | 载荷 | 服务端处理 | 进入 workflow |
|--------|---------|------|-----------|-------------|
| `dom:click` | [DOMBrowserRenderer.tsx L569](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/src/components/recorder/DOMBrowserRenderer.tsx#L569-L583) | `{ selector, userId, elementInfo, coordinates?, isSPA? }` | inputHandlers → Generator.onDOMClickAction | ✓ |
| `dom:keypress` | [DOMBrowserRenderer.tsx L642](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/src/components/recorder/DOMBrowserRenderer.tsx#L642-L647) | `{ selector, key, userId, inputType? }` | inputHandlers → Generator.onDOMKeyboardAction | ✓ |
| `input:url` | URL 表单或导航 | `string` (url) | inputHandlers → onChangeUrl | ✓ |
| `input:refresh` | 浏览器刷新按钮 | 无 | inputHandlers → onRefresh | ✓ |
| `input:back` | 后退按钮 | 无 | inputHandlers → onGoBack | ✓ |
| `input:forward` | 前进按钮 | 无 | inputHandlers → onGoForward | ✓ |
| `input:date` | 日期选择器确认 | `{ selector, value }` | inputHandlers → onDateSelection | ✓ |
| `input:time` | 时间选择器确认 | `{ selector, value }` | inputHandlers → onTimeSelection | ✓ |
| `input:datetime-local` | 日期时间选择器 | `{ selector, value }` | inputHandlers → onDateTimeLocalSelection | ✓ |
| `input:dropdown` | 下拉选择器确认 | `{ selector, value }` | inputHandlers → onDropdownSelection | ✓ |
| `input:keyup` | 按键释放 | `key: string` | inputHandlers → onKeyup | ✗ |
| `action` | 手动添加动作 | `{ action, url, selectors, args }` | inputHandlers → onGenerateAction | ✓ |
| `removeAction` | 手动删除动作 | `{ pairIndex, actionIndex }` | inputHandlers → onRemoveAction | ✗（修改） |
| `testPaginationScroll` | 分页测试 | 配置参数 | inputHandlers → onTestPaginationScroll | ✗ |

### 6.3 前端 → 服务端（浏览器同步类）

> 这类事件**不经过 Generator**，直接由 `RemoteBrowser` 处理，只操作 Playwright 浏览器本身。

| 事件名 | 发送方 | 载荷 | 服务端注册位置 | 处理逻辑 | 进入 workflow |
|--------|-------|------|--------------|---------|-------------|
| `dom:scroll` | DOMBrowserRenderer wheelHandler | `{ deltaX, deltaY }` | [RemoteBrowser.ts L228-L234](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L228-L234) | `page.mouse.wheel()` 同步滚动 | ✗ |
| `captureDirectScreenshot` | 前端截图按钮 | settings 对象 | [RemoteBrowser.ts L699-L701](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L699-L701) | 执行截图并回发结果 | ✗ |
| `changeTab` | 前端标签页切换 | tabIndex: number | [RemoteBrowser.ts L703-L706](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L703-L706) | 切换当前 page | ✗ |
| `addTab` | 前端新建标签页 | 无 | [RemoteBrowser.ts L708-L714](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L708-L714) | `context.newPage()` + 切换 | ✗ |
| `closeTab` | 前端关闭标签页 | `{ index, isCurrent }` | [RemoteBrowser.ts L716-L728](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L716-L728) | 关闭 page + 切换当前页 | ✗ |

### 6.4 服务端 → 前端（浏览器状态通知类）

| 事件名 | 发送位置 | 载荷 | 说明 |
|--------|---------|------|------|
| `rrweb-event` | [RemoteBrowser.ts L357](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L357) | rrweb 事件对象 | DOM 模式实时页面渲染（核心数据通道） |
| `domLoadingProgress` | [RemoteBrowser.ts L237-L244](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L237-L244) | `{ progress, pendingRequests, userId }` | 页面加载进度（0-100） |
| `dom-snapshot-loading` | [RemoteBrowser.ts L466](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L466) | 加载状态 | DOM 快照加载中通知 |
| `urlChanged` | [RemoteBrowser.ts L265](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L265) + [L921](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L921) | `{ url, userId }` | 页面 URL 变化通知（导航/切标签页均触发） |
| `dom-mode-enabled` | controller 初始化成功后 | `{ userId }` | 通知前端 DOM 模式已就绪 |
| `dom-mode-error` | [controller.ts L50](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/controller.ts#L50) | `{ userId, error }` | 通知前端 DOM 模式初始化失败 |
| `screenshotCaptureStarted` | [RemoteBrowser.ts L637](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L637) | 状态信息 | 截图开始通知 |
| `directScreenshotCaptured` | [RemoteBrowser.ts L655](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L655) | 截图数据 + metadata | 直接截图完成（返回 base64） |
| `screenshotError` | [RemoteBrowser.ts L629](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L629) + [L664](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L664) | 错误信息 | 截图失败通知 |

### 6.5 服务端 → 前端（工作流与 UI 类）

| 事件名 | 发送位置 | 载荷 | 说明 |
|--------|---------|------|------|
| `workflow` | [Generator.ts L342](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L342) | `WorkflowFile` | 工作流更新通知（每次 pair 变化都发） |
| `highlighter` | [Generator.ts L1205-L1210](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L1205-L1210) | `{ rect, selector, elementInfo, isDOMMode, shadowInfo }` | 元素悬停高亮数据 |
| `showDropdown` | [Generator.ts L546](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L546) | `{ coordinates, selector, options }` | 显示下拉选择器弹窗 |
| `showDatePicker` | [Generator.ts L557](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L557) | `{ coordinates, selector }` | 显示日期选择器弹窗 |
| `showTimePicker` | [Generator.ts L567](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L567) | `{ coordinates, selector }` | 显示时间选择器弹窗 |
| `showDateTimePicker` | [Generator.ts L577](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L577) | `{ coordinates, selector }` | 显示日期时间选择器弹窗 |
| `decision` | [Generator.ts L824](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L824) | `{ pair, actionType, ... }` | 需要用户决策（如 over-shadowing 确认） |
| `fileSaved` | [Generator.ts L1077](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L1077) + L1087 + L1120 + L1127 | `{ actionType }` | 工作流保存结果通知 |
| `newTab` | [Generator.ts L1241](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L1241) | 页面标题 / 'new tab' | 新标签页通知 |
| `tabHasBeenClosed` | [Generator.ts L1237](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L1237) | pageIndex | 标签页已关闭通知 |
| `paginationScrollTestResult` | [inputHandlers.ts L724](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/inputHandlers.ts#L724) | 测试结果 | 分页滚动测试结果 |

### 6.6 工作流元控制（前端 → 服务端）

| 事件名 | 注册位置 | 说明 | 进入 workflow |
|--------|---------|------|-------------|
| `setGetList` | [Generator.ts L192](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L192) | 设置列表抓取模式 | ✗ |
| `listSelector` | [Generator.ts L195](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L195) | 指定列表容器选择器 | ✗ |
| `setPaginationMode` | [Generator.ts L198](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L198) | 开启分页检测模式 | ✗ |
| `dom-mode-enabled` | [Generator.ts L204](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L204) | DOM 模式确认（前端回发 ACK） | ✗ |
| `screenshot-mode-enabled` | [Generator.ts L209](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L209) | Screenshot 模式切换（当前未使用） | ✗ |
| `save` | [Generator.ts L221](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L221) | 保存工作流到数据库 | ✗（持久化） |
| `new-recording` | [Generator.ts L226](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L226) | 重置工作流为空 | ✗ |
| `activeIndex` | [Generator.ts L231](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L231) | 更新当前选中步骤索引 | ✗ |
| `decision` | [Generator.ts L232](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L232) | 用户决策反馈 | ✗ |
| `updatePair` | [Generator.ts L254](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts#L254) | 用户手动编辑 pair | ✗ |

---

## 七、前后端职责总览

| 职责 | 前端 | 服务端 |
|------|------|--------|
| DOM 事件捕获（mousedown/keydown/wheel） | ✓（iframe 原生事件监听） | ✗ |
| 选择器生成（录制主链路） | ✓（clientSelectorGenerator，运行在 iframe） | ✗（直接消费前端结果） |
| 选择器生成（SDK/历史遗留） | ✗ | ✓（selector.ts，Playwright 内执行） |
| 元素信息采集（tagName/attributes/文本） | ✓ | ✗（直接消费前端结果） |
| 特殊输入弹窗（日期/下拉）触发 | ✓（检测后弹本地 UI） | ✓（onClick 中也有 emit，截图模式用） |
| 滚动本地同步（iframe 内） | ✓（wheelHandler 中 scrollBy） | ✓（Playwright 内 mouse.wheel） |
| 滚动录制为 workflow 动作 | ✗（不单独录制） | ✗（不单独录制） |
| Socket 事件发送（dom:click 等） | ✓ | ✓（workflow/highlighter 等回发） |
| Playwright 浏览器真实操作执行 | ✗ | ✓（page.click / page.type 等） |
| WhereWhatPair 构建 | ✗ | ✓（WorkflowGenerator） |
| Workflow 合并优化（合并连续按键） | ✗ | ✓（optimizeWorkflow） |
| Workflow 持久化存储 | ✗ | ✓（保存到数据库） |
| rrweb 页面渲染 | ✓（Replayer 消费 rrweb-event） | ✓（录制并发送 rrweb-event） |

---

## 八、关键文件索引

| 文件 | 端 | 核心职责 |
|------|----|---------|
| [src/components/recorder/DOMBrowserRenderer.tsx](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/src/components/recorder/DOMBrowserRenderer.tsx) | 前端 | DOM 模式录制组件：iframe 事件监听 + **选择器生成** + Socket 发送 |
| [src/components/browser/BrowserWindow.tsx](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/src/components/browser/BrowserWindow.tsx) | 前端 | 浏览器窗口容器：DOM 模式切换、全局状态管理 |
| [src/helpers/clientSelectorGenerator.ts](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/src/helpers/clientSelectorGenerator.ts) | 前端 | 客户端选择器生成器：单例类 + @medv/finder 算法 + 分组算法 |
| [server/src/socket-connection/connection.ts](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/socket-connection/connection.ts) | 服务端 | Socket 连接入口：注册 inputHandlers |
| [server/src/browser-management/controller.ts](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/controller.ts) | 服务端 | 浏览器控制器：创建/销毁 RemoteBrowser、DOM 模式初始化 |
| [server/src/browser-management/inputHandlers.ts](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/inputHandlers.ts) | 服务端 | **注册源 1**：用户交互事件路由 + Playwright 动作执行 |
| [server/src/browser-management/classes/RemoteBrowser.ts](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/classes/RemoteBrowser.ts) | 服务端 | **注册源 3**：滚动同步 + rrweb 录制 + 标签页/截图管理 |
| [server/src/workflow-management/classes/Generator.ts](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts) | 服务端 | **注册源 2**：模式切换 + 工作流编排器：事件 → Where-What Pair |
| [server/src/workflow-management/selector.ts](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/selector.ts) | 服务端 | 服务端选择器生成（page.evaluate 注入，SDK/历史用） |
| [server/src/workflow-management/utils.ts](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/utils.ts) | 服务端 | 选择器优先级决策（多策略选优） |
| [legacy/src/Canvas.tsx](file:///D:/fz/0601-2/solo-dogfeeding/code/106-maxun/legacy/src/Canvas.tsx) | 前端（遗留） | 旧 Screenshot 模式：Canvas 截图 + 坐标映射 + input:mousedown 发送 |
