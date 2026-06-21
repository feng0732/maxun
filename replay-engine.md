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
              ├─ [2] 防死循环: ++loopIterations > MAX_LOOP_ITERATIONS(1000) → return
              ├─ [3] 入口 waitForLoadState() 失败 → 关闭页面 + return
              ├─ [4] workflowCopy 为空则结束
              │
              ├─ [5] 匹配动作 (取最后一个)
              │     actionId = workflowCopy.length - 1
              │
              ├─ [6] 重复检查: action === lastAction ? repeatCount++ : 0
              │     └─ repeatCount > maxRepeats → throw Error 终止
              │
              └─ [7] try { carryOutSteps() } catch { continue }
                    │
                    ├─ ✅ 成功路径:
                    │   ├─ usedActions.push(action.id)
                    │   ├─ workflowCopy.splice(actionId, 1)  ← 整组移除
                    │   ├─ executedActions++ / progressUpdate
                    │   └─ loopIterations = 0  ← 死循环计数器归零
                    │
                    └─ ❌ 失败路径:
                        ├─ log error
                        └─ continue → 回到 while 顶部 (整组不移除，下轮重试)
```

**关键不变量**：
- 整组 `WhereWhatPair` 被移除 ⟺ `carryOutSteps()` 正常返回
- `loopIterations` 归零 ⟺ 有一组动作被成功移除
- `repeatCount` 累加 ⟺ 同一组动作连续被匹配（未被移除）

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

### 3.2 自定义动作详解（含逐类失败语义）

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

**失败语义**：无 try-catch
- editor 模式：early return，写空对象 `{}` 回调 → 正常返回，整组移除
- 非 editor 模式：waitForDynamicStability / ensureScriptsLoaded / page.evaluate 任一失败 → **向上抛错**，整组保留重试

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

**失败语义**：外层完整 try-catch
- 无分页模式：page.evaluate 内部还有一层 try-catch，失败返回 `[]`
- 外层 catch：写空数组 `[]` → 调用 serializableCallback → **正常返回，不抛错**
- 整组动作**正常移除**，不会触发重试

#### (3) scrapeListAuto - 自动列表选择器发现

**失败语义**：无 try-catch
- page.evaluate(window.scrapeListAuto) 失败 → **向上抛错**，整组保留重试

#### (4) scrape - 启发式抓取

**失败语义**：动作本身无 try-catch，回调有服务端隔离
- 动作内部失败（waitForDynamicStability / ensureScriptsLoaded / page.evaluate）→ **向上抛错**，整组保留重试
- 回调失败：Run 模式下 `serializableCallback` 被服务端 try-catch 包裹，reject 被吞掉；Editor 模式下无 try-catch，回调抛错会 **向上抛出**

#### (5) screenshot - 页面截图

**失败语义**：动作本身无 try-catch，但回调有服务端隔离
- 动作内部失败（waitForImagesLoaded / page.screenshot 抛错）→ **向上抛错**，整组保留重试
- 回调失败分两种场景：
  - **Run 模式**：`binaryCallback` 被服务端 try-catch 包裹（`Interpreter.ts` L709-L733），回调 reject 被 catch 吞掉，**不会上抛到 maxun-core**，截图动作正常完成
  - **Editor 模式**：`binaryCallback` 无 try-catch，`persistBinaryDataToDatabase` 内部有自己的 try-catch 吞掉，正常情况下不会抛出；但如果 `binaryCallback` 函数本身抛错 → 会 reject → **向上抛错**，整组保留重试

#### (6) scroll - 页面滚动

**失败语义**：无 try-catch
- page.evaluate(scrollTo) 失败 → **向上抛错**，整组保留重试

#### (7) enqueueLinks - 多页面并发抓取

**失败语义**：主流程无 try-catch，内部并发任务有隔离
- page.locator(...).evaluateAll 提取链接失败 / page.close 失败 → **向上抛错**，整组保留重试
- 单个链接打开后执行 runLoop 失败：内部 addJob 包裹 try-catch 吞掉 → 不影响其他链接，不抛错到外层

#### (8) script - 自定义代码注入

```typescript
try {
  const x = new AsyncFunction('page', 'log', code);
  await x(page, this.log);
} catch (error: any) {
  this.log(`Script execution failed: ${error.message}`, Level.ERROR);
  throw new Error(`Script execution error: ${error.message}`);
}
```

**失败语义**：有 try-catch，但 **catch 里重新 throw**
- 代码执行失败 → 记录 ERROR 日志 → 重新包装错误后抛出 → **向上抛错**，整组保留重试

#### (9) crawl - 全站爬取

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

**失败语义**：整体有 try-catch，但 **catch 里重新 throw**
- 内部 robots.txt 获取、单页面抓取失败有小范围 try-catch 降级
- 整体流程致命失败 → 记录 ERROR 日志 → 重新包装错误后抛出 → **向上抛错**，整组保留重试

#### (10) search - 搜索引擎抓取

流程：构造 DuckDuckGo 搜索 URL → 提取搜索结果列表 → (可选) 逐个访问结果页抓取完整内容。

**失败语义**：整体有 try-catch，但 **catch 里重新 throw**
- discover 模式下结果为 0 条不算失败，正常回调返回
- mode=scrape 时单个结果页面抓取失败有内部 try-catch，写入 error 字段继续
- 整体流程致命失败（DDG 页面打不开等）→ 记录 ERROR 日志 → 重新包装错误后抛出 → **向上抛错**，整组保留重试

#### (11) flag - 断点/暂停标记

```typescript
flag: async () => new Promise((res) => {
  this.emit('flag', page, res);
}),
```

**核心机制**：flag 返回一个 **等待 `resume()` 调用才 resolve 的 Promise**，而非抛错。
执行流程：
1. maxun-core 内 `this.emit('flag', page, res)` 将 `res`（resolve 函数）传给服务端
2. 服务端 `Interpreter.ts` 监听 `'flag'` 事件：
   - **非暂停状态** → 直接调用 `resume()` → Promise resolve → 动作正常完成
   - **暂停状态**（命中断点或用户点暂停） → 将 `resume` 存入 `interpretationResume`，等待用户操作
3. 用户通过 socket 发送 `resume` / `step` 事件 → 调用 `interpretationResume()` → Promise resolve

**失败语义**：
- 正常情况：Promise 永远不会 reject，只是**停住等待**，不抛错，不触发整组重试
- 唯一可能出问题的场景：`this.emit('flag', ...)` 如果没有监听器且 EventEmitter 设置了 `captureRejectionSymbol`，或 `interpretationResume` 被设为 `null` 后又被调用
- 实际上：服务端在 `interpretRecordingInEditor` / `InterpretRecording` 中都注册了 `'flag'` 监听器，所以 emit 不会抛错
- **结论**：flag 动作的语义是"暂停等待"，不是"失败抛错"，整组会正常移除

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

### 3.4 Playwright 原生动作的失败处理语义

每个原生动作的失败行为各不相同，以下严格对照代码事实说明：

#### (1) `goto` 导航
- **优化**：默认降级为 `domcontentloaded` 等待策略（根据后续动作决定是否追加动态稳定等待）
- **失败处理**：try-catch 完整包裹，**吞掉异常**，仅记录 WARN 日志
- **推进语义**：当前 step 视为"执行过"，继续同 Pair 内的下一个 step
- **整组影响**：**不**导致整组动作失败，只要后续 step 都完成，整组正常移除

#### (2) `click` 点击
- **重试策略**：第1次失败 → 用 `force: true` 跳过可操作性检查再试一次
- **最终失败**：两次尝试都失败 → `continue` 跳过当前 step
- **推进语义**：当前 step 跳过，继续同 Pair 内的下一个 step
- **整组影响**：**不**导致整组动作失败

```typescript
try {
  await executeAction(invokee, methodName, step.args);
} catch (error: any) {
  try {
    await executeAction(invokee, methodName, [clickArgs[0], { force: true }]);
  } catch (error: any) {
    this.log(`Click action failed: ${error.message}`, Level.WARN);
    continue;  // 跳到 for 循环的下一个 step
  }
}
```

#### (3) `waitForLoadState` 页面加载等待
- **降级策略**：请求 `networkidle`/`load` 时自动降级为 `domcontentloaded` 再尝试
- **关键细节**：catch 块内的降级重试 **没有再套 try-catch**
- **推进语义**：
  - 第一次失败 → 降级后重试
  - 降级后也失败 → **异常向上抛出**，终止 carryOutSteps
- **整组影响**：降级后再失败会冒泡到 runLoop，整组动作**不会被移除**，下一轮循环重试整组

```typescript
try {
  // 第一次尝试（已降级为 domcontentloaded）
  await executeAction(invokee, methodName, args);
} catch (error: any) {
  // catch 块内没有再 try-catch！
  await executeAction(invokee, methodName, ['domcontentloaded', { timeout: 10000 }]);
  // 上面这句如果也失败，异常直接抛出 carryOutSteps
}
```

#### (4) 其他通用原生动作（type/fill/...）
- **失败处理**：try-catch 包裹 + `continue`
- **推进语义**：当前 step 跳过，继续同 Pair 内的下一个 step
- **整组影响**：**不**导致整组动作失败

#### 小结：动作失败与整组移除的关系（总览）

| 动作 | 失败后是否抛异常 | 当前 step 处理 | 同 Pair 后续 step | 整组是否移除 |
|------|----------------|-------------|----------------|------------|
| goto | 否（吞掉） | 跳过 | 继续执行 | ✅ 正常移除 |
| click（两次都失败） | 否（continue） | 跳过 | 继续执行 | ✅ 正常移除 |
| waitForLoadState（降级后再失败） | **是** | 终止 carryOutSteps | 不再执行 | ❌ 整组保留重试 |
| 其他原生动作 | 否（continue） | 跳过 | 继续执行 | ✅ 正常移除 |
| **scrapeList** | **否（catch 吞掉）** | 写空数组 `[]` | 不再执行 | ✅ 正常移除 |
| **scrapeSchema**（editor 模式） | **否（early return）** | 写空对象 `{}` | 不再执行 | ✅ 正常移除 |
| scrapeSchema（非 editor） | **是** | 终止 carryOutSteps | 不再执行 | ❌ 整组保留重试 |
| scrape/scrapeListAuto/screenshot/scroll | **是** | 终止 carryOutSteps | 不再执行 | ❌ 整组保留重试 |
| enqueueLinks（主流程失败） | **是** | 终止 carryOutSteps | 不再执行 | ❌ 整组保留重试 |
| flag | **否（暂停等 resume）** | 暂停后继续 | resume 后继续 | ✅ 正常移除 |
| script/crawl/search | **是（catch 后 re-throw）** | 终止 carryOutSteps | 不再执行 | ❌ 整组保留重试 |

> **整组动作（WhereWhatPair）被移除的唯一条件**：`carryOutSteps()` 正常 return（没有抛出异常），此时 runLoop 会执行 `workflowCopy.splice(actionId, 1)`。
>
> **不会触发整组重试的自定义动作**：scrapeList（外层 try-catch 吞掉 + 写空数组）、scrapeSchema（editor 模式 early return）、flag（暂停等待 resume，不抛错）。其余自定义动作失败后整组保留重试。

---

## 四、失败处理链路（对照代码事实）

### 4.1 单步失败 vs 整组移除：核心推进语义

这是最容易误解的部分。关键要区分两个层次：
1. **step 层**：单个动作（`what[]` 数组中的一项）
2. **pair 层**：整组动作（一个 `WhereWhatPair`，含多个 step）

```
runLoop 主循环 (while true)
    │
    ├─ 入口 waitForLoadState() 失败 → 关闭页面 + return（整个 runLoop 结束）
    │
    ├─ 匹配 actionId = workflowCopy.length - 1
    ├─ repeatCount 检查（同动作反复执行超限则 throw）
    │
    └─ try {
         carryOutSteps(page, action.what)  ← 执行整组 step
         usedActions.push(...)              ← 标记为已用
         workflowCopy.splice(actionId, 1)   ← ★ 整组从队列移除 ★
         loopIterations = 0                 ← 死循环计数器归零
       } catch (e) {
         log(e)                             ← 记录错误
         continue                           ← 进入下一轮循环
       }
```

**整组被移除的唯一条件**：`carryOutSteps()` 正常 return（没有抛出异常）。

---

### 4.2 carryOutSteps 内部：step 失败的逐类命运

`carryOutSteps()` 用 `for (const step of steps)` 顺序执行每个 step，不同动作失败后的行为差异很大：

```
for (const step of steps) {
    │
    ├─ 分支 A: 是自定义 wawActions
    │   │
    │   ├─ (A1) scrapeList → 外层 try-catch 吞掉 → 写空数组 [] 回调 → 正常 return
    │   │
    │   ├─ (A2) scrapeSchema (editor模式) → early return, 写空 {} 回调 → 正常 return
    │   │
    │   ├─ (A3) scrapeSchema (非editor) / scrape / scrapeListAuto / screenshot / scroll
    │   │   └─ 无 try-catch → 任何一步失败直接抛 → 终止 carryOutSteps
    │   │
    │   ├─ (A4) enqueueLinks → 主流程无 try-catch → 提取链接/关闭页面失败直接抛
    │   │   └─ (内部并发链接任务有独立 try-catch 隔离，不影响外层)
    │   │
    │   ├─ (A5) flag → Promise 等待 resume() → 暂停不抛错 → resume 后正常 resolve
    │   │   └─ (几乎不会失败，除非 emit 无监听器)
    │   │
    │   └─ (A6) script / crawl / search → 外层 try-catch 但内部重新 throw
    │       └─ 失败 → 记录日志 → 包装 Error 后抛出 → 终止 carryOutSteps
    │
    └─ 分支 B: 是 Playwright 原生动作 (page.xxx)
        │
        ├─ (B1) goto
        │   └─ try { ... } catch { 只 log 不抛 } → 继续下一个 step
        │
        ├─ (B2) waitForLoadState
        │   └─ try {
        │          第一次尝试（已降级为 domcontentloaded）
        │        } catch {
        │          再试一次 domcontentloaded  ← 没有再套 try-catch！
        │          第二次失败 → 异常抛出 → 终止 carryOutSteps
        │        }
        │
        ├─ (B3) click
        │   └─ try {
        │          第一次 click
        │        } catch {
        │          try { force:true 重试 } catch { continue → 下一个 step }
        │        }
        │
        └─ (B4) 其他原生动作 (type/fill/...)
            └─ try { ... } catch { continue → 下一个 step }
```

#### 失败命运对照表（严格对照代码事实）

**A. 原生 Playwright 动作：**

| 动作 | 失败后是否抛异常 | 当前 step | 同 Pair 后续 step | 整组是否移除 | 最终结局 |
|------|----------------|---------|----------------|------------|---------|
| goto | 否（吞掉） | 跳过 | 继续执行 | ✅ 是 | 整组正常完成 |
| click（两次都失败） | 否（continue） | 跳过 | 继续执行 | ✅ 是 | 整组正常完成 |
| waitForLoadState（降级后再失败） | **是** | 终止 | 不再执行 | ❌ 否 | 下一轮循环整组重试 |
| 其他原生动作（type/fill/...） | 否（continue） | 跳过 | 继续执行 | ✅ 是 | 整组正常完成 |

**B. 自定义 wawActions（逐类细分）：**

| 动作 | try-catch | 失败行为 | 整组是否移除 | 最终结局 |
|------|-----------|---------|------------|---------|
| **scrapeList** | 外层完整 try-catch | 写空数组 `[]`，回调，**正常 return** | ✅ 是 | 整组正常完成 |
| **scrapeSchema**（editor 模式） | 无 try-catch，但 early return | 写空对象 `{}`，回调 return | ✅ 是 | 整组正常完成 |
| scrapeSchema（非 editor） | 无 try-catch | **直接抛** | ❌ 否 | 下一轮整组重试 |
| scrape | 无 try-catch | **直接抛** | ❌ 否 | 下一轮整组重试 |
| scrapeListAuto | 无 try-catch | **直接抛** | ❌ 否 | 下一轮整组重试 |
| screenshot | 无 try-catch | **直接抛** | ❌ 否 | 下一轮整组重试 |
| scroll | 无 try-catch | **直接抛** | ❌ 否 | 下一轮整组重试 |
| enqueueLinks | 主流程无 / 内部并发有 | 主流程失败时**直接抛** | ❌ 否 | 下一轮整组重试 |
| flag | 无 try-catch | **直接抛**（Promise reject） | ❌ 否 | 下一轮整组重试 |
| **script** | 有，但 catch 内 re-throw | 重新包装错误后**抛** | ❌ 否 | 下一轮整组重试 |
| **crawl** | 有，但 catch 内 re-throw | 重新包装错误后**抛** | ❌ 否 | 下一轮整组重试 |
| **search** | 有，但 catch 内 re-throw | 重新包装错误后**抛** | ❌ 否 | 下一轮整组重试 |

> **scrapeList 是自定义动作中唯一"吞掉异常 + 写空结果正常返回"的动作**。它的设计意图是：即使抓取失败，也不要中断工作流，把空结果交给上层处理。

---

### 4.3 整组重试的终止条件（maxRepeats + 死循环保护）

当整组动作反复失败（如自定义动作或 waitForLoadState 持续失败）时，`runLoop` 会一轮接一轮地重试。但有两个保护机制会最终终止：

#### 保护机制 1：maxRepeats 最大重复次数

```typescript
repeatCount = action === lastAction ? repeatCount + 1 : 0;
if (this.options.maxRepeats && repeatCount > this.options.maxRepeats) {
  throw new Error(`Action ${failedAction} exceeded max retries (${maxRepeats})`);
}
```

- **触发条件**：同一个 `WhereWhatPair` 对象被连续匹配 `maxRepeats + 1` 次
- **动作比较**：`action === lastAction` 是对象引用比较（因为动作对象未被 splice，所以引用相同）
- **后果**：抛出 Error → 终止 runLoop → 异常冒泡到顶层 → Run 标记为 failed

#### 保护机制 2：MAX_LOOP_ITERATIONS 死循环保护

```typescript
if (++loopIterations > MAX_LOOP_ITERATIONS) {  // MAX_LOOP_ITERATIONS = 1000
  this.log('Maximum loop iterations reached, terminating to prevent infinite loop', Level.ERROR);
  cleanup();
  return;  // 静默返回，不抛异常
}
```

- **触发条件**：while 循环累计超过 1000 次
- **重置时机**：**只有**整组成功执行并 splice 后，才会 `loopIterations = 0`
- **后果**：直接 return，runLoop 静默结束，**不抛异常**

> 注意：即使所有 step 都"跳过式成功"（如全部是 goto/click 失败但被吞掉），只要 carryOutSteps 正常 return，整组就会被移除，loopIterations 也会归零，不会触发死循环保护。

---

### 4.4 五层异常捕获体系

```
┌──────────────────────────────────────────────────────────────┐
│ Layer 1: server/src/task-runner.ts 顶层 try-catch             │
│   - 捕获所有未被下层吞掉的异常                                 │
│   - 更新 Run.status = 'failed'                                │
│   - 发送失败 webhook / socket 通知                            │
│   - 触发 analytics 埋点                                       │
│   - 清理浏览器资源 destroyRemoteBrowser()                     │
└──────────────────────┬───────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────┐
│ Layer 2: server/src/workflow-management/classes/Interpreter.ts│
│   - serializableCallback: try-catch 吞掉异常                  │
│   - binaryCallback: try-catch 吞掉异常                        │
│   - flushPersistenceBuffer: 指数退避重试 (最多3次)             │
│   - InterpretRecording() 本身无 try-catch                     │
│     → interpreter.run() 抛出的异常直接向上冒泡                │
└──────────────────────┬───────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────┐
│ Layer 3: maxun-core/src/interpret.ts runLoop() 外层 try-catch │
│   - 包裹 carryOutSteps() 调用                                 │
│   - catch 后只记录日志 + continue，不抛出                      │
│   - 但 maxRepeats 超限时会主动 throw Error                    │
│   - MAX_LOOP_ITERATIONS 超限时直接 return 不抛                │
│   - 循环入口 waitForLoadState() 失败: 关闭页面 + return       │
└──────────────────────┬───────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────┐
│ Layer 4: carryOutSteps() 每个 step 的 try-catch               │
│   ├─ 原生动作:                                                │
│   │   ├─ goto: 吞掉 → 继续下一个 step                         │
│   │   ├─ waitForLoadState: 先降级重试 → 再失败则向上抛         │
│   │   ├─ click: force:true 重试 → 再失败 continue            │
│   │   └─ 其他原生: 失败 → continue 下一个 step                │
│   └─ 自定义 wawActions:                                       │
│       ├─ scrapeList: 外层 try-catch 吞掉 → 写空数组返回       │
│       ├─ scrapeSchema (editor): early return 写空 {}          │
│       ├─ scrapeSchema(非editor)/scrape/.../scroll: 无 try-catch │
│       │   → 失败直接向上抛                                    │
│       ├─ flag: Promise 等 resume → 暂停不抛错，正常 resolve    │
│       └─ script/crawl/search: try-catch 后 re-throw           │
│           → 失败包装错误后向上抛                              │
└──────────────────────┬───────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────┐
│ Layer 5: 特定动作内部重试                                      │
│   - handlePagination() 分页: MAX_RETRIES=3 + RETRY_DELAY=1s  │
│     → 分页选择器失败: 3次重试后剔除该选择器，不抛错            │
│     → 分页点击操作: 3次重试后放弃本页，不抛错                  │
│   - scrapeList/crawl/search: 单条记录失败不终止整体抓取        │
│   - enqueueLinks: 单个新页面失败 try-catch 吞掉                │
└──────────────────────────────────────────────────────────────┘
```

---

### 4.5 超时控制

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

### 4.6 用户中止流程

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

### 4.7 持久化重试机制

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

### 4.8 日志与调试通道

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
