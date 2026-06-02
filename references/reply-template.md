# Web Content Reviewer Reply Template

Use these templates for scored article workflows.

## User Preference: Minimal Concise Reply

User is an AI/ML researcher who processes many articles. **After each successful ingest**, the expected reply format is:

```
落库完成 ✓
| [标题] |
| 评分: value=X, confidence=X, **product** |
| 抓取: [method] |
| commit: `hash` |
| lint: 0 errors, N tracked pages ✓ |
```

**No detailed content summary. No Phase 1 verbose output. No file listing.**

## Phase 1 Score (internal — do not send to user)

Used internally for scoring decision. Not sent as reply:

```text
## [文章标题]

概要：[200 字内]

评分：
- 价值：X/10
- 置信度：X/10
- 乘积：X

值得关注程度：[strong | worth-reading | do-not-save]
```

If product `< 49`, add one line: `未落库：[single reason]`.

## Phase 2 Success (internal closeout)

```text
Ingest status: success
Files created: [raw/articles/slug.md, entities/slug.md, ...]
Files updated: [index.md, log.md, queries/review-queue.md, ...]
Wiki pages: N -> M
Index/log: updated
Structure: updated | no structural change needed
Validation: passed ([command or checklist])
Review metadata: review_value=X, review_confidence=Y, review_recommendation=..., review_stars=...
```

## Phase 2 Failure (send this to user)

```text
落库状态：failed
原因：[single explicit reason]
```

Rules:
- Score >= 49 but no Phase 2 closeout = bug.
- Success must include actual file paths.
- Failure must clearly state that the article is not fully ingested yet.
- If validation failed, Phase 2 cannot be success.
