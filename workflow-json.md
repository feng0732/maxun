# 工作流 JSON 模型的保存和恢复关系分析

## 概述

本文档分析 maxun 项目中工作流（Workflow）JSON 模型在**节点数据生成**、**序列化持久化**、**回放执行**三个阶段之间的流转关系和代码实现。

---

## 一、核心数据模型

### 1.1 类型定义

文件位置：[maxun-core/src/types/workflow.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/maxun-core/src/types/workflow.ts)

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

文件位置：[server/src/models/Robot.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/server/src/models/Robot.ts)

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

文件位置：[server/src/browser-management/inputHandlers.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/server/src/browser-management/inputHandlers.ts)

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

文件位置：[server/src/workflow-management/classes/Generator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/server/src/workflow-management/classes/Generator.ts)

核心方法 `addPairToWorkflowAndNotifyClient`（第 295 行）：

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

文件位置：[server/src/workflow-management/classes/Generator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/server/src/workflow-management/classes/Generator.ts#L1417-L1518)

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

**优化效果**：将 10 次 `press` 按键动作优化为 1 次 `type` 批量输入动作，显著提高回放效率。

---

### 阶段 2：序列化与持久化（保存阶段）

#### 保存工作流到数据库

文件位置：[server/src/workflow-management/classes/Generator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/server/src/workflow-management/classes/Generator.ts#L1044-L1128)

`saveNewWorkflow` 方法处理保存逻辑：

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

文件位置：[server/src/workflow-management/classes/Generator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/server/src/workflow-management/classes/Generator.ts#L469-L484)

```typescript
const pair: WhereWhatPair = {
  where: { 
    url: this.getBestUrl(url),
    selectors: [selector]
  },
  what: [{
    action: 'press',
    args: [selector, encrypt(key), inputType || 'text'],  // key 被加密
  }],
};
```

---

### 阶段 3：反序列化与恢复（加载阶段）

#### 从数据库加载工作流

文件位置：[server/src/routes/workflow.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/server/src/routes/workflow.ts#L109-L150)

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

文件位置：[server/src/workflow-management/classes/Generator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/server/src/workflow-management/classes/Generator.ts#L1029-L1036)

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

文件位置：[maxun-core/src/preprocessor.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/maxun-core/src/preprocessor.ts)

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

文件位置：[server/src/workflow-management/classes/Interpreter.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/server/src/workflow-management/classes/Interpreter.ts#L14-L48)

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

文件位置：[maxun-core/src/interpret.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/maxun-core/src/interpret.ts#L2915-L2958)

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

文件位置：[maxun-core/src/interpret.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/maxun-core/src/interpret.ts#L2689-L2872)

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

文件位置：[maxun-core/src/interpret.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/maxun-core/src/interpret.ts#L550-L1953)

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
│         ├─► normalizeWorkflowUrls() - URL 标准化                        │
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
| 1 | [preprocessor.ts#L168](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/maxun-core/src/preprocessor.ts#L168) | `JSON.parse(JSON.stringify(workflow))` | 预处理时深拷贝，避免修改原始工作流 |
| 2 | [interpret.ts#L2695](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/maxun-core/src/interpret.ts#L2695) | `JSON.parse(JSON.stringify(workflow))` | 执行循环时深拷贝，维护待执行队列 |
| 3 | [Interpreter.ts#L15](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/server/src/workflow-management/classes/Interpreter.ts#L15) | `JSON.parse(JSON.stringify(workflow))` | 输入解密前深拷贝 |
| 4 | [Robot.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/server/src/models/Robot.ts) | Sequelize JSONB 自动处理 | 数据库持久化和读取 |

### 4.2 特殊格式转换

| 转换类型 | 存储形式（JSON） | 运行时形式（JS） | 处理位置 |
|---------|-----------------|-----------------|---------|
| 正则表达式 | `{ "$regex": "^https://example.com" }` | `RegExp` 对象 | [preprocessor.ts#L183-L187](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/maxun-core/src/preprocessor.ts#L183-L187) |
| 参数占位符 | `{ "$param": "username" }` | 实际参数值 | [preprocessor.ts#L171-L181](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/maxun-core/src/preprocessor.ts#L171-L181) |
| 敏感输入 | 加密字符串（AES） | 明文字符串 | [Interpreter.ts#L28-L43](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/server/src/workflow-management/classes/Interpreter.ts#L28-L43) |

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

## 六、关键代码文件索引

| 文件 | 主要职责 |
|-----|---------|
| [workflow.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/maxun-core/src/types/workflow.ts) | 核心类型定义（WhereWhatPair, Workflow 等） |
| [preprocessor.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/maxun-core/src/preprocessor.ts) | 工作流验证、参数初始化、正则转换 |
| [interpret.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/maxun-core/src/interpret.ts) | 工作流解释执行核心（run, runLoop, carryOutSteps） |
| [Generator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/server/src/workflow-management/classes/Generator.ts) | 录制时节点生成、优化、保存 |
| [Interpreter.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/server/src/workflow-management/classes/Interpreter.ts) | 服务端解释器包装、输入解密 |
| [inputHandlers.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/server/src/browser-management/inputHandlers.ts) | 用户输入事件路由 |
| [Robot.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/server/src/models/Robot.ts) | 数据库模型定义 |
| [workflow.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/109-maxun/server/src/routes/workflow.ts) | 工作流 REST API |

---

## 七、总结

工作流 JSON 模型的流转遵循以下核心原则：

1. **录制生成**：用户交互 → 细粒度 `WhereWhatPair` 节点 → 内存状态
2. **优化合并**：保存前将分散的键盘输入优化为批量 `type` 动作
3. **持久化**：通过 PostgreSQL JSONB 直接存储完整结构，ORM 处理序列化
4. **恢复加载**：从数据库读取后直接恢复为内存对象
5. **预处理**：执行前转换特殊格式（`$regex` → RegExp，`$param` → 实际值，密文 → 明文）
6. **回放执行**：深拷贝后按顺序执行每个节点的动作，从待执行队列中移除已完成项

整个流程中，**序列化**是连接内存对象和持久化存储的桥梁，**节点数据**是贯穿始终的核心载体，**回放输入**则是将录制的用户操作还原为浏览器自动化动作的最终目标。
