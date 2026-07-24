---
name: multi-llm-fallback
description: Stuck? Ask secondary LLM (Gemini, GPT, etc.) for second opinion.
type: skill
trigger:
  - user says "stuck" or "help" or "second opinion"
  - 3+ failed attempts on same error/problem
  - explicit request ("/multiLLM", "/fallback")
  - timeout or resource limit hit (graceful fallback)
---

# Skill 3: Multi-LLM Fallback Orchestrator

**Purpose:** When stuck or blocked, consult different LLM for alternative approach + recommendations.

**Inputs:**
- Current problem (description or error)
- Attempts made (code, commands, results)
- Errors encountered (stack traces, messages)
- Context (relevant files, constraints)

**Outputs:**
- Secondary LLM recommendation (alternative approach)
- Comparison: primary vs secondary approach (pros/cons)
- Next step recommendation (try path A or path B)
- Decision logged to @memory/decisions.md (capture learning)

---

## Implementation Steps

### Step 1: Detect Stuck Situation

**Explicit triggers:**
- User: "stuck" or "help" or "second opinion"
- User: "/multiLLM" or "/fallback" (command)
- User: "different approach" (implicit request)

**Implicit triggers:**
- Same error repeated 3+ times
- Failed attempts with increasing token cost (2x normal)
- Timeout on complex operation
- Resource limit hit (code execution sandbox)

Example detection:
```
Attempt 1: "Try approach A" → Error: timeout
Attempt 2: "Try approach B" → Error: same timeout
Attempt 3: "Try approach C" → Error: same timeout
→ Detect: stuck, use fallback
```

### Step 2: Collect Context

**Gather information:**
1. Problem statement (user's description)
2. Attempts made (what was tried)
3. Errors encountered (exact messages)
4. Constraints (performance, compatibility, etc.)
5. Relevant code/config (if small enough to share)
6. Desired outcome (what should happen)

**Example context:**
```
Problem: PostgreSQL replication lag on replica
Attempts:
  1. Increased max_wal_senders from 3 → 10
  2. Reduced checkpoint_timeout from 300 → 60s
  3. Tuned synchronous_commit = off
Errors:
  - Lag still 5+ minutes during peak load
  - Heartbeat shows replication working
  - Network latency stable (< 10ms)
Constraint: Can't increase sync_commit (data loss risk)
Desired: <1 minute replication lag
```

### Step 3: Call Secondary LLM

**Available fallback LLMs:**
```
Primary: Claude Opus (used in main agent)
Fallback options:
  1. Google Gemini 2.0 Flash (fast, good for brainstorm)
  2. GPT-4o Mini (OpenAI, different perspective)
  3. Grok (X, contrarian approach)
  4. Local Llama 2 (if available, privacy-preserving)
```

**Fallback order:**
1. Check config: LLM_FALLBACK_PRIMARY = env var
2. If not set: use Gemini 2.0 Flash (free, fast)
3. If Gemini fails: try GPT-4o Mini
4. If both fail: ask local LLM (if available)

**Query secondary LLM:**
```
Prompt to secondary:
"Problem: [problem]
Attempts made: [list]
Errors: [messages]
Constraints: [limitations]

What's your approach? Different from the primary LLM?
Pros/cons of your approach?
What should we try next?"
```

**Secondary LLM response:**
- Alternative approach (different strategy)
- Rationale (why this might work)
- Pros (advantages over primary approach)
- Cons (drawbacks or risks)
- Next step (concrete action)

### Step 4: Compare Approaches

**Structure comparison:**

```
┌─────────────────────────────────────────────────┐
│ PRIMARY APPROACH (Claude)                       │
├─────────────────────────────────────────────────┤
│ Strategy: Increase buffer + reduce timing      │
│ Rationale: More headroom for load spikes      │
│ Pros: Simple, safe, incremental               │
│ Cons: May not address root cause              │
│ Token cost: Medium                             │
│ Time to implement: Quick (config change)      │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ SECONDARY APPROACH (Gemini)                    │
├─────────────────────────────────────────────────┤
│ Strategy: Switch to async replication mode    │
│ Rationale: Eliminate sync bottleneck          │
│ Pros: Lower latency, higher throughput        │
│ Cons: Data loss risk if primary crashes      │
│ Token cost: Low                                │
│ Time to implement: Medium (config + test)    │
└─────────────────────────────────────────────────┘

RECOMMENDATION:
  Primary: Try if data loss acceptable
  Secondary: Better if durability critical
  Hybrid: Use async during off-peak, sync at peak
```

### Step 5: Recommend Path Forward

**Decision matrix:**
```
If priority = performance:
  → Try secondary approach (async replication)
  → Monitor data consistency during failover test

If priority = durability:
  → Try primary approach (tune buffers)
  → Verify lag under load before prod

If unsure:
  → Try secondary in staging first
  → Monitor for 24h
  → Switch to primary if issues
```

### Step 6: Log Decision

File: `@memory/decisions.md`

Entry format (multi-LLM decision):
```markdown
## [2026-07-24 17:00] bug: postgresql-replication-lag

**Title:** Resolve PostgreSQL replication lag 5+ minutes

**Because:** Replica lagging during peak load, risk of data loss

**Attempts (stuck):**
  1. Increased max_wal_senders 3 → 10 → lag unchanged
  2. Reduced checkpoint_timeout 300 → 60s → lag unchanged
  3. Tuned synchronous_commit off → data loss risk

**Primary Approach (Claude):** Increase buffer + timing
**Secondary Approach (Gemini):** Async replication + monitoring

**Did:** Combined hybrid approach: async off-peak, sync at peak

**Why:** Balance performance (async) with safety (sync during peak)

**Risk:** Complexity, mode switching could cause issues  
**Mitigation:** Automated mode switching + 24h staging test

**Expected Outcome:** <1 minute lag, zero data loss

**Status:** ✓ Completed  
**Date:** 2026-07-24 17:00  
**Task:** bugs/postgresql-replication-lag/  
**Related:** storage/postgresql (component), memory/snapshots/20260724 (past issues)  
**Tags:** database, replication, performance, multi-llm-fallback, hybrid-approach  
**Fallback LLMs:** Gemini 2.0 Flash (recommended approach)

---
```

### Step 7: Report Results

Report to user:
```
[use:: multi-llm-fallback-project] Second opinion from Gemini:

Primary (Claude): Increase buffer + tune timing
  Pros: Safe, incremental, simple
  Cons: May not fix root cause

Secondary (Gemini): Async replication mode
  Pros: Lower latency, better throughput
  Cons: Data loss risk on failure

HYBRID RECOMMENDATION:
  Use async during off-peak (better performance)
  Switch to sync during peak (protect critical data)
  Automate with cron: 22:00 → async, 06:00 → sync

Next: Test in staging environment
Cost: 0.3 (primary + secondary LLM query)
Saved: Avoided 5+ more failed attempts
```

---

## Cost Analysis

**Scenario: Stuck after 3 failed attempts**

Without fallback:
```
Attempt 1: 0.5 tokens → fail
Attempt 2: 0.5 tokens → fail
Attempt 3: 0.5 tokens → fail
Attempt 4-7: 2.0 tokens (trying random approaches)
Total: 3.5 tokens + 30 minutes wasted
```

With fallback (Skill 3):
```
Attempt 1: 0.5 tokens → fail
Attempt 2: 0.5 tokens → fail
Attempt 3: 0.5 tokens → fail
Call Gemini: 0.3 tokens (faster model)
Compare + recommend: 0.2 tokens (Claude)
Total: 2.0 tokens + 5 minutes to recommendation
Savings: 1.5 tokens (43% cheaper) + 25 min saved
```

---

## Fallback LLM Orchestration

**LLM Selection Strategy:**

| Situation | Primary | Secondary | Why |
|-----------|---------|-----------|-----|
| Performance tuning | Claude (expert) | Gemini (diverse) | Fresh perspective |
| Debugging | Claude (thorough) | GPT-4o (pattern) | Different error model |
| System design | Claude (holistic) | Grok (contrarian) | Challenge assumptions |
| Urgent (timeout) | Claude | Gemini Flash | Fastest response |
| Hardware/firmware | Claude | Local Llama | Privacy, specific knowledge |

**Fallback execution:**
```
1. Try primary (Claude) → success → return
2. Try primary → fail + retry once
3. Still fail → trigger Skill 3
4. Call secondary (Gemini)
5. Compare + recommend
6. If secondary works → use it
7. If both fail → escalate to user
8. Log attempt + recommendation to decisions.md
```

---

## Recovery + Learning

**Capture learning:**
1. Problem description + attempts
2. Primary approach (why it failed)
3. Secondary approach (why it worked or failed)
4. Chosen solution + rationale
5. Outcome (success/partial/fail)

**Patterns from multi-LLM decisions:**
- "Claude tends to be thorough but slow. Gemini is fast but risky."
- "GPT excels at pattern matching. Good for debugging."
- "Grok challenges assumptions. Use for design review."

Query history:
```bash
grep "multi-llm-fallback" @memory/decisions.md
→ Show all fallback decisions + outcomes
→ Identify which LLM combo works best for problem type
```

---

## Integration with Memory + Library

**Works with Skill 2 (Web-to-MD Library):**
- Problem → Skill 3 gets second opinion → Skill 2 caches recommendation
- Future: Similar problem → use cached approach directly
- Example: "PostgreSQL replication lag? See library/postgres-fallback-hybrid.md"

**Works with Skill 1 (Memory Auto-Updater):**
- Fallback decision logged to decisions.md
- Skill 1 snapshots decision + context
- Pattern: after 3+ fallback decisions on same topic → document pattern

---

## Testing Checklist

- [ ] Stuck detection works (3+ fails, explicit trigger)
- [ ] Context collection works (problem + attempts captured)
- [ ] Secondary LLM call works (query sent correctly)
- [ ] Response parsing works (approach + pros/cons extracted)
- [ ] Comparison works (primary vs secondary formatted)
- [ ] Recommendation works (clear next step)
- [ ] Logging works (decision saved to decisions.md)
- [ ] Cost tracking works (reports savings)
- [ ] Fallback chain works (if Gemini fails → try GPT)

---

## Example Execution

**Scenario: Stuck after 3 deployment attempts**

**Trigger:** User says "stuck on deployment"

**[Skill 3 detects stuck]:**
1. Collect: Problem (container won't start), Attempts (3 failed)
2. Extract errors: "OOMKilled", "resource limits", "ulimit"
3. Query Gemini: "Why container OOMKilled? Different approach?"
4. Gemini response: "Switch memory swap, not just limit increase"
5. Compare:
   - Claude: Increase memory limit (simple)
   - Gemini: Enable swap + memory management (flexible)
6. Recommend: Try Gemini's swap approach (lower cost, higher ceiling)
7. Log: Fallback decision → decisions.md
8. Report:

```
[use:: multi-llm-fallback-project] Stuck on deployment, second opinion:

Problem: Container OOMKilled on startup
Attempts: 3 failed (increased memory each time)

Primary (Claude): Increase container memory limit to 4GB
Secondary (Gemini): Enable swap + tune swappiness parameter

RECOMMENDATION: Try Gemini's approach (memory swap)
Why: More flexible, handles burst loads, no hard cap needed
Cost: Both approaches cheap to test
Next: Deploy with memory swap enabled

Fallback cost: 0.3 tokens (vs 2+ tokens for 5+ more attempts)
Saved: $0.15 + 30 minutes debug time
```

---

## Notes

- Skill 3 is "circuit breaker" for stuck situations
- Logs decisions for pattern learning
- Works with all LLMs (not just OpenAI/Google)
- Integration: Skill 1 + 2 capture full context for recovery
- Best used when: stuck >3 attempts OR time critical (production incident)
