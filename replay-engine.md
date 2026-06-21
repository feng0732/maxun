# 工作流回放执行引擎代码理解

## 一、整体架构总览

工作流回放引擎采用 **四层分层架构**，从任务调度到浏览器动作执行形成完整链路：

```
┌─────────────────────────────────────────────────────────┐
│  任务调度层 (Task Scheduler)                            │
│  - server/src/task-runner.ts (Graphile Worker 任务队列) │
│  - scheduler/index.ts (定时调度器)                      │
│  - routes/workflow.ts (HTTP API 触发)                   │
└──────────────────────┬──────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────┐
│  服务封装层 (Server Wrapper)                            │
│  - WorkflowInterpreter (服务端解释器)                   │
│  - RemoteBrowser (远程浏览器会话)                       │
│  - BrowserPool (浏览器池 / 资源管理)                    │
│  - controller.ts (浏览器生命周期控制器)                 │
└──────────────────────┬──────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────┐
│  核心执行层 (maxun-core Interpreter)                    │
│  - Interpreter.run() 入口                               │
│  - Interpreter.runLoop() 主循环                         │
│  - carryOutSteps() 动作分发                             │
│  - handlePagination() 分页处理                          │
└──────────────────────┬──────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────┐
│  浏览器动作层 (Browser-side Actions)                    │
│  - Playwright 原生 API (click/goto/type 等)             │
│  - scraper.js 注入脚本 (scrape/scrapeSchema/scrapeList) │
│  - 自定义动作 (crawl/search/screenshot 等)              │
└─────────────────────────────────────────────────────────┘
```

---

## 二、步骤调度链路详解

### 2.1 触发入口

工作流执行有 **三种触发方式**：

#### (1) 手动 Run 触发
入口：`server/src/task-runner.ts`

```typescript
// Graphile Worker 任务队列处理 EXECUTE_RUN 任务
[QUEUE_NAMES.EXECUTE_RUN]: async (payload: unknown) => {
  await processRunExecution(payload as ExecuteRunData);
}
```

#### (2) 定时调度触发
入口：`server/src/workflow-management/scheduler/index.ts`

```typescript
// handleRunRecording 函数创建 Run 记录并等待浏览器就绪
export async function handleRunRecording(id: string, userId: string) {
  const result = await createWorkflowAndStoreMetadata(id, userId);
  // 监听 socket 'ready-for-run' 事件触发实际执行
  socket.on('ready-for-run', readyHandler);
}
```

#### (3) 编辑器内回放
入口：`server/src/browser-management/classes/RemoteBrowser.ts`

```typescript
public interpretCurrentRecording = async (): Promise<void> => {
  const workflow = this.generator.AddGeneratedFlags(this.generator.getWorkflowFile());
  await this.interpreter.interpretRecordingInEditor(workflow, this.currentPage, ...);
}
```

---

### 2.2 Workflow 数据结构

工作流文件格式定义在 `maxun-core/src/types/workflow.ts`：

```typescript
// 单个步骤 (Where-What 对)
interface WhereWhatPair {
  id?: string
  where: Where           // 条件匹配：URL、选择器、Cookie、逻辑运算符
  what: What[]           // 动作列表：要执行的浏览器操作
}

type Workflow = WhereWhatPair[];

// WorkflowFile 是顶层结构
type WorkflowFile = {
  meta?: MetaData,
  workflow: Workflow
}
```

**关键设计**：每个步骤是 `where`(条件) + `what`(动作) 的配对。早期版本是条件匹配驱动（根据当前页面状态匹配适用的步骤），当前版本已简化为 **顺序执行** 模式。

---

### 2.3 核心调度循环 runLoop()

核心调度逻辑位于 `maxun-core/src/interpret.ts`。

#### 执行流程：

```
Interpreter.run(page, params)
    │
    ├─► Preprocessor.initWorkflow()     // 参数替换、正则编译
    ├─► ensureScriptsLoaded(page)       // 注入 scraper.js 到页面
    ├─► 设置 stopper 停止回调
    └─► concurrency.addJob(() => runLoop(page, workflow))
              │
              ▼
        runLoop() 主循环 (while true)
              │
              ├─ [1] 中止检查: isAborted / page.isClosed() / stopper
              ├─ [2] 防死循环: MAX_LOOP_ITERATIONS = 1000
              ├─ [3] waitForLoadState() 等待页面稳定
              ├─ [4] workflowCopy 为空则结束
              │
              ├─ [5] 匹配动作 (当前简化为取最后一个)
              │     actionId = workflowCopy.length - 1
              │
              ├─ [6] 重复检查: repeatCount > maxRepeats 则 throw Error
              │     └─ (repeatCount 累加的前提是 action === lastAction)
              │
              ├─ [7] 执行动作: carryOutSteps(page, action.what)
              │     │
              │     ├─ 成功: usedActions.push() → workflowCopy.splice() → loopIterations=0
              │     └─ 失败: catch 后记录日志 → continue 下一轮循环 (动作不移除)
              │
              └─ [8] (回到循环顶部)
```

#### 关键机制说明：

**(1) 并发控制 (Concurrency)**

引擎使用 `Concurrency` 类管理并行任务（如 `enqueueLinks` 打开多页面）。`runLoop` 本身通过 `concurrency.addJob()` 提交，最终等待所有并发任务完成。

**(2) Popup 处理**

在循环开始时注册 popup 监听，浏览器打开的新窗口会自动启动独立的 `runLoop` 执行相同工作流：

```typescript
const popupHandler = (popup: Page) => {
  this.concurrency.addJob(() => this.runLoop(popup, workflowCopy));
};
p.on('popup', popupHandler);
```

**(3) 进度反馈**

每次成功执行完步骤后通过 `debugChannel.progressUpdate` 回调向前端推送执行进度（失败时不推送）。

---

## 三、浏览器动作执行链路

### 3.1 动作分发器 carryOutSteps()

位于 `maxun-core/src/interpret.ts`，负责将 `what[]` 中的每个动作分发到具体执行逻辑。

#### 动作分类：

```
                    ┌──────────────────────────────┐
                    │       carryOutSteps()        │
                    └──────────────┬───────────────┘
                                   │
                 ┌─────────────────┼─────────────────┐
                 ▼                 ▼                 ▼
        ┌────────────────┐ ┌────────────────┐ ┌──────────────────┐
        │ 自定义动作     │ │ Playwright 原生 │ │ 特殊优化动作     │
        │ (wawActions)   │ │  Page API      │ │ (goto/click等)   │
        └───────┬────────┘ └───────┬────────┘ └────────┬─────────┘
                │                  │                    │
                │                  │                    │
  ┌─────────────┴──────┐     ┌─────┴──────┐       ┌────┴──────┐
  │ screenshot         │     │ page.click │       │ goto 优化 │
  │ scrape             │     │ page.type  │       │ click 重试│
  │ scrapeSchema       │     │ page.fill  │       │ wait优化  │
  │ scrapeList         │     │ ...        │       └───────────┘
  │ scrapeListAuto     │     └────────────┘
  │ crawl              │
  │ search             │
  │ enqueueLinks       │
  │ scroll             │
  │ script             │
  │ flag               │
  └────────────────────┘
```

---

### 3.2 自定义动作详解

#### (1) scrapeSchema - 结构化数据抓取

```typescript
// 流程：设置动作类型 → 等待页面稳定 → 注入脚本检查
//      → page.evaluate(window.scrapeSchema) → 累积数据 → 回调
await this.waitForDynamicStability(page, ...);
await this.ensureScriptsLoaded(page);
const scrapeResult = await page.evaluate(
  (schemaObj) => window.scrapeSchema(schemaObj), normalizedSchema
);
```

**数据累积策略**：连续的 `scrapeSchema` 结果会合并到同一行数据（除非有重复字段检测到，才另起新行）。

#### (2) scrapeList - 列表数据抓取

支持 **三种分页模式**（见 `handlePagination` 方法）：

| 分页类型 | 实现方式 |
|---------|---------|
| `scrollDown` | 滚动加载，检测高度/结果变化 |
| `scrollUp` | 向上滚动，同上 |
| `clickNext` | 点击下一页按钮，支持多个选择器自动尝试 |

关键特性：
- 去重：通过 `Set<string>` 存储已抓取条目的 JSON 签名
- 限制：尊重 `config.limit` 参数
- XPath 支持：选择器自动识别 XPath / CSS 语法
- Shadow DOM / iframe 穿透：使用 `>>` 和 `:>>` 分隔符
- 分页选择器重试：MAX_RETRIES=3，每次间隔 RETRY_DELAY=1000ms

#### (3) crawl - 全站爬取

**广度优先搜索 (BFS) 算法**：
```
起始 URL → 入队 (depth=0)
    │
    ▼
While 队列非空 && 结果数 < limit:
    ├─ 出队 {url, depth}
    ├─ 应用 robots.txt 规则检查
    ├─ 应用 include/exclude 路径规则
    ├─ 抓取页面内容 (title/text/html/metadata/links)
    └─ depth < maxDepth 时，提取页面内链接入队
```

可配置项：`mode`(domain/subdomain/path)、`limit`、`maxDepth`、`useSitemap`、`respectRobots`。

#### (4) search - 搜索引擎抓取

流程：构造 DuckDuckGo 搜索 URL → 提取搜索结果列表 → (可选) 逐个访问结果页抓取完整内容。

#### (5) flag - 断点/暂停标记

触发 EventEmitter 的 `'flag'` 事件，用于编辑器模式下的断点暂停、步进调试。

---

### 3.3 浏览器端注入脚本 scraper.js

位于 `maxun-core/src/browserSide/scraper.js`，通过 `page.addInitScript()` 在每个页面加载前注入。

暴露以下全局函数：

| 函数 | 功能 |
|------|------|
| `window.scrape(selector?)` | 启发式抓取，自动识别页面上的"有趣"元素 |
| `window.scrapeSchema(schema)` | 按 schema 定义抓取结构化字段（支持 CSS/XPath/Shadow DOM/iframe） |
| `window.scrapeList(config)` | 抓取列表数据，支持表格感知和相似元素扩展 |
| `window.scrapeListAuto(listSelector)` | 自动抓取列表元素的选择器和文本 |
| `window.scrollDown/Up(pages)` | 滚动控制 |

**selector 语法增强**：
- `>>` 分隔符：穿透 Shadow DOM（如 `custom-element >> .inner-class`）
- `:>>` 分隔符：穿透 iframe/frame（如 `iframe[name=foo] :>> .content`）
- 原生 XPath 支持：自动识别 `//`、`./` 开头的选择器

---

### 3.4 Playwright 原生动作优化

部分原生动作有特殊优化处理：

#### (1) `goto` 导航优化
```typescript
// 根据后续动作自动选择等待策略
const needsDataSoon = this.blockNeedsVisualRender(steps) 
  || this.remainingWorkflowNeedsVisualRender(remaining);
existingOpts.waitUntil = needsDataSoon ? 'networkidle' : 'domcontentloaded';
```
失败处理：try-catch 吞掉异常，仅记录 WARN 日志，**继续执行后续动作**。

#### (2) `click` 失败重试
```typescript
try {
  await page.click(selector);
} catch {
  try {
    // 重试：使用 force: true 跳过可操作性检查
    await page.click(selector, { force: true });
  } catch {
    // 两次都失败：continue 跳到下一个 step
    continue;
  }
}
```

#### (3) `waitForLoadState` 降级
请求 `networkidle` 但超时时自动降级为 `domcontentloaded`。降级后也失败则不抛异常。

#### (4) 其他通用原生动作
```typescript
try {
  await executeAction(invokee, methodName, step.args);
} catch (error: any) {
  this.log(`Action ${methodName} failed: ${error.message}`, Level.ERROR);
  continue;  // 跳到下一个 step，不抛错
}
```

---

## 四、失败处理链路（对照代码事实）

### 4.1 单个动作失败后的处理路径（核心）

这是最容易误解的部分，以下是严格对照代码事实的描述：

```
carryOutSteps() 遍历 steps[] 中的每个 step:
    │
    ├─ 动作分类
    │   ├─ Playwright 原生动作 (goto/click/wait/其他):
    │   │   ├─ goto/waitForLoadState: 内部 try-catch 吞掉 → 继续下一个 step
    │   │   ├─ click: 第1次失败 → force:true 重试 → 再失败 → continue 下一个 step
    │   │   └─ 其他原生动作: 失败 → continue 下一个 step
    │   │
    │   └─ 自定义 wawActions (scrape/scrapeList/scrapeSchema/...):
    │       └─ 无内部 try-catch → 失败直接抛出异常
    │          │
    │          └─ 异常冒泡到 runLoop() 的外层 try-catch
    │
    ▼
runLoop() 外层 catch (L2861-L2864):
    ├─ this.log(e, Level.ERROR)   // 记录错误日志
    └─ continue                   // 直接进入下一轮 while 循环
         │
         ▼
    下一轮循环发生了什么？
    ├─ [动作不移除] workflowCopy.splice(actionId, 1) 未被执行
    │   → 同一个 WhereWhatPair 仍留在 workflowCopy 中
    │
    ├─ [匹配同一动作] actionId = workflowCopy.length - 1
    │   → 仍然匹配到同一个失败的动作
    │
    ├─ [repeatCount 累加] action === lastAction → repeatCount++
    │   → 因为同一个动作对象反复被匹配
    │
    ├─ [loopIterations 累加] 成功时才会 reset loopIterations=0
    │   → 失败时不 reset，持续 +1
    │
    └─ [两种可能的结局]
        ├─ 结局 A: repeatCount > maxRepeats
        │   → throw new Error(`Action xxx exceeded max retries`)
        │   → 整个 runLoop 终止 → 异常继续向上冒泡
        │
        └─ 结局 B: loopIterations > MAX_LOOP_ITERATIONS (1000)
            → 直接 return，静默终止 runLoop
```

**结论（对照代码事实）：**

| 问题 | 答案 | 代码依据 |
|------|------|---------|
| 单个 WhereWhatPair 内某个 step 失败，会不会移除当前动作？ | **原生动作（goto/click等）失败：不会移除，继续同 Pair 内下一个 step**<br>**自定义动作（scrape/scrapeList等）失败：整个 Pair 都不会被移除** | `carryOutSteps` 内 `continue` 跳到下一个 step；`runLoop` 内 catch 后 `continue`，不执行 `splice` |
| 会不会反复重试同一个动作？ | **会**，但不是无限重试。自定义动作反复失败会触发 `maxRepeats` 保护或 `MAX_LOOP_ITERATIONS` 死循环保护 | `runLoop` L2814-L2824（maxRepeats）、L2737-L2741（1000次保护） |
| 重试是"原地重试"还是"进入下一轮循环"？ | **进入下一轮 while 循环**。不是在当前 try 块内 retry，而是走完整的循环流程（包括 waitForLoadState、匹配动作等） | `runLoop` L2864 `continue` 语句 |
| 单个 step 失败会不会终止整个工作流？ | **一般不会**，除非：① 触发 maxRepeats 超限 ② 触发 MAX_LOOP_ITERATIONS ③ 用户中止 ④ 顶层（如 browser.init）抛出致命错误 | 见五层异常捕获体系 |

---

### 4.2 五层异常捕获体系

```
┌──────────────────────────────────────────────────────────────┐
│ Layer 1: server/src/task-runner.ts 顶层 try-catch (L136)     │
│   - 捕获所有未被下层吞掉的异常                                 │
│   - 更新 Run.status = 'failed'                                │
│   - 发送失败 webhook / socket 通知                            │
│   - 触发 analytics 埋点                                       │
│   - 清理浏览器资源 destroyRemoteBrowser()                     │
└──────────────────────┬───────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────┐
│ Layer 2: server/src/workflow-management/classes/Interpreter.ts│
│   - serializableCallback: try-catch 吞掉异常 (L705-L707)      │
│   - binaryCallback: try-catch 吞掉异常 (L731-L733)            │
│   - flushPersistenceBuffer: 指数退避重试 (最多3次)             │
│   - InterpretRecording() 本身无 try-catch                     │
│     → interpreter.run() 抛出的异常直接向上冒泡                │
└──────────────────────┬───────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────┐
│ Layer 3: maxun-core/src/interpret.ts runLoop() 外层 try-catch │
│   - 包裹 carryOutSteps() 调用 (L2832-L2865)                   │
│   - catch 后只记录日志 + continue，不抛出                      │
│   - 但 maxRepeats 超限时会主动 throw Error (L2823)            │
│   - MAX_LOOP_ITERATIONS 超限时直接 return 不抛                │
│   - waitForLoadState() 失败: 关闭页面 + return (L2750-L2756)  │
└──────────────────────┬───────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────┐
│ Layer 4: carryOutSteps() 每个 step 的 try-catch               │
│   - goto/waitForLoadState: 内部 try-catch 降级 + 吞掉         │
│   - click: 失败 → force:true 重试 → 再失败 continue           │
│   - 其他原生动作: 失败 → continue 下一个 step                  │
│   - 自定义 wawActions: **无内部 try-catch**                   │
│     → 失败直接抛出到 Layer 3                                  │
└──────────────────────┬───────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────┐
│ Layer 5: 特定动作内部重试                                      │
│   - handlePagination() 分页: MAX_RETRIES=3 + RETRY_DELAY=1s  │
│     → 分页选择器失败: 3次重试后剔除该选择器，不抛错            │
│     → 分页点击操作: 3次重试后放弃本页，不抛错                  │
│   - scrapeList/crawl/search: 单条记录失败不终止整体抓取        │
│   - enqueueLinks: 单个新页面失败 try-catch 吞掉 (L619-L624)   │
└──────────────────────────────────────────────────────────────┘
```

---

### 4.3 超时控制

| 阶段 | 超时值 | 位置 |
|------|--------|------|
| 浏览器初始化 | 45s | `server/src/browser-management/classes/RemoteBrowser.ts` |
| 浏览器池等待 | 60s | `server/src/task-runner.ts` |
| Page 获取 | 15s | `server/src/task-runner.ts` |
| 工作流整体执行 | 600s (10分钟) | `server/src/task-runner.ts` |
| 单页面导航 | 15s (默认) | `maxun-core/src/interpret.ts` |
| 浏览器销毁 | 30s | `server/src/browser-management/controller.ts` |
| 脚本注入检查 | 3s | `maxun-core/src/interpret.ts` |
| 分页选择器重试间隔 | 1s | `maxun-core/src/interpret.ts` handlePagination |

所有超时均通过 `Promise.race([promise, timeoutPromise])` 模式实现。

---

### 4.4 用户中止流程

```
用户点击停止 / abort API
    │
    ▼
abortRun(runId, userId)
    │
    ├─► Run.status = 'aborting'
    ├─► interpreter.abort()  // 设置 isAborted = true
    │
    ├─► 每一层循环/动作检查 isAborted 标志
    │   ├─ runLoop 顶部检查 (L2730-L2734)
    │   ├─ carryOutSteps 入口检查 (L551-L554)
    │   ├─ scrapeSchema 内部检查 (L654-L657)
    │   ├─ scrapeList/crawl/search 内部循环检查
    │   └─ handlePagination 分页循环检查
    │
    ├─► interpreter.stopInterpretation()
    ├─► interpreter.clearState()  // 刷盘持久化缓冲区
    ├─► destroyRemoteBrowser()
    └─► Run.status = 'aborted'
```

---

### 4.5 持久化重试机制

WorkflowInterpreter 中实现了 **批量持久化 + 指数退避重试**（见 `server/src/workflow-management/classes/Interpreter.ts`）：

```
动作产生数据 → addToPersistenceBatch()
                    │
                    ├─ 条件: scrapeSchema 或 缓冲区 >= BATCH_SIZE(5)
                    │     则立即 flushPersistenceBuffer()
                    └─ 否则: scheduleBatchFlush() 延迟 3s 刷盘
                              │
                              ▼
                    flushPersistenceBuffer()
                         │
                         ├─ 成功: 清空缓冲区, 重试计数归零
                         └─ 失败: 
                             ├─ 将数据放回缓冲区头部
                             ├─ persistenceRetryCount++
                             ├─ 退避延迟 = min(5000 * 2^retry, 30000) ms
                             └─ retryCount >= 3 时丢弃数据
```

---

### 4.6 日志与调试通道

通过 `debugChannel` 回调接口，maxun-core 将内部状态推送给上层：

| 回调 | 触发时机 |
|------|---------|
| `activeId(id)` | 匹配到新步骤时，推送当前步骤索引 |
| `debugMessage(msg)` | 每条日志信息 |
| `setActionType(type)` | 执行动作前，设置动作类型 (scrapeList/screenshot/...) |
| `setActionName(name)` | 设置当前动作的用户命名 |
| `incrementScrapeListIndex()` | scrapeList 每次分页抓取后 |
| `progressUpdate(current, total, percent)` | **仅步骤成功完成后** 更新总进度 |

另外，编辑器模式下的 `flag` 事件支持 **断点暂停、步进执行、恢复** 三种调试能力。

---

## 五、完整调用链路示例

以一次 **手动触发 Run 执行** 为例，完整调用链如下：

```
[1] 前端调用 API POST /workflow/run
     ↓ server/src/routes/workflow.ts
[2] 创建 Run 记录 (status=scheduled)
     ↓ createRemoteBrowserForRun()
[3] 预留浏览器池槽位 → 异步启动浏览器初始化
     ↓ browserPool.reserveBrowserSlotAtomic()
     ↓ initializeBrowserAsync()
[4] 浏览器准备完毕 → socket emit('ready-for-run')
     ↓ Graphile Worker 投递 EXECUTE_RUN 任务
[5] processRunExecution() 开始处理
     ↓ 轮询 browserPool.getRemoteBrowser() 等待浏览器就绪 (60s timeout)
[6] 获取 Page 对象 (15s timeout)
     ↓ browser.interpreter.setRunId(runId)
[7] WorkflowInterpreter.InterpretRecording()
     ↓ processWorkflow() 解密加密输入、限制 scrapeList limit
     ↓ AddGeneratedFlags() 在每个 what 开头插入 flag 动作
[8] new Interpreter(workflow, options) 构造 maxun-core 解释器
     ↓ Preprocessor.validateWorkflow() 校验格式
[9] interpreter.run(page, params)
     ↓ Preprocessor.initWorkflow() 替换 $param / 编译 $regex
     ↓ ensureScriptsLoaded() 注入 scraper.js
     ↓ concurrency.addJob(runLoop)
[10] runLoop() 主循环迭代
      ↓ 对每个 WhereWhatPair:
      ↓ carryOutSteps(page, action.what)
[11] 动作执行 (以 scrapeList 为例)
      ↓ waitForDynamicStability() 等待网络/页面稳定
      ↓ page.evaluate(window.scrapeList, config)
      ↓ serializableCallback() 推送数据到 WorkflowInterpreter
      ↓ persistDataToDatabase() 进入批量持久化缓冲区
[12] 所有步骤完成
      ↓ concurrency.waitForCompletion()
      ↓ flushPersistenceBuffer() 确保数据落库
[13] 后处理
      ↓ processRobotOutputFormats() 处理 crawl/search 输出格式
      ↓ BinaryOutputService 上传截图到对象存储
      ↓ triggerIntegrationUpdates() 推送 Google Sheet/Airtable
      ↓ sendWebhook() 回调用户 webhook
[14] 收尾
      ↓ Run.status = 'success' / 'failed'
      ↓ socket emit('run-completed')
      ↓ destroyRemoteBrowser() 释放浏览器
```

---

## 六、核心模块文件索引

| 文件 | 职责 |
|------|------|
| `server/src/task-runner.ts` | Graphile Worker 任务队列、Run 执行总控、顶层异常捕获 |
| `server/src/workflow-management/scheduler/index.ts` | 定时调度、Run 创建流程 |
| `server/src/workflow-management/classes/Interpreter.ts` | 服务端解释器封装（数据持久化、Socket 通信、批量落库重试） |
| `server/src/browser-management/controller.ts` | 浏览器生命周期管理（创建/销毁/查询） |
| `server/src/browser-management/classes/RemoteBrowser.ts` | 单个浏览器会话封装（含 interpreter、generator） |
| `server/src/browser-management/classes/BrowserPool.ts` | 浏览器池（每用户最多 2 个浏览器、槽位预留状态机） |
| `maxun-core/src/interpret.ts` | 核心解释器：runLoop 主循环、carryOutSteps 动作分发、各动作实现 |
| `maxun-core/src/preprocessor.ts` | 工作流预处理：参数替换、正则编译、校验 |
| `maxun-core/src/types/workflow.ts` | 工作流类型定义（WhereWhatPair、Workflow 等） |
| `maxun-core/src/browserSide/scraper.js` | 浏览器端注入脚本（数据抓取函数、Shadow DOM/iframe 穿透） |
| `server/src/storage/graphileWorker.ts` | Graphile Worker 任务投递封装 |
