# 抓取结果从采集到导出持久化数据流详解

## 一、整体架构概览

数据从采集到最终导出，经历了 **六个阶段**：

```
采集层(maxun-core) → 内存暂存层(Interpreter) → 批量持久化层(Buffer) → 
数据库存储层(PostgreSQL) → 格式转换层(Post-processor) → 导出集成层
```

各阶段的核心职责：

| 阶段 | 核心模块 | 数据形态 | 存储介质 |
|------|----------|----------|----------|
| 采集层 | maxun-core/Interpreter | 原始抓取数据 | 内存（回调传递） |
| 内存暂存层 | WorkflowInterpreter | 分类结构化数据 | 内存（对象属性） |
| 批量持久化层 | persistenceBuffer | 批次数据 | 内存（队列） |
| 数据库存储层 | Run 模型 | JSONB 格式 | PostgreSQL |
| 格式转换层 | output-post-processor | 多格式输出 | 内存 + 数据库 |
| 导出集成层 | gsheet/airtable/webhook | 目标平台格式 | 外部系统 |

---

## 二、采集层：数据源头

### 2.1 数据采集入口

数据采集由 `maxun-core` 库的 `Interpreter` 类执行，通过两个回调函数向外传递数据：

- **serializableCallback**：传递可序列化数据（文本、列表、爬取结果、搜索结果）
- **binaryCallback**：传递二进制数据（截图等）

代码位置：[Interpreter.ts#L284-L330](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/classes/Interpreter.ts#L284-L330)

### 2.2 数据类型分类

采集数据分为两大类、五小类：

**可序列化数据（serializableData）：**
- `scrapeSchema`：单条/多条结构化文本提取
- `scrapeList`：列表型数据提取（表格、商品列表等）
- `crawl`：爬虫整页数据
- `search`：AI 搜索结果

**二进制数据（binaryData）：**
- `screenshot`：截图数据（base64 编码）

---

## 三、内存暂存层：数据临时汇聚

### 3.1 内存存储结构

`WorkflowInterpreter` 类在内存中维护两个核心数据结构：

#### 3.1.1 可序列化数据暂存

```typescript
public serializableDataByType: {
  scrapeSchema: Record<string, any>;   // 键为动作名称，值为数据数组
  scrapeList: Record<string, any>;     // 键为列表名称，值为数据数组
  crawl: Record<string, any>;          // 键为爬取名称，值为数据数组
  search: Record<string, any>;         // 键为搜索名称，值为数据数组
} = {
  scrapeSchema: {},
  scrapeList: {},
  crawl: {},
  search: {},
};
```

代码位置：[Interpreter.ts#L91-L102](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/classes/Interpreter.ts#L91-L102)

#### 3.1.2 二进制数据暂存

```typescript
public binaryData: { 
  name: string;       // 截图名称
  mimeType: string;   // MIME 类型
  data: string        // base64 编码数据
}[] = [];
```

代码位置：[Interpreter.ts#L114](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/classes/Interpreter.ts#L114)

### 3.2 数据命名机制

每个采集动作都会生成一个唯一名称，用于区分不同的采集结果：

- **scrapeList** → "List 1", "List 2", ...
- **scrapeSchema** → "Text 1", "Text 2", ...
- **screenshot** → "Screenshot 1", "Screenshot 2", ...

命名逻辑：[getUniqueActionName](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/classes/Interpreter.ts#L466-L495)

### 3.3 数据格式规范化

在 `serializableCallback` 中，原始数据会经过格式规范化处理：

1. 提取动作名称（从嵌套对象中解析）
2. 将数据统一展平为数组格式
3. 按类型存入 `serializableDataByType`

核心处理逻辑：[Interpreter.ts#L609-L708](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/classes/Interpreter.ts#L609-L708)

---

## 四、批量持久化层：性能优化

### 4.1 批量持久化设计

为了避免频繁的数据库写入，系统采用 **批量 + 定时** 的持久化策略：

- **批量触发**：缓冲区达到 5 条时立即刷新
- **定时触发**：每 3 秒自动刷新一次
- **scrapeSchema 特殊处理**：立即刷新（数据量通常较小）

关键参数：
```typescript
private readonly BATCH_SIZE = 5;        // 批量大小
private readonly BATCH_TIMEOUT = 3000;  // 批量超时（毫秒）
private readonly MAX_PERSISTENCE_RETRIES = 3;  // 最大重试次数
```

代码位置：[Interpreter.ts#L149-L153](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/classes/Interpreter.ts#L149-L153)

### 4.2 持久化缓冲区

```typescript
private persistenceBuffer: Array<{
  actionType: string;       // 动作类型
  data: any;                // 数据内容
  listIndex?: number;       // 列表索引（仅 scrapeList）
  timestamp: number;        // 时间戳
  creditValidated: boolean; // 信用验证标记
}> = [];
```

代码位置：[Interpreter.ts#L139-L145](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/classes/Interpreter.ts#L139-L145)

### 4.3 批量写入流程

```
数据产生 → addToPersistenceBatch → persistenceBuffer
     ↓
触发条件满足？→ 是 → flushPersistenceBuffer → 数据库事务写入
     ↓（否）
scheduleBatchFlush → 定时器等待 → 超时后刷新
```

核心方法：
- [persistDataToDatabase](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/classes/Interpreter.ts#L502-L515)：数据持久化入口
- [addToPersistenceBatch](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/classes/Interpreter.ts#L799-L809)：加入批次
- [flushPersistenceBuffer](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/classes/Interpreter.ts#L827-L952)：刷新缓冲区到数据库

### 4.4 失败重试机制

持久化失败时，采用 **指数退避** 策略重试：

```
第 1 次重试：5秒后
第 2 次重试：10秒后
第 3 次重试：20秒后
超过 3 次：丢弃数据，记录错误
```

代码位置：[Interpreter.ts#L918-L944](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/classes/Interpreter.ts#L918-L944)

### 4.5 二进制数据的特殊处理

二进制数据（截图）**不经过批量缓冲区**，而是实时直接写入数据库：

```
binaryCallback → persistBinaryDataToDatabase → 直接更新 Run.binaryOutput
```

代码位置：[persistBinaryDataToDatabase](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/classes/Interpreter.ts#L521-L560)

---

## 五、数据库存储层：数据落地

### 5.1 Run 模型数据结构

运行结果存储在 `run` 表中，核心字段为两个 JSONB 类型：

| 字段 | 类型 | 说明 |
|------|------|------|
| `serializableOutput` | JSONB | 可序列化输出数据 |
| `binaryOutput` | JSONB | 二进制输出数据（或引用 URL） |

数据模型定义：[Run.ts#L32-L33](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/models/Run.ts#L32-L33)

### 5.2 serializableOutput 结构

```json
{
  "scrapeSchema": {
    "Text 1": [{ "字段1": "值1", "字段2": "值2" }],
    "Text 2": [...]
  },
  "scrapeList": {
    "List 1": [
      { "商品名": "A", "价格": "100" },
      { "商品名": "B", "价格": "200" }
    ]
  },
  "crawl": {
    "Crawl Results": [
      { "url": "...", "markdown": "...", "html": "..." }
    ]
  },
  "search": {
    "Search Results": {
      "mode": "scrape",
      "results": [...]
    }
  }
}
```

### 5.3 binaryOutput 结构

```json
{
  "Screenshot 1": {
    "data": "base64编码数据",
    "mimeType": "image/png"
  },
  "Screenshot 2": "https://minio.example.com/bucket/key"  // 上传后变为 URL
}
```

---

## 六、格式转换层：多格式输出

### 6.1 输出格式类型

系统支持多种输出格式，定义在 [output-formats.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/constants/output-formats.ts)：

| 格式 | 说明 | 适用机器人类型 |
|------|------|----------------|
| `markdown` | Markdown 格式文本 | scrape, crawl, search |
| `html` | 原始 HTML | scrape, crawl, search |
| `text` | 纯文本 | scrape, crawl, search |
| `links` | 链接列表 | scrape, crawl, search |
| `summary` | AI 摘要 | scrape, crawl, search |
| `screenshot-visible` | 可视区域截图 | scrape, crawl, search |
| `screenshot-fullpage` | 整页截图 | scrape, crawl, search |

默认格式：`['markdown']`

### 6.2 Scrape 机器人的格式转换

对于 `scrape` 类型机器人，格式转换在任务运行器中直接执行：

```
页面 URL → convertPageToMarkdown/HTML/Text/Links/Screenshot → 格式化输出
→ 存入 serializableOutput / binaryOutput
```

核心代码：[task-runner.ts#L239-L335](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/task-runner.ts#L239-L335)

输出格式：
```typescript
// serializableOutput
{
  markdown: [{ content: "..." }],
  html: [{ content: "..." }],
  text: [{ content: "..." }],
  links: [{ url: "..." }, ...],
  summary: [{ content: "..." }]
}

// binaryOutput
{
  "screenshot-visible": { data: "base64...", mimeType: "image/png" },
  "screenshot-fullpage": { data: "base64...", mimeType: "image/png" }
}
```

### 6.3 Crawl / Search 机器人的格式转换

对于 `crawl` 和 `search` 类型机器人，数据先采集后再进行格式后处理：

**处理流程：**
1. 工作流执行，原始数据存入数据库
2. 调用 `processRobotOutputFormats` 进行后处理
3. 根据配置的输出格式进行转换和裁剪
4. 更新数据库中的结果

核心代码：[output-post-processor.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/utils/output-post-processor.ts)

**转换规则：**
- `markdown`：HTML → Markdown 转换（使用 turndown）
- `html`：保留原始 HTML
- `text`：纯文本提取
- `links`：链接提取
- `summary`：调用 LLM 生成摘要
- `screenshot-visible`：捕获可视区域截图
- `screenshot-fullpage`：捕获整页截图

**数据裁剪逻辑：**
- 不需要的格式直接从结果中删除（如不需要 html 则 `delete pageResult.html`）
- 截图数据存入 `binaryOutput`，在结果中仅保留引用 key

---

## 七、二进制存储层：MinIO 对象存储

### 7.1 为什么需要 MinIO

数据库直接存储二进制 base64 数据会导致：
- 数据库体积迅速膨胀
- 查询性能下降
- 备份恢复困难

因此系统会将二进制数据上传到 MinIO 对象存储，数据库中只保存 URL。

### 7.2 上传流程

```
Run 执行完成 → BinaryOutputService.uploadAndStoreBinaryOutput
  → 遍历 binaryOutput 中的所有项
  → 转换为 Buffer
  → 上传到 MinIO bucket
  → 生成公开访问 URL
  → 更新 Run.binaryOutput 为 URL 映射
```

核心代码：[mino.ts#L99-L191](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/storage/mino.ts#L99-L191)

### 7.3 存储结构

```
MinIO Bucket: maxun-run-screenshots
└── {runId}/
    ├── Screenshot_1.png
    ├── Screenshot_2.png
    ├── crawl-1-screenshot-visible.png
    └── ...
```

### 7.4 调用时机

二进制上传发生在 **Run 执行成功后**，集成更新前：

代码位置：[task-runner.ts#L461-L462](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/task-runner.ts#L461-L462)

---

## 八、导出集成层：外部系统对接

### 8.1 触发时机

Run 执行成功（或部分失败但有数据）后，触发集成更新：

```typescript
await triggerIntegrationUpdates(plainRun.runId, plainRun.robotMetaId);
```

代码位置：[task-runner.ts#L508](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/task-runner.ts#L508)

### 8.2 Google Sheets 导出

#### 8.2.1 数据结构转换

从数据库的 JSONB 结构转换为表格行：

| 数据类型 | Sheet 名称 | 行结构 |
|----------|------------|--------|
| scrapeSchema | Schema - {组名} | 字段名作为列 |
| scrapeList | List - {列表名} | 字段名作为列 |
| markdown | Markdown | Index, Content |
| html | HTML | Index, Content |
| crawl | Crawl - {爬取名} | 字段名作为列 |
| search | Search Results | 字段名作为列 |
| 截图 | Screenshot | Screenshot Key, Screenshot URL |

核心代码：[gsheet.ts#L73-L183](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/integrations/gsheet.ts#L73-L183)

#### 8.2.2 写入机制

- **自动建表**：Sheet 不存在时自动创建
- **表头对齐**：首行写入字段名作为表头
- **追加写入**：新数据追加到已有数据下方
- **Token 刷新**：Access Token 过期自动刷新

核心代码：[writeDataToSheet](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/integrations/gsheet.ts#L275-L380)

#### 8.2.3 任务队列

Google Sheets 更新通过任务队列异步执行，支持重试：

```
addGoogleSheetUpdateTask → googleSheetUpdateTasks 队列
→ processGoogleSheetUpdates 循环处理
→ 成功：删除任务
→ 失败：重试计数 +1，最多 5 次
→ 超限：删除任务，记录错误
```

### 8.3 Airtable 导出

#### 8.3.1 数据合并策略

Airtable 采用 **行合并** 策略，将所有类型的数据合并到同一张表中：

```
mergeRelatedData 函数：
  → 收集所有类型的数据（schema、list、markdown、html、crawl、search）
  → 按索引对齐合并为一行
  → 长度不一致时，剩余数据单独成行
```

核心代码：[mergeRelatedData](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/integrations/airtable.ts#L68-L319)

#### 8.3.2 字段自动创建

Airtable 支持字段自动创建，根据数据值推断字段类型：

| 值类型 | Airtable 字段类型 |
|--------|-------------------|
| 数字 | number |
| 布尔值 | checkbox |
| 日期 | dateTime |
| URL | url |
| 数组（对象） | multipleRecordLinks |
| 数组（非对象） | multipleSelects |
| 其他 | singleLineText |

代码位置：[inferFieldType](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/integrations/airtable.ts#L613-L623)

#### 8.3.3 写入机制

- **批量写入**：每批 10 条记录
- **重试机制**：最多 3 次重试，指数退避
- **空记录清理**：写入前后清理空记录
- **Token 刷新**：401 错误时自动刷新 token

### 8.4 Webhook 导出

#### 8.4.1 触发事件

支持的 Webhook 事件：
- `run_completed`：运行成功完成
- `run_failed`：运行失败

#### 8.4.2 Payload 结构

```json
{
  "event_type": "run_completed",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "webhook_id": "uuid",
  "data": {
    "robot_id": "...",
    "run_id": "...",
    "robot_name": "...",
    "status": "success",
    "started_at": "...",
    "finished_at": "...",
    "extracted_data": {
      "captured_texts": { "Text 1": [...] },
      "captured_lists": { "List 1": [...] },
      "crawl_data": {...},
      "search_data": {...},
      "captured_texts_count": 5,
      "captured_lists_count": 6,
      "screenshots_count": 3
    },
    "metadata": { "browser_id": "...", "user_id": 123 }
  }
}
```

#### 8.4.3 重试机制

Webhook 发送失败时采用指数退避重试：

```
第 1 次失败 → 5秒后重试
第 2 次失败 → 10秒后重试
第 3 次失败 → 20秒后重试
超过最大重试次数 → 放弃
```

代码位置：[sendWebhookWithRetry](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/routes/webhook.ts#L437-L465)

---

## 九、完整数据流时序图

```
用户/调度器
    │
    ▼
┌─────────────┐
│  创建 Run   │  →  数据库：Run 记录（status=running, 空输出）
└─────────────┘
    │
    ▼
┌─────────────┐
│ 工作流执行  │  ← maxun-core Interpreter
└─────────────┘
    │
    ├─→ serializableCallback ─→ 内存：serializableDataByType
    │                                    │
    │                                    └─→ persistenceBuffer ──批量──→ 数据库：serializableOutput
    │
    └─→ binaryCallback ───────→ 内存：binaryData
                                       │
                                       └──────────────实时──────────────→ 数据库：binaryOutput
    │
    ▼
┌─────────────┐
│  执行完成   │
└─────────────┘
    │
    ▼
┌─────────────┐
│ 格式后处理  │  →  crawl/search 类型的格式转换
└─────────────┘
    │
    ▼
┌─────────────┐
│ MinIO 上传  │  →  二进制数据 → 对象存储，URL 写回数据库
└─────────────┘
    │
    ▼
┌─────────────┐
│ 集成导出    │  ──→ Google Sheets
└─────────────┘  ──→ Airtable
                 ──→ Webhook
```

---

## 十、关键代码文件索引

| 文件 | 核心职责 |
|------|----------|
| [Interpreter.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/classes/Interpreter.ts) | 工作流解释器，数据采集与内存暂存，批量持久化 |
| [Run.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/models/Run.ts) | Run 数据模型，数据库存储结构 |
| [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/task-runner.ts) | 任务运行器，执行 Run 并协调整个数据流 |
| [output-post-processor.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/utils/output-post-processor.ts) | 输出格式后处理 |
| [output-formats.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/constants/output-formats.ts) | 输出格式常量定义 |
| [mino.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/storage/mino.ts) | MinIO 二进制存储服务 |
| [gsheet.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/integrations/gsheet.ts) | Google Sheets 集成导出 |
| [airtable.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/integrations/airtable.ts) | Airtable 集成导出 |
| [webhook.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/routes/webhook.ts) | Webhook 导出 |
| [storage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/storage.ts) | 文件系统存储工具函数 |

---

## 十一、三种触发方式对比

系统支持三种 Run 触发方式，它们在执行模型、格式转换、数据回写等方面存在显著差异。

### 11.1 触发入口总览

| 触发方式 | 入口文件 | 核心函数 | 执行模型 |
|----------|----------|----------|----------|
| **API 手动触发（SDK）** | [sdk.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/api/sdk.ts) | `POST /api/sdk/robots/:id/execute` | 同步阻塞 + 后台异步执行 |
| **定时任务触发** | [schedule-worker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/schedule-worker.ts) | `processDueSchedules` | 异步队列（Graphile Worker） |
| **后台运行器** | [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/task-runner.ts) | `processRunExecution` | 异步队列（Graphile Worker） |

---

### 11.2 触发入口详解

#### 11.2.1 API 手动触发（SDK）

**调用链路：**
```
HTTP 请求 → POST /api/sdk/robots/:id/execute
    ↓
handleRunRecording(robotId, userId, runSource, requestedFormats, promptInstructions)
    ↓
createWorkflowAndStoreMetadata → 创建 Run 记录（status=scheduled）
    ↓
建立 Socket 连接 → 监听 ready-for-run 事件
    ↓
executeRun → 实际执行工作流
    ↓
waitForRunCompletion → 轮询数据库等待 Run 完成（最多 3 小时）
    ↓
数据提取与整理 → 返回标准化 JSON 响应
```

**核心特点：**
- **支持动态参数**：可在请求体中覆盖 `formats` 和 `promptInstructions`
  ```typescript
  const requestedFormats = req.body?.formats as OutputFormats[] | undefined;
  const promptInstructions = req.body?.promptInstructions;
  ```
  代码位置：[sdk.ts#L691-L692](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/api/sdk.ts#L691-L692)

- **同步等待结果**：通过 `waitForRunCompletion` 轮询数据库，每 2 秒检查一次 Run 状态
  代码位置：[waitForRunCompletion](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/api/sdk.ts#L806-L828)

- **响应数据标准化**：返回前对数据库原始数据进行结构整理
  ```typescript
  return {
    runId: run.runId,
    status: run.status,
    data: {
      textData: run.serializableOutput?.scrapeSchema || {},
      listData: listData,           // 从 scrapeList 提取
      crawlData: crawlData,         // 从 crawl 提取
      searchData: searchData,       // 从 search 提取
      text: text,                   // 从 text[0].content 提取
      markdown: markdown,           // 从 markdown[0].content 提取
      html: html,                   // 从 html[0].content 提取
      summary: summary,             // 从 summary[0].content 提取
      promptResult: promptResult    // 从 promptResult[0].content 提取
    },
    screenshots: Object.values(run.binaryOutput || {})
  };
  ```
  代码位置：[sdk.ts#L776-L793](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/api/sdk.ts#L776-L793)

#### 11.2.2 定时任务触发

**调度流程：**
```
startScheduleWorker()
    ↓
setInterval(processDueSchedules, 30000)  // 每 30 秒轮询一次
    ↓
claimDueDbSchedules()
    ├─ pg_try_advisory_xact_lock  // 分布式锁，防止多实例重复调度
    ├─ 查询 Robot 表中 schedule.nextRunAt <= now 的记录
    ├─ FOR UPDATE SKIP LOCKED     // 行级锁，跳过已被锁定的行
    └─ 更新 schedulerClaimedAt = now 标记认领
    ↓
addJob(QUEUE_NAMES.SCHEDULED_WORKFLOW, { robotMetaId, userId })
    ↓
Graphile Worker 消费 → handleRunRecording(robotMetaId, userId)
    ↓
finalizeSchedule()
    ├─ 计算 nextRunAt（通过 cron 表达式）
    ├─ 更新 lastRunAt 和 nextRunAt
    └─ 清除 schedulerClaimedAt
```

**调度配置参数：**
```typescript
DB_SCHEDULER_BATCH_SIZE = 10;          // 每批最多认领 10 个任务
DB_SCHEDULER_POLL_MS = 30000;           // 轮询间隔 30 秒
DB_SCHEDULER_CLAIM_TIMEOUT_MS = 600000; // 认领超时 10 分钟
```

代码位置：[schedule-worker.ts#L14-L17](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/schedule-worker.ts#L14-L17)

**分布式锁机制：**
```sql
SELECT pg_try_advisory_xact_lock(43821742) AS locked
```
使用 PostgreSQL 咨询锁确保同一时刻只有一个调度实例在认领任务。

代码位置：[claimDueDbSchedules](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/schedule-worker.ts#L29-L85)

#### 11.2.3 后台运行器

**任务队列：**
```
addJob(QUEUE_NAMES.EXECUTE_RUN, { userId, runId, browserId })
    ↓
Graphile Worker 消费 → processRunExecution(data)
```

队列定义：[task-runner.ts#L37-L45](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/task-runner.ts#L37-L45)

---

### 11.3 核心差异对比表

| 对比维度 | API 手动触发（SDK） | 定时任务触发 | 后台运行器 |
|----------|---------------------|-------------|-----------|
| **执行模型** | 同步 HTTP 阻塞 + 后台异步 | 纯异步队列 | 纯异步队列 |
| **入口函数** | `handleRunRecording(robotId, userId, runSource, requestedFormats, promptInstructions)` | `handleRunRecording(robotMetaId, userId)` | `processRunExecution({userId, runId, browserId})` |
| **Run 创建时机** | 调用时同步创建 | 调度认领时创建 | 任务入队前创建 |
| **动态 formats** | ✅ 支持（请求体覆盖） | ❌ 使用机器人默认配置 | ❌ 使用机器人默认配置 |
| **动态 promptInstructions** | ✅ 支持（请求体覆盖） | ❌ 使用机器人默认配置 | ✅ 支持（通过 interpreterSettings） |
| **返回结果** | 标准化 JSON 响应（同步） | 无直接返回（异步） | 无直接返回（异步） |
| **source 标记** | `sdk` 或 `cli` | `scheduled` | `manual` |
| **Abort 检查** | ❌ 无 | ✅ 有（executeRun 中检查） | ✅ 有（关键点多次检查） |
| **格式转换时机** | 执行中同步转换 | 执行中同步转换 | 执行中同步转换 |
| **截图上传时机** | 执行完成后立即上传 | 执行完成后立即上传 | 执行完成后立即上传 |
| **集成导出时机** | 执行完成后立即触发 | 执行完成后立即触发 | 执行完成后立即触发 |
| **最大等待时间** | 3 小时（API 层） | 10 分钟（工作流执行） | 10 分钟（工作流执行） |

---

### 11.4 各环节详细差异

#### 11.4.1 格式转换（Format Conversion）

| 机器人类型 | API 手动触发 | 定时任务触发 | 后台运行器 |
|-----------|-------------|-------------|-----------|
| **scrape 类型** | 优先使用 `requestedFormats`，无则用 `recording_meta.formats`，最后默认 `['markdown']` | 仅使用 `recording_meta.formats`，默认 `['markdown']` | 仅使用 `recording_meta.formats`，默认 `['markdown']` |
| **crawl/search 类型** | 后处理阶段使用 `recording_meta.formats` | 后处理阶段使用 `recording_meta.formats` | 后处理阶段使用 `recording_meta.formats` |

**代码差异：**
- API 触发：
  ```typescript
  const rawFormats = run.interpreterSettings?.formats || recording.recording_meta.formats;
  ```
  代码位置：[scheduler/index.ts#L269](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/scheduler/index.ts#L269)

- 后台运行器：
  ```typescript
  const rawFormats = run.interpreterSettings?.formats || recording.recording_meta.formats;
  ```
  代码位置：[task-runner.ts#L242](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/task-runner.ts#L242)

#### 11.4.2 截图上传（Binary Upload）

三种触发方式的截图上传逻辑 **完全一致**，都在执行完成后调用 `BinaryOutputService.uploadAndStoreBinaryOutput`：

```typescript
const binaryOutputService = new BinaryOutputService('maxun-run-screenshots');
const uploadedBinaryOutput = Object.keys(binaryOutput).length > 0
  ? await binaryOutputService.uploadAndStoreBinaryOutput(run, binaryOutput)
  : {};
```

代码位置：
- 定时任务：[scheduler/index.ts#L650-L653](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/scheduler/index.ts#L650-L653)
- 后台运行器：[task-runner.ts#L461-L463](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/task-runner.ts#L461-L463)

#### 11.4.3 集成导出（Integration Export）

三种触发方式的集成导出逻辑 **完全一致**，都调用 `triggerIntegrationUpdates`：

```typescript
await triggerIntegrationUpdates(plainRun.runId, plainRun.robotMetaId);
```

该函数内部：
1. 添加 Google Sheets 更新任务到队列
2. 添加 Airtable 更新任务到队列
3. 异步执行 `processAirtableUpdates()` 和 `processGoogleSheetUpdates()`
4. 每个任务有 65 秒超时限制

代码位置：
- 定时任务：[scheduler/index.ts#L755](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/scheduler/index.ts#L755)
- 后台运行器：[task-runner.ts#L508](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/task-runner.ts#L508)

#### 11.4.4 数据库回写（Database Write-back）

| 触发方式 | 数据库回写时机 | 状态流转 |
|----------|----------------|----------|
| **API 手动触发** | 执行中实时批量写入 + 完成时最终更新 | scheduled → running → success/failed |
| **定时任务触发** | 执行中实时批量写入 + 完成时最终更新 | scheduled → running → success/failed |
| **后台运行器** | 执行中实时批量写入 + 完成时最终更新<br>**关键点检查 Run 是否被 Abort** | scheduled → running → success/failed/aborted |

**Abort 检查差异**：

后台运行器在多个关键点检查 Run 是否被中止：
1. 执行开始前检查 `run.status === 'aborted' || run.status === 'aborting'`
2. 工作流解释完成后检查 `await isRunAborted()`
3. 格式后处理后检查 `await isRunAborted()`

```typescript
const isRunAborted = async (): Promise<boolean> => {
  const currentRun = await Run.findOne({ where: { runId: data.runId } });
  return currentRun ? (currentRun.status === 'aborted' || currentRun.status === 'aborting') : false;
};
```

代码位置：[task-runner.ts#L381-L386](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/task-runner.ts#L381-L386)

#### 11.4.5 Webhook 触发

| 触发方式 | Webhook Payload 差异 |
|----------|---------------------|
| **API 手动触发** | 与定时任务、后台运行器一致，包含完整的 extracted_data |
| **定时任务触发** | 包含完整的 extracted_data（captured_texts、captured_lists、crawl_data、search_data） |
| **后台运行器** | 包含完整的 extracted_data，失败时额外包含 partial_data_extracted 标记 |

---

### 11.5 触发路径完整流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              触发入口层                                      │
├─────────────────────────────────┬───────────────────────────┬───────────────┤
│  API /sdk/robots/:id/execute    │  schedule-worker 轮询     │  前端手动运行  │
│  (同步等待)                     │  (30秒轮询 + 分布式锁)    │  (入队执行)    │
└─────────────────┬───────────────┴─────────────┬─────────────┴───────────────┘
                  │                             │
                  ▼                             ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          handleRunRecording 入口                             │
│  参数：(robotId, userId, runSource?, requestedFormats?, promptInstructions?)│
│  功能：创建 Run 记录 → 建立 Socket → 触发 executeRun                        │
└───────────────────────────────────────────┬─────────────────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          executeRun (scheduler/index.ts)                    │
│  scrape 机器人：直接格式转换 → 截图上传 → 集成导出                           │
│  其他机器人：InterpretRecording → 格式后处理 → 截图上传 → 集成导出            │
│  source 标记：scheduled                                                      │
└───────────────────────────────────────────┬─────────────────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        processRunExecution (task-runner.ts)                 │
│  scrape 机器人：直接格式转换 → 截图上传 → 集成导出                           │
│  其他机器人：InterpretRecording → 格式后处理 → 截图上传 → 集成导出            │
│  ✅ 多节点 Abort 检查                                                        │
│  source 标记：manual                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 11.6 触发入口代码索引

| 文件 | 触发方式 | 关键函数 |
|------|----------|----------|
| [sdk.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/api/sdk.ts) | API 手动触发 | `POST /api/sdk/robots/:id/execute`、`waitForRunCompletion` |
| [schedule-worker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/schedule-worker.ts) | 定时任务触发 | `claimDueDbSchedules`、`processDueSchedules`、`finalizeSchedule` |
| [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/scheduler/index.ts) | 定时任务执行 | `handleRunRecording`、`createWorkflowAndStoreMetadata`、`executeRun` |
| [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/task-runner.ts) | 后台运行器 | `processRunExecution`、`abortRun`、`QUEUE_NAMES` |

