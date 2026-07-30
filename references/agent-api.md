# Agent API 1.0 调用参考

这是乐途智行（英文品牌：leto AI）公开 Skill 使用的远程协议。

以下路径都相对于 `LETU_KB_BASE_URL`，都需要：

```http
Authorization: Bearer <LETU_KB_API_TOKEN>
```

Token 只能从安全环境读取，不得回显。所有 JSON 写请求使用 `Content-Type: application/json`；创建、Unit result 和 finalization 使用稳定且不复用到其他载荷的 `Idempotency-Key`。

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

结构以 `GET /api/agent/v1/schemas/unit-output/1.0` 为准：

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
