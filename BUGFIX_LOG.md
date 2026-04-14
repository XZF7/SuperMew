# Bug Fix Log

---

## [2026-04-14] 豆包模型 json_object 400 报错

### 时间戳
2026-04-14T08:50:45 UTC+8

### 报错信息
```
Error code: 400 - {'error': {'message': "<400> InternalError.Algo.InvalidParameter: 
'messages' must contain the word 'json' in some form, to use 'response_format' of type 'json_object'."}}
```

### 原因
`rag_pipeline.py` 中使用了 LangChain 的 `with_structured_output()`，该方法会自动向模型请求中附加 `response_format: json_object`。

豆包（火山引擎 ARK）模型要求：**使用 `json_object` 格式时，prompt 中必须包含 "json" 字样**，否则返回 400。

涉及两处：
- `grade_documents_node`（第173行）：使用 `GradeDocuments` 结构化输出
- `rewrite_question_node`（第206行）：使用 `RewriteStrategy` 结构化输出

### 修复方法

**文件**：`backend/rag_pipeline.py`

**第一处** — `GRADE_PROMPT` 末尾追加：
```python
"Respond in JSON format."
```

**第二处** — 路由选择 prompt 末尾追加：
```python
"请以 JSON 格式输出。"
```

### 影响范围
仅影响知识库问答触发 RAG 流程时的文档相关性评估与查询重写路由阶段，不影响普通对话。

---

## [2026-04-14] Jina Rerank 余额不足导致知识库检索无结果

### 时间戳
2026-04-14T17:30:00 UTC+8

### 报错信息
```
HTTP 403: Insufficient account balance. Top up your account at https://jina.ai/api-dashboard/key-manager.
```

### 原因
`.env` 中配置了 Jina Reranker，但账户余额不足，API 返回 403。

`rag_utils.py` 的 `retrieve_documents` 函数有两层 `except Exception` 将所有错误静默吞掉，导致：
1. Rerank 失败后虽然降级返回了检索结果
2. 但上层 `grade_documents` 节点因之前的 json_object 400 问题仍可能崩溃
3. 最终 Agent 收到空结果，回复"找不到相关信息"

### 修复方法

**文件**：`.env`

将 Rerank 相关配置注释掉，禁用 Rerank，系统自动降级为纯混合检索：
```env
# RERANK_MODEL=jina-reranker-v3
# RERANK_BINDING_HOST=https://api.jina.ai/v1/rerank
# RERANK_API_KEY=jina_xxx...
```

### 影响范围
禁用 Rerank 后，检索结果不再经过精排，召回质量略有下降，但功能完全正常。如需恢复，充值 Jina 账户后取消注释即可。

仅影响知识库问答触发 RAG 流程时的文档相关性评估与查询重写路由阶段，不影响普通对话。如需恢复，充值 Jina 账户后取消注释即可。

---

## [2026-04-14] GradeDocuments Pydantic 字段名不匹配

### 时间戳
2026-04-14T18:10:00 UTC+8

### 报错信息
```
pydantic_core._pydantic_core.ValidationError: 1 validation error for GradeDocuments
binary_score
  Field required [type=missing, input_value={'score': 'yes'}, input_type=dict]
```

### 原因
`grade_documents_node` 使用 `with_structured_output(GradeDocuments)` 时，模型返回的 JSON 字段名为 `score`，但 Pydantic 模型定义的字段名为 `binary_score`，导致验证失败，RAG 流程崩溃返回空结果。

根本原因：`GRADE_PROMPT` 没有明确告知模型输出字段名。

### 修复方法

**文件**：`backend/rag_pipeline.py`

在 `GRADE_PROMPT` 末尾明确指定字段名和格式：
```python
"Respond in JSON format with field 'binary_score', e.g. {{\"binary_score\": \"yes\"}}."
```

注意：因为 prompt 使用 `str.format()` 格式化，JSON 示例中的花括号需双写转义 `{{}}` 避免 `KeyError`。

### 影响范围
仅影响 RAG 文档相关性评估阶段，修复后 `route: generate_answer` 正常返回。