# 候选检索 Variant 与固定评测

以下路径相对于 `LETU_KB_BASE_URL`，全部携带：

```http
Authorization: Bearer <LETU_KB_API_TOKEN>
Content-Type: application/json
```

必须使用拥有 `kb:admin` 的 Token。Token、Tenant、ACL、Qdrant Collection、Vector Namespace、Index Build 闭集和服务端配置摘要不得出现在客户端输入、日志或回答中。

## 1. 使用已有评测 Dataset

列出和读取 Dataset：

```http
GET /api/evaluations/dataset-versions
GET /api/evaluations/dataset-versions/{datasetVersionId}
```

优先使用用户或管理员已经确认的 Dataset。客户端 AI 不得冒充人工审核人，也不得未经用户确认把生成的 Case 标成 `approved`。

用户明确要求创建并确认 Dataset 时，使用：

```http
POST /api/evaluations/dataset-versions

{
  "schemaVersion": "retrieval-eval-dataset/1.0",
  "name": "核心检索集",
  "cases": [
    {
      "caseId": "evc_leave_policy_001",
      "question": "年假应当如何申请？",
      "modality": "text",
      "expectationKind": "answerable",
      "importance": "critical",
      "reviewStatus": "approved",
      "reviewer": "用户确认的审核人标识",
      "positiveTargets": [
        {
          "kind": "chunk",
          "documentId": "doc_...",
          "contentBuildId": "cb_...",
          "publicationGeneration": 1,
          "chunkSetId": "ib_...",
          "chunkId": "chk_..."
        }
      ],
      "forbiddenTargets": [],
      "tags": ["leave"]
    }
  ]
}
```

Target 身份只能来自当前授权检索响应或已读取的发布资料，不得猜测。`no_answer` Case 的 `positiveTargets` 必须为空；图片或混合 Case 必须包含 Asset 或 Occurrence Target。服务端会重新验证当前权限、Publication、Chunk、Asset 和 Occurrence。

## 2. 创建并等待 Variant

当前公开候选配置：

```http
POST /api/evaluations/retrieval-variants

{
  "retrievalConfigId": "retcfg_hybrid_rrf_v2"
}
```

请求是闭合对象，不得增加 `tenantId`、`embeddingProfileId`、`indexes`、`documents`、`namespace` 或 ACL 字段。服务端从当前完整可检索闭集推导 Variant；调用者看不到内部映射。

查看：

```http
GET /api/evaluations/retrieval-variants
GET /api/evaluations/retrieval-variants/{retrievalVariantId}
```

- `building`：仍在构建；有界轮询，不创建 Target。
- `ready_not_active`：物理闭集完整，但明确没有上线。
- `stale`：Variant 所绑定的发布闭集或当前索引闭集已变化；停止等待，不得创建 Target，重新创建当前 Variant。
- 404：不存在或不可见；不要枚举 ID。

Variant 的头、预期文档闭集和 Build 关系由服务端在一个事务内登记。列表和详情会按当前 Document Read 权限重新过滤；撤权后返回空列表或 `404`，不得据此判断资源是否存在。首版只执行与当前线上相同 Embedding Profile、改变融合配置的 Variant。不同 Profile 会失败关闭，不能退回当前模型假装完成候选评测。

## 3. 创建固定 Candidate Target

```http
POST /api/evaluations/retrieval-targets

{
  "datasetVersionId": "edsv_...",
  "targetKind": "candidate",
  "retrievalVariantId": "retv_..."
}
```

响应中的 `status=candidate` 表示 Target 已冻结；不表示 Variant 已激活。Target 响应不会提供内部 Index、Namespace 或 Pin。

检查和固定查询：

```http
GET /api/evaluations/retrieval-targets/{retrievalTargetId}

POST /api/evaluations/retrieval-targets/{retrievalTargetId}/query
{
  "question": "年假应当如何申请？"
}
```

只有 `strategyApplied=hybrid_rrf` 且 `degraded=false` 可作为有效候选结果。结果只包含可公开的 Document/Build/Chunk 证据身份。

## 4. 执行三轮可恢复评测

对 `repetitionOrdinal` 依次使用 1、2、3：

```http
POST /api/evaluations/runs

{
  "datasetVersionId": "edsv_...",
  "retrievalTargetId": "evrt_...",
  "metricPolicyId": "retmetric_ranked_v1",
  "repetitionOrdinal": 1
}
```

轮询：

```http
GET /api/evaluations/runs/{evaluationRunId}
```

仅 `status=succeeded` 且响应含可验证的服务端 Report/Receipt 才算该轮完成。`invalid`、`failed`、权限代际变化、Publication 变化、降级检索或完整性错误都不能计入成功轮次。网络或 5xx 可按原请求幂等重试；4xx 应修正输入或停止。

当前阶段尚未公开 Candidate 激活 API。即使三轮成功，也只能报告“候选评测完成”；不得声称 Gate 已通过或候选已上线。
