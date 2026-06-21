# Maxun 登录态与 Cookie 注入流程代码分析

## 核心结论

**Maxun 当前不存在浏览器 Cookie 持久化保存和主动注入机制。** 登录态完全通过"录制登录操作 → 回放时重新执行登录步骤"的方式实现。每次启动 Run 都会创建一个全新的、干净的 Playwright `BrowserContext`（无任何历史 Cookie），然后通过重新执行录制的登录动作（输入用户名 → 输入密码 → 点击登录按钮等），让浏览器在执行过程中自然获得登录态 Cookie。

---

## 一、凭据保存流程（录制阶段）

### 1.1 敏感输入加密存储

录制过程中，用户的键盘输入（包括用户名、密码等敏感信息）会被 AES-256-CBC 加密后保存到工作流（Workflow）的动作参数中。

**加密入口**：[server/src/workflow-management/classes/Generator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/workflow-management/classes/Generator.ts)

- `onDOMKeyboardAction()` 方法（逐键录制时）：

```typescript
// Generator.ts L473-L477
what: [{
  action: 'press',
  args: [selector, encrypt(key), inputType || 'text'],
}],
```

- `optimizeWorkflow()` 方法（将离散按键合并为 type 动作时）：

```typescript
// Generator.ts L1503-L1507
pair.what.push({
  action: 'type',
  args: [selector, encrypt(state.value), state.type]
}, {
  action: 'waitForLoadState',
  args: ['networkidle']
});
```

**加密实现**：[server/src/utils/auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/utils/auth.ts) L26-L43

```typescript
export const encrypt = (text: string): string => {
    const ivLength = 16;
    const iv = crypto.randomBytes(ivLength);
    const algorithm = 'aes-256-cbc';
    let key = getEnvVariable('ENCRYPTION_KEY');
    // aes-256-cbc requires a 256-bit key (64 hex characters)
    const keyBuffer = Buffer.from(key, 'hex');
    const cipher = crypto.createCipheriv(algorithm, keyBuffer, iv);
    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    return `${iv.toString('hex')}:${encrypted}`;
};
```

加密输出格式：`{iv_hex}:{ciphertext_hex}`，IV 是随机生成的 16 字节。

### 1.2 isLogin 标志保存

前端在保存录制时会传递 `isLogin` 布尔值，标识该录制是否为登录流程。

**前端来源**：
- [src/components/recorder/SaveRecording.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/src/components/recorder/SaveRecording.tsx)：保存录制时发送 isLogin
- [src/context/globalInfo.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/src/context/globalInfo.tsx)：全局状态存储 `isLogin: boolean`
- [src/components/robot/RecordingsTable.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/src/components/robot/RecordingsTable.tsx)：启动录制时设置 isLogin 复选框

**后端保存**：[server/src/workflow-management/classes/Generator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/workflow-management/classes/Generator.ts) L1044-L1100

```typescript
public saveNewWorkflow = async (fileName: string, userId: number, isLogin: boolean, robotId?: string) => {
    // ...
    this.recordingMeta = {
        // ...
        isLogin: isLogin,  // 保存到 recording_meta
    }
    const robot = await Robot.create({
        // ...
        recording_meta: this.recordingMeta,
    });
};
```

**数据库存储**：[server/src/models/Robot.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/models/Robot.ts) 的 `recording_meta` JSONB 字段存储 `isLogin` 标志。

> ⚠️ **注意**：`isLogin` 标志目前仅作元数据记录，代码中并未基于此标志执行任何特殊的 Cookie 保存或注入逻辑。

---

## 二、Cookie 注入时机分析

### 2.1 每次 Run 都是全新 BrowserContext

**结论：不存在显式的 Cookie 注入步骤。**

浏览器上下文创建代码：[server/src/browser-management/classes/RemoteBrowser.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/browser-management/classes/RemoteBrowser.ts) L460-L527

```typescript
public initialize = async (userId: string): Promise<void> => {
    // ...
    const contextPromise = this.browser.newContext(contextOptions);
    this.context = await Promise.race([
        contextPromise,
        // timeout...
    ]) as BrowserContext;
    // ...
};
```

`browser.newContext()` 始终创建一个**完全隔离、全新的、无任何历史 Cookie** 的浏览器会话环境。代码全局搜索确认，不存在以下 Playwright API 调用：

- `context.addCookies()` — 不存在
- `browser.newContext({ storageState: ... })` — 不存在
- `context.storageState()` — 不存在

唯一与 Cookie 相关的代码位于 [maxun-core/src/interpret.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/maxun-core/src/interpret.ts) L271：

```typescript
cookies: (await page.context().cookies([page.url()]))
  .reduce((p, cookie) => ({
    ...p,
    [cookie.name]: cookie.value,
  }), {}),
```

此代码仅用于读取当前页面 Cookie 以支持 Workflow 的 where 条件匹配（如判断某个 Cookie 是否存在来决定是否执行某步骤），**并非用于登录态的持久化或注入**。

---

## 三、回放使用方式（Run 阶段）

### 3.1 Run 启动流程

执行入口：[server/src/task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/task-runner.ts) 的 `processRunExecution()` 函数

流程：
1. 创建/获取 RemoteBrowser 实例
2. 调用 `remoteBrowser.initialize(userId)` 创建全新 BrowserContext（干净、无 Cookie）
3. 从数据库获取录制的 Workflow
4. （可选）注入用户配置的 Credentials 覆盖录制值
5. 调用解释器 `remoteBrowser.interpreter.interpret()` 执行 Workflow
6. 执行过程中自然产生登录态 Cookie

### 3.2 凭据解密执行

**执行前解密**：[server/src/workflow-management/classes/Interpreter.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/workflow-management/classes/Interpreter.ts) L28-L33

```typescript
if ((action.action === 'type' || action.action === 'press') && Array.isArray(action.args) && action.args.length > 1) {
  try {
    const encryptedValue = action.args[1];
    if (typeof encryptedValue === 'string') {
      const decryptedValue = decrypt(encryptedValue);
      action.args[1] = decryptedValue;
    }
  }
}
```

解密后，解释器在浏览器中执行 `type`/`press` 动作，将明文输入到对应的表单字段中。

**解密实现**：[server/src/utils/auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/utils/auth.ts) L45-L60

```typescript
export const decrypt = (encryptedText: string): string => {
    const [iv, encrypted] = encryptedText.split(':');
    const algorithm = "aes-256-cbc";
    let key = getEnvVariable('ENCRYPTION_KEY');
    const keyBuffer = Buffer.from(key, 'hex');
    const decipher = crypto.createDecipheriv(algorithm, keyBuffer, Buffer.from(iv, 'hex'));
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    return decrypted;
};
```

### 3.3 额外的 Credentials 覆盖机制

用户可在 Robot 设置页面为特定 CSS 选择器配置凭据值。Run 启动时，这些值会覆盖 Workflow 中录制的原始值。

**注入位置**：[server/src/routes/storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/routes/storage.ts) 的 `handleWorkflowActions()` 函数 L290-L355

核心逻辑：

```typescript
function handleWorkflowActions(workflow: any[], credentials: Credentials) {
  // 遍历 workflow 的每个步骤
  return workflow.map(step => {
    // 遍历 step.what 中的每个动作
    for (let i = 0; i < step.what.length; i++) {
      const action = step.what[i];
      const selector = action.args[0];
      const credential = credentials[selector];

      // 如果该选择器有配置的凭据
      if (credential) {
        // 用用户配置的凭据值（加密后）替换原有的 type/press 动作
        newWhat.push({
          action: 'type',
          args: [selector, encrypt(credential.value), credential.type]
        });
        // 跳过后续原来的 type/press/waitForLoadState 动作
      }
    }
  });
}
```

调用位置：storage.ts L447-L449

```typescript
if (credentials) {
  workflow = handleWorkflowActions(workflow, credentials);
}
```

---

## 四、完整流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         录制阶段 (Recording)                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐    ┌────────────────────┐    ┌──────────────────┐ │
│  │ 用户键盘输入 │───▶│  encrypt() AES加密  │───▶│  保存到 Workflow  │ │
│  │ (用户名/密码)│    │  (auth.ts L26-43)   │    │  type/press args  │ │
│  └──────────────┘    └────────────────────┘    └────────┬─────────┘ │
│                                                         │           │
│  ┌──────────────┐    ┌────────────────────┐             ▼           │
│  │  isLogin 标志│───▶│ 保存到 recording_meta│──── Robot 数据库       │
│  └──────────────┘    └────────────────────┘                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          回放阶段 (Run)                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  1. 创建全新 BrowserContext (干净，无Cookie)                   │  │
│  │     RemoteBrowser.initialize() → browser.newContext()         │  │
│  └───────────────────────────────┬───────────────────────────────┘  │
│                                  │                                  │
│                                  ▼                                  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  2. 可选: Credentials 覆盖 (storage.ts handleWorkflowActions)  │  │
│  │     如果用户为某选择器配置了凭据值，加密后替换原始录制值         │  │
│  └───────────────────────────────┬───────────────────────────────┘  │
│                                  │                                  │
│                                  ▼                                  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  3. 解释器执行前解密 (Interpreter.ts L28-33)                   │  │
│  │     decrypt() → 还原明文用户名/密码                            │  │
│  └───────────────────────────────┬───────────────────────────────┘  │
│                                  │                                  │
│                                  ▼                                  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  4. Playwright 按顺序执行录制动作:                              │  │
│  │     • goto 登录页面                                            │  │
│  │     • type 用户名 → 浏览器输入                                 │  │
│  │     • type 密码 → 浏览器输入                                   │  │
│  │     • click 登录按钮 → 提交表单                                │  │
│  │     • 服务器返回 Set-Cookie → 浏览器自动保存到 Context         │  │
│  └───────────────────────────────┬───────────────────────────────┘  │
│                                  │                                  │
│                                  ▼                                  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  5. 后续步骤在已登录状态下执行                                  │  │
│  │     (Cookie 存在于 BrowserContext 内存中，随 Run 结束销毁)      │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 五、关键代码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
| AES 加密 | [server/src/utils/auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/utils/auth.ts) | L26-L43 |
| AES 解密 | [server/src/utils/auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/utils/auth.ts) | L45-L60 |
| 录制时加密键盘输入 | [server/src/workflow-management/classes/Generator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/workflow-management/classes/Generator.ts) | L476 |
| 优化时加密合并的 type 动作 | [server/src/workflow-management/classes/Generator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/workflow-management/classes/Generator.ts) | L1506 |
| 保存 isLogin 标志 | [server/src/workflow-management/classes/Generator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/workflow-management/classes/Generator.ts) | L1099 |
| 创建全新 BrowserContext | [server/src/browser-management/classes/RemoteBrowser.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/browser-management/classes/RemoteBrowser.ts) | L522 |
| 执行前解密凭据 | [server/src/workflow-management/classes/Interpreter.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/workflow-management/classes/Interpreter.ts) | L28-L33 |
| Credentials 覆盖注入 | [server/src/routes/storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/routes/storage.ts) | L290-L355 |
| 读取 Cookie 用于 where 匹配 | [maxun-core/src/interpret.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/maxun-core/src/interpret.ts) | L271 |
| Robot 数据模型 | [server/src/models/Robot.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/models/Robot.ts) | - |
| Run 执行入口 | [server/src/task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/task-runner.ts) | processRunExecution() |

---

## 六、设计特点与限制

### 设计特点
1. **无状态设计**：每次 Run 完全独立，避免了跨任务的登录态污染
2. **凭据加密**：敏感数据 AES-256-CBC 加密存储，符合安全要求
3. **灵活覆盖**：支持通过 Credentials 机制在运行时替换录制的凭据值，便于多环境/多账号使用

### 当前限制
1. **每次都需重新登录**：无 Cookie 持久化，Run 开始必须重新执行完整登录流程，增加了执行时间
2. **isLogin 标志未被实际使用**：虽然录制时保存了 `isLogin`，但后续代码未基于此标志做特殊处理（如提取保存 Cookie）
3. **登录态随 Run 销毁**：BrowserContext 在 Run 结束后即销毁，无法在多个 Run 间共享登录态
4. **无会话续期机制**：如果登录态在长 Run 中过期，没有自动续期能力

---

## 七、凭据覆盖时机详解（编辑配置保存流程）

### 7.1 核心设计：凭据在编辑保存时"烧录"到 Workflow

**关键发现**：`handleWorkflowActions()` 并非在 Run 启动时调用，而是在**编辑配置保存时**就被调用，将凭据直接"烧录"到 Workflow 中保存。

### 7.2 完整流程

**前端触发**：[src/components/robot/pages/RobotEditPage.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/src/components/robot/pages/RobotEditPage.tsx) L1203-L1326

```typescript
const handleSave = async () => {
  // 构造 credentials payload
  const credentialsForPayload = Object.entries(credentials).reduce(
    (acc, [selector, info]) => {
      const enforceType = info.type === "password" ? "password" : "text";
      acc[selector] = {
        value: info.value,
        type: enforceType,
      };
      return acc;
    },
    {} as Record<string, CredentialInfo>
  );

  const payload: any = {
    name: robot.recording_meta.name,
    limits: scrapeListLimits.map(...),
    credentials: credentialsForPayload,  // 明文传递凭据
    targetUrl: targetUrl,
    workflow: updatedWorkflow,
    formats: ...,
  };

  // 调用更新 API
  const success = await updateRecording(robot.recording_meta.id, payload);

  if (success) {
    // 保存成功后立即启动 Run
    handleStart(robot);
    // ...
  }
};
```

**前端 API**：[src/api/storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/src/api/storage.ts) L115-L130

```typescript
export const updateRecording = async (id: string, data: { 
  name?: string; 
  limits?: Array<{pairIndex: number, actionIndex: number, argIndex: number, limit: number}>;
  credentials?: Credentials; 
  targetUrl?: string;
  workflow?: any[];
  formats?: OutputFormats[];
}): Promise<boolean> => {
  const response = await axios.put(`${apiUrl}/storage/recordings/${id}`, data);
  return response.status === 200;
};
```

**后端处理**：[server/src/routes/storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/routes/storage.ts) L373-L556

```typescript
router.put('/recordings/:id', requireSignIn, async (req: AuthenticatedRequest, res) => {
  const { id } = req.params;
  const { name, limits, credentials, targetUrl, workflow: incomingWorkflow, formats } = req.body;

  // 从数据库获取原始 robot
  const robot = await Robot.findOne({ where: { 'recording_meta.id': id, userId: req.user!.id } });
  let workflow = robot.recording.workflow;

  // ... URL 规范化等处理 ...

  // ⚠️ 关键：凭据覆盖发生在这里（保存前）
  if (credentials) {
    workflow = handleWorkflowActions(workflow, credentials);
  }

  // ... limits 处理 ...

  // 将修改后的 workflow 保存到数据库
  const updates: any = {
    recording: { ...robot.recording, workflow: normalizeWorkflowUrls(workflow) },
    recording_meta: updatedMeta,
  };

  await Robot.update(updates, {
    where: { 'recording_meta.id': id, userId: req.user!.id }
  });

  return res.status(200).json({ message: 'Robot updated successfully', robot });
});
```

### 7.3 handleWorkflowActions 核心逻辑

[server/src/routes/storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/routes/storage.ts) L290-L355

```typescript
function handleWorkflowActions(workflow: any[], credentials: Credentials) {
  return workflow.map(step => {
    if (!step.what) return step;

    const newWhat: any[] = [];
    const processedSelectors = new Set<string>();

    for (let i = 0; i < step.what.length; i++) {
      const action = step.what[i];
      const selector = action.args[0];
      const credential = credentials[selector];

      if (!credential) {
        newWhat.push(action);
        continue;
      }

      // 该选择器有配置的凭据值
      if (action.action === 'click') {
        newWhat.push(action);
        // 如果 click 后面跟着 type/press，用凭据值替换
        if (!processedSelectors.has(selector) &&
            i + 1 < step.what.length &&
            (step.what[i + 1].action === 'type' || step.what[i + 1].action === 'press')) {
          
          // 用加密后的凭据值替换
          newWhat.push({
            action: 'type',
            args: [selector, encrypt(credential.value), credential.type]
          });
          newWhat.push({
            action: 'waitForLoadState',
            args: ['networkidle']
          });

          processedSelectors.add(selector);
          // 跳过后续原来的 type/press/waitForLoadState 动作
          while (i + 1 < step.what.length && ...) { i++; }
        }
      } else if ((action.action === 'type' || action.action === 'press') &&
                 !processedSelectors.has(selector)) {
        // 直接替换 type/press 动作的值
        newWhat.push({
          action: 'type',
          args: [selector, encrypt(credential.value), credential.type]
        });
        newWhat.push({
          action: 'waitForLoadState',
          args: ['networkidle']
        });
        processedSelectors.add(selector);
        // 跳过后续重复动作
        while (...) { i++; }
      } else {
        newWhat.push(action);
      }
    }

    return { ...step, what: newWhat };
  });
}
```

### 7.4 关键结论

1. **凭据不单独存储**：`credentials` 参数本身不会保存到数据库，而是通过 `handleWorkflowActions()` 处理后，将加密后的凭据值直接替换 Workflow 中对应的 `type`/`press` 动作的 `args[1]`
2. **保存即覆盖**：凭据覆盖发生在编辑保存阶段，而非 Run 启动阶段
3. **明文传输**：前端通过 HTTP 明文发送 credentials 到后端（依赖 HTTPS 保护）
4. **立即生效**：保存成功后调用 `handleStart(robot)` 立即启动 Run，此时 Workflow 中已包含新的凭据值

---

## 八、回放入口完整调用链

### 8.1 入口触发点

**前端触发位置**：保存成功后立即启动
[src/components/robot/pages/RobotEditPage.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/src/components/robot/pages/RobotEditPage.tsx) L1310

```typescript
if (success) {
  setRerenderRobots(true);
  notify("success", t("robot_edit.notifications.update_success"));
  handleStart(robot);  // 立即启动 Run
  // ...
}
```

**前端 Run API**：[src/api/storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/src/api/storage.ts) L288-L302

```typescript
export const createRunForStoredRecording = async (id: string, settings: RunSettings): Promise<CreateRunResponse> => {
  const response = await axios.put(
    `${apiUrl}/storage/runs/${id}`,
    { ...settings });  // settings 包含 formats、params 等，不包含 credentials
  // ...
};
```

### 8.2 后端 Run 启动入口

[server/src/routes/storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/routes/storage.ts) L997-L1131

```typescript
router.put('/runs/:id', requireSignIn, async (req: AuthenticatedRequest, res) => {
  const recording = await Robot.findOne({
    where: {
      'recording_meta.id': req.params.id,
      userId: req.user.id,
    },
    raw: true
  });

  const runId = uuid();
  const canCreateBrowser = await browserPool.hasAvailableBrowserSlots(req.user.id, "run");

  if (canCreateBrowser) {
    // 1. 创建浏览器（异步初始化）
    const browserId = await createRemoteBrowserForRun(req.user.id);
    
    // 2. 创建 Run 记录
    await Run.create({
      status: 'running',
      name: recording.recording_meta.name,
      robotId: recording.id,
      robotMetaId: recording.recording_meta.id,
      startedAt: new Date().toLocaleString(),
      finishedAt: '',
      browserId: browserId, 
      interpreterSettings: req.body,  // 保存运行时设置（formats、params 等）
      log: '',
      runId,
      runByUserId: req.user.id,
      serializableOutput: {},
      binaryOutput: {},
    });

    // 3. 加入执行队列
    const jobId = await addJob(QUEUE_NAMES.EXECUTE_RUN, {
      userId: req.user.id,
      runId: runId,
      browserId: browserId,
    }, { maxAttempts: 1 });

    return res.send({
      browserId: browserId, 
      runId: runId,
      robotMetaId: recording.recording_meta.id,
      queued: false 
    }); 
  } else {
    // 浏览器槽位不足，进入排队状态
    const browserId = uuid(); 
    await Run.create({
      status: 'queued',  // 标记为排队
      // ...
      interpreterSettings: req.body,
      log: 'Run queued - waiting for available browser slot',
      // ...
    });
    
    return res.send({
      browserId: browserId,
      runId: runId,
      robotMetaId: recording.recording_meta.id,
      queued: true 
    });
  } 
});
```

### 8.3 interpreterSettings 的作用

`interpreterSettings` 在 Run 启动时保存，主要包含：
- `formats`：输出格式（markdown、html、screenshot 等）
- `params`：工作流参数
- `robotType`：机器人类型（doc-extract、doc-parse 等）
- `promptInstructions`：LLM 提示词指令
- **不包含 credentials**：因为凭据已在编辑保存时烧录到 Workflow 中

---

## 九、运行队列启动流程（Graphile Worker）

### 9.1 队列架构

Maxun 使用 [Graphile Worker](https://github.com/graphile/worker) 作为后台任务队列，基于 PostgreSQL 实现。

**队列名称定义**：[server/src/task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/task-runner.ts) L37-L45

```typescript
export const QUEUE_NAMES = {
  INITIALIZE_BROWSER_RECORDING: 'initialize-browser-recording',
  DESTROY_BROWSER: 'destroy-browser',
  INTERPRET_WORKFLOW: 'interpret-workflow',
  STOP_INTERPRETATION: 'stop-interpretation',
  EXECUTE_RUN: 'execute-run',           // Run 执行
  ABORT_RUN: 'abort-run',               // Run 中止
  SCHEDULED_WORKFLOW: 'scheduled-workflow',
} as const;
```

### 9.2 入队操作

[server/src/storage/graphileWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/storage/graphileWorker.ts) L61-L71

```typescript
export async function addJob(
  taskIdentifier: string,
  payload: Record<string, unknown>,
  options?: { maxAttempts?: number; runAt?: Date; jobKey?: string },
): Promise<string> {
  if (!workerUtils) {
    throw new Error('Graphile Worker utils not initialized');
  }
  const job = await workerUtils.addJob(taskIdentifier, payload, options);
  return String(job.id);
}
```

### 9.3 任务处理器注册

[server/src/task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/task-runner.ts) L631-L675

```typescript
const taskList: TaskList = {
  [QUEUE_NAMES.INITIALIZE_BROWSER_RECORDING]: async (payload: unknown) => {
    const data = payload as InitializeBrowserData;
    initializeRemoteBrowserForRecording(data.userId);
  },

  [QUEUE_NAMES.DESTROY_BROWSER]: async (payload: unknown) => {
    const data = payload as DestroyBrowserData;
    await destroyRemoteBrowser(data.browserId, data.userId);
  },

  [QUEUE_NAMES.EXECUTE_RUN]: async (payload: unknown) => {
    await processRunExecution(payload as ExecuteRunData);  // 核心执行入口
  },

  [QUEUE_NAMES.ABORT_RUN]: async (payload: unknown) => {
    const data = payload as AbortRunData;
    await abortRun(data.runId, data.userId);
  },

  [QUEUE_NAMES.SCHEDULED_WORKFLOW]: async (payload: unknown) => {
    const data = payload as ScheduledWorkflowData;
    await handleRunRecording(data.robotMetaId, data.userId);
  },
};
```

### 9.4 Worker 启动

[server/src/task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/task-runner.ts) 后续代码初始化 Worker 并监听任务。

---

## 十、浏览器上下文创建与时序关联

### 10.1 浏览器创建流程（非阻塞异步）

**创建入口**：[server/src/browser-management/controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/browser-management/controller.ts) L114-L137

```typescript
export const createRemoteBrowserForRun = (userId: string): string => {
  const id = uuid();

  // 1. 原子性预留浏览器槽位
  const slotReserved = browserPool.reserveBrowserSlotAtomic(id, userId, "run");
  if (!slotReserved) {
    throw new Error('User has reached maximum browser limit');
  }

  // 2. ⚠️ 异步初始化浏览器（非阻塞，立即返回 browserId）
  initializeBrowserAsync(id, userId)
    .catch((error: any) => {
      logger.log('error', `Unhandled error in initializeBrowserAsync: ${error.message}`);
      browserPool.failBrowserSlot(id);
    });
  
  return id;  // 立即返回，不等待浏览器初始化完成
};
```

### 10.2 浏览器初始化异步流程

`initializeBrowserAsync()` 内部最终调用：
[server/src/browser-management/classes/RemoteBrowser.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/browser-management/classes/RemoteBrowser.ts) L460-L582

```typescript
public initialize = async (userId: string): Promise<void> => {
  // ...
  // 创建全新 BrowserContext（干净、无 Cookie）
  const contextPromise = this.browser.newContext(contextOptions);
  this.context = await Promise.race([
    contextPromise,
    new Promise<never>((_, reject) => {
      setTimeout(() => reject(new Error('Context creation timed out after 15s')), 15000);
    })
  ]) as BrowserContext;
  // ... 创建 Page、初始化 rrweb 等 ...
};
```

### 10.3 完整时序图

```
前端 (RobotEditPage.handleSave)
    │
    ├─ 构造 payload (credentials 明文)
    │
    ▼
PUT /storage/recordings/:id (后端)
    │
    ├─ 从 DB 读取原始 workflow
    ├─ handleWorkflowActions(workflow, credentials)
    │   └─ 加密凭据值并替换 workflow 中的 type/press 动作
    ├─ 将修改后的 workflow 保存到 Robot.recording
    │
    ▼
前端 handleStart(robot)
    │
    ▼
PUT /storage/runs/:id (后端)
    │
    ├─ T0: 检查浏览器槽位
    ├─ T1: createRemoteBrowserForRun()
    │   ├─ 预留槽位
    │   └─ 异步调用 initializeBrowserAsync()
    │       └─ [后台] RemoteBrowser.initialize()
    │           └─ browser.newContext() → 全新 BrowserContext
    │
    ├─ T2: 创建 Run 记录，status='running'
    │   └─ 保存 interpreterSettings (formats、params 等，无 credentials)
    │
    ├─ T3: addJob(EXECUTE_RUN, {runId, browserId})
    │   └─ 任务入队到 Graphile Worker
    │
    ▼
Graphile Worker (后台任务)
    │
    ├─ T4: 消费任务 → processRunExecution()
    │
    ├─ T5: 轮询等待浏览器就绪（最多 60s）
    │   └─ while (!browserPool.getRemoteBrowser(browserId)) { sleep(2s) }
    │
    ├─ T6: 从 Robot 读取 recording（已包含加密凭据）
    │
    ├─ T7: browser.interpreter.InterpretRecording()
    │   ├─ processWorkflow() → decrypt() 还原明文凭据
    │   └─ Playwright 依次执行动作:
    │       ├─ goto 登录页
    │       ├─ type 用户名
    │       ├─ type 密码
    │       ├─ click 登录按钮
    │       └─ 浏览器获得登录态 Cookie
    │
    └─ T8: 执行完成 → 更新 Run 状态 → 销毁浏览器
```

### 10.4 关键时序关联

| 阶段 | 时间点 | 关键操作 | 与凭据的关联 |
|------|--------|----------|-------------|
| 编辑保存 | T-编辑 | `handleWorkflowActions()` 加密凭据并替换 workflow | 凭据被烧录到 workflow |
| Run 启动 | T0-T3 | 创建浏览器 → 创建 Run → 入队 | 不涉及凭据，workflow 已包含凭据 |
| 浏览器初始化 | T1-异步 | `browser.newContext()` 创建干净上下文 | 无 Cookie，等待后续登录动作 |
| 任务执行 | T4-T5 | 轮询等待浏览器就绪 | 不涉及凭据 |
| 解释执行 | T6-T7 | 读取 workflow → 解密 → 执行登录动作 | 解密凭据并在浏览器中输入 |
| 执行完成 | T8 | 销毁 BrowserContext | Cookie 随上下文销毁 |

### 10.5 设计权衡

**并行初始化的优点**：
- 浏览器初始化与任务入队并行进行，减少总等待时间
- 利用浏览器启动的"冷启动"时间进行数据库操作和队列调度

**潜在问题**：
- 如果浏览器初始化失败，任务已经入队，需要在 `processRunExecution()` 中处理失败状态
- 轮询等待增加了复杂度和延迟

---

## 十一、更新：关键代码位置速查（补充）

| 功能 | 文件 | 行号 |
|------|------|------|
| AES 加密 | [server/src/utils/auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/utils/auth.ts) | L26-L43 |
| AES 解密 | [server/src/utils/auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/utils/auth.ts) | L45-L60 |
| 录制时加密键盘输入 | [server/src/workflow-management/classes/Generator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/workflow-management/classes/Generator.ts) | L476 |
| 优化时加密合并的 type 动作 | [server/src/workflow-management/classes/Generator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/workflow-management/classes/Generator.ts) | L1506 |
| 保存 isLogin 标志 | [server/src/workflow-management/classes/Generator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/workflow-management/classes/Generator.ts) | L1099 |
| 创建全新 BrowserContext | [server/src/browser-management/classes/RemoteBrowser.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/browser-management/classes/RemoteBrowser.ts) | L522 |
| 执行前解密凭据 | [server/src/workflow-management/classes/Interpreter.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/workflow-management/classes/Interpreter.ts) | L28-L33 |
| **凭据覆盖（编辑保存时）** | [server/src/routes/storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/routes/storage.ts) | L290-L355 |
| **编辑配置保存入口** | [server/src/routes/storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/routes/storage.ts) | L373-L556 |
| **Run 启动入口** | [server/src/routes/storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/routes/storage.ts) | L997-L1131 |
| **Run 执行核心** | [server/src/task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/task-runner.ts) | L130-L581 |
| **任务队列注册** | [server/src/task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/task-runner.ts) | L631-L675 |
| **创建浏览器（Run）** | [server/src/browser-management/controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/browser-management/controller.ts) | L114-L137 |
| **Graphile Worker 入队** | [server/src/storage/graphileWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/storage/graphileWorker.ts) | L61-L71 |
| 前端编辑保存 | [src/components/robot/pages/RobotEditPage.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/src/components/robot/pages/RobotEditPage.tsx) | L1203-L1326 |
| 前端更新 API | [src/api/storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/src/api/storage.ts) | L115-L130 |
| 前端 Run API | [src/api/storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/src/api/storage.ts) | L288-L302 |
| 读取 Cookie 用于 where 匹配 | [maxun-core/src/interpret.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/maxun-core/src/interpret.ts) | L271 |
| Robot 数据模型 | [server/src/models/Robot.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/115-maxun/server/src/models/Robot.ts) | - |

---

## 十二、更新：设计特点与限制（补充）

### 新增设计特点
4. **提前烧录**：凭据在编辑保存时就被加密合并到 Workflow 中，Run 启动时无需额外处理
5. **并行初始化**：浏览器初始化与任务入队并行进行，优化了冷启动时间
6. **队列解耦**：通过 Graphile Worker 队列实现请求处理与实际执行的解耦

### 新增限制
5. **凭据明文传输**：编辑保存时 credentials 以明文通过 HTTP 发送（需 HTTPS 保护）
6. **无凭据历史**：每次保存都会覆盖 Workflow 中的值，无法追溯凭据变更历史
7. **排队任务无浏览器**：进入 `queued` 状态的 Run 不会立即创建浏览器，出队时才创建
8. **异步初始化风险**：浏览器初始化可能失败，但任务已入队，需额外的失败处理逻辑
