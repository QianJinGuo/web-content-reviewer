# Web Content Reviewer Reply Template

Use these templates for scored article workflows.

## Phase 2 Success

```text
**落库状态**：✅ 成功
**知识库状态**：✅ 新建或更新（file: raw/articles/[slug].md, entities/[slug].md）
**Wiki 页数**：N → M
**索引/结构**：已更新 index/log；如有需要，已整理对应 section / query 页面
**Lint 结果**：结构校验通过（index/header/counts/duplicates）
**Review 元数据**：`review_value=X`, `review_confidence=Y`, `review_recommendation=...`, `review_stars=...`
```

## Phase 2 Failure

```text
**落库状态**：❌ 失败
**知识库状态**：未完成落库（file: [已创建/已更新文件，若无则写 none]）
**Wiki 页数**：未确认或未更新
**索引/结构**：未完成整理 / 被阻塞
**Lint 结果**：失败或未运行
**阻塞原因**：[一句话说明]
```

Rules:
- Score ≥ 49 but no Phase 2 closeout = bug.
- Success must include actual file paths.
- Failure must clearly state that the article is not fully ingested yet.
