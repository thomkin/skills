---
name: memory-auto-updater
description: Auto-save decisions + snapshots. Prune rolling memory at ~150 lines.
type: skill
trigger:
  - task completed (04_Verification.md done)
  - manual save request ("save memory" / "update memory")
  - session end (periodic checkpoint)
---

# Skill 1: Memory Auto-Updater

**Purpose:** Auto-save learnings + maintain rolling memory without manual invocation.

**Inputs:**
- Task folder path (bugs/[slug]/ or features/[slug]/)
- Optional: manual trigger + topic/note

**Outputs:**
- Memory files updated (decisions.md, topic-specific files)
- Snapshot created if threshold hit (~150 lines)
- Report: "Memory updated: [files]. Snapshot: [timestamp] (pruned N lines)"

---

## Implementation Steps

### Step 1: Detect Task Completion

**Trigger condition:**
- User states "04_Verification complete" OR
- All 4 step files exist + marked complete OR
- Manual: "save memory" or "memory update"

**Extract task metadata:**
```
task_type: "bug" or "feature"
task_slug: "fix-headscale-timeout"
task_path: "bugs/fix-headscale-timeout/"
completion_date: 2026-07-24T15:30:00Z
```

### Step 2: Extract Decision from 02_Planning.md

Parse 02_Planning.md:
```
[DECISION]
Because: [problem description]
Did: [solution chosen]
Risk: [mitigations if any]
Result: [expected outcome]
```

Example extraction:
```
Because: Headscale client timeout on discovery
Did: Increase timeout from 5s → 15s in client config
Risk: Slower detection of dead servers (mitigated by heartbeat)
Result: Clients won't timeout during peak load
```

### Step 3: Append to decisions.md

File: `@memory/decisions.md`

Format: `[YYYY-MM-DD HH:MM] [type] [slug] — [decision summary]`

Entry template:
```markdown
## [2026-07-24 15:30] bug: fix-headscale-timeout
**Task:** bugs/fix-headscale-timeout/
**Because:** Clients timeout during discovery
**Did:** Increase timeout 5s → 15s
**Risk:** Slower dead-server detection (mitigated: heartbeat)
**Outcome:** Clients stable under peak load
**Learned:** Timeout thresholds tightly coupled to network latency
```

Append to end of decisions.md with date + link to task.

### Step 4: Update Topic-Specific Files

Identify topic from task + 02_Planning.md:
```
bug: fix-headscale-timeout → topic = "headscale"
feature: add-cloudflare-r2 → topic = "cloudflare"
```

File: `@memory/[topic].md`

If file doesn't exist, create with header:
```markdown
# [Topic] Learnings

[entries go here]
```

Append new fact:
```markdown
### [Date] Headscale Client Timeout
- Timeout config: client-discovery timeout
- Threshold: 5s → 15s (adjustable per network)
- Trigger: High load, slow responses
- Fix: Increase timeout + add heartbeat check
- Result: Stable under peak load, proper fallback
```

### Step 5: Check Memory.md Line Count

File: `@memory/MEMORY.md`

Count lines:
```
if line_count > 150:
  create_snapshot()
  prune_memory()
else:
  done()
```

MEMORY.md contains: index entries (one line each) linking to all memory files.

Example:
```markdown
# Memory Index

- [Decisions](decisions.md) — task decisions + outcomes
- [Headscale](topic/headscale.md) — headscale component learnings
- [Cloudflare](topic/cloudflare.md) — Cloudflare setup patterns
- [Feedback: Autonomy](feedback_autonomy.md) — rules for autonomous execution
[... more entries ...]
```

### Step 6: Snapshot Mechanism (If >150 lines)

**Trigger:** MEMORY.md exceeds ~150 lines

**Create snapshot:**
```
File: @memory/snapshots/YYYYMMDD-HHMM_label.md
Content: Copy current entire @memory/ (all files)
Include: Timestamp, line count, entries included
```

Snapshot template:
```markdown
---
name: snapshot-20260724-1530-phase1-complete
description: Phase 1 complete. 6 skills implemented + tested.
date: 2026-07-24T15:30:00Z
line_count_before: 168
line_count_after: 92
entries_archived: 76
---

# Snapshot: Phase 1 Complete (2026-07-24 15:30)

## MEMORY.md Index (before)
[copy of full MEMORY.md entries that were pruned]

## decisions.md Pruned Entries
[copy of old decisions (older than 2 weeks)]

## Archived Topic Files
[list of topic files moved to snapshots/archived/]

## Current State
[link to fresh MEMORY.md after prune]
```

### Step 7: Prune Rolling Memory

**Goal:** Keep MEMORY.md ~150 lines by archiving old entries

**Prune rules:**

1. **Age-based:** Remove entries older than 2 weeks
   - Keep: last 14 days
   - Archive: older than 14 days

2. **Retention:** Always keep most-recent decision per topic
   - If topic only has old entries: keep 1 most-recent
   - Preserve pattern even if old

3. **Snapshot preservation:** Before pruning, copy to snapshots/

**Prune process:**

```
Step 1: Identify old entries (>14 days old)
Step 2: Copy to snapshots/YYYYMMDD-HHMM_[label].md
Step 3: Remove from MEMORY.md index
Step 4: Move old topic files to snapshots/archived/[topic].md
Step 5: Keep current entries in @memory/[topic].md
Step 6: Recount lines, report results
```

Example output:
```
Memory pruned:
  Removed: 76 lines (decisions older than 14 days)
  Archived: 3 topic files → snapshots/archived/
  Snapshot: snapshots/20260724-1530_phase1-complete.md
  New line count: 92 (from 168)
  Next threshold: 150 lines
```

### Step 8: Report to User

Report format:
```
[use:: memory-auto-updater-project] Memory updated:
  ✓ decisions.md (+1 entry)
  ✓ topic/headscale.md (+1 fact)
  ✓ MEMORY.md index (92 lines → within threshold)
  ✓ Snapshot: snapshots/20260724-1530_phase1-complete.md

Learned: Headscale timeout tuning 5s→15s stable under load
Next: Review snapshot if session ends
```

---

## Integration with Workflow

**When runs:** After step 4 (Verification complete)

**Workflow trigger:**
1. User completes 04_Verification.md
2. Skill 1 auto-activates (keyword: "verify complete" or step 4 done)
3. Extracts decision from 02_Planning.md
4. Appends to decisions.md + topic file
5. Checks MEMORY.md line count
6. If >150 lines: snapshot + prune
7. Reports completion

**Integration with Skill 5 (Decision Logger):**
- Skill 1 handles snapshot/prune
- Skill 5 handles "because A, did B" pattern extraction
- Both feed same decisions.md file (Skill 1 organizes, Skill 5 logs patterns)

---

## Memory File Structure

```
@memory/
  ├── MEMORY.md              # Index (~150 line rolling threshold)
  ├── decisions.md           # All task decisions + outcomes
  ├── feedback_autonomy.md   # Rules for autonomous execution
  ├── feedback_skills.md     # How to load + apply skills
  ├── feedback_commit_rules.md # Git discipline rules
  ├── topic/
  │   ├── headscale.md       # Headscale component learnings
  │   ├── cloudflare.md      # Cloudflare setup patterns
  │   └── ...
  ├── library/
  │   ├── external-runbooks/ # Cached PDFs/links
  │   └── baselines/         # Thresholds, metrics, SLOs
  └── snapshots/
      ├── YYYYMMDD-HHMM_label.md
      └── archived/
          ├── headscale.md   # Old topic file
          └── ...
```

---

## Pruning Algorithm (Detailed)

**Input:** Current MEMORY.md, decisions.md, topic files

**Output:** Pruned state, snapshot created

```
1. Parse MEMORY.md → extract index entries
2. For each entry:
   a. Get file (decisions.md, topic/[name].md, feedback_*.md)
   b. Read file content
   c. For each entry in file:
      - Extract date (if present)
      - Calculate age = now - date
      - If age > 14 days: mark for archive
      - If age <= 14 days: keep in memory

3. Archive marked entries:
   a. Copy to snapshots/YYYYMMDD-HHMM_[label].md
   b. Remove from current file
   c. If file empty after removal: move entire file to snapshots/archived/

4. Recount lines in MEMORY.md
5. If still > 150 lines: repeat prune (lower threshold to 10 days)

6. Update MEMORY.md index:
   - Remove entries for archived files
   - Update line counts if shown
   - Add pointer to latest snapshot

7. Report:
   - Lines removed
   - Files archived
   - Snapshot location
   - New line count
```

---

## Testing Checklist

- [ ] Decision extraction works (parses 02_Planning.md)
- [ ] decisions.md append works (format correct, date added)
- [ ] Topic file creation works (creates if missing)
- [ ] Topic file update works (appends fact with date)
- [ ] Line count detection works (detects > 150 lines)
- [ ] Snapshot creation works (files copied, timestamp added)
- [ ] Prune age detection works (correctly identifies >14 days)
- [ ] Prune removal works (old entries deleted)
- [ ] MEMORY.md index update works (reflects pruned state)
- [ ] Report format correct (shows what updated + pruned)

---

## Example Execution

**Scenario:** Task completed, memory grows beyond threshold

**Before:**
```
@memory/MEMORY.md: 168 lines
@memory/decisions.md: 35 decisions (oldest: 2026-07-10)
@memory/topic/headscale.md: 28 entries (oldest: 2026-07-08)
```

**Trigger:** User: "04_Verification complete"

**[Skill 1 executes]:**
1. Extract decision from bugs/fix-headscale-timeout/02_Planning.md
2. Append to decisions.md (+1 entry → 36 total)
3. Update topic/headscale.md (+1 fact → 29 total)
4. Check MEMORY.md: 168 lines → OVER THRESHOLD
5. Create snapshot: snapshots/20260724-1530_phase1-complete.md
6. Prune: Remove entries >14 days old (6 decisions, 3 topic facts)
7. Update MEMORY.md: Remove archived entries (recount: 92 lines)
8. Report:

```
[use:: memory-auto-updater-project] Memory updated:
  ✓ decisions.md (+1 entry: fix-headscale-timeout)
  ✓ topic/headscale.md (+1 fact: timeout tuning)
  ✓ Snapshot: snapshots/20260724-1530_phase1-complete.md
  ✓ Pruned: 76 lines (>14 days old)
  ✓ New count: 92 lines (below 150 threshold)

Next task will have fresh memory base. Snapshot stored for recovery.
```

---

## Notes

- Skill 1 runs after EVERY task completion (automatic, invisible)
- Snapshots are git-tracked (version control acts as backup)
- Pruning is reversible (snapshots kept for 6 months minimum)
- Integration with Skill 5: both append to decisions.md (Skill 1 organizes, Skill 5 logs patterns)
- Works with Phase 2 + all subsequent phases
