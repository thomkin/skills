---
name: workflow-runner
description: Create task folders + enforce 4-step workflow (Analysis → Planning → Execution → Verification).
type: skill
trigger:
  - user creates new task (bug or feature)
  - user says "new bug [name]" or "new feature [name]"
  - any development work started
---

# Skill 4: 4-Step Workflow Runner

**Purpose:** Auto-create task structure + enforce workflow

**Inputs:**
- Task type (bug or feature)
- Task name/slug
- Initial description

**Outputs:**
- Folder created: bugs/[slug]/ or features/[slug]/
- 4 files created + templates injected
- Task state tracked (current step = 1)
- Report: "Created [task]. Start: 01_Analysis.md"

---

## Implementation Steps

### 1. Parse Input

Extract from user message:
- type: "bug" or "feature"
- slug: kebab-case (3-50 chars)
- description: initial task description

**Validation:**
- slug: no spaces, no underscores, kebab-case only
- type: only "bug" or "feature"
- duplicate check: folder doesn't already exist

### 2. Create Folder Structure

```bash
if type == "bug":
  mkdir -p bugs/[slug]/
elif type == "feature":
  mkdir -p features/[slug]/

touch [path]/01_Analysis.md
touch [path]/02_Planning.md
touch [path]/03_Execution.md
touch [path]/04_Verification.md
```

### 3. Inject Templates

Fill each file with template sections:

**01_Analysis.md:**
- Task (restate in 1 line)
- Dependencies (files, services, APIs)
- Impacted Files (what's affected + why)
- Plausibility Check (for bugs: does it make sense?)
- Open Questions (to resolve in planning)

**02_Planning.md:**
- Problem Statement (clear terms)
- Solution Approach (high-level plan)
- Logic Map (step-by-step or pseudocode)
- Schema/Type Changes (migrations, types)
- File Changes (which files, in what order)
- Error Handling (how to handle errors)
- Testing Strategy (how to verify)
- Risks/Gotchas (risks + mitigations)
- [USER APPROVAL GATE] (requires approval before execution)

**03_Execution.md:**
- File changes (per file, show diffs or full replacements)
- Shell Commands (commands to run)
- Migrations (if applicable)
- Build Verification (build commands + expected output)

**04_Verification.md:**
- Architecture Cross-Check (matches ARCHITECTURE.md patterns?)
- Build Status (builds OK? Tests pass? No errors?)
- Integration Test (manual flow verification)
- Result checklist (bug fixed? No new issues? Ready to commit?)

### 4. Track Task State

Store current state:
```
task_slug: fix-headscale-client-timeout
task_type: bug
current_step: 1
created: 2026-07-24T12:30:00Z
```

Can store in hidden file `.workflow-state` (git-ignored).

### 5. Report to User

```
✓ Created task: bugs/fix-headscale-client-timeout/

Files:
  ✓ 01_Analysis.md
  ✓ 02_Planning.md
  ✓ 03_Execution.md
  ✓ 04_Verification.md

Next step: Fill 01_Analysis.md

What to cover:
  1. What dependencies needed?
  2. Which files impacted?
  3. Does bug/feature make sense?
  4. Open questions to resolve in planning?

Ready? Edit: bugs/fix-headscale-client-timeout/01_Analysis.md
```

---

## Workflow Enforcement

### After Step 1 (Analysis Complete)

```
✓ 01_Analysis.md complete

Ready for 02_Planning.md? (yes/no)
```

User completes Analysis, you guide to Planning.

### After Step 2 (Planning Complete)

```
✓ 02_Planning.md complete

[USER APPROVAL GATE]

Plan looks good?
  approved → proceed to 03_Execution.md
  needs-rework → update 02_Planning.md + resubmit
```

No execution without approval. Prevents wrong direction.

### After Step 3 (Execution Complete)

```
✓ 03_Execution.md complete

Proceeding to 04_Verification.md

Verify:
  - Solution matches ARCHITECTURE.md?
  - Builds without errors?
  - Tests pass?
  - Integration test works?
```

### After Step 4 (Verification Complete)

```
✓ All 4 steps complete

Ready to commit?
  yes → proceed to git commit (with user approval)
  no → which step needs work?
```

---

## Example Execution

**User:** "new bug: headscale clients timeout"

**[Skill 4 executes]:**
1. Parse: type=bug, slug=headscale-clients-timeout
2. Create: bugs/headscale-clients-timeout/
3. Create 4 files + inject templates
4. Track: step=1
5. Report:

```
✓ Created task: bugs/headscale-clients-timeout/

Files:
  ✓ 01_Analysis.md
  ✓ 02_Planning.md
  ✓ 03_Execution.md
  ✓ 04_Verification.md

Next: Analyze the issue
  - What's timing out? (login? node discovery?)
  - Which dependencies? (Headscale server, clients, network)
  - Which files? (components/headscale/CONTEXT.md, deployment/ansible/)
  - Plausible? (Yes: timeout often caused by slow response, network issues)
  - Open questions? (Is timeout consistent? All clients or specific OS?)

Go: bugs/headscale-clients-timeout/01_Analysis.md
```

---

## Testing Checklist

- [ ] Folder creation works (bugs/ or features/)
- [ ] 4 files created with templates
- [ ] Slug validation works (rejects invalid)
- [ ] Duplicate detection works (can't create same task twice)
- [ ] Step tracking works (shows current_step=1 after creation)
- [ ] Approval gate enforced (can't proceed from step 2 without approval)
- [ ] Templates inject task name correctly
- [ ] Report is clear + actionable

---

## Notes

- Skill 6 (Skill Loader) loads this skill automatically when "new bug" or "new feature" detected
- Works with bugs/, features/, researcher/, components/*/tasks/ folders
- Paired with Skill 1 (Memory Auto-Updater): logs task creation to @memory/decisions.md
- Templates must match CLAUDE.md (keep in sync)
