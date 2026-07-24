# Agent API 1.0 调用参考

这是乐途智行（英文品牌：leto AI）公开 Skill 使用的远程协议。

以下路径都相对于 `LETU_KB_BASE_URL`，都需要：

```http
Authorization: Bearer <LETU_KB_API_TOKEN>
```

Token 只能从安全环境读取，不得回显。所有 JSON 写请求使用 `Content-Type: application/json`；创建、Unit result 和 finalization 使用稳定且不复用到其他载荷的 `Idempotency-Key`。

## 1. 创建并上传源文件

```http
POST /api/agent/v1/work-orders
Idempotency-Key: source-<stable-id>

{
  "operation": "create_document",
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

更新时改为 `operation=update_document` 并提供 `documentId`。主文件必须位于清单根目录；附件可以是安全相对路径。使用响应中每个 `uploads[].url`、`method` 和 `headers` 上传原始字节，不做 Base64。

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

Markdown Unit 已在 `input.text` 中提供正文。图片与 PDF page Unit 从 `/input` 获取服务端签发的图片字节。

处理图片或 PDF page 时必须把信息分成三层：

- `visibleText`：图片 OCR 原文，只进入 Asset 元数据。
- `detailedDescription`：图片的使用场景、画面内容、讲述内容和表达含义。
- `localBlocks`：整理后的知识正文，只保留适合人工阅读和 RAG 的事实、关系、流程、结论与业务含义。

禁止把 `visibleText` 或 OCR 逐字稿作为 paragraph 提交。服务端可以校验结构与引用，但不会调用模型替客户端判断正文是否具有语义价值，因此客户端 AI 必须在提交前完成这项检查。

### 登记服务端已有图片

`sourcePath` 必须原样取自 Unit 的 `input.sourcePath` 或 `input.pageImagePath`：

```http
POST /api/agent/v1/work-orders/{workOrderId}/assets

{
  "sourcePath": "authoring/inputs/page-0001.jpg",
  "detailedDescription": "完整、具体且可独立理解的图片描述",
  "visibleText": "图片内可见文字；没有则为空字符串",
  "imageType": "document",
  "sourceSha256": "sha256:<对应 sourceSlices 的哈希>"
}
```

### 上传客户端裁剪或生成的正式图片

先声明：

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

按响应 URL 与 headers PUT 原始图片字节。上传响应返回正式 `assetId`。

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

使用响应中的 `method`、`url` 和 `headers` PUT 原始字节，不使用 Base64。支持的媒体、Evidence 类型、保留策略和 Work Order 级预算以 `GET /api/agent/v1/capabilities` 为准。服务端复验 Content-Type、大小、SHA-256、UTF-8/JSON/JSONL 或图片签名；相同上传对象可安全重试。

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

`validation_queued` 只表示已进入服务端复验队列。继续轮询，直到 `published`、`rejected` 或 `superseded`。失败时保留错误码和有界诊断，修正当前 Work Order；不得宣称已经写入知识库。

发布阶段由服务端确定性生成 `index/chunks.jsonl` 与 `index/manifest.json`，客户端 AI 不提交这两个文件。Index Manifest 的 `embeddingStatus=not_configured` 表示 Chunk Artifact 已可供 RAG/向量库消费，但尚未生成或写入任何 Embedding；不得向用户宣称向量索引已经完成。
