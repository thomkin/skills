---
name: skill-loader
description: Auto-load skills + fallback to file. Self-report which loaded.
type: skill
trigger:
  - task mentions skill keyword (Playwright, Yocto, compression, etc.)
  - manual skill request (/caveman, /superpower, etc.)
  - response generation (check if skill should apply)
  - skill not found in runtime
---

# Skill 6: Skill Loader (Auto)

**Purpose:** Load skills on-demand. Fallback to file if not in runtime. Self-report source.

**Inputs:**
- Skill name (explicit or inferred from task)
- Fallback mode (re-read from file if needed)
- Optional: skill version/tag

**Outputs:**
- Skill loaded (from runtime or file)
- Rules applied to current task/response
- Report marker: `[use:: skillname-loaded]` or `[use:: skillname-project]` or `[use:: skillname-global]`
- What skill did: "Applied [skillname]: [specific actions taken]"

---

## Implementation Steps

### Step 1: Detect Skill Needed

**Explicit request:**
```
User: "/caveman"
→ skill_needed = "caveman"

User: "new bug: ..."
→ skill_needed = "workflow-runner" (detected from "new")
```

**Keyword detection:**
- "Playwright", "E2E", "test" → playwright-electron
- "Yocto", "kas", "bitbake" → yocto-xilinx
- "compression", "brief", "terse", "short" → caveman
- "stuck", "help", "second opinion" → multi-llm-fallback
- "memory", "decision", "save" → memory-auto-updater or decision-logger
- "PDF", "fetch", "web link", "cache" → web-to-md-library
- "new bug", "new feature" → workflow-runner

### Step 2: Check Runtime

```python
if skill_name in runtime.loaded_skills:
  skill_loaded = True
  source = "runtime"
else:
  skill_loaded = False
  source = None
```

### Step 3a: If Loaded in Runtime

```
1. Apply skill rules to current task/response
2. Execute skill logic
3. Report: [use:: skillname-loaded]
4. Report what skill did: "Applied [skillname]: [specific actions]"

Example:
  [use:: caveman-loaded] Applied caveman compression: removed 40% filler, kept all substance
```

### Step 3b: If Not Loaded (Fallback)

**Priority 1: Check project-specific skills**
```
path = ./skills/[skillname].md
if exists:
  read file
  parse frontmatter
  extract rules
  source = "project"
  goto Step 4
```

**Priority 2: Check global skills**
```
path = $LLM_SKILLS_DIR/[skillname].md
if exists:
  read file
  parse frontmatter
  extract rules
  source = "global"
  goto Step 4
```

**If not found:**
```
Report: "❌ Skill '[skillname]' not found"
Check: ./skills/README.md or ~/.llm-skills/
```

### Step 4: Apply Skill Rules

Parse frontmatter (YAML):
```yaml
---
name: caveman
description: Ultra-compressed communication
type: skill
trigger:
  - user requests terse output
  - compression needed
rules:
  - drop articles (a, an, the)
  - drop filler (just, really, basically)
  - fragments OK
  - keep substance (code, data, reasoning)
---
```

Extract `rules` section from file. Apply to current response:
- Remove articles
- Remove filler
- Keep technical accuracy
- Format: fragments + bullet points

### Step 5: Report Source + Action

**Report marker at start of response:**

```
[use:: skillname-source] Applied [skillname]: [what happened]
```

Examples:
```
[use:: caveman-loaded] Compressed output: removed 35% filler
[use:: caveman-project] Compressed output: dropped articles + filler
[use:: caveman-global] Compressed output: fragments OK, substance kept
[use:: workflow-runner-global] Created task: bugs/fix-headscale-timeout/
```

Format: `[use:: skillname-location]` where location is:
- `loaded` (already in runtime)
- `project` (loaded from ./skills/)
- `global` (loaded from ~/.llm-skills/)

---

## Skill Detection Keywords

| Keyword | Skill | Priority |
|---------|-------|----------|
| "Playwright", "E2E", "test" | playwright-electron | 1 |
| "Yocto", "kas", "bitbake" | yocto-xilinx | 1 |
| "terse", "brief", "short", "compress" | caveman | 1 |
| "multi-step", "planning", "workflow" | superpower | 2 |
| "stuck", "help", "different approach" | multi-llm-fallback | 2 |
| "new bug", "new feature" | workflow-runner | 1 |
| "memory", "decision", "save learning" | memory-auto-updater or decision-logger | 2 |
| "fetch", "PDF", "web link" | web-to-md-library | 2 |

Priority 1 = high confidence, auto-load  
Priority 2 = medium confidence, load if mentioned

---

## Example 1: Caveman Skill (Already Loaded)

**User:** "explain database connection pooling"

**[Skill 6 detects]:**
- No explicit request for skill
- But caveman skill is loaded (default)
- Should caveman apply? YES (response generation = default)

**[Apply caveman]:**
```
[use:: caveman-loaded] Pool reuse open DB connections. No new connection per request. Skip handshake overhead.
```

---

## Example 2: Workflow Runner (Not Loaded, Project Has It)

**User:** "new bug: login timeout"

**[Skill 6 detects]:**
- Keyword "new bug" detected
- skill_needed = "workflow-runner"
- Check runtime: NOT LOADED
- Check ./skills/workflow-runner.md: NOT FOUND (devops doesn't have it)
- Check ~/.llm-skills/workflow-runner.md: FOUND

**[Apply from global]:**
```
[use:: workflow-runner-global] Created task: bugs/login-timeout/
  01_Analysis.md
  02_Planning.md
  03_Execution.md
  04_Verification.md
  
Next: Fill 01_Analysis.md
```

---

## Example 3: Custom Project Skill

**User:** "write Ansible playbook for MongoDB"

**[Skill 6 detects]:**
- Keyword "Ansible" detected
- skill_needed = "ansible"
- Check runtime: NOT LOADED
- Check ./skills/ansible.md: FOUND

**[Apply from project]:**
```
[use:: ansible-project] Loaded Ansible playbook skill
  - Playbook structure (roles, tasks, handlers)
  - Best practices for MongoDB setup
  - Example: replication, backup, monitoring
```

---

## Testing Checklist

- [ ] Skill detection works (keywords recognized)
- [ ] Runtime check works (knows if skill loaded)
- [ ] File lookup works (./skills/ first, then ~/.llm-skills/)
- [ ] Frontmatter parsing works (extracts name, description, rules)
- [ ] Rules application works (e.g., caveman removes articles)
- [ ] Self-report works (`[use:: skillname-location]` printed first)
- [ ] Fallback works (if ./skills/ missing, uses ~/.llm-skills/)
- [ ] Error handling works (clear message if skill not found)

---

## Integration with Phase 1

- Works with Skill 4 (Workflow Runner): auto-loads when "new bug/feature" detected
- Enables all other skills: loads from global when project skills missing
- Auto-triggers: no user action needed (fully automatic)
- Self-reports: user knows which skill loaded + from where

## Notes

- Skill 6 itself is a skill (meta!)
- Enable Skill 6 first to bootstrap other skills
- Always check ./skills/ first (project can override globals)
- If skill file missing: clear error "Skill '[name]' not found in ./skills/ or ~/.llm-skills/"
- Self-report marker must appear FIRST in response (before any other output)

