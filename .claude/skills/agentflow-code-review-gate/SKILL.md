---
name: agentflow-code-review-gate
description: Quality-gate Stage-2 code review. Reviews a diff across three lenses (correctness bugs, CLAUDE.md adherence, missed simplifications) in one pass and emits a machine-readable REVIEW-VERDICT line. Not for interactive use — invoked by quality_gate.run_stage2_review via claude -p. Does NOT invoke the /code-review plugin.
---

# Code-Review Gate (Stage 2)

**Gate:** Must pass BEFORE a ticket advances past code review.

This skill is the quality gate's owned, diff-shaped code reviewer. It does NOT
invoke the `/code-review` plugin (that is PR-centric, comments on GitHub, and
fans out 5+ subagents — wrong shape and side-effects for a gate).

## Procedure

1. You receive a unified diff on **stdin**.
2. Review the diff yourself in a single structured pass across **three lenses**,
   considering only lines the diff modifies:
   - **Correctness** — real bugs: null/None derefs, off-by-one, wrong
     conditionals, unhandled errors, resource leaks, race conditions.
   - **CLAUDE.md adherence** — violations of the repo's stated rules (atomic
     writes, no bare except, status-check before .json(), etc.).
   - **Missed simplifications** — only those a senior engineer would block on,
     not stylistic nits.
3. A finding is **blocking** only if it is a real defect on a modified line.
   Ignore: pre-existing issues, nitpicks, lint/type/format, untouched lines,
   general coverage/docs concerns not mandated by CLAUDE.md, likely false
   positives. Count surviving blocking findings → `N`.
4. Briefly narrate each blocking finding (file:line + one sentence) BEFORE the
   verdict line, so the gate's captured detail is actionable.

## Output Contract (machine-readable verdict)

`quality_gate.run_stage2_review` reads the **verdict from stdout, not the exit
code**. Emit, as the **final line** and **exactly once**, one of:

```
REVIEW-VERDICT: PASS
REVIEW-VERDICT: FAIL: <machine_reason>
```

- `PASS` only when `N == 0`.
- On failure, `<machine_reason>` is a single snake_case token — prefer the
  blocking count, e.g. `3_blocking`.
- Emit the line literally, once, as the **last non-empty line** of your output.
  The gate parses only that final line — do NOT place any text after it, and do
  NOT repeat the literal string `REVIEW-VERDICT:` anywhere earlier. Omitting the
  line → gate fails closed `code_review_no_verdict`.
- Your exit code is ignored for the verdict; a non-zero exit is read as
  `code_review_tooling_failed`, never a veto.
