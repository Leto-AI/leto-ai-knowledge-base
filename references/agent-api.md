# Agent API 3.0 调用参考

这是乐途智行（英文品牌：leto AI）公开 Skill 使用的远程协议。

以下路径都相对于 `LETU_KB_BASE_URL`，都需要：

```http
Authorization: Bearer <LETU_KB_API_TOKEN>
```

Token 只能从安全环境读取，不得回显。所有 JSON 写请求使用 `Content-Type: application/json`；创建、Unit result 和 finalization 使用稳定且不复用到其他载荷的 `Idempotency-Key`。

## 目录

- [1. 创建并上传源文件](#1-创建并上传源文件)
- [Office Authoring v3](#office-authoring-v3)
- [2. 处理 Unit](#2-处理-unit)
- [3. Finalization 与结果](#3-finalization-与结果)

所有任务先调用 `GET /api/agent/v1/bootstrap`，并只执行
`actions.submission.available=true` 时返回的提交协议。连接和重定向必须遵守
[connection-security.md](connection-security.md)。

必须先读取 `actions.submission.apiContract` 与
`actions.submission.schemaBundle`。Contract 是完整操作目录，Schema Bundle 是请求和
响应结构的唯一机器来源。按 `workflow` 中的 `operationId` 执行；发送 JSON 前用
`requestSchemaRef` 校验，收到成功 JSON 后用对应 `responseSchemaRef` 校验。二进制请求
按 `binaryRequest`、声明时的字节数/SHA-256，以及服务端返回的固定 Header 执行。Schema
失败时不得猜字段或继续下一阶段，应保留响应和稳定错误码后纠正当前请求。

新建文档还必须读取 Bootstrap 的 `tenant.spaceBindings`。只有一个绑定时直接使用其
`spaceId`；多个绑定时使用用户明确指定的目标，没有明确选择就先确认。不能提交未返回
的 ID，也不能把 `includeDescendants=true` 解释为可以离开该空间树。更新已有文档不
提交 `targetSpaceId`，文档移动属于独立后台管理操作。

独立 Schema 端点有两种返回形状，必须先看返回对象本身：

- 若根对象就是标准 JSON Schema（例如具有 `type`、`oneOf`、`$ref` 等关键字），
  直接编译根对象。
- 若根对象是契约封套，并包含 `endpoint`、`request`、`response`，发送请求时只编译
  `request`，验证成功响应时只编译 `response`。例如
  `retrieval-search/2.0` 的请求 Schema 是 `.request`、响应 Schema 是 `.response`；
  `retrieval-index-status/1.0` 只有响应，因此编译 `.response`。

不要把带 `endpoint` 的整个契约封套交给 Ajv 等 Validator 来验证业务 JSON；那只会
验证“契约文档”的形状，不能证明 API 响应符合业务 Schema。Schema 中的 `$ref` 必须
仍按同一个 Bundle/文档解析，不能手工删除。

登记图片前还必须读取
`GET /api/agent/v1/schemas/asset-metadata/1.0`。`imageType` 的闭集为
`photo`、`diagram`、`chart`、`screenshot`、`document`、
`decorative`、`unknown`。不要提交 `table` 等未声明值；表格页面截图通常用
`document`，用于表达统计关系的图表才用 `chart`。

## 1. 创建并上传源文件

```http
POST /api/agent/v1/work-orders
Idempotency-Key: source-<stable-id>

{
  "operation": "create_document",
  "targetSpaceId": "space_company",
  "sources": [
    {
      "logicalPath": "guide.pdf",
      "role": "primary",
      "mediaType": "application/pdf",
      "size": 12345,
      "sha256": "sha256:<64 lowercase hex>"
    }
  ]
}
```

更新时改为 `operation=update_document` 并提供 `documentId`。主文件必须位于清单根目录；附件可以是安全相对路径。使用响应中每个 `uploads[].url`、`method` 和 `headers` 上传原始字节，不做 Base64。当前 URL 是同源相对路径，上传请求同时携带本页开头的通用 Bearer；若 URL 不是同源相对路径则按连接安全契约停止。

完成后：

```http
POST /api/agent/v1/work-orders/{workOrderId}/source-seal
{}

POST /api/agent/v1/work-orders/{workOrderId}/prepare
{}
```

## Office Authoring v3

DOCX、PPTX、XLSX 只提交原件，且原件必须是唯一 `primary`。客户端不得解析 OOXML
生成结构清单，不得声明或上传 `letu-office-authoring-manifest.json`；服务端会拒绝该旧
协议。`source-seal` 后调用 `prepare`，服务端隔离 Office Worker 会在不调用 LLM、
不执行宏、不刷新外链和不运行公式的前提下安全解析原件，并确定性生成全部可信 Unit。

服务端按以下结构划分：

- Word：按标题层级形成 `word_section`，公开当前 Section 的 `paragraphIds`、
  `tableIds`、`noteIds` 与原生图片。
- PowerPoint：每张幻灯片形成 `presentation_slide`，公开当前页的 `shapeIds`、
  `noteIds` 与原生图片。
- Excel：按有界范围形成 `spreadsheet_range`，公开 `range`、`cellAddresses`、
  `chartIds`、缓存值与公式事实。遇到尚未支持且可能被静默遗漏的 Drawing 对象时，
  `prepare` 失败关闭，不发布残缺文档。

`prepare` 后的 Office Unit 使用 `external-authoring-unit/3.0`。客户端 AI 只处理服务端
公开的 `source`、`input`、`requiredSourceObjectIds` 与 `sourceAssets`，不得猜测或扩展
来源闭集。每个 `localBlock` 和 `imagePlacement` 必须带属于当前 Unit 的精确
`sourceAnchor`；可用类型是 `word_paragraph`、`word_table`、`word_note`、
`presentation_shape`、`presentation_note`、`spreadsheet_cell` 和
`spreadsheet_range`。空 Word Section 没有任何 Paragraph/Table/Note 时，允许且只允许
`word_section/sectionId`；空 Slide 没有任何 Shape/Note 时，允许且只允许
`presentation_slide/slideId`。

Unit 是有界传输和提交单元，不等于完整业务容器。超大 Section、Slide 或 Range 会被
确定性拆成多个顺序 Unit，每个 Unit 最多公开 16,000 个必需来源对象，且服务端会按真实
UTF-8 JSON 字节复核最小闭合输出不超过 4 MiB；客户端必须逐个处理，不得按
`sectionId`、`slideId` 或 Range 名称自行去重。单个不可继续拆分的大文字对象可能使用
`text/plain` 二进制输入；二进制 `input` 必须同时核对
响应中的 `inputMediaType` 和 `inputSha256`。Spreadsheet 的 JSON input 使用
`office-spreadsheet-authoring-input/1.0`，其闭合 Schema 从 Bootstrap 的
`authoringInputSchemas.spreadsheet_range` 获取；读取当前 Unit 的 `range`、`formulas`、
`charts` 和 `chartDependencies`，禁止执行公式或刷新外链。Chart 的输出归属已由服务端按
锚点绑定当前 Range，但数据来源不一定在当前 Range：客户端必须按 `chartId` 一一关联
dependency，遍历全部 `sourceRanges`、`rangeShards`、`cells` 和 `formulas`，包括跨 Sheet
依赖，并据此整理图表表达的事实、趋势、比较和业务含义。不得只读标题或当前 Cell 后猜测，
也不得跨 Unit 移动或合并 Chart。

`chartDependencies` 是服务端从原生工作簿生成并绑定输入 SHA 的只读事实上下文，其中跨
Sheet Cell 不属于当前 Unit 的 `requiredSourceObjectIds`，不得擅自写进 coverage。
因此 coverage 不能证明客户端读过或理解了 dependency；客户端 AI 必须在提交前另外自检：
Chart 与 dependency 的 `chartId` 集合和顺序一致，每个来源范围至少有一个 shard，所有
shard 的 cells/formulas 都已参与语义整理。任何缺失、错配或无法理解都应停止并报告，不能
提交一个仅在结构上通过 coverage 的残缺结果。

输出还必须包含精确 coverage 回执：`coverage.sourceObjectIds` 与 Unit 的
`requiredSourceObjectIds` 集合必须完全相同，不得漏项、重复或伪造；`mappings` 必须为
每个 `localBlock` 和 `imagePlacement.localKey` 提供唯一映射，列出该输出实际处理的来源
对象。全部必需对象至少被映射一次，每个 mapping 还必须包含对应 `sourceAnchor` 的对象。
这形成机器可审计的处理责任闭包，但不是语义正确性的自动证明。
因此禁止把全部来源 ID 无差别挂到一个并未实际表达它们的摘要，只为让集合校验通过；
每个 mapping 列出的来源事实必须真实反映在对应输出里。语义质量由客户端 AI 的逐对象
理解、Skill 约束和后续人工/模型抽检负责，服务端只对来源闭集、锚点和回执结构失败关闭。

```json
{
  "expectedGeneration": 1,
  "output": {
    "localBlocks": [{
      "localKey": "summary",
      "type": "paragraph",
      "text": "该章节说明产品定位与三个核心能力。",
      "sourceAnchor": {
        "kind": "word_paragraph",
        "paragraphId": "word_paragraph_<server-id>"
      }
    }],
    "imagePlacements": [],
    "coverage": {
      "sourceObjectIds": [
        "word_paragraph_<server-id>",
        "word_table_<server-id>"
      ],
      "mappings": [{
        "localKey": "summary",
        "sourceObjectIds": [
          "word_paragraph_<server-id>",
          "word_table_<server-id>"
        ]
      }]
    }
  }
}
```

若 Unit 包含 `sourceAssets`，逐个从其 `inputUrl` 读取原生图片字节，并调用 Contract 的
`promoteUnitSourceAsset` 操作提交唯一 `detailedDescription`、OCR `visibleText` 与
`imageType`。该操作复用服务端已验证字节，不需要也不允许客户端再次上传图片。返回的
`assetId` 必须通过带来源锚点的 `imagePlacements` 放入正文；任何非装饰原生图片缺少描述
或放置都会导致 Finalization 失败。

收到 `UNIT_SOURCE_ANCHOR_REQUIRED`、`UNIT_SOURCE_ANCHOR_INVALID` 或
`UNIT_SOURCE_COVERAGE_INVALID` 时，按 `recommendedAction` 重新读取当前 Unit，只修正
当前结果，不重建 Work Order。服务端负责可信结构、Canonical Markdown、Package、
Chunk、Embedding、索引、权限和发布；客户端 AI 负责语义理解、重构和图片描述。

## 2. 处理 Unit

```http
GET /api/agent/v1/work-orders/{workOrderId}/units/next
GET /api/agent/v1/work-orders/{workOrderId}/units/{unitId}
GET /api/agent/v1/work-orders/{workOrderId}/units/{unitId}/input
```

Markdown Unit 已在 `input.text` 中提供正文。图片与 PDF page Unit 使用响应中的同源
`input.inputUrl` 获取服务端签发的图片字节；不得假设或请求服务端 Workspace 路径。

`units/next` 和 Unit detail 响应中的 `unitGeneration` 是当前并发控制代际。提交结果时必须将刚读取的精确值映射为请求字段 `expectedGeneration`。不要使用循环序号、客户端计数器或自行执行 `+1`；重试前若不确定，重新读取 Unit detail。

处理图片或 PDF page 时必须把信息分成三层：

- `visibleText`：图片 OCR 原文，只进入 Asset 元数据。
- `detailedDescription`：图片的使用场景、画面内容、讲述内容和表达含义。
- `localBlocks`：整理后的知识正文，只保留适合人工阅读和 RAG 的事实、关系、流程、结论与业务含义。

禁止把 `visibleText` 或 OCR 逐字稿作为 paragraph 提交。服务端可以校验结构与引用，但不会调用模型替客户端判断正文是否具有语义价值，因此客户端 AI 必须在提交前完成这项检查。
发布后服务端为了 RAG 召回，可以把 Asset 的 `visibleText` 和
`detailedDescription` 投影进 `index/chunks.jsonl` 或 Search Chunk。该派生投影不是
`document.md`，客户端不得将它回写为 Unit paragraph、`localBlocks` 或 Markdown。

### 登记正式图片

原始图片、PDF 页图、客户端裁剪图或生成图统一先声明。若使用 Unit 原图，先从
`input.inputUrl` 下载字节，再原样上传；不要传服务端内部路径：

```http
POST /api/agent/v1/work-orders/{workOrderId}/asset-uploads

{
  "mediaType": "image/png",
  "size": 1234,
  "sha256": "sha256:<64 lowercase hex>",
  "detailedDescription": "完整、具体且可独立理解的图片描述",
  "visibleText": "",
  "imageType": "chart",
  "sourceSha256": "sha256:<来源页或源图片哈希>"
}
```

按响应 URL 与 headers PUT 原始图片字节，并携带通用 Bearer。上传响应返回正式 `assetId`。

### 上传处理 Evidence

只有用户授权保留、且能支持解析结论或故障追溯的过程产物才作为 Evidence。先声明：

```http
POST /api/agent/v1/work-orders/{workOrderId}/evidence-uploads

{
  "mediaType": "application/json",
  "size": 1234,
  "sha256": "sha256:<64 lowercase hex>",
  "type": "ocr_result",
  "retention": "build_lifetime"
}
```

使用响应中的 `method`、`url` 和 `headers` PUT 原始字节，并携带通用 Bearer，不使用 Base64。支持的媒体、Evidence 类型、保留策略和 Work Order 级预算以 `GET /api/agent/v1/capabilities` 为准。服务端复验 Content-Type、大小、SHA-256、UTF-8/JSON/JSONL 或图片签名；相同上传对象可安全重试。

文件数、媒体类型、字节数、图片几何、Unit、Asset 和 Evidence 配额均由服务端强制执行；
客户端不得通过拆分文件或篡改元数据绕过限制。每次处理前以
`GET /api/agent/v1/capabilities` 返回的当前约束为准，收到配额或媒体校验错误时应缩小、
转换或重新选择用户明确授权的输入后再提交。

`build_lifetime` 会进入 `evidence/evidence-index.json` 和最终发布包；`temporary` 只用于构建期且不进入发布包。禁止上传 Token、凭据、环境转储、源文件副本、脚本、可执行文件、归档、网络抓包或用户未授权文件。服务端不会执行或解压 Evidence。

Evidence 身份同时绑定 `type`、`retention` 与内容 SHA-256。相同内容和类型可以分别登记为 `temporary` 与 `build_lifetime`，二者具有不同 `evidenceId`；上传顺序不改变身份或最终保留结果。

发布包及其索引、报告和 Manifest 都由服务端生成并校验。客户端不得手工增加、删除或
修改这些派生文件；变更处理输入后应重新执行 Finalization。

### 提交语义结果

结构以 Bootstrap 返回的 Unit Output Schema 为准；当前提交协议统一从 Bootstrap 获取
`/api/agent/v1/schemas/unit-output/3.0`，其中非 Office 仍是 v1 Unit，Office 使用 v3
输出：

```http
PUT /api/agent/v1/work-orders/{workOrderId}/units/{unitId}/result
Idempotency-Key: unit-<unitId>-generation-<n>

{
  "expectedGeneration": 1,
  "output": {
    "localBlocks": [
      {"localKey": "title", "type": "heading", "level": 1, "text": "标题"},
      {"localKey": "body", "type": "paragraph", "text": "该图用于说明申请审批流程：员工提交后由主管审核，通过后进入归档；其核心含义是审批责任和状态流转必须可追踪。"}
    ],
    "imagePlacements": [
      {"localKey": "figure-1", "assetId": "asset_<id>", "afterLocalBlockKey": "body"}
    ]
  }
}
```

上例中的 `1` 只用于展示字段形状，必须替换为该 Unit 最近一次响应的
`unitGeneration`，即 `unitGeneration → expectedGeneration`。

错误示例：

```json
{
  "localKey": "ocr-copy",
  "type": "paragraph",
  "text": "申请 提交申请 主管审批 审批通过 归档 返回 修改 驳回……"
}
```

上例只是把图片 OCR 重新写入正文。正确做法是把这段原始文字放入 Asset 的
`visibleText`，将流程关系和业务结论写入 `localBlocks`，并用 `imagePlacements`
引用图片。

## 3. Finalization 与结果

```http
POST /api/agent/v1/work-orders/{workOrderId}/finalizations
Idempotency-Key: finalize-<workOrderId>-1
{}

GET /api/agent/v1/work-orders/{workOrderId}
GET /api/agent/v1/work-orders/{workOrderId}/summary
```

用户明确放弃且 Work Order 尚未进入验证时，只按 Contract 的 `cancel` 操作取消：

```http
POST /api/agent/v1/work-orders/{workOrderId}/cancel
Idempotency-Key: cancel-<workOrderId>-<stable-intent>
Content-Type: application/json

{}
```

同一个取消意图始终复用同一个 Key；成功响应必须为 `status=cancelled`。进入验证、
已发布或其他终态后不得把取消当作成功补偿，也不得更换 Key 反复尝试。

`validation_queued` 只表示已进入服务端复验队列。继续有界轮询，直到
`published`、`rejected`、`validation_failed`、`superseded`、`cancelled` 或
`expired`。后五种都是失败终态；保留错误码和有界诊断，不得宣称已经写入知识库。
HTTP 410 表示 Work Order 已经过期，立即停止轮询，不复用旧 ID。只有
`published` 且同时存在 Acceptance Receipt 与 Publication Receipt 才算成功。

发布阶段由服务端确定性生成 `index/chunks.jsonl` 与 `index/manifest.json`，客户端
AI 不提交这两个文件。只有同一 Work Order 返回 `published` 且同时存在 Acceptance
Receipt 与 Publication Receipt 才能宣称提交成功。Skill Token 不具备通用文档、
任务、Package 或实时索引读取权限；需要查阅知识或索引时，用户必须另行提供 Query
Token，并重新读取该 Token 的 Bootstrap。Work Order 属于临时处理流程，保留期结束后
可能返回 410，不得依赖它作为长期状态入口。
