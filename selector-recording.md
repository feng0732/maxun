# 录制阶段选择器生成机制分析

## 一、整体架构概览

录制阶段的核心目标是将用户在浏览器中的交互事件（点击、输入等）转化为可在回放阶段稳定执行的元素定位信息。整个数据流分为 6 个关键阶段：

```
用户浏览器交互
      ↓
[1] 前端事件捕获 + 坐标采集
      ↓  socket.io (dom:click / dom:keypress 等)
[2] 服务端事件路由分发 (inputHandlers.ts)
      ↓
[3] WorkflowGenerator 编排处理 (Generator.ts)
      ↓  Playwright page.evaluate()
[4] 浏览器端元素定位：坐标 → HTMLElement (穿透 Shadow DOM / Iframe)
      ↓
[5] 多策略选择器并行生成 (@medv/finder 算法)
      ↓
[6] 选择器优先级决策 → 嵌入 Where-What Pair workflow
```

---

## 二、事件捕获与传输链路

### 2.1 前端事件源

用户的浏览器交互通过录制脚本注入页面后监听 DOM 事件，采集的信息包括：
- **坐标信息**：`{ x, y }` 鼠标点击位置（核心定位依据）
- **事件类型**：`click`、`keypress`、`input`、`change` 等
- **元素辅助信息**：标签名、类名、文本内容等

### 2.2 Socket.io 事件路由

服务端通过 [inputHandlers.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/inputHandlers.ts) 注册事件监听：

| Socket 事件         | 对应处理函数                  | 说明                   |
|---------------------|-----------------------------|------------------------|
| `dom:click`         | `onDOMClickAction`          | DOM 点击动作           |
| `dom:keypress`      | `onDOMKeyboardAction`       | 键盘按键动作           |
| `input:url`         | `onUrlChangeAction`         | URL 导航               |
| `input:date`        | `onDOMInputAction`          | 日期输入               |
| `input:dropdown`    | `onDOMDropdownAction`       | 下拉选择               |

所有事件处理器通过 `handleWrapper()` 包装，确保仅在浏览器活跃且非解释执行状态时才处理输入。

---

## 三、WorkflowGenerator 核心编排

[Generator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts) 是录制阶段的大脑，负责将原始事件转化为结构化的 Where-What Pair。

### 3.1 核心方法调用链

以点击事件 `onDOMClickAction` 为例：

```
onDOMClickAction(page, data)
    ↓
generateSelector(page, coordinates, action)  ← 关键：生成选择器
    ├─ getElementInformation(page, coordinates)  ← 获取元素基础信息
    ├─ getSelectors(page, coordinates)           ← 生成多策略选择器
    └─ getBestSelectorForAction(action)          ← 决策最优选择器
    ↓
构建 WhereWhatPair:
  {
    where: { url, selectors: [bestSelector] },
    what:  [{ action: "click", args: [{ ... }] }]
  }
    ↓
addPairToWorkflowAndNotifyClient(pair, page)
    ↓
workflow 优化（如将连续 keypress 合并为 type）
```

### 3.2 Where-What Pair 结构

工作流采用条件-动作对的结构：

```typescript
interface WhereWhatPair {
  where: {
    url: string;           // 页面 URL 匹配条件
    selectors?: string[];  // 元素选择器条件（回放时验证元素存在）
  };
  what: Array<{
    action: string;       // "click" | "type" | "scroll" 等
    args: any[];          // 动作参数
  }>;
}
```

---

## 四、元素定位：从坐标到 DOM 元素

选择器生成的第一步是将屏幕坐标 `(x, y)` 映射到准确的 HTMLElement。这一逻辑在 [selector.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/selector.ts) 的 `getElementInformation()` 和 `getSelectors()` 中通过 `page.evaluate()` 在浏览器端执行。

### 4.1 标准元素定位流程

```
document.elementsFromPoint(x, y)
      ↓  获取该坐标下所有堆叠元素
findDeepestElement()
      ↓  选择 DOM 树中最深层（最具体）的元素
[A 标签特殊处理] 若父元素是 <a>，则升级到 <a> 标签
      ↓
traverseShadowDOM()  ← 穿透 Shadow DOM
      ↓
处理 iframe / frame 穿透
      ↓
返回目标 HTMLElement
```

### 4.2 Shadow DOM 穿透

使用 `elementFromPoint` + `shadowRoot` 递归遍历（最大深度 4 层）：

```typescript
function traverseShadowDOM(element: HTMLElement): HTMLElement {
  let current = element;
  let shadowRoot = current.shadowRoot;
  while (shadowRoot && depth < MAX_SHADOW_DEPTH) {
    const shadowElement = shadowRoot.elementFromPoint(x, y);
    if (!shadowElement || shadowElement === current) break;
    deepest = shadowElement;
    current = shadowElement;
    shadowRoot = current.shadowRoot;
  }
  return deepest;
}
```

### 4.3 Iframe / Frame 穿透

对于 iframe 和传统 frame，通过坐标偏移计算相对位置后递归进入：

```
检测到元素是 <iframe> 或 <frame> 或在 frameset 中
      ↓
计算点击在 iframe 内的相对坐标: (iframeX = x - iframeRect.left)
      ↓
iframeDocument.elementFromPoint(iframeX, iframeY)
      ↓
递归 traverseShadowDOM()
      ↓
若内部还有 iframe，继续嵌套（最大 4 层）
```

---

## 五、@medv/finder 算法：自底向上 CSS 选择器生成

项目采用 `@medv/finder` 算法的本地实现（内嵌在 `getSelectors()` 中），核心思想是**自底向上搜索 + 惩罚分数排序 + 唯一性验证**。

### 5.1 惩罚分数体系 (Penalty)

| 选择器类型   | 惩罚分 | 说明                     |
|-------------|--------|--------------------------|
| `#id`       | 0      | 最优，ID 选择器          |
| `[attr=val]`| 0.5    | 属性选择器               |
| `.class`    | 1      | 类名选择器               |
| `tag`       | 2      | 标签选择器               |
| `*`         | 3      | 通配符（兜底）           |
| `:nth-child(i)` | +1 | 位置伪类（附加惩罚）     |

目标：找到**能够唯一确定该元素**的**惩罚分数总和最小**的选择器组合。

### 5.2 自底向上搜索 (bottomUpSearch)

从目标元素开始，逐层向父元素遍历，为每一层生成候选 Node 列表：

```
目标元素 <button class="btn primary" id="submit">
  level[0] 候选: [#submit, .btn, .primary, button, *]
      ↓
父元素 <form class="login-form">
  level[1] 候选: [.login-form, form, *]
      ↓
祖先元素 <div class="container">
  level[2] 候选: [.container, div, *]
```

每一层候选按优先级排序：`id > 属性 > 类名 > 标签 > *`

### 5.3 三级 Limit 降级策略

搜索按 Limit 分为三级，逐级降级以平衡性能和结果质量：

| Limit    | 策略                                         |
|----------|----------------------------------------------|
| `All`    | 每层保留所有候选 + nth-child 变体             |
| `Two`    | 每层仅保留最优 1 个候选 + nth-child 变体      |
| `One`    | 每层仅保留最优 1 个候选，必要时加 nth-child   |

调用链：`bottomUpSearch(All) → 失败则 fallback → bottomUpSearch(Two) → 失败则 fallback → bottomUpSearch(One)`

### 5.4 组合生成与唯一性验证

将各层候选进行笛卡尔积组合，按惩罚分升序排列后逐一验证：

```
对于候选路径 path:
  1. selector(path) → 组装为 CSS 选择器字符串
  2. rootDocument.querySelectorAll(css).length
     - == 1: 唯一匹配 ✓ 成功返回
     - == 0: 异常（选择不到元素）
     - > 1: 不唯一，尝试下一个
```

相邻层级用 `>`（直接子元素），其余用空格（后代元素）连接。

### 5.5 优化阶段 (optimize)

找到唯一选择器后，尝试递归删除中间层节点以获得更短的选择器：

```
原始: div.container > form.login-form > button#submit
            ↓ 尝试删除 .login-form
验证: div.container > button#submit 是否唯一？
            ↓ 是
优化后: div.container > button#submit
```

---

## 六、多策略选择器并行生成

单次录制不会只生成一个选择器，而是并行生成 **10+ 种不同策略**的选择器，组成 `Selectors` 对象，由 [selector.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/selector.ts) `genSelectors()` 函数负责：

```typescript
interface Selectors {
  id?: string | null;                    // #id 选择器
  generalSelector?: string | null;       // 通用 finder 结果（默认配置）
  attrSelector?: string | null;          // 允许任意属性的 finder 结果
  testIdSelector?: string | null;        // 测试 ID 属性（data-testid 等）
  text?: string;                         // 元素文本内容
  href?: string;                         // href 属性值
  hrefSelector?: string | null;          // 基于 href 的选择器
  accessibilitySelector?: string | null; // aria-label / alt / title
  formSelector?: string | null;          // name / placeholder / for
  relSelector?: string | null;           // rel 属性
  iframeSelector?: {                     // iframe 穿透选择器
    full: string;
    isIframe: boolean;
  } | null;
  shadowSelector?: {                     // Shadow DOM 穿透选择器
    full: string;
    mode: string;  // "open" | "closed"
  } | null;
}
```

### 6.1 特殊选择器生成逻辑

**Shadow DOM 选择器**（`>>` 分隔符）：
```
hostSelector >> shadowInnerSelector
例: .my-component >> .btn-primary
```
通过递归获取 shadow root 路径，每一层单独调用 finder，然后用 `>>` 连接。

**Iframe 选择器**（`:>>` 分隔符）：
```
iframeSelector :>> innerElementSelector
例: iframe[name="main"] :>> .submit-btn
```
类似 Shadow DOM，每级 iframe 单独生成选择器，用 `:>>` 连接。

---

## 七、选择器优先级决策

[utils.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/utils.ts) 中 `getBestSelectorForAction()` 根据**动作类型**和**标签类型**决定最终使用哪个选择器。

### 7.1 决策优先级总表

#### Click / Hover / DragAndDrop 动作

| 标签类型   | 优先级顺序（从高到低）                                                          |
|-----------|------------------------------------------------------------------------------|
| `<input>`  | testId → id → formSelector → accessibilitySelector → generalSelector → attrSelector |
| `<a>`      | testId → id → hrefSelector → accessibilitySelector → generalSelector → attrSelector |
| `<span>/<em>/<cite>/<b>/<strong>` | testId → id → accessibilitySelector → hrefSelector → textSelector → generalSelector → attrSelector |
| 其他       | testId → id → accessibilitySelector → hrefSelector → generalSelector → attrSelector |

全局最高优先级（所有标签共享）：
1. `iframeSelector.full`（若在 iframe 中）
2. `shadowSelector.full`（若在 Shadow DOM 中）

#### Input / Keydown 动作

优先级：`testId → id → formSelector → accessibilitySelector → generalSelector → attrSelector`

（shadowSelector 同样享有最高优先级）

### 7.2 设计原则

- **稳定性优先**：`data-testid` > `id` > 可访问性属性 > 类名/标签
- **语义匹配**：链接用 `href`，表单用 `name/placeholder`，文本元素考虑 `textSelector`
- **特殊优先**：iframe/Shadow DOM 选择器一旦存在即最高优先级，因为普通 CSS 选择器无法穿透

---

## 八、列表模式与元素分组

当处于列表抓取模式（`getList = true`）时，选择器生成逻辑发生变化：

### 8.1 非唯一选择器 (getNonUniqueSelectors)

列表模式下不再追求唯一选择器，而是生成能匹配**所有同类元素**的通用选择器。此时：
- `findContainerElement()` 替代 `findDeepestElement()`，向上找容器节点
- 表格的 `<td>/<th>` 自动升级到 `<table>`
- 通过结构指纹（ElementFingerprint）进行相似度分组

### 8.2 元素指纹 (ElementFingerprint)

[clientSelectorGenerator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/src/helpers/clientSelectorGenerator.ts) 中定义：

```typescript
interface ElementFingerprint {
  tagName: string;                    // 标签名
  normalizedClasses: string;          // 归一化类名（去除动态ID类）
  childrenCount: number;              // 子元素数量
  childrenStructure: string;          // 子元素结构
  attributes: string;                 // 属性签名
  depth: number;                      // DOM 深度
  textCharacteristics: {              // 文本特征
    hasText: boolean;
    textLength: number;
    hasLinks: number;
    hasImages: number;
    hasButtons: number;
  };
  signature: string;                  // 综合签名
}
```

### 8.3 分组算法

1. **表格特殊处理**：`tbody > tr` 直接强制分组（相同父 table）
2. **结构相似度计算**：指纹相似度 ≥ 0.7 阈值归为一组
3. **祖先桶聚类**：向上找 1~5 层祖先，将同祖先下的相似元素聚为一组（确保空间邻近）
4. **最小分组大小**：≥ 2 个元素才认为是有效列表组

---

## 九、关键文件索引

| 文件 | 核心职责 |
|------|---------|
| [server/src/workflow-management/classes/Generator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/classes/Generator.ts) | 工作流编排器：事件 → Where-What Pair |
| [server/src/workflow-management/selector.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/selector.ts) | 服务端选择器生成：@medv/finder + 多策略并行 |
| [server/src/workflow-management/utils.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/workflow-management/utils.ts) | 选择器优先级决策 |
| [server/src/browser-management/inputHandlers.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/server/src/browser-management/inputHandlers.ts) | Socket.io 事件路由 |
| [src/helpers/clientSelectorGenerator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/src/helpers/clientSelectorGenerator.ts) | 客户端选择器生成 + 元素分组算法 |
| [maxun-core/src/browserSide/scraper.js](file:///d:/fz/0601-2/solo-dogfeeding/code/106-maxun/maxun-core/src/browserSide/scraper.js) | 浏览器端抓取：`>>` / `:>>` 解析执行 |
