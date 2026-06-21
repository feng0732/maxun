# 工作流回放执行引擎代码理解

## 一、整体架构总览

工作流回放引擎采用 **四层分层架构**，从任务调度到浏览器动作执行形成完整链路：

```
┌─────────────────────────────────────────────────────────┐
│  任务调度层 (Task Scheduler)                            │
│  - task-runner.ts (Graphile Worker 任务队列)            │
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
入口：[task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/server/src/task-runner.ts#L660-L662)

```typescript
// Graphile Worker 任务队列处理 EXECUTE_RUN 任务
[QUEUE_NAMES.EXECUTE_RUN]: async (payload: unknown) => {
  await processRunExecution(payload as ExecuteRunData);
}
```

#### (2) 定时调度触发
入口：[scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/server/src/workflow-management/scheduler/index.ts#L860-L909)

```typescript
// handleRunRecording 函数创建 Run 记录并等待浏览器就绪
export async function handleRunRecording(id: string, userId: string) {
  const result = await createWorkflowAndStoreMetadata(id, userId);
  // 监听 socket 'ready-for-run' 事件触发实际执行
  socket.on('ready-for-run', readyHandler);
}
```

#### (3) 编辑器内回放
入口：[RemoteBrowser.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L863-L894)

```typescript
public interpretCurrentRecording = async (): Promise<void> => {
  const workflow = this.generator.AddGeneratedFlags(this.generator.getWorkflowFile());
  await this.interpreter.interpretRecordingInEditor(workflow, this.currentPage, ...);
}
```

---

### 2.2 Workflow 数据结构

工作流文件格式定义在 [workflow.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/maxun-core/src/types/workflow.ts#L49-L60)：

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

核心调度逻辑位于 [interpret.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/maxun-core/src/interpret.ts#L2689-L2872)。

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
              ├─ [6] 断点/重复检查
              │     ├─ 触发 debugChannel.activeId 回调
              │     └─ 超过 maxRepeats 则抛出错误
              │
              ├─ [7] 执行动作: carryOutSteps(page, action.what)
              │
              ├─ [8] 标记完成: usedActions.push()
              ├─ [9] 从 workflowCopy 中移除已执行步骤
              └─ [10] 重置 loopIterations
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

每次执行完步骤后通过 `debugChannel.progressUpdate` 回调向前端推送执行进度。

---

## 三、浏览器动作执行链路

### 3.1 动作分发器 carryOutSteps()

位于 [interpret.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/maxun-core/src/interpret.ts#L550-L1953)，负责将 `what[]` 中的每个动作分发到具体执行逻辑。

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

位于 [scraper.js](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/maxun-core/src/browserSide/scraper.js)，通过 `page.addInitScript()` 在每个页面加载前注入。

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

#### (2) `click` 失败重试
```typescript
try {
  await page.click(selector);
} catch {
  // 重试：使用 force: true 跳过可操作性检查
  await page.click(selector, { force: true });
}
```

#### (3) `waitForLoadState` 降级
请求 `networkidle` 但超时时自动降级为 `domcontentloaded`。

---

## 四、失败处理链路

### 4.1 多层异常捕获

```
┌──────────────────────────────────────────────────┐
│ Layer 1: task-runner.ts 顶层 try-catch           │
│   - 更新 Run.status = 'failed'                    │
│   - 发送失败 webhook / socket 通知                │
│   - 触发 analytics 埋点                          │
│   - 清理浏览器资源                                │
└──────────────────────┬───────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────┐
│ Layer 2: WorkflowInterpreter 层                  │
│   - serializableCallback / binaryCallback 异常   │
│   - 持久化缓冲区刷盘失败重试                      │
│   - 加密输入解密失败降级                          │
└──────────────────────┬───────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────┐
│ Layer 3: Interpreter.runLoop()                   │
│   - 单步骤失败: catch 后 continue 继续下一步      │
│   - maxRepeats: 同一步骤重复执行超限则抛错终止     │
│   - isAborted: 用户中止标志立即停止               │
│   - MAX_LOOP_ITERATIONS: 死循环保护(1000次)      │
└──────────────────────┬───────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────┐
│ Layer 4: carryOutSteps() 单动作容错              │
│   - goto/waitForLoadState 失败降级                │
│   - click 失败用 force:true 重试                 │
│   - 其他通用动作: 失败记录日志后 continue          │
└──────────────────────┬───────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────┐
│ Layer 5: 特定动作内部重试                        │
│   - scrapeList 分页: MAX_RETRIES=3 + 退避延迟    │
│   - search: DuckDuckGo 结果提取重试              │
│   - crawl: 单 URL 失败不终止整个爬取             │
└──────────────────────────────────────────────────┘
```

---

### 4.2 超时控制

| 阶段 | 超时值 | 位置 |
|------|--------|------|
| 浏览器初始化 | 45s | [RemoteBrowser.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/server/src/browser-management/classes/RemoteBrowser.ts#L420) |
| 浏览器池等待 | 60s | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/server/src/task-runner.ts#L131) |
| Page 获取 | 15s | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/server/src/task-runner.ts#L132) |
| 工作流整体执行 | 600s (10分钟) | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/server/src/task-runner.ts#L401) |
| 单页面导航 | 15s (默认) | [interpret.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/maxun-core/src/interpret.ts#L1888) |
| 浏览器销毁 | 30s | [controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/server/src/browser-management/controller.ts#L158) |
| 脚本注入检查 | 3s | [interpret.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/maxun-core/src/interpret.ts#L2886) |

所有超时均通过 `Promise.race([promise, timeoutPromise])` 模式实现。

---

### 4.3 用户中止流程

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
    │   ├─ runLoop 顶部检查
    │   ├─ carryOutSteps 每步检查
    │   ├─ scrapeList/crawl/search 内部循环检查
    │   └─ handlePagination 分页循环检查
    │
    ├─► interpreter.stopInterpretation()
    ├─► interpreter.clearState()  // 刷盘持久化缓冲区
    ├─► destroyRemoteBrowser()
    └─► Run.status = 'aborted'
```

---

### 4.4 持久化重试机制

WorkflowInterpreter 中实现了 **批量持久化 + 指数退避重试**（见 [Interpreter.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/server/src/workflow-management/classes/Interpreter.ts#L827-L952)）：

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

### 4.5 日志与调试通道

通过 `debugChannel` 回调接口，maxun-core 将内部状态推送给上层：

| 回调 | 触发时机 |
|------|---------|
| `activeId(id)` | 匹配到新步骤时，推送当前步骤索引 |
| `debugMessage(msg)` | 每条日志信息 |
| `setActionType(type)` | 执行动作前，设置动作类型 (scrapeList/screenshot/...) |
| `setActionName(name)` | 设置当前动作的用户命名 |
| `incrementScrapeListIndex()` | scrapeList 每次分页抓取后 |
| `progressUpdate(current, total, percent)` | 步骤完成后更新总进度 |

另外，编辑器模式下的 `flag` 事件支持 **断点暂停、步进执行、恢复** 三种调试能力。

---

## 五、完整调用链路示例

以一次 **手动触发 Run 执行** 为例，完整调用链如下：

```
[1] 前端调用 API POST /workflow/run
     ↓ routes/workflow.ts
[2] 创建 Run 记录 (status=scheduled)
     ↓ createRemoteBrowserForRun()
[3] 预留浏览器池槽位 → 异步启动浏览器初始化
     ↓ browserPool.reserveBrowserSlotAtomic()
     ↓ initializeBrowserAsync()
[4] 浏览器准备完毕 → socket emit('ready-for-run')
     ↓ Graphile Worker 投递 EXECUTE_RUN 任务
[5] processRunExecution() 开始处理
     ↓ 轮询 browserPool.getRemoteBrowser() 等待浏览器就绪
[6] 获取 Page 对象
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
| [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/server/src/task-runner.ts) | Graphile Worker 任务队列、Run 执行总控 |
| [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/server/src/workflow-management/scheduler/index.ts) | 定时调度、Run 创建流程 |
| [Interpreter.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/server/src/workflow-management/classes/Interpreter.ts) | 服务端解释器封装（数据持久化、Socket 通信） |
| [controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/server/src/browser-management/controller.ts) | 浏览器生命周期管理（创建/销毁/查询） |
| [RemoteBrowser.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/server/src/browser-management/classes/RemoteBrowser.ts) | 单个浏览器会话封装（含 interpreter、generator） |
| [BrowserPool.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/server/src/browser-management/classes/BrowserPool.ts) | 浏览器池（每用户最多 2 个浏览器、槽位预留） |
| [maxun-core/interpret.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/maxun-core/src/interpret.ts) | 核心解释器：runLoop、carryOutSteps、动作实现 |
| [maxun-core/preprocessor.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/maxun-core/src/preprocessor.ts) | 工作流预处理：参数替换、正则编译、校验 |
| [maxun-core/types/workflow.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/maxun-core/src/types/workflow.ts) | 工作流类型定义 |
| [maxun-core/browserSide/scraper.js](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/maxun-core/src/browserSide/scraper.js) | 浏览器端注入脚本（数据抓取函数） |
| [storage/graphileWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/110-maxun/server/src/storage/graphileWorker.ts) | Graphile Worker 任务投递封装 |
