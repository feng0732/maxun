# 工作流 JSON 模型的保存和恢复关系分析

## 概述

本文档分析 maxun 项目中工作流（Workflow）JSON 模型在**节点数据生成**、**序列化持久化**、**回放执行**三个阶段之间的流转关系和代码实现。所有描述严格对照实际代码行为。

---

## 一、核心数据模型

### 1.1 类型定义

文件位置：`maxun-core/src/types/workflow.ts`

```typescript
// 最顶层的工作流文件结构
export type WorkflowFile = {
  meta?: MetaData,           // 元数据（名称、描述等）
  workflow: Workflow         // 工作流主体
};

// 工作流是 WhereWhatPair 的数组
export type Workflow = WhereWhatPair[];

// 每个工作流节点（核心数据结构）
export interface WhereWhatPair {
  id?: string;               // 节点唯一标识
  where: Where;              // 执行条件（何时执行）
  what: What[];              // 动作列表（执行什么）
}

// 条件定义 - 描述"在什么状态下执行"
export type Where = {
  url?: string | { $regex: string };    // URL 匹配
  cookies?: Record<string, string>;     // Cookie 匹配
  selectors?: string[];                 // 页面元素选择器
  $and?: Where[];                       // 逻辑与
  $or?: Where[];                        // 逻辑或
  $not?: Where;                         // 逻辑非
  $after?: string;                      // 某动作之后
  $before?: string;                     // 某动作之前
};

// 动作定义 - 描述"具体执行什么操作"
export type What = {
  action: string;           // 动作名称（如 click, fill, goto, scrape）
  args?: any[];             // 动作参数
  name?: string;            // 动作名称（用户自定义）
  actionId?: string;        // 动作唯一标识
};
```

### 1.2 数据库存储模型

文件位置：`server/src/models/Robot.ts`

```typescript
// Robot 模型中工作流的持久化字段
recording_meta: {
  type: DataTypes.JSONB,     // PostgreSQL JSONB 类型
  allowNull: false,
},
recording: {
  type: DataTypes.JSONB,     // 存储完整的 WorkflowFile
  allowNull: false,
},
```

**关键设计**：使用 PostgreSQL 的 `JSONB` 类型直接存储完整的 JSON 结构，无需额外的表关联，简化了读写操作。

---

## 二、代码执行顺序：从录制到回放的完整流程

### 阶段 1：节点数据生成（录制阶段）

#### 输入来源：用户交互事件

文件位置：`server/src/browser-management/inputHandlers.ts`

用户在前端浏览器中的所有操作通过 Socket.IO 事件发送到后端：

```
用户操作 → Socket 事件 → 输入处理器 → Generator 生成节点
```

| 事件名称 | 对应动作 | 生成的节点类型 |
|---------|---------|--------------|
| `dom:click` | 点击元素 | `{ action: 'click', args: [selector] }` |
| `dom:keypress` | 键盘输入 | `{ action: 'press', args: [selector, encryptedKey] }` |
| `input:url` | URL 跳转 | `{ action: 'goto', args: [url] }` |
| `input:date` | 日期选择 | `{ action: 'fill', args: [selector, value] }` |
| `input:dropdown` | 下拉选择 | `{ action: 'selectOption', args: [selector, value] }` |
| `action` | 自定义动作 | 如 scrape, scrapeList, screenshot 等 |

#### 节点生成器：WorkflowGenerator

文件位置：`server/src/workflow-management/classes/Generator.ts`

核心方法 `addPairToWorkflowAndNotifyClient`（第 295 行附近）：

```typescript
private addPairToWorkflowAndNotifyClient = async (pair: WhereWhatPair, page: Page) => {
  // 1. 检查是否有相同 selector 的节点，合并动作
  if (pair.where.selectors && pair.where.selectors[0]) {
    const match = selectorAlreadyInWorkflow(pair.where.selectors[0], this.workflowRecord.workflow);
    if (match) {
      // 合并到已有节点的 what 数组
      this.workflowRecord.workflow[matchedIndex].what =
        this.workflowRecord.workflow[matchedIndex].what.concat(pair.what);
      matched = true;
    }
  }

  // 2. 检查是否有"遮蔽"关系（同URL同页面的节点）
  if (!matched) {
    const handled = await this.handleOverShadowing(pair, page, this.generatedData.lastIndex || 0);
    if (!handled) {
      // 添加 waitForLoadState 动作确保页面稳定
      pair.what.push({
        action: 'waitForLoadState',
        args: ['networkidle'],
      });
      // 插入到 workflow 数组
      this.workflowRecord.workflow.splice(this.generatedData.lastIndex || 0, 0, pair);
    }
  }

  // 3. 通知前端更新
  this.socket.emit('workflow', this.workflowRecord);
};
```

**内存状态**：生成的节点保存在 `Generator.workflowRecord` 中，类型为 `WorkflowFile`。

#### 工作流优化：输入状态合并

文件位置：`server/src/workflow-management/classes/Generator.ts`（第 1417 行 `optimizeWorkflow`）

`optimizeWorkflow` 方法在保存前执行，将分散的键盘输入优化为批量输入：

```typescript
private optimizeWorkflow = (workflow: WorkflowFile) => {
  const inputStates = new Map<string, InputState>();

  // 遍历所有动作，收集输入状态
  for (const pair of workflow.workflow) {
    let currentIndex = 0;
    while (currentIndex < pair.what.length) {
      const condition = pair.what[currentIndex];

      // 1. 处理光标位置（click 带 cursorIndex）
      if (condition.action === 'click' && condition.args?.[2]?.cursorIndex !== undefined) {
        const selector = condition.args[0];
        const cursorIndex = condition.args[2].cursorIndex;
        let state = inputStates.get(selector) || {
          selector,
          value: '',
          type: 'text',
          cursorPosition: -1
        };
        state.cursorPosition = cursorIndex;
        inputStates.set(selector, state);
        pair.what.splice(currentIndex, 1);  // 删除 click 动作
        continue;
      }

      // 2. 处理按键输入（press 动作）
      if (condition.action === 'press' && condition.args?.[1]) {
        const [selector, encryptedKey, type] = condition.args;
        const key = decrypt(encryptedKey);
        let state = inputStates.get(selector) || {
          selector,
          value: '',
          type: type || 'text',
          cursorPosition: -1
        };

        // 根据光标位置构建完整输入值
        if (key.length === 1) {
          // 普通字符输入
          if (state.cursorPosition === -1) {
            state.value += key;
          } else {
            state.value =
              state.value.slice(0, state.cursorPosition) +
              key +
              state.value.slice(state.cursorPosition);
            state.cursorPosition++;
          }
        } else if (key === 'Backspace') {
          // 退格键处理
          if (state.cursorPosition > 0) {
            state.value =
              state.value.slice(0, state.cursorPosition - 1) +
              state.value.slice(state.cursorPosition);
            state.cursorPosition--;
          }
        }
        // ... Delete 等其他按键处理

        inputStates.set(selector, state);
        pair.what.splice(currentIndex, 1);  // 删除 press 动作
        continue;
      }

      currentIndex++;
    }
  }

  // 3. 将合并后的输入转化为 type 动作
  for (const [selector, state] of inputStates.entries()) {
    if (state.value) {
      for (let i = workflow.workflow.length - 1; i >= 0; i--) {
        const pair = workflow.workflow[i];
        pair.what.push({
          action: 'type',
          args: [selector, encrypt(state.value), state.type]
        }, {
          action: 'waitForLoadState',
          args: ['networkidle']
        });
        break;
      }
    }
  }

  return workflow;
};
```

**优化效果**：将 N 次 `press` 按键动作优化为 1 次 `type` 批量输入动作，显著提高回放效率。

---

### 阶段 2：序列化与持久化（保存阶段）

#### 保存工作流到数据库

文件位置：`server/src/workflow-management/classes/Generator.ts`（第 1044 行 `saveNewWorkflow`）

```typescript
public saveNewWorkflow = async (fileName: string, userId: number, isLogin: boolean, robotId?: string) => {
  // 1. 优化工作流（如上述）
  const recording = this.optimizeWorkflow(this.workflowRecord);

  // 2. URL 标准化
  const normalizedRecording = {
    ...recording,
    workflow: normalizeWorkflowUrls(recording.workflow),
  };

  // 3. 构建元数据
  this.recordingMeta = {
    name: trimmedFileName,
    id: uuid(),
    createdAt: this.recordingMeta.createdAt || new Date().toLocaleString(),
    pairs: normalizedRecording.workflow.length,
    updatedAt: new Date().toLocaleString(),
    params: this.getParams() || [],
    type: this.recordingMeta.type || 'extract',
    isLogin: isLogin,
  };

  // 4. 保存到数据库（Sequelize 自动序列化 JSONB）
  const robot = await Robot.create({
    userId,
    recording_meta: this.recordingMeta,
    recording: normalizedRecording,  // 直接传入对象，Sequelize 自动 JSON 序列化
  });
};
```

**序列化发生处**：
1. **ORM 层自动序列化**：Robot 模型的 `recording` 和 `recording_meta` 字段定义为 `JSONB` 类型，Sequelize 在保存时自动调用 `JSON.stringify()`。
2. **显式深拷贝**：在 `optimizeWorkflow` 和 `processWorkflow` 中使用 `JSON.parse(JSON.stringify(obj))` 进行深拷贝，确保不修改原始对象。

#### 敏感数据加密

在录制阶段，键盘输入的值会被加密存储：

文件位置：`server/src/workflow-management/classes/Generator.ts`

```typescript
const pair: WhereWhatPair = {
  where: {
    url: this.getBestUrl(url),
    selectors: [selector]
  },
  what: [{
    action: 'press',
    args: [selector, encrypt(key), inputType || 'text'],  // key 被 encrypt() 加密
  }],
};
```

---

### 阶段 3：反序列化与恢复（加载阶段）

#### 从数据库加载工作流

文件位置：`server/src/routes/workflow.ts`

```typescript
// PUT /workflow/:browserId/:id
router.put('/:browserId/:id', requireSignIn, async (req: AuthenticatedRequest, res) => {
  // 1. 从数据库读取（Sequelize 自动反序列化 JSONB）
  const robot = await Robot.findOne({
    where: {
      'recording_meta.id': req.params.id,
      userId: req.user.id,
    },
    raw: true
  });

  const { recording, recording_meta } = robot;  // recording 已是 WorkflowFile 对象

  // 2. 恢复到 Generator 的内存状态
  if (recording && recording.workflow) {
    browser.generator.updateWorkflowFile(recording, recording_meta);
    const workflowFile = browser.generator.getWorkflowFile();
    return res.send(workflowFile);
  }
});
```

**反序列化发生处**：
1. **ORM 层自动反序列化**：从数据库读取 `JSONB` 字段时，Sequelize 自动调用 `JSON.parse()` 转换为 JavaScript 对象。

#### 更新生成器状态

文件位置：`server/src/workflow-management/classes/Generator.ts`

```typescript
public updateWorkflowFile = (workflowFile: WorkflowFile, meta: MetaData) => {
  this.recordingMeta = meta;
  const params = this.checkWorkflowForParams(workflowFile);
  if (params) {
    this.recordingMeta.params = params;
  }
  this.workflowRecord = workflowFile;  // 直接赋值恢复
};
```

---

### 阶段 4：预处理与初始化（执行前准备）

#### 工作流预处理

文件位置：`maxun-core/src/preprocessor.ts`

`Preprocessor.initWorkflow` 方法在执行前调用：

```typescript
static initWorkflow(workflow: Workflow, params?: ParamType): Workflow {
  // 1. 深拷贝（JSON 序列化 + 反序列化）
  let workflowCopy = JSON.parse(JSON.stringify(workflow));  // 关键序列化点

  // 2. 初始化参数：替换 { $param: "paramName" } 为实际值
  if (params) {
    workflowCopy = initSpecialRecurse(
      workflowCopy,
      '$param',
      (paramName) => {
        if (params && params[paramName]) {
          return params[paramName];
        }
        throw new SyntaxError(`Unspecified parameter found ${paramName}.`);
      },
    );
  }

  // 3. 初始化正则：替换 { $regex: "pattern" } 为 RegExp 对象
  workflowCopy = initSpecialRecurse(
    workflowCopy,
    '$regex',
    (regex) => new RegExp(regex),
  );

  return workflowCopy;
}
```

**关键转换**：
- JSON 中的 `{ "$regex": "^https://example.com" }` → JavaScript `RegExp` 对象
- JSON 中的 `{ "$param": "username" }` → 实际参数值

#### 输入解密处理

文件位置：`server/src/workflow-management/classes/Interpreter.ts`

```typescript
function processWorkflow(workflow: WorkflowFile, checkLimit: boolean = false): WorkflowFile {
  // 1. 深拷贝
  const processedWorkflow = JSON.parse(JSON.stringify(workflow)) as WorkflowFile;

  // 2. 遍历所有动作，解密敏感输入
  processedWorkflow.workflow.forEach((pair) => {
    pair.what.forEach((action) => {
      // 解密 type 和 press 动作中的加密值
      if ((action.action === 'type' || action.action === 'press') &&
          Array.isArray(action.args) && action.args.length > 1) {
        try {
          const encryptedValue = action.args[1];
          if (typeof encryptedValue === 'string') {
            const decryptedValue = decrypt(encryptedValue);
            action.args[1] = decryptedValue;  // 替换为明文
          }
        } catch (error) {
          // ... 错误处理
        }
      }
    });
  });

  return processedWorkflow;
}
```

---

### 阶段 5：回放执行（运行阶段）

#### 核心执行循环

文件位置：`maxun-core/src/interpret.ts`

```typescript
public async run(page: Page, params?: ParamType): Promise<void> {
  // 1. 预处理：初始化参数和正则
  this.initializedWorkflow = Preprocessor.initWorkflow(this.workflow, params);

  // 2. 加载浏览器端脚本
  await this.ensureScriptsLoaded(page);

  // 3. 启动主执行循环
  this.concurrency.addJob(() => this.runLoop(page, this.initializedWorkflow!));
  await this.concurrency.waitForCompletion();
}
```

#### runLoop 主循环

文件位置：`maxun-core/src/interpret.ts`

```typescript
private async runLoop(p: Page, workflow: Workflow) {
  // 1. 深拷贝工作流（不修改原始数据）
  let workflowCopy: Workflow = JSON.parse(JSON.stringify(workflow));  // 关键序列化点

  // 2. 移除特殊选择器标记
  workflowCopy = this.removeSpecialSelectors(workflowCopy);

  const usedActions: string[] = [];

  while (true) {
    // 3. 检查终止条件
    if (p.isClosed() || !this.stopper || workflowCopy.length === 0) {
      return;
    }

    // 4. 获取当前页面状态（用于匹配 where 条件）
    // const pageState = await this.getState(p, workflowCopy, selectors);

    // 5. 匹配可执行的动作（原设计：applicable 检查，现简化为顺序执行）
    // const actionId = workflowCopy.findIndex((step) => {
    //   return this.applicable(step.where, pageState, usedActions);
    // });

    // 当前简化实现：从数组末尾开始执行
    const actionId = workflowCopy.length - 1;
    const action = workflowCopy[actionId];

    if (action) {
      // 6. 执行动作
      await this.carryOutSteps(p, action.what, workflowCopy);
      usedActions.push(action.id ?? 'undefined');

      // 7. 从待执行列表中移除已执行的动作
      workflowCopy.splice(actionId, 1);
    } else {
      return;
    }
  }
}
```

#### 动作执行器

文件位置：`maxun-core/src/interpret.ts`

`carryOutSteps` 方法执行具体的动作：

```typescript
private async carryOutSteps(page: Page, steps: What[], currentWorkflow?: Workflow) {
  // 自定义动作映射（如 scrape, screenshot, scroll 等）
  const wawActions: Record<CustomFunctions, (...args: any[]) => void> = {
    screenshot: async (params, nameOverride) => { /* ... */ },
    scrape: async (selector) => { /* ... */ },
    scrapeSchema: async (schema, actionName) => { /* ... */ },
    scrapeList: async (config, actionName) => { /* ... */ },
    scroll: async (pages) => { /* ... */ },
    // ... 更多自定义动作
  };

  for (const step of steps) {
    if (step.action in wawActions) {
      // 执行自定义动作
      const params = !step.args || Array.isArray(step.args) ? step.args : [step.args];
      await wawActions[step.action as CustomFunctions](...(params ?? []));
    } else {
      // 执行 Playwright 原生方法（如 click, fill, goto 等）
      const levels = String(step.action).split('.');
      const methodName = levels[levels.length - 1];
      let invokee: any = page;
      for (const level of levels.splice(0, levels.length - 1)) {
        invokee = invokee[level];
      }
      await invokee[methodName](...(step.args ?? []));
    }
  }
}
```

---

## 三、节点数据、序列化和回放输入的关系图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              录制阶段                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  用户交互（点击、输入）                                                  │
│         │                                                               │
│         ▼                                                               │
│  Socket 事件 (dom:click, dom:keypress, ...)                              │
│         │                                                               │
│         ▼                                                               │
│  inputHandlers.ts 接收事件                                               │
│         │                                                               │
│         ▼                                                               │
│  Generator 生成 WhereWhatPair 节点                                       │
│         │  { id, where: {url, selectors}, what: [{action, args}] }       │
│         ▼                                                               │
│  内存状态: Generator.workflowRecord (WorkflowFile)                       │
│         │                                                               │
│         ▼                                                               │
│  optimizeWorkflow() 优化输入：                                           │
│    - 收集 click (cursorIndex) 记录光标位置                               │
│    - 收集 press 按键构建完整输入值                                       │
│    - 替换为 type 批量输入动作                                            │
│         │                                                               │
│         ▼                                                               │
│  内存状态: 优化后的 WorkflowFile                                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            序列化保存阶段                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Generator.saveNewWorkflow()                                            │
│         │                                                               │
│         ├─► normalizeWorkflowUrls() - URL 规范化                        │
│         │                                                               │
│         ▼                                                               │
│  Robot.create({                                                         │
│    recording_meta: metaData,     /* JSONB 字段 */                       │
│    recording: workflowFile      /* JSONB 字段 */                        │
│  })                                                                     │
│         │                                                               │
│         ▼  Sequelize ORM 自动序列化                                     │
│  PostgreSQL JSONB 存储                                                  │
│    - recording_meta: {"name": "...", "id": "...", ...}                  │
│    - recording: {"workflow": [{"id": "...", "where": {...}, "what": [...]}]}  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            反序列化恢复阶段                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Robot.findOne() 读取数据库                                              │
│         │  Sequelize 自动反序列化 JSONB                                  │
│         ▼                                                               │
│  JavaScript 对象: { recording_meta, recording }                          │
│         │                                                               │
│         ▼                                                               │
│  Generator.updateWorkflowFile(recording, recording_meta)                 │
│         │                                                               │
│         ▼                                                               │
│  内存状态恢复: Generator.workflowRecord = recording                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          预处理初始化阶段                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  processWorkflow(workflow)                                              │
│         │                                                               │
│         ├─► JSON.parse(JSON.stringify()) 深拷贝                         │
│         │                                                               │
│         └─► 解密 type/press 动作中的加密值                               │
│                                                                         │
│         ▼                                                               │
│  Preprocessor.initWorkflow(workflow, params)                             │
│         │                                                               │
│         ├─► JSON.parse(JSON.stringify()) 深拷贝                         │
│         │                                                               │
│         ├─► 替换 $param 为实际参数值                                     │
│         │    { $param: "username" } → "user123"                         │
│         │                                                               │
│         └─► 替换 $regex 为 RegExp 对象                                   │
│              { $regex: "^https://" } → /^https:\/\//                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            回放执行阶段                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Interpreter.run(page, params)                                          │
│         │                                                               │
│         ▼                                                               │
│  runLoop(page, initializedWorkflow)                                     │
│         │                                                               │
│         ├─► JSON.parse(JSON.stringify()) 深拷贝工作流                   │
│         │                                                               │
│         ├─► 循环执行：                                                  │
│         │     while (workflowCopy.length > 0) {                         │
│         │                                                               │
│         │       // 原设计：获取页面状态 + 条件匹配                        │
│         │       // const pageState = getState(page)                     │
│         │       // const match = applicable(step.where, pageState)       │
│         │                                                               │
│         │       // 当前实现：顺序执行                                    │
│         │       const action = workflowCopy[workflowCopy.length - 1]     │
│         │                                                               │
│         │       // 执行动作                                              │
│         │       carryOutSteps(page, action.what)                         │
│         │         │                                                     │
│         │         ├─► 自定义动作: wawActions[action]                    │
│         │         │   (scrape, screenshot, scroll, etc.)                │
│         │         │                                                     │
│         │         └─► Playwright 方法: page[action](...args)            │
│         │             (click, fill, goto, etc.)                         │
│         │                                                               │
│         │       // 移除已执行的动作                                      │
│         │       workflowCopy.splice(actionId, 1)                        │
│         │     }                                                         │
│         │                                                               │
│         └─► 完成，返回结果                                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 四、关键序列化点汇总

### 4.1 JSON 序列化/反序列化位置

| 位置 | 文件 | 代码 | 用途 |
|-----|------|------|------|
| 1 | `maxun-core/src/preprocessor.ts` | `JSON.parse(JSON.stringify(workflow))` | 预处理时深拷贝，避免修改原始工作流 |
| 2 | `maxun-core/src/interpret.ts` | `JSON.parse(JSON.stringify(workflow))` | 执行循环时深拷贝，维护待执行队列 |
| 3 | `server/src/workflow-management/classes/Interpreter.ts` | `JSON.parse(JSON.stringify(workflow))` | 输入解密前深拷贝 |
| 4 | `server/src/models/Robot.ts` | Sequelize JSONB 自动处理 | 数据库持久化和读取 |

### 4.2 特殊格式转换

| 转换类型 | 存储形式（JSON） | 运行时形式（JS） | 处理位置 |
|---------|-----------------|-----------------|---------|
| 正则表达式 | `{ "$regex": "^https://example.com" }` | `RegExp` 对象 | `maxun-core/src/preprocessor.ts` |
| 参数占位符 | `{ "$param": "username" }` | 实际参数值 | `maxun-core/src/preprocessor.ts` |
| 敏感输入 | 加密字符串（AES-256-CBC） | 明文字符串 | `server/src/workflow-management/classes/Interpreter.ts` |

---

## 五、设计特点与技术权衡

### 5.1 优点

1. **JSON 优先设计**：全程使用 JSON 作为数据交换格式，简化了前后端通信和数据库存储
2. **声明式工作流**：`where` 条件 + `what` 动作的声明式设计，理论上支持复杂的状态匹配
3. **优化的输入处理**：录制时逐键记录，保存时合并为批量输入，兼顾准确性和执行效率
4. **安全加密**：敏感输入在持久化前加密，保护用户隐私
5. **深拷贝保护**：多个环节使用 `JSON.parse(JSON.stringify())` 确保数据不可变性

### 5.2 可改进点

1. **条件匹配简化**：原设计的 `applicable()` 状态匹配机制被简化为顺序执行，可能影响复杂场景的灵活性
2. **序列化性能**：多次 `JSON.parse(JSON.stringify())` 在大工作流时可能有性能影响
3. **错误恢复**：执行中断后没有完善的检查点恢复机制
4. **类型安全**：`any[]` 类型的 args 缺乏严格的类型检查

---

## 六、关键代码文件索引（GUI 录制路径）

| 文件 | 主要职责 |
|-----|---------|
| `maxun-core/src/types/workflow.ts` | 核心类型定义（WhereWhatPair, Workflow 等） |
| `maxun-core/src/preprocessor.ts` | 工作流验证、参数初始化、正则转换 |
| `maxun-core/src/interpret.ts` | 工作流解释执行核心（run, runLoop, carryOutSteps） |
| `server/src/workflow-management/classes/Generator.ts` | 录制时节点生成、优化、保存 |
| `server/src/workflow-management/classes/Interpreter.ts` | 服务端解释器包装、输入解密 |
| `server/src/browser-management/inputHandlers.ts` | 用户输入事件路由 |
| `server/src/models/Robot.ts` | 数据库模型定义 |
| `server/src/routes/workflow.ts` | 工作流 REST API |

---

## 七、总结（GUI 录制路径）

工作流 JSON 模型的流转遵循以下核心原则：

1. **录制生成**：用户交互 → 细粒度 `WhereWhatPair` 节点 → 内存状态
2. **优化合并**：保存前将分散的键盘输入优化为批量 `type` 动作
3. **持久化**：通过 PostgreSQL JSONB 直接存储完整结构，ORM 处理序列化
4. **恢复加载**：从数据库读取后直接恢复为内存对象
5. **预处理**：执行前转换特殊格式（`$regex` → RegExp，`$param` → 实际值，密文 → 明文）
6. **回放执行**：深拷贝后按顺序执行每个节点的动作，从待执行队列中移除已完成项

整个流程中，**序列化**是连接内存对象和持久化存储的桥梁，**节点数据**是贯穿始终的核心载体，**回放输入**则是将录制的用户操作还原为浏览器自动化动作的最终目标。

---

## 八、SDK 简化工作流：从输入到保存模型的完整代码链路

除了上述的 GUI 录制方式外，Maxun 还提供了 SDK 接口允许开发者通过编程方式提交简化工作流。这一路径与 GUI 录制在**输入生成阶段**完全不同，但在**序列化保存**和后续阶段共用相同的基础设施。

### 8.1 API 入口：POST /api/sdk/robots

#### 8.1.1 路由挂载关系（为什么带 /api 前缀）

文件位置：`server/src/server.ts`

Maxun 后端使用 Express，路由分两套体系挂载：

```
Express app
    │
    ├── 直接挂载的路由（无前缀）：
    │     ├── /webhook     ← server/src/routes/webhook.ts
    │     ├── /record      ← server/src/routes/record.ts
    │     ├── /workflow    ← server/src/routes/workflow.ts   （GUI 录制用）
    │     ├── /storage     ← server/src/routes/storage.ts
    │     ├── /auth        ← server/src/routes/auth.ts
    │     └── /proxy       ← server/src/routes/proxy.ts
    │
    └── /api 前缀下的路由（api/ 目录统一挂载）：
          └── /sdk/...     ← server/src/api/sdk.ts
```

挂载代码（`server/src/server.ts` 第 127-135 行）：

```typescript
// 遍历 api/ 目录下的所有文件，统一挂载到 /api 前缀下
readdirSync(path.join(__dirname, 'api')).forEach((r) => {
  const route = require(path.join(__dirname, 'api', r));
  const router = route.default || route;
  if (typeof router === 'function') {
    app.use('/api', router);   // 关键：所有 api/ 目录的路由都加 /api 前缀
  }
});
```

**为什么带 /api 前缀**：
- **分层设计**：`api/` 目录下的路由被设计为对外的 API 接口（如 SDK），统一加 `/api` 前缀与内部 GUI 路由区分
- **自动挂载**：通过 `readdirSync` 遍历 `api/` 目录自动挂载，新增文件无需手动改 `server.ts`
- **命名空间隔离**：`/api/*` 对外，`/workflow`、`/auth` 等对内（GUI 前端用），职责明确

因此，SDK 路由在 `sdk.ts` 中定义的是 `/sdk/robots`，经过 `app.use('/api', router)` 挂载后，**外部访问的完整路径是 `/api/sdk/robots`**。

#### 8.1.2 处理流程

文件位置：`server/src/api/sdk.ts`

SDK 用户提交的是"简化格式"的工作流，API 路由处理以下核心步骤：

```
SDK 请求体（简化格式）
    │
    ▼
1. 结构校验（meta + workflow 必须存在）
    │
    ├── 类型为 'scrape' 时：跳过选择器补全，仅验证 URL
    │
    └── 其他类型时：执行 WorkflowEnricher.enrichWorkflow()
          │
          ├── 选择器补全 + 验证
          ├── 输入类型检测 + 加密
          └── scrapeSchema/scrapeList 字段自动探测
    │
    ▼
2. normalizeWorkflowUrls() - URL 规范化
    │
    ▼
3. 构建 recording_meta（元数据）
    │
    ▼
4. Robot.create() - 写库保存
```

核心代码：

```typescript
// 步骤 1：根据类型决定是否需要选择器补全
if (type === 'scrape') {
  // scrape 类型：无需补全，直接从 meta.url 获取入口 URL
  enrichedWorkflow = [];
  extractedUrl = normalizeRobotUrl((workflowFile.meta as any).url);
} else {
  // 其他类型（extract 等）：执行完整的选择器补全流程
  const enrichResult = await WorkflowEnricher.enrichWorkflow(
    workflowFile.workflow,  // 用户提交的简化工作流
    user.id
  );
  if (!enrichResult.success) {
    return res.status(400).json({ error: "Workflow validation failed", details: enrichResult.errors });
  }
  enrichedWorkflow = normalizeWorkflowUrls(enrichResult.workflow!);
  extractedUrl = enrichResult.url ? normalizeRobotUrl(enrichResult.url) : undefined;
}

// 步骤 2-3：构建元数据
const robotMeta = {
  name: workflowFile.meta.name,
  id: metaId,
  createdAt: new Date().toISOString(),
  pairs: enrichedWorkflow.length,
  type,
  url: extractedUrl,
  formats: normalizedFormats,
  // ... 其他 LLM 相关字段
};

// 步骤 4：写库保存（与 GUI 录制路径完全相同）
const robot = await Robot.create({
  id: robotId,
  userId: user.id,
  recording_meta: robotMeta,
  recording: {
    workflow: normalizeWorkflowUrls(enrichedWorkflow)  // 再次规范化 URL
  }
});
```

---

### 8.2 选择器补全：WorkflowEnricher.enrichWorkflow

文件位置：`server/src/sdk/workflowEnricher.ts`

这是 SDK 路径区别于 GUI 录制的**核心差异化步骤**。它启动一个真实浏览器来验证和补全用户提交的简化选择器。

#### 整体执行流程

```
simplifiedWorkflow（用户提交的简化节点数组）
    │
    ▼
1. 提取入口 URL（遍历 where.url 字段）
    │
    ▼
2. 启动远程浏览器 createRemoteBrowserForValidation()
    │
    ▼
3. 初始化 SelectorValidator → page.goto(url) 打开页面
    │
    ▼
4. 遍历每个简化节点 step：
    │
    ├─► 遍历每个动作 action：
    │     │
    │     ├── type 动作：输入检测 + 加密
    │     │     ├── selector 加入 selectors 集合
    │     │     ├── 值加密 encrypt(value)
    │     │     └── 自动检测 inputType（或使用用户提供的）
    │     │
    │     ├── scrapeSchema 动作：字段选择器补全
    │     │     ├── 对每个字段调用 validateSchemaFields()
    │     │     └── 返回 { tag, isShadow, selector, attribute }
    │     │
    │     ├── scrapeList 动作：列表自动探测
    │     │     ├── autoDetectListFields() 自动识别子字段
    │     │     └── autoDetectPagination() 自动识别分页
    │     │
    │     └── 其他动作（click 等）：原样保留
    │
    └─► 将收集的 selectors 写入 enrichedStep.where.selectors
    │
    ▼
5. 关闭浏览器，返回 enrichedWorkflow
```

#### 8.2.1 type 动作：输入检测 + 加密

文件位置：`server/src/sdk/workflowEnricher.ts`

```typescript
if (action.action === 'type') {
  const [selector, value, providedInputType] = action.args;

  selectors.push(selector);  // 收集选择器到 where.selectors

  // 关键：值加密（与 GUI 录制路径使用相同的 encrypt 函数）
  const encryptedValue = encrypt(value);

  if (!providedInputType) {
    // 自动检测输入框类型
    const inputType = await validator.detectInputType(selector);
    enrichedStep.what.push({
      ...action,
      args: [selector, encryptedValue, inputType]  // 补全第三参数
    });
  } else {
    enrichedStep.what.push({
      ...action,
      args: [selector, encryptedValue, providedInputType]
    });
  }

  // 自动追加 waitForLoadState 动作（与 GUI 录制路径一致）
  enrichedStep.what.push({ action: 'waitForLoadState', args: ['networkidle'] });
}
```

**与 GUI 录制对比**：
- GUI 录制：逐键 `press` → 优化为 `type`（带加密值）
- SDK 路径：直接 `type` → 立即加密（跳过逐键录制和优化阶段）

最终存储格式**完全一致**，均为 `{ action: 'type', args: [selector, encryptedValue, inputType] }`。

**detectInputType 返回值（代码确认）**：

文件位置：`server/src/sdk/selectorValidator.ts`

```typescript
const inputType = await element.evaluate((el) => {
  if (el instanceof HTMLInputElement) {
    return el.type || 'text';
  }
  if (el instanceof HTMLTextAreaElement) {
    return 'textarea';
  }
  if (el instanceof HTMLSelectElement) {
    return 'select';
  }
  return 'text';
});
```

即：
- `<input>` 元素返回其 `type` 属性（如 `text`/`password`/`email`/`number`/`search` 等），缺失时返回 `'text'`
- `<textarea>` 返回 `'textarea'`
- `<select>` 返回 `'select'`
- 其他元素返回 `'text'`

#### 8.2.2 scrapeSchema 动作：字段选择器补全

文件位置：`server/src/sdk/workflowEnricher.ts`

用户可能只提交了简单的字段名-选择器映射：
```json
{
  "action": "scrapeSchema",
  "args": [{
    "title": "h1.product-title",
    "price": "//div[@class='price']"
  }]
}
```

经过 `validateSchemaFields()` 补全后，每个字段获得额外的元数据：
```json
{
  "action": "scrapeSchema",
  "actionId": "text-uuid",
  "args": [{
    "title": {
      "tag": "H1",
      "isShadow": false,
      "selector": "h1.product-title",
      "attribute": "innerText"
    },
    "price": {
      "tag": "DIV",
      "isShadow": false,
      "selector": "//div[@class='price']",
      "attribute": "innerText"
    }
  }]
}
```

验证由 `server/src/sdk/selectorValidator.ts` 的 `validateSelector()` 完成，包括：
- 检查选择器是否匹配到元素（`count() !== 0`）
- 获取元素的 `tagName`（H1, DIV, A 等）
- 检查元素是否在 Shadow DOM 中（`isShadow`）
- 确认属性类型（默认 `innerText`，可为 `href`, `src` 等）

#### 8.2.3 scrapeList 动作：列表自动探测

文件位置：`server/src/sdk/workflowEnricher.ts`

用户可能只提交了列表容器选择器（itemSelector），系统自动：
1. **autoDetectListFields()**：遍历列表项的子元素，自动识别标题、价格、图片、链接等常见字段
2. **autoDetectPagination()**：检测是否存在下一页按钮，识别分页类型（`none`/`click`/`scroll`/`url_param`）

补全后的 `scrapeList` 动作：
```json
{
  "action": "scrapeList",
  "actionId": "list-uuid",
  "args": [{
    "fields": { "Title": {...}, "Price": {...}, "Image": {...} },
    "listSelector": "div.product-card",
    "pagination": { "type": "click", "selector": "button.next" },
    "limit": 100
  }]
}
```

---

### 8.3 输入加密：AES-256-CBC 对称加密

文件位置：`server/src/utils/auth.ts`

SDK 路径与 GUI 录制路径使用**完全相同的加密函数**，确保回放时解密逻辑的一致性。

#### encrypt 实现（代码确认）

```typescript
export const encrypt = (text: string): string => {
  const ivLength = 16;
  const iv = crypto.randomBytes(ivLength);           // 随机 16 字节 IV
  const algorithm = 'aes-256-cbc';

  // 从环境变量 ENCRYPTION_KEY 获取密钥（64 位十六进制 = 256 位）
  let key = getEnvVariable('ENCRYPTION_KEY');
  if (!key || key.length !== 64) {
    key = crypto.randomBytes(32).toString('hex');   // 生成临时密钥（警告）
  }
  const keyBuffer = Buffer.from(key, 'hex');

  const cipher = crypto.createCipheriv(algorithm, keyBuffer, iv);
  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');

  // 输出格式: "IV(hex):密文(hex)" 拼接
  return `${iv.toString('hex')}:${encrypted}`;
};
```

**加密特征（代码确认）**：
- **算法**：AES-256-CBC（256 位密钥，密码分组链接模式）
- **IV 处理**：每次加密生成随机 16 字节 IV，拼接在密文前（相同明文 → 不同密文）
- **密钥来源**：环境变量 `ENCRYPTION_KEY`，要求 64 个十六进制字符（对应 256 位）；缺失或长度不对时临时生成并打印警告
- **输出格式**：`ivHex:ciphertextHex`，用冒号分隔两部分（每部分均为十六进制字符串）

#### decrypt 实现（回放时调用，代码确认）

```typescript
export const decrypt = (encryptedText: string): string => {
  const [iv, encrypted] = encryptedText.split(':');  // 按冒号拆分 IV 和密文
  const algorithm = "aes-256-cbc";

  let key = getEnvVariable('ENCRYPTION_KEY');
  if (!key || key.length !== 64) {
    key = crypto.randomBytes(32).toString('hex');
  }
  const keyBuffer = Buffer.from(key, 'hex');

  const decipher = crypto.createDecipheriv(algorithm, keyBuffer, Buffer.from(iv, 'hex'));
  let decrypted = decipher.update(encrypted, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  return decrypted;
};
```

**加密发生的三个位置（代码确认）**：

| 位置 | 场景 | 文件 |
|-----|------|------|
| 1 | SDK `type` 动作补全时 | `server/src/sdk/workflowEnricher.ts`（`encrypt(value)` 调用） |
| 2 | GUI 录制键盘 `press` 动作时 | `server/src/workflow-management/classes/Generator.ts`（`encrypt(key)` 调用） |
| 3 | GUI `optimizeWorkflow` 合并生成 `type` 动作时 | `server/src/workflow-management/classes/Generator.ts`（`encrypt(state.value)` 调用） |

**解密发生的位置（代码确认）**：
- `server/src/workflow-management/classes/Interpreter.ts` 的 `processWorkflow()`：所有工作流回放前统一解密 `type` 和 `press` 动作的 `args[1]`

---

### 8.4 URL 规范化：严格按代码确认的行为

URL 规范化在 SDK 路径中被调用多次。代码库中实际存在两个不同的规范化函数，用途不同，以下描述严格基于实际代码。

#### 8.4.1 normalizeRobotUrl：存储用 URL 规范化（代码确认）

该函数在三处独立定义但实现完全一致：`server/src/api/sdk.ts`、`server/src/workflow-management/classes/Generator.ts`、`server/src/routes/storage.ts`。

```typescript
const normalizeRobotUrl = (rawUrl: string): string => {
  const normalizedUrl = new URL(rawUrl.trim());
  if (!['http:', 'https:'].includes(normalizedUrl.protocol)) {
    throw new Error('Invalid URL protocol');
  }
  normalizedUrl.search = normalizedUrl.searchParams.toString();
  return normalizedUrl.toString();
};
```

**实际执行的操作（严格按代码）**：
1. `rawUrl.trim()`：去除字符串前后空白字符
2. `new URL(...)`：用 WHATWG URL 构造器解析，解析失败直接抛错
3. 协议校验：仅允许 `http:` 或 `https:`，否则抛 `Invalid URL protocol`
4. `normalizedUrl.search = normalizedUrl.searchParams.toString()`：将查询字符串经 `searchParams` 重新序列化写回（规范化 percent-encoding；参数顺序保持原样，不做排序）
5. `normalizedUrl.toString()`：输出完整 URL 字符串

**代码未执行的操作（明确排除）**：
- 未做 `host.toLowerCase()` 主机名小写转换
- 未做 `pathname.replace(/\/$/, '')` 尾斜杠移除
- 未修改端口号、用户名、密码、hash 等其他 URL 组成部分

#### 8.4.2 normalizeUrl：比较用 URL 规范化（代码确认，仅用于名称去重）

仅定义于 `server/src/api/sdk.ts`，用于 `findExistingRobotByName()` 中比较机器人是否已存在。

```typescript
const normalizeUrl = (raw: string): string => {
  try {
    const u = new URL(raw);
    u.search = u.searchParams.toString();
    return `${u.protocol}//${u.host.toLowerCase()}${u.pathname.replace(/\/$/, '')}${u.search}`;
  } catch {
    return raw.toLowerCase().trim();
  }
};
```

**实际执行的操作（严格按代码）**：
1. `new URL(raw)`：解析 URL，失败则降级为 `raw.toLowerCase().trim()`
2. `u.search = u.searchParams.toString()`：将查询字符串经 `searchParams` 重新序列化写回（规范化 percent-encoding；参数顺序保持原样，不做排序）
3. `u.host.toLowerCase()`：主机名转小写
4. `u.pathname.replace(/\/$/, '')`：移除 pathname 末尾的单个斜杠
5. 手动拼接：`protocol + // + host(小写) + pathname(去尾斜杠) + search`

**用途**：仅用于**判断是否已有同名机器人**时的 URL 等值比较，不用于写入存储。

#### 8.4.3 normalizeWorkflowUrls：遍历工作流规范化（代码确认）

```typescript
const normalizeWorkflowUrls = (workflow: any[] = []): any[] =>
  workflow.map((pair: any) => ({
    ...pair,
    where: pair?.where
      ? {
          ...pair.where,
          ...(typeof pair.where.url === 'string' && pair.where.url !== 'about:blank'
            ? { url: normalizeRobotUrl(pair.where.url) }
            : {}),
        }
      : pair?.where,
    what: Array.isArray(pair?.what)
      ? pair.what.map((action: any) => {
          if (
            action.action === 'goto' &&
            Array.isArray(action.args) &&
            typeof action.args[0] === 'string' &&
            action.args[0] !== 'about:blank'
          ) {
            return {
              ...action,
              args: [normalizeRobotUrl(action.args[0]), ...action.args.slice(1)],
            };
          }

          if (
            (action.action === 'scrape' || action.action === 'crawl') &&
            Array.isArray(action.args) &&
            action.args[0] &&
            typeof action.args[0] === 'object' &&
            typeof action.args[0].url === 'string' &&
            action.args[0].url !== 'about:blank'
          ) {
            return {
              ...action,
              args: [
                {
                  ...action.args[0],
                  url: normalizeRobotUrl(action.args[0].url),
                },
                ...action.args.slice(1),
              ],
            };
          }

          return action;
        })
      : pair?.what,
  }));
```

**实际遍历并调用 `normalizeRobotUrl` 的三个位置（严格按代码）**：
1. **`pair.where.url`**：当类型为 `string` 且值不为 `'about:blank'` 时
2. **`goto` 动作的 `args[0]`**：当 `args` 是数组、`args[0]` 为 `string` 且值不为 `'about:blank'` 时
3. **`scrape` 或 `crawl` 动作的 `args[0].url`**：当 `args` 是数组、`args[0]` 为对象、`args[0].url` 为 `string` 且值不为 `'about:blank'` 时

**特殊规则（代码确认）**：值为字符串 `'about:blank'` 时**跳过规范化**（作为起始标记保留原字面量）。

#### 8.4.4 SDK 路径中的规范化调用时序（代码确认）

```
enrichWorkflow 完成
    │
    ├─► enrichedWorkflow = normalizeWorkflowUrls(enrichResult.workflow)
    │     第 1 次：对 enricher 返回的工作流整体遍历规范化
    │
    ▼
extractedUrl = normalizeRobotUrl(enrichResult.url)
    │     第 2 次：对入口 URL 单独规范化（写入 recording_meta.url）
    │
    ▼
写入 Robot.create 时:
    recording.workflow = normalizeWorkflowUrls(enrichedWorkflow)
          第 3 次：写库前再次对工作流遍历规范化
```

---

### 8.5 写库保存：Robot.create 的最终写入

文件位置：`server/src/models/Robot.ts`

SDK 路径与 GUI 录制路径在**写库阶段完全一致**，均通过 Sequelize 的 `Robot.create()` 写入 PostgreSQL。

#### 写入数据结构（代码确认）

```typescript
Robot.create({
  id: robotId,                    // UUID 主键
  userId: user.id,                // 关联用户

  // --- JSONB 字段 1: 元数据 ---
  recording_meta: {
    name: "My Robot",             // 用户定义的名称
    id: metaId,                   // 业务 ID（与 Robot.id 不同）
    createdAt: "2026-06-21T...",
    updatedAt: "2026-06-21T...",
    pairs: 3,                     // 工作流节点数
    params: [],                   // 提取出的参数占位符
    type: "extract",              // extract / scrape / crawl / search
    url: "https://example.com",   // 入口 URL
    formats: ["markdown", "json"],// 输出格式
    isLLM: false,                 // 是否 LLM 生成
    // ... LLM 配置字段（可选）
  },

  // --- JSONB 字段 2: 工作流本体 ---
  recording: {
    workflow: [                   // WhereWhatPair[] 数组
      {
        where: { url: "about:blank", selectors: [] },
        what: [
          { action: "goto", args: ["https://example.com"] },
          { action: "waitForLoadState", args: ["networkidle"] }
        ]
      },
      {
        where: {
          url: "https://example.com",
          selectors: ["input.search"]
        },
        what: [
          { action: "type", args: ["input.search", "ivHex:encryptedValue", "text"] },
          { action: "waitForLoadState", args: ["networkidle"] },
          { action: "scrapeList", actionId: "list-uuid", args: [{
            fields: { ... }, listSelector: "...", pagination: { ... }
          }]}
        ]
      }
    ]
  },

  // --- 其他关联字段 ---
  google_sheet_email: null,
  schedule: null,
  webhooks: null,
  // ...
})
```

#### Sequelize JSONB 序列化过程

```
JavaScript 对象 (recording_meta, recording)
    │
    ▼  Sequelize DataTypes.JSONB 处理器
JSON.stringify(obj) 自动序列化
    │
    ▼  PostgreSQL 驱动
转换为 PostgreSQL JSONB 二进制格式
    │
    ▼
写入数据库列
```

**关键点（代码确认）**：应用层代码**不需要显式调用 `JSON.stringify()`**。Sequelize 的 `DataTypes.JSONB` 类型定义会在保存时自动完成序列化，读取时自动完成 `JSON.parse()` 反序列化。

#### 与 GUI 录制路径的写入对比

| 对比项 | GUI 录制路径 | SDK 简化路径 |
|-------|------------|------------|
| 入口 | `Generator.saveNewWorkflow()` | `POST /api/sdk/robots` |
| 节点生成 | 实时生成（Socket 事件驱动） | 一次性提交（HTTP 请求体） |
| 选择器来源 | 浏览器自动录制 | 用户提交 + 浏览器补全（enrichWorkflow） |
| 输入处理 | 逐键 press → optimizeWorkflow 合并为 type | 直接 type → encrypt 加密 + detectInputType 补全 |
| URL 规范化 | `normalizeWorkflowUrls()` 一次 | `normalizeWorkflowUrls()` 两次 + `normalizeRobotUrl()` 一次 |
| 写库函数 | `Robot.create()` | `Robot.create()` |
| 存储格式 | **完全一致** | **完全一致** |
| 回放加载 | **完全一致** | **完全一致** |
| 执行引擎 | **完全一致** | **完全一致** |

---

### 8.6 SDK 简化工作流的完整关系图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      SDK 提交：简化工作流输入                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  POST /api/sdk/robots                                                   │
│  Body: {                                                                │
│    meta: { name: "...", type: "extract", ... },                         │
│    workflow: [                                                          │
│      { where: { url: "..." }, what: [                                   │
│        { action: "type", args: ["#search", "keyword"] },   ← 简化格式    │
│        { action: "scrapeList", args: [{ itemSelector: "..." }] }        │
│      ]}                                                                 │
│    ]                                                                    │
│  }                                                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                 步骤 1：WorkflowEnricher 选择器补全                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 启动浏览器：createRemoteBrowserForValidation()                      │
│  2. 导航到 URL：page.goto(url)                                          │
│  3. 遍历每个动作进行补全：                                                │
│                                                                         │
│  ┌─ type 动作 ──────────────────────────────────────────────┐           │
│  │  selector → 加入 where.selectors 集合                    │           │
│  │  value    → encrypt(value) 加密                          │           │
│  │             （AES-256-CBC，随机 16 字节 IV，输出 ivHex:密文Hex）       │
│  │  inputType→ validator.detectInputType() 自动检测          │           │
│  │             （HTMLInputElement.type / 'textarea' /        │           │
│  │              'select' / 'text'）                          │           │
│  │  自动追加: waitForLoadState(networkidle)                 │           │
│  └───────────────────────────────────────────────────────────┘           │
│                                                                         │
│  ┌─ scrapeSchema 动作 ──────────────────────────────────────┐           │
│  │  每个字段 → validateSelector() 验证                       │           │
│  │    补全: { tag, isShadow, selector, attribute }          │           │
│  │    选择器 → 加入 where.selectors                          │           │
│  └───────────────────────────────────────────────────────────┘           │
│                                                                         │
│  ┌─ scrapeList 动作 ────────────────────────────────────────┐           │
│  │  autoDetectListFields() → 自动识别子字段                  │           │
│  │  autoDetectPagination() → 自动识别分页                    │           │
│  └───────────────────────────────────────────────────────────┘           │
│                                                                         │
│  enrichedStep.where.selectors = [收集的选择器数组]                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│           步骤 2：URL 规范化（严格按代码确认的行为）                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  normalizeRobotUrl 实际执行：                                            │
│    1. rawUrl.trim()                     ← 去前后空白                    │
│    2. new URL(...) 解析 + 协议校验        ← 仅 http/https                │
│    3. search = searchParams.toString()   ← 规范化 percent-encoding          │
│    4. .toString() 输出                   ← 完整 URL 字符串               │
│    （代码未执行：主机小写 / 去尾斜杠，两者仅 normalizeUrl 比较时才做）       │
│                                                                         │
│  normalizeWorkflowUrls 遍历三处：                                        │
│    ① pair.where.url                      (字符串且 ≠ 'about:blank')     │
│    ② goto.args[0]                        (字符串且 ≠ 'about:blank')     │
│    ③ scrape/crawl.args[0].url            (字符串且 ≠ 'about:blank')     │
│                                                                         │
│  第 1 次: normalizeWorkflowUrls(enrichedWorkflow)  ← enricher 输出后     │
│  第 2 次: normalizeRobotUrl(enrichResult.url)      ← 入口 URL 单独       │
│  第 3 次: normalizeWorkflowUrls(enrichedWorkflow)  ← 写库前再遍历一次     │
│                                                                         │
│  特殊规则: 值为字面量 "about:blank" → 跳过规范化（保留作起始标记）         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                   步骤 3：构建 recording_meta 元数据                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  robotMeta = {                                                          │
│    name, id(uuid), createdAt, updatedAt,                                │
│    pairs: enrichedWorkflow.length,                                      │
│    type, url(规范化后的), formats,                                      │
│    isLLM, promptInstructions, ...(LLM 配置)                             │
│  }                                                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     步骤 4：Robot.create 写库保存                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Robot.create({                                                         │
│    id: uuid(),                                                          │
│    userId: user.id,                                                     │
│    recording_meta: robotMeta,    ← DataTypes.JSONB 自动 JSON.stringify   │
│    recording: {                   ← DataTypes.JSONB 自动 JSON.stringify   │
│      workflow: normalizeWorkflowUrls(enrichedWorkflow)                  │
│    }                                                                    │
│    ...其他字段                                                           │
│  })                                                                     │
│         │                                                               │
│         ▼  Sequelize ORM 层                                            │
│  JSON.stringify(recording_meta) → PostgreSQL JSONB                      │
│  JSON.stringify(recording)      → PostgreSQL JSONB                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     后续：与 GUI 录制路径完全共用                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  • 从数据库加载：Robot.findOne() → Sequelize 自动 JSON.parse 反序列化     │
│  • 回放前：processWorkflow() 解密 AES 加密的 type/press 值              │
│  • 预处理：Preprocessor.initWorkflow() 转换 $regex/$param              │
│  • 执行：Interpreter.run() → runLoop() → carryOutSteps()                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 8.7 关键代码文件索引（SDK 路径）

| 文件 | 主要职责 | 关键函数/路由 |
|-----|---------|-------------|
| `server/src/server.ts` | Express 路由挂载入口 | `app.use('/api', router)` 统一给 api/ 目录加 /api 前缀 |
| `server/src/api/sdk.ts` | SDK API 路由，统一入口 | `POST /api/sdk/robots`（外部完整路径，内部定义为 `/sdk/robots`，经 `/api` 前缀挂载）、`normalizeRobotUrl`、`normalizeWorkflowUrls`、`normalizeUrl`（仅比较用） |
| `server/src/sdk/workflowEnricher.ts` | 选择器补全核心 | `enrichWorkflow()`、`generateWorkflowFromPrompt()` |
| `server/src/sdk/selectorValidator.ts` | 浏览器端选择器验证 | `validateSelector()`、`detectInputType()`、`validateSchemaFields()` |
| `server/src/utils/auth.ts` | 敏感数据加解密 | `encrypt()` (AES-256-CBC)、`decrypt()` |
| `server/src/models/Robot.ts` | 数据库模型（与 GUI 共享） | `Robot.init()` 中 JSONB 字段定义 |

---

### 8.8 SDK 路径核心结论（严格代码确认）

1. **两种路径，统一存储**：SDK 简化工作流经过补全、加密、规范化后，写入数据库的 `WhereWhatPair` 结构与 GUI 录制路径**完全一致**

2. **选择器补全的本质**：启动真实浏览器验证用户提交的选择器，补全 `tag`、`isShadow`、`attribute` 等元数据，并将选择器集合写入 `where.selectors` 字段

3. **加密一致性**：SDK 的 `type` 动作直接调用 `encrypt()`，跳过了 GUI 的逐键 `press` → 优化为 `type` 的过程，但加密算法（AES-256-CBC，输出 `ivHex:ciphertextHex`）和存储格式与 GUI 路径完全相同

4. **URL 规范化的明确边界**：
   - `normalizeRobotUrl` 做：`trim()` → 协议校验 → `searchParams.toString()` 规范化 percent-encoding → `.toString()` 输出（参数顺序保持原样，不做排序）
   - `normalizeRobotUrl` **不做**：主机名小写、尾斜杠去除（这两者仅存在于 `normalizeUrl`，且仅用于同名机器人比较场景）
   - `normalizeWorkflowUrls` 遍历：`where.url`、`goto.args[0]`、`scrape/crawl.args[0].url` 三处，字面量 `'about:blank'` 一律跳过
   - SDK 路径中调用时序：enricher 输出后 1 次、入口 URL 1 次、写库前再 1 次

5. **后续阶段零差异**：一旦写入数据库，加载、预处理、回放执行的所有步骤，SDK 路径与 GUI 录制路径**完全共用相同的代码**
