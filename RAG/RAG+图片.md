#  Image 提取与前端渲染完整链路

[[OCR]]  

> 本文档涵盖来料检验模块中图片从 RAGFlow 提取、前后端交互、入库、前端渲染到页面刷新后持久化渲染的完整链路。后续维护只需阅读本文档即可理解前后端协调逻辑。

  

---

  

## 0. 核心概念

  

| 术语 | 说明 |

|------|------|

| `imageId` | RAGFlow 内部图片 ID（字符串），需通过后端代理获取图片二进制 |

| `imageUrl` | Markdown chunk content 中提取的图片 URL（以 `http` 开头的完整 URL），可直接作为 `<Image>` src |

| `aiMatchedImageIds` | 数据库字段（`String[]`），**同时存储 imageId 和 imageUrl**，不区分类型 |

| `sources` | 同步结果中每个检索来源的信息，包含各自的 imageId/imageUrls |

  

**URL/ID 判断规则**：`value.startsWith('http')` → URL，否则为 imageId。

  

---

  

## 1. 后端提取（sync-standards.ts）

  

### 1.1 数据来源

  

RAGFlow 检索返回的 `RetrievalChunk` 类型：

  

```typescript

interface RetrievalChunk {

  documentId?: string;

  documentName?: string;

  content: string | { markdown?: string; ... };  // 可能是对象或字符串

  similarity?: number;

  imageId?: string;  // RAGFlow chunk 解析时自动提取的文档图片引用

}

```

  

图片数据有两个来源：

- **imageId**：RAGFlow chunk 自带的 `imageId` 字段

- **imageUrl**：从 chunk content 的 Markdown 文本中提取（正则匹配 `![alt](url)` 格式）

  

### 1.2 提取逻辑

  

位置：`sync-standards.ts` ~line 828-833

  

```typescript

// 取 rerank top1 chunk

const top1Chunk = reranked[0];

  

// imageId：只取 top1 chunk 的 imageId

const imageIds = top1Chunk?.imageId ? [top1Chunk.imageId] : [];

  

// imageUrl：从 top1 chunk content 的 Markdown 文本中提取

const imageUrls = top1Chunk

  ? extractMarkdownImageUrls(ragflowChunkText(top1Chunk.content))

  : [];

```

  

**关键决策**：imageIds 和 imageUrls 都**只取 rerank top1 chunk**，其他 chunk 的图片丢弃。

  

### 1.3 返回类型（SyncIncomingInspectionResultRow）

  

位置：`apps/api/src/services/incoming-inspection/types.ts`

  

```typescript

export type SyncIncomingInspectionResultRow = {

  rowIndex: number;

  proposedStandard: string;

  found: boolean;

  retrievalScore?: number;

  rerankScore?: number;

  /** 每条来源携带各自的 imageId 和 imageUrls（展示用） */

  sources: Array<{

    documentName?: string;

    documentId?: string;

    excerpt: string;

    imageId?: string;

    imageUrls: string[];

  }>;

  /** rerank top1 chunk 关联的图片 ID（入库用） */

  imageIds: string[];

  /** rerank top1 chunk content 中提取的 Markdown 图片 URL（入库用，优先） */

  imageUrls: string[];

  error?: string;

  ragflowQuestion: string;

};

```

  

两处图片字段用途不同：

  

| 字段 | 来源 | 用途 |

|------|------|------|

| `sources[].imageId` | 每个 chunk 各自的 imageId | 前端来源 Popover 中展示图片 |

| `sources[].imageUrls` | 每个 chunk content 中提取 | 前端来源 Popover 中展示图片（优先） |

| `imageIds` | 仅 rerank top1 chunk | 用户点击「采纳」时入库 |

| `imageUrls` | 仅 rerank top1 chunk content | 用户点击「采纳」时入库（优先） |

  

---

  

## 2. 前端接收与渲染（incoming-inspection-sync-append-columns.tsx）

  

### 2.1 数据类型（IncomingSyncDiff）

  

```typescript

export type IncomingSyncDiff = {

  rowIndex: number;

  id: string;                          // DB 行 ID

  beforeValue: string;                 // 原始标准值

  proposedStandard: string;            // LLM 提取的标准值

  found: boolean;

  retrievalScore?: number;

  rerankScore?: number;

  sources: Array<{

    documentName?: string;

    documentId?: string;

    excerpt: string;

    imageId?: string;

    imageUrls?: string[];

  }>;

  error?: string;

  ragflowQuestion: string;

  draftAfter?: string;                 // 用户手工编辑后的值

  status?: 'pending' | 'accepted' | 'rejected' | 'edited';

  materialName?: string;

  inspectionCategory?: string;

  inspectionItem?: string;

  imageIds?: string[];                 // rerank top1 chunk 关联的图片 ID（同步结果 / DB 加载）

  imageUrls?: string[];                // rerank top1 chunk content 中提取的 Markdown 图片 URL

  selectedImages?: Array<{             // 用户选中的图片（用于采纳时入库，多选）

    type: 'url' | 'id';

    value: string;

  }>;

};

```

  

### 2.2 渲染位置 A：来源 Popover 中的图片

  

在「来源(N)」按钮弹出的 Collapse 面板中，每条来源摘录下方渲染图片：

  

```tsx

{ ((s.imageUrls && s.imageUrls.length > 0) || s.imageId) && (

  <div style={{ display: 'flex', flexWrap: 'wrap', gap: 8 }}>

    {s.imageUrls && s.imageUrls.length > 0 ? (

      // imageUrl 优先：直接作为 src

      s.imageUrls.map((url, idx) => (

        <Image key={url} src={url} alt={`参考图片 ${idx + 1}`} ... />

      ))

    ) : (

      // fallback：通过代理获取 imageId

      <Image

        key="proxy"

        src={`/api-proxy/v1/incoming-inspections/images/${s.imageId}`}

        alt="参考图片" ...

      />

    )}

  </div>

)}

```

  

### 2.3 渲染位置 B：J 列图片缩略图

  

J 列渲染器收集图片的完整逻辑：

  

```typescript

// 收集图片：优先取 top1 sources 的图片，其次 fallback 到 DB 加载的 imageIds

const allImages: Array<{ type: 'url' | 'id'; value: string }> = [];

const top1 = d.sources[0];

if (top1) {

  // 同步场景：sources 有数据

  if (top1.imageUrls && top1.imageUrls.length > 0) {

    for (const url of top1.imageUrls) {

      allImages.push({ type: 'url', value: url });

    }

  } else if (top1.imageId) {

    allImages.push({ type: 'id', value: top1.imageId });

  }

} else if (d.imageIds && d.imageIds.length > 0) {

  // DB 加载场景（页面刷新后）：sources 为空，从 imageIds 渲染

  // 按 http 前缀区分 URL/ID

  for (const img of d.imageIds) {

    allImages.push({ type: img.startsWith('http') ? 'url' : 'id', value: img });

  }

}

```

  

每张图片可点击勾选（checkbox），选中状态影响采纳时入库的图片。

  

### 2.4 渲染位置 C：采纳时入库

  

用户点击「采纳」时的图片提交逻辑（**有 URL 只存 URL，无则存 ID**）：

  

```typescript

let saveImageIds: string[] | undefined;

if (d.selectedImages && d.selectedImages.length > 0) {

  // 用户选了图片：直接取所有 value（URL 和 ID 混存）

  saveImageIds = d.selectedImages.map(s => s.value);

} else {

  // 未选择：URL 优先，有 URL 只存 URL，无则存 ID

  const urls = d.imageUrls ?? [];

  const ids = d.imageIds ?? [];

  saveImageIds = urls.length > 0 ? urls : (ids.length > 0 ? ids : undefined);

}

const ok = await onSaveToDb(record.key, displayValue, d.id, saveImageIds);

if (ok) {

  // 采纳成功后写回 diff：更新 imageIds 为实际保存的值，清除选中状态

  onDiffsChange(patchDiff(diffs, rowId, {

    draftAfter: undefined,

    status: isEdited ? 'edited' : 'accepted',

    imageIds: saveImageIds,

    selectedImages: undefined,

  }));

}

```

  

### 2.5 onSaveToDb 回调（page.tsx）

  

```typescript

onSaveToDb: async (rowIndex, standard, id, imageIds) => {

  await apiJson('/v1/incoming-inspections/standard', 'PATCH', {

    id,

    aiMatchedResult: standard,

    aiMatchedImageIds: imageIds?.length ? imageIds : undefined,

  });

}

```

  

**注意**：`imageIds` 为空（`undefined` 或 `[]`）时不传 `aiMatchedImageIds`，后端不执行 `set`，DB 中原有值保持不变（不会被清空）。

  

---

  

## 3. 后端 API 端点

  

### 3.1 PATCH `/v1/incoming-inspections/standard` — 采纳入库

  

位置：`apps/api/src/routes/v1.ts` ~line 1162-1210

  

```typescript

// 请求体

{

  id: string;

  aiMatchedResult?: string;

  aiMatchedImageIds?: string[];  // 同时包含 imageId 和 imageUrl

}

  

// 数据库更新

await app.prisma.incomingInspection.update({

  where: { id: body.id },

  data: {

    aiMatchedResult: body.aiMatchedResult,

    aiMatchedImageIds: { set: body.aiMatchedImageIds },

  },

});

```

  

### 3.2 GET `/v1/incoming-inspections/home-sheet` — 加载表格数据

  

位置：`apps/api/src/services/incoming-inspection-home-sheet.ts`

  

返回结构：

  

```typescript

type IncomingInspectionHomeSheetRow = {

  type: 'detail';

  cells: string[];       // A-G 列：物料名称、料号、检验类别、检验项目、标准、来源、采纳结果

  id?: string;

  aiMatchedImageIds?: string[];  // AI 匹配关联的图片（供刷新后重建 diffs）

};

```

  

查询：

  

```typescript

const items = await prisma.incomingInspection.findMany({

  where: { status: 'active' },

  orderBy: [{ materialName: 'asc' }, { inspectionCategory: 'asc' }],

});

  

const rows = items.map((it) => ({

  type: 'detail',

  cells: [

    it.materialName, it.materialCode, it.inspectionCategory,

    it.inspectionItem, it.standard, it.source,

    it.aiMatchedResult ?? '',

  ],

  id: it.id,

  aiMatchedImageIds: it.aiMatchedImageIds,

}));

```

  

### 3.3 POST `/v1/incoming-inspections/sync-standards` — 同步标准

  

位置：`apps/api/src/routes/v1.ts` ~line 991

  

调用 `syncIncomingInspectionStandards()`，返回 `SyncIncomingInspectionResultRow[]`，前端据此构建 `IncomingSyncDiff[]`。

  

---

  

## 4. 图片代理接口（fetch-image.ts）

  

### 4.1 代理 URL

  

```

GET /api-proxy/v1/incoming-inspections/images/{imageId}

```

  

### 4.2 代理逻辑

  

```

前端请求 → 后端代理 → RAGFlow 内部图片接口

                              ↓

              {RAGFLOW_IMAGE_BASE_URL}/v1/document/image/{imageId}

```

  

处理流程：

  

| 情况 | 返回 |

|------|------|

| `imageId` 为空或空白 | 1x1 透明 PNG 占位图 |

| RAGFlow 请求成功 (200) | 原始图片二进制，带 `Cache-Control: public, max-age=3600` |

| RAGFlow 请求失败 (非 200) | 1x1 透明 PNG 占位图 |

| 网络异常 | 1x1 透明 PNG 占位图 |

  

占位图使用 base64 编码的透明 PNG（`TRANSPARENT_PNG` 常量）。

  

---

  

## 5. 数据库 Schema

  

位置：`apps/api/prisma/schema.prisma` ~line 479-518

  

```prisma

model IncomingInspection {

  id                   String   @id @default(uuid()) @db.Uuid

  materialName         String   @map("material_name")

  materialCode         String   @map("material_code")

  inspectionCategory   String   @map("inspection_category")

  inspectionItem       String   @map("inspection_item") @db.Text

  standard             String   @map("standard") @db.Text

  source               String   @map("source")

  

  aiMatchedStandard    String   @default("") @map("ai_matched_standard") @db.Text

  aiMatchedImageIds    String[] @default([]) @map("ai_matched_image_ids")  // 同时存 imageId 和 imageUrl

  aiMatchedResult      String   @default("") @map("ai_matched_result") @db.Text

  

  remark               String?  @default("") @db.Text

  status               String   @default("active") @map("status")

  version              Int      @default(1) @map("version")

  tags                 String[] @default([]) @map("tags")

  

  createdAt            DateTime @default(now()) @map("created_at")

  updatedAt            DateTime @updatedAt @map("updated_at")

  

  @@index([materialCode])

  @@index([inspectionCategory])

  @@index([status])

  @@map("incoming_inspections")

}

```

  

**注意**：`aiMatchedImageIds` 是 `String[]` 类型，URL 和 ID 混存在同一字段中，前端通过 `startsWith('http')` 判断类型。

  

---

  

## 6. 完整数据流

  

### 6.1 场景 A：首次同步 → 采纳

  

```

RAGFlow 检索

  ↓

RetrievalChunk[].imageId + chunk content 中的 Markdown imageUrls

  ↓

rerank 重排

  ├─ top1 chunk.imageId → resultRow.imageIds       （入库用）

  ├─ top1 chunk content 中提取 imageUrls → resultRow.imageUrls  （入库用，优先）

  └─ 每个 chunk.imageId/imageUrls → resultRow.sources[]  （展示用）

  ↓

前端接收 SyncIncomingInspectionResultRow[] → 转为 IncomingSyncDiff[]

  ├─ sources[].imageUrls → 来源 Popover 中渲染 <Image>（优先）

  ├─ sources[].imageId → 来源 Popover 中渲染 <Image>（fallback）

  ├─ J 列图片缩略图 → sources[0] 的图片（imageUrls 优先，无则 imageId）

  └─ 用户可点击勾选图片（selectedImages）

  ↓

用户点击「采纳」

  ├─ 选了图片 → selectedImages.map(s => s.value)  （URL/ID 混存）

  └─ 未选图片 → URL 优先，有 URL 只存 URL，无则存 ID

      有 imageUrls → saveImageIds = imageUrls

      无 imageUrls 有 imageIds → saveImageIds = imageIds

      都为空 → 不存（DB 不变）

  ↓

PATCH /v1/incoming-inspections/standard

  { aiMatchedImageIds: [...] }   // 覆盖式 set，不是累加

  ↓

数据库 incoming_inspections.ai_matched_image_ids = [...]

  ↓

采纳成功后 patchDiff 写回 diff

  { status: 'accepted'/'edited', imageIds: saveImageIds, selectedImages: undefined }

  （供导出功能读取正确的图片）

```

  

### 6.2 场景 B：页面刷新后

  

```

数据库 incoming_inspections.ai_matched_image_ids (有数据)

  ↓

GET /v1/incoming-inspections/home-sheet

  → 返回 rows: { cells, id, aiMatchedImageIds }

  ↓

前端 loadIncomingInspectionForPanel

  → 重建 IncomingSyncDiff[]:

    {

      id: r.id,

      imageIds: r.aiMatchedImageIds,    // DB 中的图片

      sources: [],                       // DB 没有 sources

      status: r.cells[6] ? 'accepted' : 'pending',

    }

  ↓

J 列渲染 → diffForGridRow(diffs, rowId) 找到 diff

  ↓

sources 为空 → fallback 到 d.imageIds → 按 http 前缀区分 URL/ID

  ├─ URL (http...) → 直接作为 <Image> src

  └─ ID (非 http) → /api-proxy/v1/incoming-inspections/images/{id}

```

  

---

  

## 7. 图片渲染优先级总结

  

| 场景 | 图片来源 | 判断规则 |

|------|----------|----------|

| 来源 Popover | `sources[].imageUrls` → `sources[].imageId` | sources 有数据时优先取 sources |

| J 列缩略图（同步后） | `sources[0].imageUrls` → `sources[0].imageId` | 同上 |

| J 列缩略图（刷新后） | `d.imageIds[]` 按 `startsWith('http')` 区分 | sources 为空时 fallback |

| 采纳入库（勾选） | `selectedImages` | 用户选择优先 |

| 采纳入库（未勾选） | `imageUrls` → `imageIds` | URL 优先，有 URL 只存 URL，无则存 ID |

| 导出 | `selectedImages` → `diff.imageIds` → `sources[0]` | 三级 fallback |

  

---

  

## 8. 设计意图

  

| 决策 | 原因 |

|------|------|

| URL 和 ID 混存 `aiMatchedImageIds` 同一字段 | 不改动数据库结构；前端按 `http` 前缀判断类型即可 |

| imageUrl 优先于 imageId | URL 可直接渲染，无需代理请求，速度更快 |

| 采纳覆盖式写入（`set`） | 每次采纳代表最新的确认结果，不累加旧值 |

| 未传图片不执行 `set` | 避免无意识清空 DB 中的已有图片 |

| 采纳成功后 patchDiff 写回 imageIds | 导出功能能读取到正确的图片，与入库一致 |

| imageId 通过后端代理获取 | 避免前端直接访问 RAGFlow 内部网络的跨域和认证问题 |

| 失败回退占位图 | 保证 UI 布局不因图片缺失而错乱 |

| 刷新后从 DB 重建 diffs | 让 J 列等 append columns 在无同步缓存时仍能渲染 |

  

---

  

## 9. 相关代码位置

  

| 文件 | 职责 |

|------|------|

| `apps/api/src/services/incoming-inspection/sync-standards.ts` | RAGFlow 检索、rerank、LLM 提取、imageId/imageUrls 提取 |

| `apps/api/src/services/incoming-inspection/types.ts` | `SyncIncomingInspectionInputRow` / `SyncIncomingInspectionResultRow` 类型定义 |

| `apps/api/src/services/incoming-inspection-home-sheet.ts` | home-sheet API 返回 aiMatchedImageIds，供刷新后重建 diffs |

| `apps/api/src/services/incoming-inspection/fetch-image.ts` | 图片代理接口 handler |

| `apps/api/src/routes/v1.ts` | PATCH standard（采纳入库）、GET home-sheet、POST sync-standards |

| `apps/api/prisma/schema.prisma` | IncomingInspection 模型，aiMatchedImageIds 字段定义 |

| `apps/web/src/lib/incoming-inspection-sync-append-columns.tsx` | 来源 Popover 图片渲染、J 列缩略图、图片勾选、采纳入库逻辑 |

| `apps/web/src/app/page.tsx` | loadIncomingInspectionForPanel（DB 加载重建 diffs）、onSaveToDb（采纳回调）、同步结果构建 diff（含 imageUrls）、导出逻辑 |

  

---

  

────────────────┬───────────────────────────────────────────────────────────────┐

  │      场景      │                           导出用图                            │

  ├────────────────┼───────────────────────────────────────────────────────────────┤

  │ 用户勾选了图片 │ selectedImages（用户选的）                                    │

  ├────────────────┼───────────────────────────────────────────────────────────────┤

  │ 未勾选直接采纳 │ diff.imageIds（已更新为实际保存的 URL 或 ID）                 │

  ├────────────────┼───────────────────────────────────────────────────────────────┤

  │ 页面刷新后导出 │ diff.imageIds（从 DB 的 aiMatchedImageIds 重建，包含 URL+ID） │

  └────────────────┴───────────────────────────────────────────────────────────────┘

  

## 10. 常用 SQL

  

```sql

-- 查询存了 URL 的记录（即 aiMatchedImageIds 中包含 http 开头的值）

SELECT id, ai_matched_image_ids

FROM incoming_inspections

WHERE ai_matched_image_ids::text LIKE '%http%';

  

-- 统计有多少条记录存了 URL

SELECT COUNT(*) FROM incoming_inspections

WHERE EXISTS (

  SELECT 1 FROM unnest(ai_matched_image_ids) AS img

  WHERE img LIKE 'http%'

);

  

-- 查看某条记录的图片数据

SELECT id, material_name, ai_matched_image_ids

FROM incoming_inspections

WHERE id = '<uuid>';

```

  

## 11. 图片选择与存库完整操作逻辑

  

### 11.1 图片来源决策

  

每行图片来自 **rerank top1 chunk**（见 section 1.2），有两种形态：

  

| 形态 | 字段 | 来源 |

|------|------|------|

| URL | `imageUrls` | 从 top1 chunk content 的 Markdown 中提取 `![alt](url)` |

| ID | `imageIds` | RAGFlow chunk 自带的 `imageId` |

  

**注意**：只取 top1 的图片，其他 chunk 的图片丢弃。

  

### 11.2 图片选择交互（前端）

  

用户在 J 列「选择图例」中有两种操作：

  

| 操作 | 结果 |

|------|------|

| **不勾选任何图片** | 采纳时使用默认逻辑（见 11.3） |

| **手动勾选图片** | 采纳时存用户选中的图片 |

  

勾选状态存储在 `IncomingSyncDiff.selectedImages` 中：

```typescript

selectedImages?: Array<{ type: 'url' | 'id'; value: string }>;

```

  

### 11.3 采纳时的图片入库逻辑

  

位置：`incoming-inspection-sync-append-columns.tsx` 采纳按钮 onClick

  

```typescript

let saveImageIds: string[] | undefined;

if (d.selectedImages && d.selectedImages.length > 0) {

  // 用户手动勾选了图片 → 存用户选的值

  saveImageIds = d.selectedImages.map(s => s.value);

} else {

  // 未勾选 → URL 优先，有 URL 只存 URL，无则存 ID

  const urls = d.imageUrls ?? [];

  const ids = d.imageIds ?? [];

  saveImageIds = urls.length > 0 ? urls : (ids.length > 0 ? ids : undefined);

}

```

  

**决策优先级**：

1. 用户勾选了图片 → 存用户选的全部 value

2. 未勾选 → 优先存 `imageUrls`（Markdown URL）

3. 无 URL → 存 `imageIds`（ID 或 URL 混存）

4. 都为空 → `undefined`（不存，DB 保持不变）

  

### 11.4 入库 API

  

前端调用：`page.tsx` 中的 `onSaveToDb` 回调

```typescript

onSaveToDb: async (rowIndex, standard, id, imageIds) => {

  await apiJson('/v1/incoming-inspections/standard', 'PATCH', {

    id,

    aiMatchedResult: standard,

    aiMatchedImageIds: imageIds?.length ? imageIds : undefined,

  });

}

```

  

后端处理：`v1.ts` PATCH `/v1/incoming-inspections/standard`

```typescript

await app.prisma.incomingInspection.update({

  where: { id: body.id },

  data: {

    aiMatchedResult: body.aiMatchedResult,

    aiMatchedImageIds: { set: body.aiMatchedImageIds },  // 覆盖式 set

  },

});

```

  

**关键点**：

- `set` 是**覆盖式写入**，不是追加

- `imageIds` 为空（`undefined` 或 `[]`）时不传字段 → DB 中原有值保持不变

  

### 11.5 重试（重新同步）逻辑

  

**只点重试，不点采纳**：

- 重新跑 sync 流程，更新前端 diff 数据

- 状态变回 `pending`

- **DB 中旧数据保持不变**（不会写库）

  

**重试后再点采纳**：

- 图片按 11.3 的决策逻辑重新计算

- **覆盖写入** DB 中的 `ai_matched_image_ids`（set 操作）

  

**重试后新结果没有图片**：

- `saveImageIds = undefined`

- 后端不传 `aiMatchedImageIds` 字段

- **DB 中旧图片保持不变**（不会被清空）

  

### 11.6 图片入库示例

  

| 场景 | 图片数据 | 入库结果 |

|------|---------|---------|

| 用户勾选了 2 张图 | `['url1', 'id-xxx']` | `aiMatchedImageIds = ['url1', 'id-xxx']` |

| 未勾选，top1 有 URL | `imageUrls=['url1', 'url2']` | `aiMatchedImageIds = ['url1', 'url2']` |

| 未勾选，top1 无 URL 有 ID | `imageUrls=[], imageIds=['id-xxx']` | `aiMatchedImageIds = ['id-xxx']` |

| 未勾选，top1 无图片 | 都为空 | 不传字段，DB 不变 |

| 重试后采纳，新结果有图片 | `['url3']` | `aiMatchedImageIds = ['url3']`（旧值被覆盖） |

| 重试后采纳，新结果无图片 | `undefined` | 不传字段，DB 旧图片保留 |

  

### 11.7 页面刷新后的图片渲染

  

位置：`page.tsx` → `loadIncomingInspectionForPanel`

  

从 DB 加载的 `aiMatchedImageIds` 重建 diff：

```typescript

{

  id: r.id,

  imageIds: r.aiMatchedImageIds,  // DB 中的图片（URL + ID 混存）

  sources: [],                     // DB 没有 sources

  status: r.cells[6] ? 'accepted' : 'pending',

}

```

  

J 列渲染时的 fallback 逻辑：

```typescript

if (d.imageIds && d.imageIds.length > 0) {

  for (const img of d.imageIds) {

    allImages.push({ type: img.startsWith('http') ? 'url' : 'id', value: img });

  }

}

```

  

URL（`http` 开头）直接作为 `<Image>` src，ID 通过代理 `/api-proxy/v1/incoming-inspections/images/{id}` 获取。