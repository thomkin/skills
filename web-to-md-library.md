---
name: web-to-md-library
description: Cache PDFs + web links as markdown. Reuse cache on future reference.
type: skill
trigger:
  - user says "fetch [URL]" or "PDF" or "link"
  - research task references external resource
  - second reference to cached resource
  - manual: "cache this resource"
---

# Skill 2: Web-to-MD Library Builder

**Purpose:** Cache external resources (PDFs, URLs, docs) as markdown. Reuse cache on future reference (cheap token cost).

**Inputs:**
- URL (web link) or PDF file path
- Optional: topic/category, cache metadata
- Manual trigger or auto-detect on research task

**Outputs:**
- Cached file: @memory/library/[slug].md (markdown)
- Index updated: @memory/MEMORY.md (link + metadata)
- Report: "Cached to library/[slug]. Next: uses cache (~0.1 cost vs 1.0 web fetch)"

---

## Implementation Steps

### Step 1: Detect Resource Fetch

**Auto-detection:**
- User: "fetch https://example.com/doc"
- User: "PDF at /path/to/document.pdf"
- User: "link: https://..." (in task description)
- Task mentions external resource (http://, https://, .pdf)

**Manual trigger:**
- User: "cache this resource"
- User: "/cache-library [URL]"

### Step 2: Extract Slug + Metadata

From URL or file:
```
URL: https://docs.example.com/guides/timeout-tuning.html
  → slug = "example-timeout-tuning"
  → domain = "docs.example.com"
  → category = "guides"

File: /path/to/postgresql-replication.pdf
  → slug = "postgresql-replication"
  → domain = "local-file"
  → category = "database"
```

**Slug rules:**
- lowercase, kebab-case
- max 50 chars
- remove extension (.pdf, .html, etc.)
- unique (append suffix if duplicate: -1, -2, etc.)

### Step 3: Fetch + Parse Content

**For URLs (HTTP/HTTPS):**
```
1. Fetch page
2. Extract: title, description, main content
3. Identify: code blocks, links, warnings, examples
4. Remove: tracking, ads, navigation
5. Parse: tables, lists, headings
6. Output: clean markdown
```

**For PDFs:**
```
1. Extract text from PDF
2. Preserve: structure (headings, sections)
3. Extract: tables, images (as descriptions)
4. Remove: header/footer, page numbers
5. Identify: warnings, code samples
6. Output: markdown with sections + examples
```

**Content extraction structure:**
```markdown
# [Document Title]

**Source:** [URL or file path]  
**Fetched:** 2026-07-24 16:30  
**Category:** [guides / api-docs / research / etc.]

## TL;DR
[1-3 sentence summary]

## Key Sections
[Main content, organized by heading]

## Code Examples
[Any code samples, preserved as-is]

## Warnings / Important Notes
[Any critical info, highlighted]

## Related Links
[Links mentioned in document]

## Metadata
- Source URL: [full URL]
- Cache date: 2026-07-24
- Expiry hint: Update if source changed (manual check)
```

### Step 4: Save to Library

**File location:** `@memory/library/[slug].md`

**Directory structure:**
```
@memory/library/
  ├── example-timeout-tuning.md
  ├── postgresql-replication.md
  ├── cloudflare-api-reference.md
  └── README.md (index of all cached resources)
```

**File format:** Standard markdown with metadata header (frontmatter optional)

### Step 5: Update Index

File: `@memory/MEMORY.md`

Add entry under "## Library" section:
```markdown
## Library

- [Timeout Tuning Guide](library/example-timeout-tuning.md) — docs.example.com/guides
- [PostgreSQL Replication](library/postgresql-replication.md) — postgresql.org
- [Cloudflare API Reference](library/cloudflare-api-reference.md) — api.cloudflare.com
```

Also update library README:
```markdown
# Cached Resources

## By Category

### Guides
- [Timeout Tuning](example-timeout-tuning.md) — fetched 2026-07-24

### API References
- [Cloudflare API](cloudflare-api-reference.md) — fetched 2026-07-22

### Research Papers
- [PostgreSQL Replication](postgresql-replication.md) — fetched 2026-07-20

## Cache Statistics
- Total resources: 3
- Total size: ~150KB (compressed)
- Last updated: 2026-07-24
- Average age: 2 days
```

### Step 6: Cost Comparison

**First reference (fetch from web):**
```
Cost: ~1.0 token budget (web fetch + parsing + storage)
Time: 5-10 seconds (network + parsing)
```

**Cached reference:**
```
Cost: ~0.1 token budget (read markdown file)
Time: <100ms (file I/O)
Savings: 90% cost, 50x faster
```

Report on cache hit:
```
[use:: web-to-md-library-cached] Using cached: library/example-timeout-tuning.md
Cost savings: 0.9 tokens (90% cheaper than web fetch)
Last updated: 2026-07-24 (fresh)
```

### Step 7: Cache Expiry + Refresh

**Expiry rules:**
- Default: 7 days (automatic reminder to refresh)
- Research docs: 14 days (less frequent changes)
- API references: 3 days (frequent updates)
- Custom: user-specified (for rarely-changed docs)

**Refresh on demand:**
- User: "refresh library/example-timeout-tuning.md"
- Skill 2 re-fetches → compares with cache → updates if changed
- Report: "Updated: 5 sections changed, 2 new examples"

**Stale marker:**
```markdown
⚠️ **Cache Note:** Last fetched 2026-07-20 (4 days old)
Source may have changed. Refresh? (user can trigger manual refresh)
```

### Step 8: Report to User

Report on cache save:
```
[use:: web-to-md-library-project] Cached resource:
  File: library/example-timeout-tuning.md
  Source: docs.example.com/guides/timeout-tuning.html
  Size: ~2KB
  Sections: 5 (TL;DR, Key Concepts, Examples, Warnings, Links)
  
Cache hit savings: Next reference ~0.1 cost vs 1.0 web fetch
Expiry: 7 days (refresh: "refresh library/example-timeout-tuning.md")
```

---

## Integration with Memory System

**Works with Phase 2 (Memory Auto-Updater):**
1. Skill 2 caches resource → library/[slug].md created
2. Skill 2 updates MEMORY.md → adds library entry
3. Skill 1 detects change → updates @memory/ snapshot if >150 lines
4. Snapshot preserves entire library (cached resources survive session)

**Works with Skill 5 (Decision Logger):**
- Decision references external resource → Skill 2 auto-caches
- Future: "Have we solved timeout problems?" → grep decisions.md → links to library/example-timeout-tuning.md
- Library enables reproducible research (decisions linked to sources)

---

## Search + Retrieval

**Find cached resources:**
```bash
# List all cached resources
ls @memory/library/

# Search by topic
grep -l "timeout\|replication\|cloudflare" @memory/library/*.md

# Search within cached resource
grep "connection pool" @memory/library/postgresql-replication.md
```

**Browser-style access:**
- MEMORY.md index → clickable links to library files
- Topic files (topic/headscale.md) → can reference library entries
- Decisions can link to sources: `See library/postgresql-replication.md for details`

---

## Example Execution

**Scenario 1: First fetch (expensive)**

User: "fetch https://docs.example.com/guides/timeout-tuning.html"

**[Skill 2 executes]:**
1. Parse URL → slug = "example-timeout-tuning"
2. Fetch page → extract content
3. Parse: TL;DR, sections, examples, warnings
4. Save to library/example-timeout-tuning.md
5. Update MEMORY.md index
6. Report:

```
[use:: web-to-md-library-project] Cached resource:
  File: library/example-timeout-tuning.md
  Source: docs.example.com/guides/timeout-tuning.html
  Size: ~2.5KB
  Sections: 6 (TL;DR, Overview, Tuning Strategy, Examples, Warnings, Links)
  
Cost: 1.0 (web fetch + parse)
Next fetch: 0.1 (cache hit)
```

**Scenario 2: Second reference (cheap cache hit)**

User (in new task): "How to tune timeout? Reference Skill 2 docs on timeout"

**[Skill 2 detects cache]:**
1. Check library/ for "timeout"
2. Find library/example-timeout-tuning.md
3. Load from disk (not web)
4. Return cached content
5. Report:

```
[use:: web-to-md-library-cached] Using cached: library/example-timeout-tuning.md
  Fetched: 2026-07-24 (fresh, 0 days old)
  Sections: 6 available
  Cost: 0.1 (90% savings vs web fetch)
  
To refresh: "refresh library/example-timeout-tuning.md"
```

**Scenario 3: Research task (multiple resources)**

Task: Investigate PostgreSQL replication patterns

**[Skill 2 executes multiple times]:**
1. Fetch: postgresql.org/docs/replication
   → Cache: library/postgresql-replication.md (0.1 cost after first fetch)
2. Fetch: wiki.postgresql.org/replication-faq
   → Cache: library/postgresql-faq.md
3. Fetch: github.com/postgres/postgres/replication-guide
   → Cache: library/postgres-replication-guide.md

Result: 3 resources cached (~0.2 cost vs 3.0 web fetches = 93% savings)

---

## Cache Organization

**By category (library/README.md):**
- Guides (how-tos, tutorials)
- API References (endpoints, schemas)
- Research Papers (academic, whitepapers)
- Configuration Examples (setup, deployment)
- Troubleshooting (error analysis, solutions)
- Specifications (standards, formats)

**By component (library/[topic]/[slug].md):**
```
library/
  ├── headscale/
  │   ├── client-config.md
  │   ├── deployment-guide.md
  │   └── troubleshooting.md
  ├── postgresql/
  │   ├── replication.md
  │   ├── monitoring.md
  │   └── tuning.md
  └── cloudflare/
      ├── api-reference.md
      ├── waf-rules.md
      └── failover-guide.md
```

---

## Testing Checklist

- [ ] URL parsing works (extract domain, slug)
- [ ] PDF parsing works (extract text + structure)
- [ ] Content extraction works (TL;DR, sections, examples)
- [ ] Markdown formatting correct (headings, code blocks)
- [ ] File save works (@memory/library/[slug].md created)
- [ ] Index update works (MEMORY.md + library/README.md updated)
- [ ] Cache hit works (second reference uses file, not web)
- [ ] Cost tracking works (reports 0.1 vs 1.0)
- [ ] Expiry detection works (stale marker if >7 days)
- [ ] Refresh works (manual re-fetch + compare)

---

## Notes

- Skill 2 runs automatically on resource mention (or manual trigger)
- Cache is git-tracked (survives session, shared across team)
- Expiry hints remind to refresh (but don't force)
- Works with any LLM (markdown is portable)
- Integration: Skill 1 snapshots preserve library (full recovery possible)
