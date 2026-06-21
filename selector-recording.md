# 录制阶段选择器生成机制分析

## 一、核心概念：为什么容易混淆？

录制阶段选择器的抽象之所以绕，是因为项目中存在**两套选择器生成器**和**两种录制模式**，它们在不同的时机、不同的运行环境中被调用，承担不同的职责。

| 维度 | 客户端选择器生成器 | 服务端选择器生成器 |
|------|------------------|------------------|
| 所在文件 | [clientSelectorGenerator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/src/helpers/clientSelectorGenerator.ts) | [selector.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/selector.ts) |
| 运行环境 | 前端浏览器（iframe 内） | 服务端 Playwright 控制的浏览器 |
| 核心算法 | @medv/finder（内嵌实现） | @medv/finder（内嵌实现，代码几乎相同） |
| 实例方式 | 单例 `clientSelectorGenerator` | 函数集合，通过 `page.evaluate()` 注入执行 |
| 主要用途 | DOM 模式下事件捕获 + 高亮显示 + 分组检测 | Screenshot 模式下选择器生成 + workflow 构建 |

> **关键理解**：两套代码逻辑同源（都是 @medv/finder 算法的内嵌实现），但运行在不同的浏览器上下文中，服务于不同的录制阶段。

---

## 二、两种录制模式

### 2.1 DOM 模式 vs Screenshot 模式

录制存在两种交互模式，它们的选择器生成链路完全不同：

| 模式 | 触发条件 | 选择器由谁生成 | 数据流向 |
|------|---------|--------------|---------|
| **DOM 模式** | `dom-mode-enabled` 事件 | 前端 `clientSelectorGenerator` | 前端生成 → socket 传输 → 服务端直接使用 |
| **Screenshot 模式** | `screenshot-mode-enabled` 事件 | 服务端 `selector.ts` | 前端传坐标 → 服务端 Playwright evaluate 重新生成 |

`WorkflowGenerator` 通过 `isDOMMode` 标志位跟踪当前模式：

```typescript
// Generator.ts L159
private isDOMMode: boolean = false;

// L203-L213
private initializeDOMListeners() {
  this.socket.on('dom-mode-enabled', () => {
    this.isDOMMode = true;
  });
  this.socket.on('screenshot-mode-enabled', () => {
    this.isDOMMode = false;
  });
}
```

---

## 三、前端侧：录制组件与客户端选择器生成器

### 3.1 前端录制组件 DOMBrowserRenderer

[DOMBrowserRenderer.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/src/components/recorder/DOMBrowserRenderer.tsx) 是前端录制的核心组件，它在 iframe 中渲染录制页面，并直接监听 DOM 事件。

**核心职责**：
1. 在 iframe 文档上绑定原生事件监听器（mousedown、keydown、wheel 等）
2. 调用 `clientSelectorGenerator` 生成选择器
3. 通过 Socket.io 将事件 + 选择器发送给服务端
4. 处理元素高亮、列表分组可视化等 UI 交互

**事件监听器注册**（setupIframeInteractions）：

```typescript
// L752-L760
handlers.mousedown = mouseDownHandler;     // 点击/选择
handlers.mouseup = mouseUpHandler;
handlers.mousemove = mouseMoveHandler;    // 悬停高亮
handlers.wheel = wheelHandler;            // 滚动
handlers.keydown = keyDownHandler;        // 键盘输入
handlers.keyup = keyUpHandler;
handlers.click = clickHandler;
handlers.submit = preventDefaults;
handlers.beforeunload = preventDefaults;
```

### 3.2 客户端选择器生成器 clientSelectorGenerator

[clientSelectorGenerator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/src/helpers/clientSelectorGenerator.ts) 是一个单例类，在前端 iframe 上下文中运行。

**导出方式**：
```typescript
// L4378-L4379
export { ClientSelectorGenerator };
export const clientSelectorGenerator = new ClientSelectorGenerator();
```

**核心公共方法**：

| 方法 | 输入 | 输出 | 调用时机 |
|------|------|------|---------|
| `generateSelector(iframeDoc, coords, action)` | Document + 坐标 + 动作类型 | 最优选择器字符串 | 点击/键盘事件发生时 |
| `generateDataForHighlighter(coords, iframeDoc, ...)` | 坐标 + Document | 高亮数据（rect、selector、elementInfo、groupInfo） | 鼠标移动/悬停时 |
| `getElementInformation(iframeDoc, coords, ...)` | Document + 坐标 | ElementInfo 对象 | 事件发生时获取元素详情 |
| `generateSelectorsFromElement(element, iframeDoc)` | HTMLElement | 多策略 Selectors 对象 | 从元素直接生成 |
| `getChildSelectors(iframeDoc, parentSelector)` | 父选择器 | 子元素选择器数组 | 列表模式下 |
| `analyzeElementGroups(iframeDoc)` | Document | 无（内部状态更新） | 列表模式每次悬停时 |

### 3.3 前端调用时机与数据流

以点击事件为例，前端 `mouseDownHandler` 的完整流程：

```
用户在 iframe 中点击
      ↓
mouseDownHandler 触发
      ├─ 计算点击坐标 (iframeX, iframeY)
      │
      ├─ [捕获模式判断] isInCaptureMode (getText || getList)
      │   ├─ 是 → 调用 onElementSelect（选择元素，不发送点击）
      │   └─ 否 → 继续录制流程
      │
      ├─ [链接特殊处理] 检测 target.closest("a[href]")
      │   └─ 是 → 阻止默认跳转，记录 isSPA 标志
      │
      ├─ 生成选择器
      │   └─ clientSelectorGenerator.generateSelector(
      │          iframeDoc, {x, y}, ActionType.Click
      │       )
      │
      ├─ 获取元素信息
      │   └─ clientSelectorGenerator.getElementInformation(...)
      │
      ├─ [特殊输入判断]
      │   ├─ SELECT 标签 → 触发 onShowDropdown（弹出下拉选择器）
      │   ├─ date/time/datetime-local input → 触发日期选择器
      │   └─ 普通元素 → 发送 dom:click 事件
      │
      └─ Socket 发送
          └─ socket.emit("dom:click", {
               selector,         // 前端已生成好的选择器
               userId,
               elementInfo,      // 元素详情
               coordinates,      // 相对坐标
               isSPA             // 是否 SPA 链接
             })
```

---

## 四、Socket 事件载荷详解

### 4.1 主要事件与载荷格式

| Socket 事件 | 发送方 | 载荷结构 |
|------------|--------|---------|
| `dom:click` | 前端 → 服务端 | `{ selector, userId, elementInfo, coordinates?, isSPA? }` |
| `dom:keypress` | 前端 → 服务端 | `{ selector, key, userId, inputType? }` |
| `dom:scroll` | 前端 → 服务端 | `{ deltaX, deltaY }` |
| `input:url` | 前端 → 服务端 | `string` (url) |
| `input:date` | 前端 → 服务端 | `{ selector, value }` |
| `input:dropdown` | 前端 → 服务端 | `{ selector, value }` |
| `highlighter` | 服务端 → 前端 | `{ rect, selector, elementInfo, isDOMMode, shadowInfo }` |
| `workflow` | 服务端 → 前端 | `WorkflowFile` 完整工作流 |

### 4.2 dom:click 载荷详解

```typescript
// DOMBrowserRenderer.tsx L569-L583
socket.emit("dom:click", {
  selector,            // 已生成的最优选择器（字符串）
  userId: user?.id || "unknown",
  elementInfo,         // ElementInfo 对象（标签名、属性、文本等）
  coordinates: {       // 相对元素坐标
    x: relativeX,
    y: relativeY
  },
  isSPA: false         // 是否为 SPA 导航
});
```

> **重要**：`dom:click` 事件中已经包含了**前端生成好的 selector**，服务端的 `onDOMClickAction` **直接使用这个 selector**，不再重新生成。

### 4.3 dom:keypress 载荷详解

```typescript
// DOMBrowserRenderer.tsx L642-L647
socket.emit("dom:keypress", {
  selector,        // 焦点元素的选择器
  key: keyboardEvent.key,
  userId: user?.id || "unknown",
  inputType: elementInfo?.attributes?.type || "text",
});
```

---

## 五、服务端：事件路由与 Workflow 构建

### 5.1 事件路由：inputHandlers.ts

[inputHandlers.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/inputHandlers.ts) 是服务端的 Socket 事件注册中心。

**注册入口**：
```typescript
// L871-L887
const registerInputHandlers = (socket: Socket, userId: string) => {
    socket.on("input:keyup", (data) => onKeyup(data, userId));
    socket.on("input:url", (data) => onChangeUrl(data, userId));
    socket.on("input:date", (data) => onDateSelection(data, userId));
    socket.on("dom:click", (data) => onDOMClickAction(data, userId));
    socket.on("dom:keypress", (data) => onDOMKeyboardAction(data, userId));
    // ...
};
```

### 5.2 包装器模式：handleWrapper

所有事件处理器都通过 `handleWrapper` 包装，确保：
1. 浏览器实例存在且活跃
2. 不在解释执行（回放）状态
3. 当前 Page 有效

```typescript
// L28-L57
const handleWrapper = async (handleCallback, userId, args?) => {
    const id = browserPool.getActiveBrowserId(userId, "recording");
    if (id) {
        const activeBrowser = browserPool.getRemoteBrowser(id);
        // 回放中则忽略输入
        if (activeBrowser?.interpreter.interpretationInProgress()) return;
        const currentPage = activeBrowser?.getCurrentPage();
        if (currentPage && activeBrowser) {
            await handleCallback(activeBrowser, currentPage, args);
        }
    }
};
```

### 5.3 选择器如何进入工作流：onDOMClickAction 完整流程

以 `dom:click` 事件为例，服务端处理链路：

```
socket.on("dom:click", data)
      ↓
onDOMClickAction(data, userId)        [inputHandlers.ts]
      ↓
handleWrapper(handleClickAction, ...) [inputHandlers.ts]
      ↓
handleClickAction(activeBrowser, page, data)
      ├─ 步骤1: 移除 target="_blank" 防止新标签
      │    └─ page.evaluate(sel) → document.querySelector(sel)
      │
      ├─ 步骤2: Playwright 执行真实点击
      │    ├─ input 元素且有坐标 → page.mouse.click(坐标)
      │    └─ 普通元素 → page.click(selector)
      │
      └─ 步骤3: 生成 Workflow Pair
           └─ generator.onDOMClickAction(page, data)
                └─ addPairToWorkflowAndNotifyClient(pair, page)
                     └─ socket.emit("workflow", workflowRecord)
```

### 5.4 WorkflowGenerator 中的构建逻辑

[Generator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts) 的 `onDOMClickAction` 直接使用前端传来的 selector：

```typescript
// L426-L458
public onDOMClickAction = async (page, data) => {
  const { selector, url, elementInfo, coordinates } = data;

  // 直接使用前端传来的 selector 构建 Pair
  const pair: WhereWhatPair = {
    where: { 
      url: this.getBestUrl(url),
      selectors: [selector]    // selector 进入 where 条件
    },
    what: [{
      action: 'click',
      args: [selector],       // selector 进入 what 动作参数
    }],
  };

  // input/textarea 附加坐标信息
  if (elementInfo && coordinates && isInputElement) {
    pair.what[0].args.push(
      { position: coordinates },
      { cursorIndex: 0 }
    );
  }

  // 记录状态
  this.generatedData.lastUsedSelector = selector;
  this.generatedData.lastAction = 'click';

  // 加入工作流并通知前端
  await this.addPairToWorkflowAndNotifyClient(pair, page);
};
```

> **关键点**：`onDOMClickAction` **不重新生成选择器**，它直接消费前端通过 socket 发送的 `selector`。选择器在前端 `mouseDownHandler` 中已经生成完毕。

### 5.5 addPairToWorkflowAndNotifyClient：合并与通知

选择器进入工作流后，`addPairToWorkflowAndNotifyClient` 做智能合并：

```typescript
// L295-L344
private addPairToWorkflowAndNotifyClient = async (pair, page) => {
  // 1. 检查是否已存在相同 where 选择器的 Pair
  //    → 存在：将 what 追加到已有 Pair（同页面多步操作合并）
  //    → 不存在：继续
  let matched = selectorAlreadyInWorkflow(
    pair.where.selectors[0],
    this.workflowRecord.workflow
  );
  if (matched) {
    matched.what = matched.what.concat(pair.what);
    return;
  }

  // 2. 处理 over-shadowing（元素同时可见导致的规则覆盖）
  const handled = await this.handleOverShadowing(pair, page, ...);
  if (!handled) {
    // 3. 追加 waitForLoadState 动作
    pair.what.push({ action: 'waitForLoadState', args: ['networkidle'] });
    // 4. 插入工作流数组
    this.workflowRecord.workflow.splice(index, 0, pair);
  }

  // 5. 通知前端更新
  this.socket.emit('workflow', this.workflowRecord);
};
```

---

## 六、Screenshot 模式：服务端主导的选择器生成

### 6.1 触发时机

当处于 Screenshot 模式（非 DOM 模式）时，前端只发送点击坐标，服务端通过 `onClick(coordinates, page)` 方法生成选择器。

### 6.2 服务端选择器生成链路

```
onClick(coordinates, page)           [Generator.ts L511]
      ↓
generateSelector(page, coords, action)
      ├─ getElementInformation(page, coords)  ← 获取元素信息
      │   └─ page.evaluate(...)  ← 在 Playwright 浏览器中执行
      │
      ├─ getSelectors(page, coords)           ← 生成多策略选择器
      │   └─ page.evaluate(...)  ← 在 Playwright 浏览器中执行
      │        └─ finder 算法（同客户端实现）
      │
      └─ getBestSelectorForAction(action)     ← 决策最优选择器
           ↓
      构建 WhereWhatPair
           ↓
      addPairToWorkflowAndNotifyClient
```

### 6.3 服务端 selector.ts 的角色

[selector.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/selector.ts) 是服务端的选择器工具集合，所有函数都通过 `page.evaluate()` 将代码注入 Playwright 控制的浏览器中执行。

**导出函数**：

| 函数 | 说明 |
|------|------|
| `getElementInformation(page, coords, listSelector, getList)` | 获取元素基础信息 |
| `getSelectors(page, coords)` | 生成多策略选择器（10+ 种） |
| `getRect(page, coords, listSelector, getList)` | 获取元素位置大小 |
| `getNonUniqueSelectors(page, coords, listSelector)` | 列表模式非唯一选择器 |
| `getChildSelectors(page, parentSelector)` | 获取子元素选择器 |
| `selectorAlreadyInWorkflow(selector, workflow)` | 检查选择器是否已在工作流中 |
| `isRuleOvershadowing(pair, workflow, page)` | 检查规则覆盖 |

---

## 七、完整链路对比图

### 7.1 DOM 模式链路（前端主导）

```
前端浏览器 (iframe 内)
┌─────────────────────────────────────────┐
│ DOMBrowserRenderer 组件                 │
│  ├─ mousedown 事件监听                  │
│  ├─ clientSelectorGenerator.generateSelector()
│  │   └─ @medv/finder 算法               │
│  ├─ clientSelectorGenerator.getElementInformation()
│  └─ socket.emit("dom:click", {
│        selector,    ← 已生成的选择器
│        elementInfo,
│        coordinates
│     })
└───────────────────┬─────────────────────┘
                    │ Socket.io
                    ▼
服务端 (Node.js)
┌─────────────────────────────────────────┐
│ inputHandlers.ts                        │
│  └─ onDOMClickAction → handleClickAction│
│        ├─ Playwright 执行点击           │
│        │   └─ page.click(selector)      │
│        └─ WorkflowGenerator             │
│             └─ onDOMClickAction         │
│                  └─ 直接使用 selector    │
│                     构建 WhereWhatPair  │
│                     加入 workflow        │
└─────────────────────────────────────────┘
```

### 7.2 Screenshot 模式链路（服务端主导）

```
前端 (只有截图)
┌─────────────────────────────────────────┐
│ 用户点击截图                            │
│  └─ socket.emit("click", coordinates)   │
└───────────────────┬─────────────────────┘
                    │ Socket.io
                    ▼
服务端 (Node.js + Playwright)
┌─────────────────────────────────────────┐
│ WorkflowGenerator.onClick(coords, page) │
│  ├─ page.mouse.click(coords)            │
│  │  ↓                                   │
│  ├─ generateSelector(page, coords)      │
│  │   ├─ getElementInformation()         │
│  │   │   └─ page.evaluate(...)          │
│  │   ├─ getSelectors()                  │
│  │   │   └─ page.evaluate(...)          │
│  │   │      └─ @medv/finder 算法        │
│  │   └─ getBestSelectorForAction()      │
│  │                                      │
│  └─ 构建 WhereWhatPair → 加入 workflow  │
└─────────────────────────────────────────┘
```

---

## 八、前后端职责总览

| 职责 | 前端 | 服务端 |
|------|------|--------|
| DOM 事件捕获 | ✓（iframe 内原生监听） | ✗ |
| 选择器生成（DOM 模式） | ✓（clientSelectorGenerator） | ✗（直接使用前端结果） |
| 选择器生成（Screenshot 模式） | ✗ | ✓（selector.ts + Playwright） |
| 元素高亮显示 | ✓ | ✓（highlighter 事件下发数据） |
| 列表分组检测 | ✓（analyzeElementGroups） | ✓（服务端也有相关逻辑） |
| Socket 事件发送 | ✓ | ✓（双向通信） |
| Workflow 构建与管理 | ✗ | ✓（WorkflowGenerator） |
| Playwright 浏览器操作 | ✗ | ✓ |
| 工作流优化（合并按键等） | ✗ | ✓ |
| 持久化存储 | ✗ | ✓ |

---

## 九、关键文件索引

| 文件 | 端 | 核心职责 |
|------|----|---------|
| [src/components/recorder/DOMBrowserRenderer.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/src/components/recorder/DOMBrowserRenderer.tsx) | 前端 | 录制组件：iframe 事件监听 + Socket 发送 |
| [src/helpers/clientSelectorGenerator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/src/helpers/clientSelectorGenerator.ts) | 前端 | 客户端选择器生成器：单例类 + 分组算法 |
| [server/src/browser-management/inputHandlers.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/inputHandlers.ts) | 服务端 | Socket 事件路由 + Playwright 动作执行 |
| [server/src/workflow-management/classes/Generator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts) | 服务端 | 工作流编排器：事件 → Where-What Pair |
| [server/src/workflow-management/selector.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/selector.ts) | 服务端 | 服务端选择器生成（page.evaluate 注入） |
| [server/src/workflow-management/utils.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/utils.ts) | 服务端 | 选择器优先级决策 |
| [maxun-core/src/browserSide/scraper.js](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/maxun-core/src/browserSide/scraper.js) | 浏览器端 | 回放时选择器执行：`>>` / `:>>` 解析 |
