---
name: web-content-reviewer
description: "Peer-review any web article or blog post and save high-quality ones to Obsidian wiki using llm-wiki standards. Double scoring: article value + confidence. Strict standards, no shallow content."
version: "1.0"
author: user
related_skills: [llm-wiki]
---

# Web Content Reviewer

Peer-review any web article or blog post and save high-quality ones to the Obsidian wiki.

## Trigger

When the user sends ANY web article or blog post link (WeChat mp.weixin.qq.com, 微信公众号, 知乎 zhihu.com, 掘金 juejin.cn, CSDN, 博客园, Medium, dev.to, GitHub issues/README, ArXiv, or any public URL) and asks to review/summarize/save/ingest it into the knowledge base.

## Role

Act as a strict peer reviewer. Judge every article independently. Save only high-quality content to the Obsidian wiki at `~/wiki` (which IS the Obsidian vault). Reject shallow, repetitive, or low-credibility content.

## Double Scoring System

### 1. Article Value (0-10)
Content quality and incremental value to the knowledge base.

| Score | Meaning |
|-------|---------|
| 9-10 | Must save: source-code-level depth / new domain / strong cognitive refresh |
| 7-8 | Worth saving: systematic knowledge increment, fills a gap |
| 5-6 | Reference only: local highlights, similar content already in knowledge base |
| 0-4 | Do not save: shallow, repetitive, low technical value |

### 2. Confidence (0-10)
Credibility of the article / author / source. How verifiable are the claims?

| Score | Source Type | Characteristics |
|-------|-------------|-----------------|
| 9-10 | First-hand source code analysis, official tech blog, GitHub-verified | Has specific code, data, citations; not secondhand |
| 7-8 | Original work from known tech communities (Alibaba Cloud/Taobao/DataSTUDIO/etc.) | Author named with proven practice; content cross-verifiable |
| 5-6 | Industry observation/reviews/secondhand interpretation | No source code/data, opinion-based, relies on author's judgment |
| 3-4 | Republished summary, quick news | Secondhand, no independent verification, factual doubts |
| 0-2 | Marketing content, unverified rumors | Not credible |

## Decision Matrix

**Threshold rule: Value × Confidence ≥ 49 must be met to save.**

| Value × Confidence | Action |
|-------------------|--------|
| ≥ 49 | Save |
| < 49 | ❌ Do not save — no exceptions |

> Important: Even if Value ≥ 7 and Confidence ≥ 7 individually, if the product is < 49, do NOT save. Example: Value 6 × Confidence 7 = 42 < 49 → do not save.

No "middle ground" — articles either meet the threshold or they don't.

**Strict rule: If threshold (≥49) IS met, the article MUST be saved. Do not skip, do not skip for later, do not note it and move on. Every qualifying article must be immediately入库.**

When saving a reviewed article into the wiki, persist the review result as frontmatter metadata:
- `review_value: X`
- `review_confidence: Y`
- `review_recommendation: strong | worth-reading | reference`
- `review_stars: 1-5` (optional quick visual signal)

Do **not** store these as tags. Tags are reserved for topic taxonomy; review signals are ranking metadata.

Use this normalization so review metadata stays comparable across articles:
- `review_recommendation: strong` when `value >= 8` and `confidence >= 7`
- `review_recommendation: worth-reading` when the article is saved but does not meet the `strong` bar
- `review_recommendation: reference` only for lower-priority scored notes kept outside the main wiki workflow
- `review_stars: 5` for `value >= 9` and `confidence >= 8`
- `review_stars: 4` for strong saves that do not reach the top tier
- `review_stars: 3` for standard worthwhile saves
- Avoid `review_stars: 1-2` for main wiki ingest unless you intentionally keep low-priority reviewed material

## Review Criteria

1. **Depth** — Source-code-level analysis, specific data, principle-level insights? Reject empty summaries/news flashes/basic popular science
2. **Uniqueness** — New knowledge not in my wiki? Does it surpass or repeat existing articles?
3. **Practical Value** — Guides practice (tool usage, architecture design, tech decisions, code writing)?
4. **Technical Content** — Author with practical experience (source code > official release > republished info)?
5. **Timeliness** — Key node in current AI/Agent wave (Self-Evolving, RL training loops, tool abstraction, etc.)?

## Content Extraction Techniques

### WeChat/MpWeixin Articles

WeChat articles (mp.weixin.qq.com) are JS-rendered — `browser_navigate` snapshot does NOT capture full article text. Use this approach:

1. `browser_navigate(url)` — navigate to the WeChat article URL (URLs like `mp.weixin.qq.com/s/xxx` are redirect URLs; navigate first to resolve the final URL)
2. **For short articles** (< 8000 chars): `browser_console` with `document.body.innerText` — returns full text in one call
3. **For long articles** (≥ 8000 chars): Use `document.body.innerText.slice(offset, limit)` with 8000-char windows, call multiple times with different offsets (e.g., `slice(0, 8000)`, `slice(8000, 16000)`, `slice(16000, 24000)`), then concatenate
4. **Multi-scroll + retry on empty**: If the article has lazy-loaded images/comments below fold, scroll down with `browser_scroll(direction="down")` 2-3 times before extracting — this triggers WeChat's lazy rendering
   - **IMPORTANT**: If `browser_console` returns empty on the first call (even after scrolling), WAIT a moment and retry. WeChat JS rendering may not have completed. Re-navigating to the URL and immediately calling `browser_console` also works as a fallback.
   - **Worked pattern**: `browser_navigate(url)` → `browser_scroll(direction="down")` × 2 → `browser_console`

5. **Image carousel articles (图片轮播)**: Some WeChat articles present content primarily as a slideshow of images (typically 5-15 images, identified by numbered pagination buttons like "1" "2" "3" ... in the snapshot). Signs:
   - Heading exists but `document.body.innerText` returns only the title/tags/author with no article body
   - Snapshot shows multiple page-turner buttons (数字 1-11)
   - Most content is inside `<img>` tags rendered on a canvas
   - **OCR limitation**: vision tools (vision_analyze) consistently fail to extract readable text from WeChat article screenshots — do not rely on OCR for these articles
   - **Decision**: Skip scoring, inform user "图片轮播文章，正文在图片中无法提取"
   - **If user wants to save anyway**: Save images to `assets/images/[topic]/` directory with sequential names (01-cover.jpg, 02.jpg, etc.) and create a markdown index file with `![](assets/images/...)` embeds. Update log.md with action "save" (not "ingest") and increment Total pages by count of files saved.

### GitHub Blob Pages

When viewing source files on GitHub (`github.com/user/repo/blob/main/path.md`):
- **Problem**: The page includes GitHub headers, line numbers, and UI — not clean content; blob URL also requires login for some repos
- **Solution**: Navigate to the **raw content URL**: `raw.githubusercontent.com/user/repo/main/path.md`
- This gives clean markdown/text without any GitHub UI wrapper
- Works for any file GitHub can render (`.md`, `.py`, `.txt`, `.json`, etc.)
- **Always try raw first** if blob URL shows login wall or empty content

### 小红书 (Xiaohongshu) Articles

- 小红书 requires login/authentication and blocks anonymous access with IP risk errors
- If you encounter error 300012 ("IP存在风险，请切换可靠网络环境后重试") or login wall — skip and notify the user
- Alternative: ask the user to paste the article content directly

### Notion Fallback (Low-Value Articles)

If an article scores below threshold but the user wants to save it somewhere:
1. Use the Notion API to create a page under the user's Notion home
2. **Root page ID**: `34c0350f-4ef6-8160-a15d-ea85e3388083` (user's AI learning root page)
3. Create via Python script — write the payload to a temp file, then `python3 /tmp/script.py`
4. Example blocks structure:
   ```python
   blocks = [
       {"object": "block", "type": "heading_2", "heading_2": {"rich_text": [{"type": "text", "text": {"content": "文章信息"}}]}},
       {"object": "block", "type": "paragraph", "paragraph": {"rich_text": [{"type": "text", "text": {"content": "URL: ..."}}]}},
       {"object": "block", "type": "heading_2", "heading_2": {"rich_text": [{"type": "text", "text": {"content": "核心要点"}}]}},
       {"object": "block", "type": "bulleted_list_item", "bulleted_list_item": {"rich_text": [...]}},
   ]
   ```
5. API endpoint: `POST https://api.notion.com/v1/pages` with header `Authorization: Bearer $NOTION_API_KEY`
6. The new page ID is returned in the API response — construct URL: `https://www.notion.so/{page_id_parsed}`

## Wiki Save Workflow (MUST use llm-wiki skill)

**CRITICAL:** Every save to the wiki MUST follow the `llm-wiki` skill standard. Load it first via `skill_view(name="llm-wiki")` before saving. The llm-wiki skill defines:
- Frontmatter schema (title, created, updated, type, tags, sources, confidence)
- Optional review metadata (`review_value`, `review_confidence`, `review_recommendation`, `review_stars`)
- File layout (raw/ + entities/ + comparisons/ + concepts/)
- Provenance markers (^[[raw/...]] on claims from specific sources)
- Cross-reference requirements (minimum 2 wikilinks per page)
- Index/log update requirements
- Tag taxonomy constraints

For each article judged worthy:

1. **Load llm-wiki skill** via `skill_view(name="llm-wiki")` — read the full skill before saving
2. **Navigate** to the WeChat article URL with browser
3. **Extract** full text via `browser_console` expression: `document.body.innerText`
4. **Create** files following llm-wiki conventions:
   - `raw/articles/[slug].md` — original article archive with raw frontmatter (source_url, ingested, sha256)
   - `entities/[slug].md` — entity page with full llm-wiki frontmatter + review metadata + structured summary + wikilinks to related entities
   - `comparisons/[slug].md` — (if comparing multiple tools/systems) side-by-side comparison page
5. **Cross-reference** — new entity pages MUST link to at least 2 existing wiki pages via `[[wikilinks]]`
6. **Update and organize** the current wiki:
   - update `index.md` (Total pages +N, add entry alphabetically under the right section)
   - update `log.md` (append ingest record with file list)
   - if the new article reveals a better navigation surface, add or refresh query/navigation pages
   - if the touched section is getting crowded or stale, reorganize summaries/structure while you are there
7. **Run structural lint** after saving — use the `llm-wiki` completion rule; do not report success before it passes
8. **Reply** with the standard review format

## Reply Format

**IMPORTANT — Two-Phase Reply:**

### Phase 1: Score first (no ingestion yet)
```
## [文章标题]

**概要**：（200字内）

**评分**：
- 价值：X/10
- 置信度：X/10

**值得关注程度**：⭐⭐⭐（3-5个⭐）
```
→ If score < 49: stop here. One-line reason why not saved.

### Phase 2: Ingest and confirm (only if score ≥ 49)
After completing steps 4–7 (file creation + index/log update + lint), reply with:

Template:

```text
references/reply-template.md
```

**STRICT RULES:**
- Phase 2 reply MUST include whether the article was successfully ingested.
- If ingestion failed, say so explicitly and do not imply the wiki is up to date.
- Phase 2 reply MUST include the actual file paths created or updated.
- Phase 2 success closeout MUST include wiki page delta, index/log or structure update status, lint result, and review metadata.
- If the agent only sends Phase 1 without Phase 2 for a score ≥49 article, that is a bug — the ingestion steps were skipped.
- Prefer the success/failure templates in `references/reply-template.md` over ad hoc wording.

If the target wiki maintains a review queue page such as `queries/review-queue.md`,
keep the reply metadata compatible with that queue by using the normalized
`review_recommendation` / `review_stars` mapping from `llm-wiki`.

## Batch Processing Safety Rule

When processing multiple articles in sequence, **do not begin scoring a new article until Phase 2 of the previous article is fully complete** (file creation + index update + log written + user confirmed). The scoring decision (≥49) is NOT the completion signal — file creation is.

**The failure mode this prevents:** In batch processing, it's easy to mentally conflate "decided to ingest" with "actually created the files." This happened when processing 4+ articles in sequence — Phase 1 was sent for all articles immediately, but file creation for articles 3 and 4 was skipped because Phase 2 of articles 1 and 2 hadn't been confirmed before starting articles 3 and 4.

**Practical rule:** Before scoring article N+1, verify that article N's log.md entry exists and contains the actual file paths. If not, complete article N's ingestion first.


## Duplicate Detection

When the same topic appears multiple times (e.g., Karpathy LLM Wiki covered by InfoQ, AGI Hunt, and AI寒武纪 all in the same session):
- **First report (highest confidence source)** → evaluate and decide
- **Second report (same content, different source)** → if content is identical, reject quickly unless it adds new incremental information
- **Third report (same content again)** → reject unless it has genuinely new information
- If in doubt, compare key claims against the existing wiki content before saving

**Same creator, multiple publishers**: When the same person (e.g., Boris Cherny on Claude Code, Karpathy on LLM Wiki) is covered by different publishers — save the most authoritative version, reject others as duplicates even if wording differs. Primary-source articles where the tool creator speaks for themselves always take priority over secondhand interpretations.

## Value Score Adjustments

- **Primary-source tip/trick articles**: A "feature list" article from the tool creator themselves (e.g., Boris Cherny sharing his own Claude Code workflow) may be underweighted at Value 6. Upward-adjust to Value 7 when: (a) source is the actual tool author, AND (b) contains non-obvious engineering details (specific production scale, workaround flags, undocumented combinations). 6×8=48 borderline → 7×8=56 saves.
- **GitHub README-backed WeChat articles**: WeChat articles that are based on or directly reference GitHub README content get +1 Confidence boost (claims are cross-verifiable against source code/commits).

## Merge Logic

If an article covers similar topic as existing wiki content:
- **Same topic, richer content** → Overwrite the existing entity with expanded content, update frontmatter updated date
- **Same topic, different angle** → Merge unique insights into existing entity, note the new source
- **Duplicate without 增量** → Skip saving, inform user "已有更完整的版本"

## Key Files Reference

- Wiki location: `~/wiki` (this IS the Obsidian vault)
- Wiki structure: `entities/`, `raw/`, `comparisons/`, `concepts/`, `queries/`, `_archive/`
- Wikilink format: always use `[[subdir/basename]]` (e.g., `[[entities/hermes-agent]]`), never bare `[[basename]]`
- After saving: always update `index.md` (Total pages +N) and `log.md` (append ingest record)
