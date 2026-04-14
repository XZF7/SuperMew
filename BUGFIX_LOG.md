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
