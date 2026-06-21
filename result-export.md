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

## 十一、四层架构职责边界与 scrape/非 scrape 差异

> **修正说明**：之前的分析将 API/SDK/定时任务/后台队列的职责边界混淆了。实际上系统存在 **四套独立的执行代码** 和 **两个同名但不同的 handleRunRecording**。

### 11.1 四层架构与职责边界

| 层级 | 所在文件 | 核心职责 | **不做什么** |
|------|----------|----------|-------------|
| **API 同步等待层** | [sdk.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/api/sdk.ts)、[record.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/api/record.ts) | 接收 HTTP 请求、创建 Run 记录、建立 Socket、同步轮询等待结果、格式化 HTTP 响应 | 不直接执行工作流 |
| **执行层** | [record.ts#L720-L1314](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/api/record.ts#L720-L1314)、[scheduler/index.ts#L186-L832](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/scheduler/index.ts#L186-L832)、[task-runner.ts#L130-L582](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/task-runner.ts#L130-L582) | 实际执行浏览器操作、格式转换、截图上传、集成导出、数据库回写、Webhook 发送 | （注意：三套代码高度重复） |
| **调度层** | [schedule-worker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/schedule-worker.ts) | 定时轮询 Robot 表、分布式锁认领、计算下次执行时间、派发任务到队列 | 不执行工作流、不处理格式转换 |
| **后台队列层** | [task-runner.ts#L631-L675](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/task-runner.ts#L631-L675) | Graphile Worker 基础设施：任务队列、并发控制、失败重试、任务分发 | 不包含业务逻辑，只做任务路由 |

#### 关键澄清：两个同名但不同的 handleRunRecording

代码中存在 **两个不同文件、不同参数** 的 `handleRunRecording` 函数，极易混淆：

| 函数 | 所在文件 | 调用方 | 参数 | 用途 |
|------|----------|--------|------|------|
| `handleRunRecording(id, userId, runSource, requestedFormats, promptInstructions)` | [record.ts#L1403](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/api/record.ts#L1403) | API 路由、SDK 路由 | 5 个参数，后 3 个可选 | API/MCP/CLI/SDK 手动触发 |
| `handleRunRecording(id, userId)` | [scheduler/index.ts#L860](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/scheduler/index.ts#L860) | SCHEDULED_WORKFLOW 队列 | 仅 2 个参数 | 定时任务触发 |

同样，`createWorkflowAndStoreMetadata` 也有两个版本：

| 函数 | 所在文件 | Run 初始状态 | 是否写入动态参数 |
|------|----------|-------------|-----------------|
| `createWorkflowAndStoreMetadata(id, userId, runSource, requestedFormats, promptInstructions)` | [record.ts#L552](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/api/record.ts#L552) | `status: 'running'` | ✅ 将 formats 和 promptInstructions 写入 interpreterSettings |
| `createWorkflowAndStoreMetadata(id, userId)` | [scheduler/index.ts#L42](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/scheduler/index.ts#L42) | `status: 'scheduled'` | ❌ 不写入动态参数 |

---

### 11.2 三套重复的执行层代码

执行层的核心逻辑（scrape 格式转换、非 scrape 工作流执行、截图上传、集成导出、Webhook）在 **三个文件中重复实现**：

| 文件 | 核心函数 | 触发场景 | source 标记 | 最大重试次数 |
|------|----------|----------|-------------|-------------|
| [api/record.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/api/record.ts) | `executeRun(id, userId)` | API/MCP/CLI 手动触发 | `"api"` | 0（无重试） |
| [workflow-management/scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/scheduler/index.ts) | `executeRun(id, userId)` | 定时任务触发 | `"scheduled"` | 3 次（retryCount 检查） |
| [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/task-runner.ts) | `processRunExecution(data)` | EXECUTE_RUN 队列（文档机器人、前端手动运行） | `"manual"` | 由 Graphile Worker 控制 |

三套代码的业务逻辑 **几乎完全相同**，差异仅在：
- 错误日志的前缀不同
- telemetry capture 的 source 字段不同
- Abort 检查粒度不同（task-runner.ts 最多节点）
- task-runner.ts 额外支持 doc-extract/doc-parse 文档机器人

---

### 11.3 scrape vs 非 scrape 的核心差异（最重要）

**scrape 类型和非 scrape 类型（extract/crawl/search）走的是完全不同的执行路径**，这是之前文档最大的遗漏：

| 对比维度 | scrape 机器人 | 非 scrape 机器人（extract/crawl/search） |
|----------|--------------|----------------------------------------|
| **是否经过 InterpretRecording** | ❌ **不经过**，直接调用格式转换函数 | ✅ **经过**，调用 `browser.interpreter.InterpretRecording()` |
| **是否经过批量持久化缓冲区** | ❌ **不经过**，数据直接组装 | ✅ **经过**，通过 persistenceBuffer 批量写库 |
| **数据写库时机** | 全部完成后 **一次性 update** | 执行中 **实时批量写库**（Interpreter 回调）+ 完成后追加 update |
| **格式转换入口** | 直接调用 `convertPageToMarkdown/HTML/Text/Links/Screenshot()` | InterpretRecording → `processRobotOutputFormats()`（仅 crawl/search） |
| **截图获取方式** | `convertPageToScreenshot(url, page, fullPage)` 直接生成 | InterpretRecording 的 `binaryCallback` 回调收集 |
| **Interpreter.setRunId()** | ❌ 不调用 | ✅ 调用，用于实时持久化绑定 |
| **输出格式** | markdown/html/text/links/screenshot-visible/screenshot-fullpage/summary/promptResult | scrapeSchema/scrapeList/crawl/search + 后处理派生格式 |

#### 差异详解：格式转换

**scrape 机器人路径**（完全绕开 Interpreter）：
```typescript
// api/record.ts / scheduler/index.ts / task-runner.ts 中都有这段逻辑
if (recording.recording_meta.type === 'scrape') {
    const formats = run.interpreterSettings?.formats || recording.recording_meta.formats || ['markdown'];
    
    // 直接调用格式转换函数，不经过 InterpretRecording
    if (formats.includes('markdown')) {
        markdown = await convertPageToMarkdown(url, currentPage);
        serializableOutput.markdown = [{ content: markdown }];
    }
    if (formats.includes('html')) {
        html = await convertPageToHTML(url, currentPage);
        serializableOutput.html = [{ content: html }];
    }
    if (formats.includes('text')) {
        text = await convertPageToText(url, currentPage);
        serializableOutput.text = [{ content: text }];
    }
    if (formats.includes('links')) {
        links = await convertPageToLinks(url, currentPage);
        serializableOutput.links = links.map(link => ({ url: link }));
    }
    if (formats.includes('screenshot-visible')) {
        buf = await convertPageToScreenshot(url, currentPage, false);
        binaryOutput['screenshot-visible'] = { data: buf.toString('base64'), mimeType: 'image/png' };
    }
    if (formats.includes('screenshot-fullpage')) {
        buf = await convertPageToScreenshot(url, currentPage, true);
        binaryOutput['screenshot-fullpage'] = { data: buf.toString('base64'), mimeType: 'image/png' };
    }
    if (formats.includes('summary')) {
        summaryText = await summarizeMarkdown(markdown, llmConfig);
        serializableOutput.summary = [{ content: summaryText }];
    }
    // promptInstructions → executeBrowserAgent()
}
```
代码位置（以 API 层为例）：[record.ts#L805-L958](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/api/record.ts#L805-L958)

**非 scrape 机器人路径**（经过 Interpreter + 批量持久化）：
```typescript
// 1. 设置 Run ID，启用实时持久化
browser.interpreter.setRunId(plainRun.runId);

// 2. 调用工作流解释器执行 —— 内部通过回调实时批量写库
const interpretationInfo = await browser.interpreter.InterpretRecording(
    AddGeneratedFlags(recording.recording),
    currentPage,
    (newPage) => currentPage = newPage,
    plainRun.interpreterSettings
);

// 3. crawl/search 类型额外进行格式后处理
if (robotType === 'crawl' || robotType === 'search') {
    const processedOutput = await processRobotOutputFormats({
        robotType,
        outputFormats,
        categorizedOutput,
        currentPage,
        initialBinaryOutput,
        llmConfig,
    });
    // 后处理结果追加写库
}
```
代码位置（以 API 层为例）：[record.ts#L1112-L1167](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/api/record.ts#L1112-L1167)

#### 差异详解：截图上传

| 机器人类型 | 截图数据来源 | 上传时机 |
|-----------|-------------|---------|
| **scrape** | `convertPageToScreenshot()` 直接返回 Buffer | 所有格式转换完成后 → 组装 serializableOutput + binaryOutput → 一次性写库 → 立即上传 MinIO |
| **非 scrape** | InterpretRecording 的 `binaryCallback` 回调 → 暂存 interpreter.binaryData → **实时写库**（persistBinaryDataToDatabase） | InterpretRecording 完成后从数据库读取已有 binaryOutput → 加上后处理新增截图 → 统一上传 MinIO → 更新数据库 |

**关键区别**：scrape 的截图数据 **从未进入 Interpreter 的内存数据结构**，完全在 executeRun 函数内局部变量中流转；非 scrape 的截图在 Interpreter 内部产生，通过回调实时写入数据库。

#### 差异详解：集成导出

scrape 和非 scrape 的集成导出逻辑 **完全相同**，都调用同一个 `triggerIntegrationUpdates()` 函数：
1. scrape：格式转换全部完成后调用
2. 非 scrape：InterpretRecording + 格式后处理全部完成后调用

该函数内部逻辑：
```typescript
addGoogleSheetUpdateTask(runId, { robotId, runId, status: 'pending', retries: 5 });
addAirtableUpdateTask(runId, { robotId, runId, status: 'pending', retries: 5 });
withTimeout(processAirtableUpdates(), 65000, 'Airtable update');
withTimeout(processGoogleSheetUpdates(), 65000, 'Google Sheets update');
```

代码位置：[record.ts#L665-L689](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/api/record.ts#L665-L689)

#### 差异详解：数据库回写

| 机器人类型 | 写库次数 | 写库内容 | 写库时机 |
|-----------|---------|---------|---------|
| **scrape** | **2 次** | 1. status = running（创建时）<br>2. status = success/failed + serializableOutput + binaryOutput（全部完成后一次性） | 全部格式转换完成后 |
| **非 scrape** | **N + 2 次以上** | 1. status = running（创建时）<br>2. 多次实时写入 scrapeSchema/scrapeList/crawl/search（批量缓冲区触发）<br>3. 多次实时写入 binaryOutput（截图回调触发）<br>4. crawl/search 后处理结果追加 update<br>5. status = success/failed + log + 最终 binaryOutput | 执行过程中持续写入 + 最终写入 |

数据库写库次数差异图示：
```
scrape:
  [创建] → running → [等待格式转换] → [一次性写入所有输出] → success/failed
            1次                                  1次                  共2次

非 scrape (extract/crawl/search):
  [创建] → running → InterpretRecording(执行中持续批量写库) → 后处理 → 最终状态
            1次         N次(每次满5条或3秒) + M次(截图)      1次     1次   共N+M+3次
```

---

### 11.4 完整触发路径（修正版）

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                            API 同步等待层（不执行）                                │
│  [record.ts] POST /api/record/run/:id                                            │
│  [sdk.ts]    POST /api/sdk/robots/:id/execute                                    │
│        │                                                                         │
│        ▼                                                                         │
│  handleRunRecording(record.ts, 5参数)                                             │
│        │  → createWorkflowAndStoreMetadata(创建 Run, status='running')           │
│        │  → 建立 Socket 连接，监听 ready-for-run                                  │
│        ▼                                                                         │
│  waitForRunCompletion（仅 SDK，每 2 秒轮询数据库）                                 │
│        │                                                                         │
│        ▼                                                                         │
│  标准化响应（提取 textData/listData/crawlData/searchData/text/markdown/html...） │
└──────────────────────────────────┬───────────────────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              调度层（不执行）                                      │
│  [schedule-worker.ts]                                                            │
│        │                                                                         │
│        ▼                                                                         │
│  setInterval(processDueSchedules, 30000)  // 每 30 秒轮询                        │
│        │                                                                         │
│        ▼                                                                         │
│  claimDueDbSchedules()                                                           │
│    ├─ pg_try_advisory_xact_lock(43821742)    // 分布式咨询锁                      │
│    ├─ FOR UPDATE SKIP LOCKED                  // 行级锁                          │
│    └─ 更新 schedulerClaimedAt = now           // 认领标记                        │
│        │                                                                         │
│        ▼                                                                         │
│  addJob(QUEUE_NAMES.SCHEDULED_WORKFLOW, { robotMetaId, userId })                 │
│        │                                                                         │
│        ▼                                                                         │
│  finalizeSchedule() → 计算 nextRunAt、更新 lastRunAt                             │
└──────────────────────────────────┬───────────────────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           后台队列层（只做路由）                                   │
│  Graphile Worker (task-runner.ts)                                                │
│        │                                                                         │
│        ├─ QUEUE_NAMES.SCHEDULED_WORKFLOW → handleRunRecording(scheduler/, 2参数) │
│        │       → createWorkflowAndStoreMetadata(创建 Run, status='scheduled')    │
│        │       → 建立 Socket → 监听 ready-for-run                                │
│        │                                                                         │
│        └─ QUEUE_NAMES.EXECUTE_RUN → processRunExecution()  // 文档机器人等       │
└──────────────────────────────────┬───────────────────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                          执行层（三套重复代码）                                    │
│                                                                                  │
│  if (type === 'scrape') {                  if (type !== 'scrape') {              │
│    // 直接格式转换                                // InterpretRecording           │
│    convertPageToMarkdown()                       browser.interpreter.setRunId()  │
│    convertPageToHTML()                            browser.interpreter.           │
│    convertPageToText()                              InterpretRecording()         │
│    convertPageToLinks()                           // 内部批量实时写库             │
│    convertPageToScreenshot()                      ↓                             │
│    summarizeMarkdown()                        crawl/search:                     │
│    executeBrowserAgent()                        processRobotOutputFormats()     │
│  }                                            }                                  │
│        │                                                                         │
│        ▼                                                                         │
│  BinaryOutputService.uploadAndStoreBinaryOutput() → MinIO                        │
│        │                                                                         │
│        ▼                                                                         │
│  sendWebhook() → 外部 Webhook URL                                                │
│        │                                                                         │
│        ▼                                                                         │
│  triggerIntegrationUpdates() → Google Sheets + Airtable                         │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

### 11.5 scrape vs 非 scrape 差异汇总表

| 环节 | scrape | extract | crawl | search |
|------|--------|---------|-------|--------|
| **经过 InterpretRecording** | ❌ | ✅ | ✅ | ✅ |
| **经过批量持久化缓冲区** | ❌ | ✅ | ✅ | ✅ |
| **实时写库** | ❌（一次性） | ✅ | ✅ | ✅ |
| **调用 processRobotOutputFormats** | ❌ | ❌ | ✅ | ✅ |
| **输出到 serializableOutput 的 key** | markdown/html/text/links/summary/promptResult/scrape | scrapeSchema/scrapeList | crawl + 派生的 markdown/html/text/links/summary | search + 派生的 markdown/html/text/links/summary |
| **截图数据来源** | convertPageToScreenshot() | binaryCallback | binaryCallback + 后处理新增 | binaryCallback + 后处理新增 |
| **setRunId() 调用** | ❌ | ✅ | ✅ | ✅ |
| **格式参数来源** | interpreterSettings.formats → recording_meta.formats → 默认 markdown | recording_meta.formats | interpreterSettings.formats → recording_meta.formats → 默认 markdown | interpreterSettings.formats → recording_meta.formats（discover 模式默认空数组，scrape 模式默认 markdown） |

---

### 11.6 触发入口代码索引（修正版）

| 文件 | 层级 | 关键函数 | 说明 |
|------|------|----------|------|
| [sdk.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/api/sdk.ts) | API 同步等待层 | `POST /api/sdk/robots/:id/execute`、`waitForRunCompletion` | SDK 入口：调用 record.ts 的 handleRunRecording + 同步轮询 |
| [record.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/api/record.ts) | API 同步等待层 + 执行层 | `handleRunRecording(5参数)`、`POST /api/record/run/:id`、`createWorkflowAndStoreMetadata(5参数)`、`executeRun(id, userId)` | **两套职责在同一文件**：API 入口 + 实际执行逻辑 |
| [schedule-worker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/schedule-worker.ts) | 调度层 | `claimDueDbSchedules`、`processDueSchedules`、`finalizeSchedule` | 只负责定时扫描、分布式锁、派发任务到队列 |
| [scheduler/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/workflow-management/scheduler/index.ts) | 执行层 | `handleRunRecording(2参数)`、`createWorkflowAndStoreMetadata(2参数)`、`executeRun(id, userId)` | 定时任务的实际执行逻辑（与 record.ts 中 executeRun 高度重复） |
| [task-runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/113-maxun/server/src/task-runner.ts) | 后台队列层 + 执行层 | `QUEUE_NAMES`、`processRunExecution`、`abortRun`、Graphile Worker 任务列表 | **两套职责在同一文件**：Worker 基础设施 + EXECUTE_RUN 队列的实际执行逻辑 |


