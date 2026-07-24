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

## 4. 创建服务端冻结的三槽 Cohort

客户端只提交 Dataset 和固定 Target：

```http
POST /api/evaluations/cohorts

{
  "datasetVersionId": "edsv_...",
  "retrievalTargetId": "evrt_..."
}
```

禁止提交 `evaluationRunIds`、重复次数、Runner、稳定性策略或指标策略。服务端以实验身份内容寻址，在一个 `BEGIN IMMEDIATE` 中创建不可变 Cohort Intent、固定顺序 1/2/3 的三个槽位及其 Run Job；相同实验跨用户、Token 或重试都返回同一 Cohort，不能重抽样或挑选有利结果。

轮询 Cohort：

```http
GET /api/evaluations/cohorts/{evaluationCohortId}
```

终态含义：

- `sealed_stable`：三个固定槽位均成功，且完整有序 Top-50 证据身份投影一致；浮点分数和通道数组顺序不参与稳定性比较。
- `sealed_unstable`：三个槽位均成功，但至少一轮有序证据身份不同。不得重建 Cohort 再试。
- `failed` / `invalid`：任一槽失败或权限、发布、配置、完整性 Fence 失效；剩余槽会被围栏并产生签名终态回执。

只有 `sealed_stable` 且服务端签名回执验证通过，才能报告“候选稳定性评测完成”。网络或 5xx 可按同一请求幂等重试；4xx 应修正输入或停止。`POST /api/evaluations/runs` 仅供管理员诊断单轮问题，不属于候选发布评测闭环，Skill 不得用它代替 Cohort。

## 5. 登记与读取 Current Baseline

Current Baseline 是后续 Gate 比较的线上基准，不是 Candidate 的别名。只有用户明确要求建立或更新线上基准时执行。先创建服务端 Current Target：

```http
POST /api/evaluations/retrieval-targets

{
  "datasetVersionId": "edsv_...",
  "targetKind": "current",
  "retrievalConfigId": "retcfg_hybrid_rrf_v1"
}
```

再按第 4 节为这个 Current Target 创建并等待一个 `sealed_stable` Cohort。Baseline 只接受 Current Target；Candidate Target 即使稳定也必须被拒绝，不能绕过后续 Gate。

读取现有槽位及其 `generation`：

```http
GET /api/evaluations/baselines
GET /api/evaluations/baselines/{datasetVersionId}/retrieval_gate_v1
```

首次登记使用 `expectedGeneration=0`；更新必须提交刚读取的精确 generation：

```http
PUT /api/evaluations/baselines/{datasetVersionId}/retrieval_gate_v1

{
  "evaluationCohortId": "evco_...",
  "expectedGeneration": 0
}
```

服务端以事务 CAS 修改槽位，并分别签名不可变 Baseline Fact 和当前 Slot assignment。即使重复设置同一个 Baseline，也必须提交当前 generation；陈旧 generation 返回 `409 EVALUATION_BASELINE_GENERATION_CONFLICT`，应重新读取并让用户确认，不能盲目覆盖。请求不得增加报告、指标、Target、Tenant、权限或签名字段。

登记时服务端会在提交前重新捕获当前 Publication、Index 与授权快照。Target 只是创建时标记为 `current`、但现网已经发布新版本或切换索引时，会返回 `409 EVALUATION_TARGET_STALE` 且不推进 generation；客户端必须创建新的 Current Target 和 Cohort，不能复用历史 Target。

Current Baseline 成功登记仍不等于 Candidate Gate 通过。

## 6. 创建并读取签名 Gate Decision

只有 Candidate Cohort 已进入终态，且已经读取当前 Baseline generation 后，才能请求服务端判定：

```http
POST /api/evaluations/gates/retrieval_gate_v1/decisions

{
  "datasetVersionId": "edsv_...",
  "candidateCohortId": "evco_...",
  "expectedBaselineGeneration": 1
}
```

请求是闭合对象。禁止增加指标、阈值、报告、Run、Target、Variant、Activation Plan、Tenant、Namespace、权限、签名或服务端摘要。服务端自行读取并验证 Dataset、三槽 Baseline/Candidate 证据、当前权限与发布/索引闭集，并使用固定 `retrieval_gate_v1` 数值政策生成不可变签名 Decision。

返回只包含可公开摘要：

```json
{
  "decisionId": "evgd_...",
  "datasetVersionId": "edsv_...",
  "gatePolicyId": "retrieval_gate_v1",
  "baselineGeneration": 1,
  "candidateCohortId": "evco_...",
  "candidateVariantId": "retv_...",
  "outcome": "PASS",
  "reasonCodes": [],
  "createdAt": "2026-07-24T12:00:00.000Z",
  "reused": false
}
```

读取并重新鉴权：

```http
GET /api/evaluations/gates/decisions/{decisionId}
```

- `PASS`：候选在固定政策下未退化且安全检查通过。只能报告“通过门禁，尚未上线”。
- `FAIL`：候选证据有效，但至少一项候选安全或质量检查失败。原样报告 `reasonCodes`，不要重试抽样。
- `INVALID`：基线或候选证据不满足可比较条件。必须先修复基线、稳定性或证据完整性，不能解释为候选失败，也不能继续激活。
- `409 EVALUATION_GATE_BASELINE_CONFLICT`：Baseline 已推进。重新读取 Baseline，并让用户确认是否针对新基线重新判定。

Decision 响应不会暴露内部 Projection、Run Evidence、Activation Plan、Index Build 或 Namespace。客户端不得探测这些数据。

当前尚未公开 Candidate 激活 API。即使 Decision 为 `PASS`，没有一次性 Permit 和 Route 激活回执也不得声称候选已上线。
