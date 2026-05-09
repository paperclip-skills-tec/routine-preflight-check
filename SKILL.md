---
name: routine-preflight-check
description: Use when activating, modifying, or re-enabling a Paperclip routine — before enabling a trigger, after changing routine code, when resolving a "never fired" execution issue, or before marking a blocked routine fix as done.
---

# Routine Preflight Check

## Overview

Before relying on a Paperclip routine, run a structured preflight check across six areas. Skipping this causes cycles of "never fired → blocked → fix committed → blocked again" — a pattern that burned 6+ issues on the Release Readiness Audit routine ([TEC-1326](/TEC/issues/TEC-1326) through [TEC-1603](/TEC/issues/TEC-1603)).

**Emit a preflight report as an issue comment. Block activation if any critical check fails.**

---

## Checklist

### Check 1 — Trigger Validation

Fetch the routine:
```
GET /api/companies/{companyId}/routines/{routineId}
```

Verify:
- At least one trigger has `status: active`
- For `schedule` triggers: parse the `cronExpression` and confirm next fire time is in the future
- For `webhook` triggers: confirm the URL is reachable
- For `api` triggers: confirm the calling system is configured

**Critical block:** No active trigger → routine will never fire regardless of script correctness.

---

### Check 2 — Environment Dependencies

Read the routine's script or agent instructions. List every CLI tool or binary invoked (e.g. `gh`, `node`, `jq`, `curl`, `python`). For each, verify availability in the execution environment:

```bash
which <tool>
# Or in the routine's execution context:
command -v <tool> && echo "found" || echo "MISSING"
```

If the routine runs on a remote host or container, run verification there — not just on the development machine.

**Critical block:** Any required binary absent → routine silently fails or emits cryptic "command not found" errors.

---

### Check 3 — Credential and Config Dependencies

Identify every environment variable, API token, or secret the routine reads. Confirm:
- The variable exists and is non-empty in the execution environment
- The credential has not expired
- The value reflects the current deployment — not a value only present on an unmerged branch

```bash
printenv | grep -E "(API_KEY|TOKEN|SECRET|_URL)" | sed 's/=.*/=***/'
```

**Critical block:** Missing or unset credential → auth failures, not "missing env var" errors. Often misdiagnosed as code bugs.

---

### Check 4 — Prerequisite Data

Some routines depend on external data being present before they produce useful output:
- Labels (e.g. `release:*` must exist on relevant issues)
- Issue queues (e.g. `in_review` queue must have candidates)
- External configs, seed data, or integration state the routine queries

Verify this data exists before activating the schedule.

**Non-critical block:** Routine fires and exits cleanly but produces zero output, which may be mistaken for a trigger or code failure.

---

### Check 5 — Merge Verification

If the routine script or agent instructions were changed on a feature branch, verify the branch is merged to the deployment target **before** marking the fix as done:

```bash
git log origin/main..HEAD -- <routine-script-path>
# Should be empty — no commits ahead of main
```

Or check via Paperclip issue status: the PR/commit must be deployed, not just committed.

**Critical block:** A committed but unmerged fix is not a deployed fix. This is the #1 source of repeat blocks — the agent sees "fix done" and re-enables the trigger, but the routine runs the old code.

---

### Check 6 — Dry Run

Where feasible, trigger the routine once manually before relying on the schedule:

```
POST /api/companies/{companyId}/routines/{routineId}/trigger
```

Or invoke the routine agent directly on a real (or test) issue. Confirm:
- Exit status is non-error
- Output is plausible (not empty, not error messages)
- No unexpected side effects

**Non-critical block if skipped:** Document the reason in the report. Skipping increases risk of schedule-triggered failures generating watchdog alerts.

---

## Preflight Report Template

Post this comment on the routine's activation or fix issue after completing all checks:

```markdown
## Routine Preflight Report — {routine name}

| Check | Result | Notes |
|---|---|---|
| 1. Trigger validation | ✅ PASS / ❌ FAIL | e.g. "cron `0 9 * * 1` — next fire 2026-05-12 09:00 UTC" |
| 2. Environment dependencies | ✅ PASS / ❌ FAIL | e.g. "`gh`, `node`, `jq` all found at expected paths" |
| 3. Credential / config | ✅ PASS / ❌ FAIL | e.g. "`GITHUB_TOKEN` present and non-empty" |
| 4. Prerequisite data | ✅ PASS / ⚠️ SKIP / ❌ FAIL | e.g. "3 issues with `release:*` label found" |
| 5. Merge verification | ✅ PASS / ❌ FAIL | e.g. "branch merged to main 2026-05-07" |
| 6. Dry run | ✅ PASS / ⚠️ SKIP / ❌ FAIL | e.g. "triggered manually, processed 2 issues, no errors" |

**Overall: ✅ READY TO ACTIVATE / ❌ BLOCKED**

[If blocked: state which check(s) failed, the fix required, and the owner.]
```

---

## Blocking Rules

**Do NOT activate (or re-enable) a routine trigger if any critical check fails:**

| Check | Critical? | Blocks activation? |
|---|---|---|
| Trigger validation | Yes | Yes — no trigger = no execution |
| Environment dependencies | Yes | Yes — missing binary = silent failure |
| Credential / config | Yes | Yes — missing cred = auth failure |
| Prerequisite data | No | Warn but may proceed; document assumption |
| Merge verification | Yes | Yes — unmerged fix = stale code runs |
| Dry run | No | Warn if skipped; document reason |

If any critical check fails: patch the issue to `blocked`, name the fix owner and action in the comment, and do not re-enable the trigger until re-running the full preflight.

---

## Common Mistakes

| Mistake | Reality |
|---|---|
| "The fix is committed, so I'll re-enable the trigger" | Committed ≠ merged ≠ deployed. Run Check 5. |
| "It worked on my machine" | The execution environment may differ. Run Check 2 on the host where the routine runs. |
| "The trigger looks right in the UI" | A trigger can be `active` but have an invalid cron expression or a past end date. Verify next fire time. |
| "The routine ran before, so credentials are fine" | Tokens expire. Check non-empty and not expired, not just present. |
| "Zero output means the routine is broken" | Often means Check 4 data isn't there yet. Verify prerequisite data before debugging code. |

---

*TEC Custom Skill — maintained by the Deltek Technical Services Engineering team.*
