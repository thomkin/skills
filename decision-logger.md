---
name: decision-logger
description: Track causality "because A, did B" for pattern recognition + audit trail.
type: skill
trigger:
  - planning approved (02_Planning.md approved)
  - execution complete (03_Execution.md done)
  - user explicit decision ("decided: ...", "chose ...")
  - task completion (04_Verification.md done)
---

# Skill 5: Decision Logger

**Purpose:** Log decisions with reasoning for pattern matching + future reference.

**Inputs:**
- Task folder (bugs/[slug]/ or features/[slug]/)
- Decision context (from 02_Planning.md or user statement)
- Optional: related decisions, status, constraints

**Outputs:**
- Decision logged to @memory/decisions.md (structured format)
- Pattern tags extracted (topic, constraint, outcome)
- Report: "Logged decision: [title]. Searchable by [tags]"

---

## Implementation Steps

### Step 1: Detect Decision Point

**Automatic triggers:**
1. 02_Planning.md marked "approved"
   - Extract design decision from "Solution Approach" section
2. 03_Execution.md marked "done"
   - Extract implementation decision from "File Changes" section
3. 04_Verification.md marked "done"
   - Extract outcome decision from "Result Checklist" section

**Manual triggers:**
- User: "decided: [title] because [reason]"
- User: "chose [option A] over [option B] because [reason]"

### Step 2: Parse Decision Structure

Extract from 02_Planning.md (design decision):
```
Problem Statement: [what's wrong]
Solution Approach: [high-level plan]
Logic Map: [step-by-step]
Error Handling: [how to handle errors]
```

Map to decision structure:
```
Title: "Increase Headscale timeout from 5s to 15s"
Because: "Clients timeout during peak load discovery phase"
Did: "Modified client config discovery timeout: 5s → 15s"
Why: "Timeout was too tight for network latency + GC pauses"
Risk: "Slower detection of dead servers"
Mitigation: "Added heartbeat check (separate 30s timeout)"
Expected Outcome: "Stable client connections under peak load"
Related: bug/fix-headscale-heartbeat (future)
Status: "approved" (before execution)
```

Extract from 03_Execution.md (implementation decision):
```
File Changes: [what code changed]
Rationale: [why this approach]
Trade-offs: [alternatives considered]
```

Map to:
```
Title: "Use config-based timeout instead of hardcoded"
Because: "Allows tuning per environment without code change"
Did: "Moved timeout constant to config file, load at startup"
Why: "Dev/UAT/Prod have different network profiles"
Trade-off: "Extra file I/O (mitigated: cached on load)"
Status: "completed"
```

### Step 3: Structured Logging Format

File: `@memory/decisions.md`

Entry template (Markdown with YAML frontmatter):
```markdown
## [2026-07-24 15:30] bug: fix-headscale-timeout

**Title:** Increase Headscale timeout 5s → 15s

**Because:** Clients timeout during peak load discovery

**Did:** Modified client-config discovery timeout threshold

**Why:** Timeout too tight for network latency + GC pauses

**Risk:** Slower dead-server detection  
**Mitigation:** Separate heartbeat check (30s timeout)

**Expected Outcome:** Stable client connections under peak load

**Status:** ✓ Completed  
**Date:** 2026-07-24 15:30  
**Task:** bugs/fix-headscale-timeout/  
**Related:** bug/fix-headscale-heartbeat (future)

---
```

### Step 4: Extract Pattern Tags

Identify searchable patterns:
```
Topic: "headscale"
Component: "client"
Problem Type: "timeout"
Constraint: "network-latency"
Solution Type: "increase-threshold"
Outcome: "stable-under-load"
Risk Level: "low"
```

Tags enable future search:
```
Search: "Have we solved timeout problems before?"
  → grep "Problem Type: timeout" decisions.md
  → Find: fix-headscale-timeout (5s→15s), fix-db-timeout (30s→60s)

Search: "How did we handle network latency?"
  → grep "Constraint: network-latency" decisions.md
  → Find: 3 related decisions + mitigations
```

### Step 5: Append to decisions.md

Format: One decision per entry, newest first

Structure:
```markdown
# Decision Log

## [2026-07-24 15:30] bug: fix-headscale-timeout
[full entry as above]

## [2026-07-24 12:00] feature: add-cloudflare-r2
[full entry]

## [2026-07-23 18:45] bug: fix-db-replication-lag
[full entry]
```

Metadata for each entry:
- Date + time (ISO format)
- Task type (bug/feature) + slug
- Extraction source (02_Planning / 03_Execution / 04_Verification / manual)

### Step 6: Link Related Decisions

Parse "Related:" field for cross-references:

```
Related: bug/fix-headscale-heartbeat (future)
         feature/add-headscale-monitoring (related)
         memory/snapshots/20260724-1530 (context)
```

Enable bidirectional links:
- If related decision already logged: add backlink
- If related decision not yet logged: create entry stub ("to be completed")

### Step 7: Enable Search + Retrieval

Decision log becomes queryable:

**Search by topic:**
```bash
grep "Topic: headscale" @memory/decisions.md
→ Show all headscale-related decisions
```

**Search by problem type:**
```bash
grep "Problem Type: timeout" @memory/decisions.md
→ Show all timeout-related solutions
```

**Search by status:**
```bash
grep "Status: completed" @memory/decisions.md
→ Show all implemented decisions
```

**Search by date range:**
```bash
grep "2026-07-2[34]" @memory/decisions.md
→ Show decisions from past 2 days
```

Patterns emerge:
- "We solve network timeouts by increasing threshold + adding heartbeat"
- "Headscale components need 5-15s tolerance for discovery"
- "Config-based thresholds work better than hardcoded"

### Step 8: Status Tracking

Decision lifecycle:
```
Status: "proposed"      → suggested in 02_Planning
Status: "approved"      → approved by human (gate passed)
Status: "in-progress"   → 03_Execution started
Status: "✓ completed"   → 04_Verification done
Status: "blocked"       → can't execute (document blocker)
Status: "reverted"      → undone (document reason)
```

Use status to track:
- What's been tried (completed + reverted)
- What's current (in-progress)
- What's blocked (waiting on external)

### Step 9: Report to User

Report format:
```
[use:: decision-logger-project] Logged decision:
  Title: Increase Headscale timeout 5s → 15s
  Because: Peak load discovery timeout
  Status: ✓ Completed
  Tags: headscale, timeout, network-latency
  Searchable: grep "Topic: headscale" decisions.md
  
Related: fix-headscale-heartbeat (future)
```

---

## Pattern Recognition Examples

**Example 1: Timeout Pattern**

Log decision #1 (2026-07-20):
```
Title: Fix DB connection timeout
Because: Slow network latency to Postgres cluster
Did: Increase timeout 30s → 60s
Risk: Slower error detection
Mitigation: Add monitor alert at 45s
```

Log decision #2 (2026-07-24):
```
Title: Fix Headscale client timeout
Because: Slow network latency during discovery
Did: Increase timeout 5s → 15s
Risk: Slower server detection
Mitigation: Add heartbeat check
```

Query: `grep "Problem Type: timeout" decisions.md`
Result: 2 timeout problems, both solved by "increase + mitigate"
Pattern: "Timeout issues solved by threshold + parallel health check"

**Example 2: Config vs Hardcoded**

Log decision #3 (2026-07-10):
```
Title: Move port to config file
Because: Different ports per environment
Did: config.yaml + env-specific overrides
Why: Avoid code changes per environment
```

Log decision #4 (2026-07-24):
```
Title: Move timeout to config file
Because: Different timeouts per environment
Did: config.yaml + env-specific overrides
Why: Avoid code changes per environment
```

Query: `grep "Solution Type: config-based" decisions.md`
Result: Pattern detected: "Environment-specific values belong in config"
Future: "Should this feature be in config or code?" → "Config, we've done this pattern twice"

---

## Integration with Skill 1 (Memory Auto-Updater)

**Skill 5 → Skill 1 pipeline:**
1. Skill 5 logs decision to decisions.md
2. Skill 1 (after task completion) reads decisions.md
3. Skill 1 extracts topic + adds to topic file
4. Skill 1 checks line count, snapshots if needed
5. Skill 1 prunes old entries

Both work on same decisions.md:
- Skill 5 logs structured decisions
- Skill 1 organizes + maintains rolling memory

---

## Decision File Structure

```markdown
# Decision Log

Format: [YYYY-MM-DD HH:MM] [type]: [slug]

## [2026-07-24 15:30] bug: fix-headscale-timeout
**Title:** [one-line decision]
**Because:** [what triggered the decision]
**Did:** [what was chosen/implemented]
**Why:** [rationale, trade-offs]
**Risk:** [downsides, mitigations]
**Expected Outcome:** [what should improve]
**Status:** ✓ Completed | approved | in-progress | blocked | reverted
**Date:** 2026-07-24 15:30
**Task:** bugs/fix-headscale-timeout/
**Related:** [other tasks/decisions]
**Tags:** headscale, timeout, network-latency, solution-type:increase-threshold

---

[next decision entry]
```

---

## Searching Decisions (Examples)

**Query: "timeout"**
```bash
grep -i "timeout\|Because.*timeout\|Title.*timeout" @memory/decisions.md
```
Result: All timeout-related decisions (3 found)

**Query: "status:completed"**
```bash
grep "Status: ✓ Completed" @memory/decisions.md
```
Result: All implemented decisions (27 found)

**Query: "headscale AND timeout"**
```bash
grep -A10 "Topic: headscale" @memory/decisions.md | grep "timeout"
```
Result: Headscale timeout decisions (2 found)

**Query: "Why: config-based"**
```bash
grep "Why.*config\|Solution Type: config-based" @memory/decisions.md
```
Result: All config-based decisions, showing pattern

---

## Testing Checklist

- [ ] Decision extraction from 02_Planning works
- [ ] Decision extraction from 03_Execution works
- [ ] Structured format correct (all fields present)
- [ ] Tags extracted correctly (topic, problem-type, solution-type)
- [ ] Related links created bidirectional
- [ ] Status tracking accurate (proposed → approved → completed)
- [ ] decisions.md format parseable (grep works)
- [ ] Search queries return correct results
- [ ] Report format clear + actionable

---

## Example Execution

**Scenario:** Task completes, Skill 5 logs decision

**Trigger:** 04_Verification.md marked complete, planning section:
```
"Increase timeout 5s → 15s to handle peak load discovery"
"Clients were timing out. Solution: tune timeout threshold"
"Risk: slower dead-server detection. Mitigation: heartbeat check"
```

**[Skill 5 executes]:**
1. Extract title: "Increase Headscale timeout 5s → 15s"
2. Parse: Because (peak load), Did (threshold change), Why (network latency)
3. Extract tags: headscale, timeout, network-latency, solution-type:increase-threshold
4. Append to decisions.md with all fields + date
5. Create backlink to related heartbeat decision (if exists)
6. Report:

```
[use:: decision-logger-project] Logged decision:
  Title: Increase Headscale timeout 5s → 15s
  Because: Clients timeout during peak load discovery
  Did: Modified client-config threshold (tunable per env)
  Status: ✓ Completed
  Tags: headscale, timeout, network-latency
  Related: bug/fix-headscale-heartbeat (future)

Searchable:
  grep "Topic: headscale" @memory/decisions.md
  grep "Problem Type: timeout" @memory/decisions.md
  grep "Status: completed" @memory/decisions.md
```

---

## Notes

- Skill 5 logs decisions DURING workflow (steps 2-4)
- Skill 1 organizes decisions AFTER workflow (step 4 done, memory update)
- Patterns emerge after 5-10 logged decisions
- Decisions are immutable (no edits, add reverted note if changed)
- Works with all task types (bug/feature/researcher)
