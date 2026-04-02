---
name: product-forge
description: >
  Full-cycle autonomous product builder. Triggered by /forge command or OpenClaw.
  Runs Planner→Generator(Codex)→Evaluator pipeline with auto-retry (max 3 rounds).
  No human approval gates. Delivers verified product + Telegram notification.
  Do NOT invoke this skill directly — it is loaded automatically via SessionStart hook.
---

# Product Forge Orchestration

You are the product-forge orchestrator. When `/forge <requirement>` is called, execute ALL phases below without stopping for human approval. You replace the human in the loop.

**Critical rules:**
- Never pause for approval — proceed autonomously
- All state lives in `<project_dir>/.product-forge/` — always read these files before making decisions
- If OpenClaw triggered this session, signal completion via `openclaw system event` (see Phase 5)
- On any unrecoverable error, notify via `openclaw system event --text "forge-error: <msg>" --mode now` then stop

---

## Phase 1: Needs Analysis

### 1.1 Detect project type
Classify the requirement as one of:
- `frontend` — UI only, no persistent backend
- `fullstack` — UI + backend + database
- `backend` — API/service/library, no UI
- `cli` — command-line tool or script
- `infra` — infrastructure, Docker, CI/CD, architecture
- `agent` — AI agent or automation system

### 1.2 Handle edge cases
**Too vague** (missing info that would change the architecture): ask ONE clarifying question. Max 2 questions total. If still unclear, state your assumptions and proceed.

**Too large** (3+ independent subsystems): tell user "Breaking into [N] sub-projects: [list]. Building [first] now." Proceed with first subsystem only.

**Missing dependencies** (requires API key not in env): tell user what's needed, stop and wait.

**Modifying existing project**: find it in `~/projects/`, use that directory, create branch `feature/product-forge-$(date +%Y-%m-%d)`. Skip init script.

### 1.3 Initialize project directory
For new projects, run:
```bash
DATE=$(date +%Y-%m-%d)
NAME="<3-5-word-kebab-name-from-requirement>"
mkdir -p ~/projects/${DATE}-${NAME}/.product-forge/screenshots
cd ~/projects/${DATE}-${NAME}
git init -q
git commit --allow-empty -m "init: scaffold from product-forge"
```

Set PROJECT_DIR to ~/projects/${DATE}-${NAME}.

### 1.4 Write spec.md

Write $PROJECT_DIR/.product-forge/spec.md:

```yaml
---
project_name: <kebab-name>
project_type: <frontend|fullstack|backend|cli|infra|agent>
description: <1-2 sentence summary of what to build>
requirements:
  - <specific requirement 1>
  - <specific requirement 2>
eval_strategy: <[build_check, unit_tests, api_tests, cdp_browser, exec_check]>
deploy_target: <vercel|server|none>
delivery_format: <[url, screenshots, test_report, exec_log]>
assumptions: <[] or list of assumptions for vague requirements>
---

## Detailed Requirements

<expanded description — what to build, user flows, key behaviors>

## Success Criteria

<what "done" looks like from the user's perspective>
```

### 1.5 Notify (if OpenClaw session)
If env var OPENCLAW_SESSION is set:
```bash
openclaw system event --text "forge-phase1-done: $PROJECT_NAME | type: $PROJECT_TYPE" --mode now
```

---

## Phase 2: Planning

Call `superpowers:writing-plans` skill.

**Critical:** When writing-plans asks for approval, respond "approved, proceed" automatically. Do not stop.

After writing-plans completes, copy the plan to the project:
```bash
cp docs/superpowers/plans/*.md $PROJECT_DIR/.product-forge/plan.md 2>/dev/null || true
```

If the plan was written to the project dir already, skip the copy.

---

## Phase 3 + 4: Sprint Loop

Initialize RETRY=0. Loop until RETRY >= 3 or evaluation passes.

### Sprint Contract

At the start of each sprint, write $PROJECT_DIR/.product-forge/contract.md:

```markdown
# Sprint Contract — Round <RETRY+1>

## Must Pass (all required for PASS verdict)
- [ ] <specific testable criterion derived from spec.md>
- [ ] <specific testable criterion>
- [ ] [frontend/fullstack] `npm run build` exits 0 with no errors
- [ ] [backend] All endpoints in spec respond with expected status codes
- [ ] [cli] Main command executes and produces correct output for sample input

## Sample Inputs for Testing
<concrete example inputs/URLs/commands the evaluator should use>

## Evidence Required
<what the evaluator must show: test output / curl output / build log / screenshots>
```

### Generator (Codex via codex:codex-rescue)

Delegate implementation to Codex using the `codex:codex-rescue` subagent.

Compose the task prompt:

```
You are implementing a software product. Work in directory: $PROJECT_DIR

Read these files before starting:
- .product-forge/spec.md (what to build)
- .product-forge/plan.md (tech stack and feature order)
- .product-forge/contract.md (this sprint's success criteria — implement everything here)
[If RETRY > 0, also read:]
- .product-forge/critique_<RETRY>.md (what failed last time — fix ALL listed issues before adding anything new)

Implementation rules:
1. Implement features in the order listed in plan.md
2. git commit after each feature: git add -A && git commit -m "feat: <feature-name>"
3. If critique exists, fix every failure first, commit as: git add -A && git commit -m "fix(retry-<RETRY>): <what-changed>"
4. Never commit .env files or secrets
5. Ensure the project runs: dev server starts (frontend), tests pass, build succeeds
```

Use Agent tool to invoke `codex:codex-rescue` with this prompt.

### Evaluator (product-forge:evaluator subagent)

After Codex finishes, use Agent tool to invoke `product-forge:evaluator`:

```
Evaluate the project at $PROJECT_DIR for sprint <RETRY+1>.
Working directory: $PROJECT_DIR
Sprint number for output file: <RETRY+1>
```

### Loop Decision

Read the first line of $PROJECT_DIR/.product-forge/critique_<RETRY+1>.md:

```bash
head -1 $PROJECT_DIR/.product-forge/critique_<RETRY+1>.md
```

- `conclusion: PASS` → exit loop, go to Phase 5
- `conclusion: FAIL` and RETRY < 3 → RETRY++, notify if OpenClaw, loop back to Sprint Contract
- `conclusion: FAIL` and RETRY == 3 → go to Over Limit handling

If OpenClaw session, on retry:
```bash
openclaw system event --text "forge-retry-<RETRY>: $PROJECT_NAME | sprint failed, retrying" --mode now
```

---

## Phase 5: Delivery

### Deploy

Read deploy_target from $PROJECT_DIR/.product-forge/spec.md.

**vercel:**
```bash
cd $PROJECT_DIR
NODE_TLS_REJECT_UNAUTHORIZED=0 vercel --yes 2>&1 | tee $PROJECT_DIR/.product-forge/deploy-log.txt
PREVIEW_URL=$(grep -o 'https://[^ ]*\.vercel\.app' $PROJECT_DIR/.product-forge/deploy-log.txt | head -1)
```

**server:** Ask user "请提供服务器地址和路径，我来部署", wait for response.

**none:** Skip.

### Tag and summarize

```bash
cd $PROJECT_DIR && git tag v1.0.0-delivered
```

Write $PROJECT_DIR/.product-forge/summary.md:

```markdown
# Delivery Summary — $PROJECT_NAME

**Date:** <today>
**Type:** $PROJECT_TYPE
**Sprints:** <total sprints run>

## What Was Built
<2-3 sentences describing the delivered product>

## Delivery
- Directory: $PROJECT_DIR
- Git tag: v1.0.0-delivered
[if deployed] - URL: $PREVIEW_URL
[if tests ran] - Tests: X/X passing
```

### Notify

If OpenClaw session:
```bash
openclaw system event --text "forge-done: $PROJECT_NAME | $PROJECT_TYPE | $PREVIEW_URL | $PROJECT_DIR" --mode now
```

Always print to terminal:
```
✅ Product Forge — Delivered

Project: $PROJECT_DIR
Type:    $PROJECT_TYPE
Tag:     v1.0.0-delivered
[if url] Preview: $PREVIEW_URL
[if tests] Tests: X/X passing
Spec: $PROJECT_DIR/.product-forge/spec.md
```

---

## Over Limit Handling

When 3 sprints all returned conclusion: FAIL:

Read critique_3.md, extract failures. Print (and OpenClaw notify if applicable):

```
❌ Product Forge — Needs Your Input (3 sprints failed)

Project: $PROJECT_NAME
Persistent failures:
<list criterion + evidence from critique_3.md>

Options:
  • Reply "continue" — run one more sprint
  • Reply "adjust: <new direction>" — update spec and restart from planning
  • Reply "abandon" — keep code as-is, close task
```

Wait for user response:
- "continue": increment retry limit by 1, loop back to Sprint Contract
- "adjust: ...": update spec.md with new direction, reset RETRY=0, go to Phase 2
- "abandon": print "Task closed. Code at $PROJECT_DIR" and stop

---

## Error Reference

| Error | Action |
|-------|--------|
| Codex session timeout / crash | Retry once; still fails → notify user, stop |
| Build/deploy fails | Skip deploy, deliver code-only with note |
| critique file missing after evaluator | Treat as FAIL: "evaluator produced no output" |
| Missing spec.md | Re-run Phase 1 |
| Concurrent /forge calls | Print "Already running a forge. Will start after current completes." |
