---
name: web-content-reviewer
description: "Strictly review web articles and save only qualifying high-value sources to the Obsidian wiki using llm-wiki completion gates. Use for URL review, scoring, ingest decisions, wiki save workflows, review metadata, and explicit success/failure closeouts."
version: "1.1"
author: user
related_skills: [llm-wiki, tinyfish-web-agent]
---

# Web Content Reviewer

Peer-review web articles and ingest only high-quality, credible material into the wiki.

## 评分策略选择指南（2026-05-12 实践总结）

### 背景

评分 pipeline 有两个引擎：**Heuristic Scoring**（确定性、零成本）和 **LLM Scoring**（更智能、但依赖 API 稳定性）。

实际使用中发现：
- **MiniMax-M2.7 结构化输出持续损坏** — 返回 ` thinks` XML 标签包裹的 evaluation-prompt 而非 `{"review_value":N,"review_confidence":N}` JSON。即使 strip 标签后提取也可能失败。这不是偶发问题，已持续多轮 cron（横跨数天）。
- **Heuristic 对不同来源有系统性偏差** — 微信公众号文章因 HTML→Markdown 转换丢失标题层级、代码围栏、外链，天然得分低（max 42/49）。RSS 文章格式较好，得分可达 56+。

### 决策原则

| 情况 | 推荐方案 | 理由 |
|------|----------|------|
| LLM API 结构化输出可靠 | LLM 评分 | 更准确，理解深层技术价值 |
| LLM API 不稳定/返回非 JSON | **纯 Heuristic** | 确定性，无 API flakiness |
| 来源为微信公众号 | **Heuristic ≥25** | 格式特殊导致 baseline 偏低 |
| 来源为 RSS/Newsletter | **Heuristic ≥30** | 格式较好，阈值可设高 |
| 混合来源、需要一致性 | **纯 Heuristic + 按源阈值** | 同一标准、不同基线 |

### 统一评分门槛

```
所有来源统一：value × confidence >= 49
```

**硬性长度门槛**（char_count 为 markdown 原文 stripped 后的字符数）：

| char_count | 决策 |
|------------|------|
| < 1500     | **最严格审核**：要求极高的技术密度和独特性才可入库（char_count 越小越难通过） |
| 1500-3000  | 正常评分，需 value × confidence >= 49 |
| > 3000     | 正常评分 |

> 注意：微信文章因 HTML→Markdown 转换丢失标题层级、代码块、外链，格式损失不影响评分门槛。heuristic/llm 均使用同一乘积门槛，不再按源分别设阈值。

### 为什么纯 Heuristic 在特定环境下是最佳实践

1. **确定性** — 同一篇文章永远得同一分，可预测、可 debug
2. **零成本** — 不需要调 API，适合批量场景（80 篇/20min）
3. **快** — 毫秒级评分，无网络延迟
4. **简单** — 一个函数、多个阈值、零回退路径
5. **LLM 的优势被 API 不可靠性抵消** — 一个返回 ` thinks` 标签的 LLM 比 heuristic 更不可靠

### 何时应该用 LLM

- LLM provider 能稳定输出结构化 JSON（已验证过，不是 MiniMax 当前状态）
- 需要理解**文章深层的技术价值**（heuristic 只能检测表面特征）
- 单篇评估，不涉及批量
- 愿意接受偶尔的 API 失败回退

### URL 去重：只用黑名单，不用 frontmatter

```
ingested: 字段不可靠！
  - wechat-mp-rss-extractor 写入 frontmatter 的 ingested: 日期不代表真正入库
  - cron 中断时可能只写了 frontmatter 没创建 raw/articles/ 文件
  - 正确做法：只查 raw/articles/*.md 的 source_url:/url: 字段
```

## Trigger

Use this skill when the user provides a web article, blog post, GitHub page, paper page, WeChat article, Zhihu post, Juejin/CSDN/Blogyuan page, Medium/dev.to post, or public URL and asks to review, summarize, save, or ingest it into the knowledge base.

## Outcome Contract

The workflow has two phases:
1. Score first and explain the decision.
2. If `value * confidence >= 49`, immediately complete wiki ingest before moving on.

No middle state counts as success. A qualifying article that was scored but not saved is a workflow failure.

## Scoring Gate

Use double scoring:
- `review_value` (0-10): article depth and incremental value to the wiki.
- `review_confidence` (0-10): credibility and verifiability.

**Mandatory pre-scoring step — classify content type first:**

1. Classify article type: `technical` / `analysis` / `news` / `tutorial` / `lifestyle` / `marketing`
2. Apply v/c mismatch corrections before computing v×c gate (see rubric)
3. Apply frontier-inclusive principle when article captures new perspectives from credible sources
4. Then evaluate `review_value * review_confidence >= 49`

**Note on borderline cases:** `46 <= v×c <= 52` does NOT require human confirmation — auto-ingest is correct. The frontier-inclusive principle and v/c mismatch corrections together handle most false-positive risks. Trust the rubric.

Save threshold:

```text
review_value * review_confidence >= 49
```

If the product is below 49, do not save unless the user explicitly asks for a non-wiki fallback. Use `references/scoring-rubric.md` for detailed rubrics and edge adjustments.

## Required Workflow

1. Load and follow `llm-wiki` before any wiki write.
2. Extract the article using `references/extraction-playbook.md`.
   - If extraction fails (Cloudflare, CAPTCHA, empty body), fallback to **tinyfish-web-agent**.
   - If extraction fails due to **paywall/subscription** (WeChat shows <1KB truncated content with no further paragraphs accessible), report extraction failure directly. Do not attempt TinyFish — the full content is inaccessible without credentials.
3. Score with the rubric and send Phase 1.
4. If below threshold, stop with the reason and do not create wiki files.
5. If threshold is met, check strategic alignment using `references/strategic-alignment.md`.
6. Save through `references/wiki-save-workflow.md`.
7. Persist review metadata in frontmatter, never tags.
8. Update `index.md`, `log.md`, and any relevant query/navigation page.
9. Run structural validation before Phase 2 success.
10. Reply using `references/reply-template.md`.

## Review Metadata

Use the following normalization for review metadata (canonical source — also duplicated in `llm-wiki` skill's `references/review-frontmatter-snippet.md`):

```text
review_stars <= 2 -> do not save  (一票否决，优先于 v×c)
review_value >= 9 and confidence >= 8 -> strong, stars 5
review_value >= 8 and confidence >= 7 -> strong, stars 4
review_value * review_confidence >= 49 -> worth-reading, stars 3
reference-only retained notes -> reference, stars 1-2  (不推荐入库)
```

> [!CAUTION]
> `review_stars <= 2` is a **hard veto** — it triggers `do not save` regardless of v×c score.
> This prevents lifestyle/软性 content (employee perks, personal stories, product showcases)
> from sneaking in via inflated v×c, even when the LLM assigns value=7, confidence=8 (v×c=56).
> stars is the LLM's own honest quality signal and should always be checked first.

## Pitfalls

### 软性内容借 v×c 灌水入库（2026-05-20）

**症状**：value=7, confidence=8, stars=3 的生活方式/个人分享类文章，v×c=56 过线入库，但实际知识密度极低。

**根因**：LLM 对宏观概念（"人才竞争""员工福利"）的泛化会人为抬高 v×c，但 stars 是诚实评分，不受来源认证加成。

**修复**：评分时先检查 stars，stars≤2 直接否决，无论 v×c 是否过线。此规则已在上述 Review Metadata 中硬编码。

### 格式欺骗：news 类文章借结构感将 c 虚标至 8（2026-05-21）

**症状**：量子位 AI 教育新闻文章，v=6, c=8 = 48，本应被 Gate 拦住，但 c=8 来自"格式完整"（案例列表、来源标注、OpenAI 背书），并非技术深度。最终入库后发现无任何技术细节。

**根因**：Confidence rubric 定义 c=8 为"Original work from known technical communities / Named author, practical experience"。新闻报道可以有 named examples 和 credible-looking citations，但这些佐证的是"事件"而非"可复用的技术知识"。c 衡量的是评分者对信息*可验证性和可操作性*的信心，不是格式完整度。

**触发路径**：
1. 文章为 news 类型（有案例、事件、案例演示，无技术细节）
2. Scorer 看到结构完整（标题层级、列表、来源标注）→ 给 c 高分
3. 同时 v 给了 6（中等价值，有一定信息量）→ 6×8=48，刚好在阈值线下 1 分
4. 边界案例未被拦截，自动入库

**修复（三层防御）**：
1. Content-type detection 必须作为评分第一步：news 类文章 confidence 上限为 6
2. v/c mismatch correction：c≥8 + v≤5 → cap v at c-2（格式欺骗信号）
3. 边界案例 46≤v×c≤52 必须上报用户确认，不得自动入库

**鉴别方法**：问自己——"读完这篇文章，我能直接复用/验证什么？"如果答案只是"知道有这么个事"，则是 news 类，confidence 不应超过 6。

Persist review metadata in entity page frontmatter only (never in tags):

```yaml
review_value: 8
review_confidence: 7
review_recommendation: strong   # strong | worth-reading | reference
review_stars: 4                 # optional, 1-5 for quick visual scanning
```

## Completion Gate

Phase 2 success requires all of the following:
- raw source and synthesized wiki files created or updated correctly
- review metadata persisted where applicable
- `index.md` and `log.md` updated
- structural/navigation changes considered and applied when useful
- validation passed
- actual file paths included in the closeout

If any item is missing, Phase 2 must be a failure closeout with a single explicit blocker.

## Batch Safety

When multiple URLs are provided, process one article through full Phase 2 closeout before scoring the next. The scoring decision is not the completion signal. See `references/batch-safety.md`.

## Failure Semantics

- If extraction fails, try **TinyFish** (`tinyfish-web-agent` skill) as fallback before reporting failure.
- If TinyFish also fails, report extraction failure and do not score.
- If the article is an image carousel that cannot be extracted, say so explicitly.
- If validation fails, do not claim the wiki is updated.
- If a qualifying article is not ingested, say `落库状态：failed` and name the blocker.
## WeChat Inbox Processing (Batch Mode)

当从 `raw/wechat-inbox/` 批量处理微信文章时：

### 流程

1. **检查已入库**：先对比 `entities/` 和 `raw/articles/`，跳过已存在的文章
2. **Heuristic 预过滤**（可选）：用文件内容长度、关键词快速排除明显低价值文章（减少 LLM 调用量）
3. **构建批量评分 prompt**：将待评分文章的前 3000 字符 + 标题组合，写入 `raw/email-inbox/wechat-batch-scoring-prompt.md`
4. **调用 LLM 评分**：使用 Baidu Qianfan API（`deepseek-v4-flash` 模型），endpoint `https://qianfan.baidubce.com/v2/coding/chat/completions`
5. **解析 JSON 结果**：`value × confidence >= 49` → 入库；否则拒绝
6. **入库处理**：对通过评分的文章，从 inbox 移动到 `raw/articles/`，创建 entity 页面，更新 `index.md`
7. **清理 inbox**：删除已处理的文件（无论入库还是拒绝）
8. **清理临时文件**：删除 `wechat-batch-scoring-prompt.md`

### 评分 API 配置

```python
api_url = 'https://qianfan.baidubce.com/v2/coding/chat/completions'
api_key = 'YOUR_BCE_V3_API_KEY'  # 在 config.yaml 的 custom_providers 下
model = 'deepseek-v4-flash'
```

### 批量评分 prompt 格式

```
你是一个技术文章质量评估助手。请评估以下文章是否值得入库到技术知识库。

评分标准：
- 价值(1-10)：技术深度、独特见解、实用价值、信息密度
- 置信度(1-10)：内容是否清晰可判断
- 入库：value×confidence ≥ 49 或有独特技术洞察

只输出JSON数组，不要输出其他内容：
[{"title": "...", "value": N, "confidence": N, "ingest": true/false, "reason": "..."}]

---
## {文章标题}
{前3000字符内容}
---
```

### 注意事项

- **不要对已入库文章重复评分**：先检查 `entities/` 和 `raw/articles/`，避免重复工作
- **批量大小**：19-20 篇/次是合理上限（响应长度 ~3000-6000 chars，API 超时 ~60-120s）
- **Heuristic 预过滤**：文件 <1KB、纯标题/空内容、明显新闻/公告 → 直接拒绝，不调 LLM
- **评分后清理**：无论入库还是拒绝，都要从 inbox 删除已处理文件

### WeChat 文件名含空格处理

微信文件名常见空格（如 `腾讯员工公寓曝光竟是这样的布置.md`），使用 xargs pipeline：

```bash
cd /Users/jinguo/wiki/raw/wechat-inbox
ls -1 | grep "filename" | xargs -I {} cp "{}" /Users/jinguo/wiki/raw/articles/
```

### MiniMax API ` thinks` 标签处理

`minimax-cn/MiniMax-M2-7` 返回的 JSON 响应中，`content` 字段值被包裹在 ` thinks` XML 标签内。必须先 strip 再提取 JSON：

```python
raw = re.sub(r'<thinks>.*?</thinks>', '', raw, flags=re.DOTALL)
data = json.loads(raw)
```

### MiniMax 配额耗尽时的即时回退（2026-05-17 实践）

当 MiniMax 返回 `NotEnoughCvError (code 11210)` 时，**不要停掉 pipeline**，立即切换到 `deepseek-chat` 继续评分：

```python
# 检测配额耗尽
if "NotEnoughCvError" in error or error.get("code") == 11210:
    # 切换到 deepseek-chat（需配置 deepseek provider）
    model = "deepseek-chat"
    # 继续评分，不中断批量流程
```

**注意**：这与 `thinks` 标签问题不同——配额耗尽是账户级限制（Token Plan Plus 15000/15000），会返回明确的 HTTP 错误码 11210，而非格式损坏。MiniMax 配额通常每日/每月重置，回退模型只是临时方案。

### Xunfei MAAS API 端点状态（2026-05-17 更新）

旧端点 `maas.lance4.ai` 已失效（DNS NXDOMAIN）。正确端点 `https://maas-coding-api.cn-huabei-1.xf-yun.com/v2` **可用于简单评分请求**（如 JSON 评分 prompt），但可能返回 One API HTML 页面用于复杂请求。

配置在 `config.yaml` 的 `custom_providers` 下，name 为 `maas`，model 为 `astron-code-latest`。

**降级策略**：当 MiniMax 配额耗尽时，先尝试 Xunfei endpoint 做简单评分。如果返回 HTML 非 JSON，再切换到 `deepseek-chat`。不要反复尝试不可用的 Xunfei 端点。

### Jina Reader 批量抓取 Cloudflare 拦截（2026-05-17 实践）

当一次性通过 Jina Reader 抓取 20+ URL 时，大部分会返回 46 字符（Cloudflare 拦截）。这不是内容问题而是限流。应对策略：
- 单次批量不超过 ~10 个 URL
- 若多数返回 46 字符，切换到 curl 逐个抓取剩余 URL
- 关键 URL（高价值/高评分候选）优先用 curl 直接抓取

## Progressive References

- `references/scoring-rubric.md` — value/confidence scoring tables and adjustment rules.
- `references/extraction-playbook.md` — extraction methods and source-specific failure modes.
- `references/wiki-save-workflow.md` — llm-wiki save steps, metadata, index/log, validation.
- `references/batch-safety.md` — multi-article sequencing and skipped-ingest prevention.
- `references/reply-template.md` — mandatory Phase 1 and Phase 2 response formats.

## References

- `references/wechat-inbox-handling.md` — Handling Chinese filenames with spaces in shell commands
