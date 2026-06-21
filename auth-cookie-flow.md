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
