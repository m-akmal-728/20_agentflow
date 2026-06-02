---
name: mission
description: "Epic-tier orchestrator above /goal. A mission is a per-project, stamp-keyed north-star that turns an ordered ladder of goal-stubs into goals (active.json). v1 is SHADOW-ONLY: /mission next is operator-initiated; the goal-complete chain only proposes, never auto-advances. Subcommands: status, show, new, next, gate, unblock, clear."
argument-hint: "[status | show | new | next | gate | unblock | clear]"
---

# /mission — Campaign Orchestrator (v1 shadow-only)

A mission sits ABOVE `/goal`: it names the north-star and orchestrates an ordered
sequence of goals to an outcome. `/mission next` is the only thing that *creates*
goals (writes `active.json`) — the job `/goal` deliberately refuses.

**v1 is shadow-only (spec D4):** `/mission next` is operator-initiated; the
goal-complete→advance chain only records a would-advance judgment and suggests the
command — it never auto-writes `active.json`. Hands-off advance is v1.1, hard-gated
behind a calibration evaluator that does not yet exist.

## State files

```
~/.local/state/goal-stop/missions/<project>.json          # the mission (the score)
~/.local/state/goal-stop/missions/runs/<key>.json         # durable run records
~/.local/state/goal-stop/missions/KILLFILE                # presence = stop mutating ops
~/.local/state/goal-stop/missions/<project>.json.bak-<ts> # /mission clear archive
```

`<key>` = `<project>__<goal_id>__<attempt>` (collision-safe, spec §7).

## Resolve which mission

Every invocation starts by sourcing the shared resolver — it sets `$MISSION_PROJECT`
and `$MISSION_FILE` by PROJECT (never from a raw US-shortcode stamp, spec §6):

```bash
source "$HOME/.claude/scripts/stamp-context.sh"
# Provides $STAMP, $MISSION_PROJECT, $MISSION_FILE (empty => no mission resolved).
[ -n "$MISSION_FILE" ] || { echo "No mission resolves for this session. /stamp a project first."; exit 0; }
```

## Behavior

### `/mission` or `/mission status`
`python3 ~/.claude/scripts/mission_parse.py status --path "$MISSION_FILE"`

### `/mission show`
`python3 ~/.claude/scripts/mission_parse.py show --path "$MISSION_FILE"`

### `/mission new`  (heavy creator)
Author a mission JSON from north-star + problem + outcome + an ordered list of goal
stubs (`order`, `goal_id`, `headline`, `exit_when`, `status:"pending"`, `attempt:0`,
`block_reason:null`) + a `done_gate` manifest. If handed a stub list, use it; else
propose a ladder and get operator approval BEFORE writing. Write with a tempfile +
`os.replace` (never a partial file). One mission per project dir (refuse if
`$MISSION_FILE` already exists and is `active`/`blocked`).

### `/mission next`  (operator-initiated orchestrator — the ordered algorithm)
Run the spec §4 algorithm IN ORDER, fail-closed at every step:

1. **Preconditions + pre-decomposition tripwires** — one call:
   ```bash
   python3 ~/.claude/scripts/mission_parse.py next-check --path "$MISSION_FILE"
   ```
   - `{"ok": false, "refuse": …}` → print the refusal and STOP. (killfile / mission
     not active / active.json incomplete / no pending goal / reject-escalation.)
   - `{"ok": true, "target": {…}, "reject_count": N}` → continue with `target`.

2. **Decompose** (the ONLY LLM step). From `target.headline` + `target.exit_when`,
   draft a `scope_uss[]` skeleton — each US an object `{"id","order","status":"pending",
   "rationale","paths":[…]}`. `paths[]` is your best estimate of files each US will
   touch (drives the step-3 sensitive scan). Judge decomposability HERE: if the stub is
   too vague, has no verifier, or the DOR sits at 7.0–7.3 borderline → do NOT proceed;
   block it instead (novel-kind), tell the operator what to resolve, and STOP.

3. **Post-decomposition tripwire + two-phase write** — write the decomposed
   `scope_uss[]` to a temp JSON, then:
   ```bash
   python3 ~/.claude/scripts/mission_parse.py next-commit \
     --path "$MISSION_FILE" --goal "$TARGET_GOAL_ID" --scope-uss-file /tmp/mission-scope.json
   ```
   - `{"ok": false, "tripwire": "sensitive-surface", …}` → the goal is now `blocked`
     and the mission `blocked`; print the HIL-handoff and STOP. Nothing was spawned.
   - `{"ok": false, "refuse": …}` → refuse-overwrite/lock; print and STOP.
   - `{"ok": true, "active_goal": …, "attempt": N}` → `active.json` was written.

4. **Hand to /goal** — if the session is stamped AND `$MISSION_PROJECT` matches the
   stamp's project, `/goal` will auto-execute the spawned USs (its existing stamp-gate).
   If unstamped, tell the operator to run `/goal`. `/mission` does not invoke
   `sh:execute` itself.

### `/mission gate`  (MANUAL — always)
Run the `on_close` gate: invoke the wired `agentflow:dod` / `sh:spec-panel` / `sh:test`
/ deploy+Playwright-smoke across the whole surface, judge against `done_gate.all_of`.
- PASS → `python3 ~/.claude/scripts/mission_parse.py gate --path "$MISSION_FILE" --passed`
- FAIL (e.g. smoke red) → `python3 ~/.claude/scripts/mission_parse.py gate --path "$MISSION_FILE" --reason smoke_red`,
  then surface the spec §8 rollback sequence: (1) mission stays `active` (re-runnable);
  (2) run the project's `deploy.sh rollback`; (3) restore prior docs from the pre-gate
  state; (4) the FAIL is recorded. NEVER mark complete on a red smoke.

### `/mission unblock`  (operator-only recovery)
After the operator resolves a goal's `block_reason`:
`python3 ~/.claude/scripts/mission_parse.py unblock --path "$MISSION_FILE"`
Transitions the blocked goal → `pending`, clears `block_reason`, mission → `active`.
A subsequent `/mission next` re-spawns it (bumping `attempt`). Never automatic.

### `/mission clear`  (archive, never delete)
`python3 ~/.claude/scripts/mission_parse.py clear --path "$MISSION_FILE"` — renames to
`.bak-<ts>`. Confirm first.

## Anti-goals (spec §9)
- Does NOT define quality criteria — wires existing gates.
- Does NOT auto-fire link ② in v1 — shadow-only (proposes); auto-advance is v1.1.
- Does NOT overwrite a non-complete `active.json` — refuse, no `--force` in v1.
- Does NOT skip a blocked goal — goals are sequential; a block halts the mission.
- Does NOT delete state — `clear` archives via rename.
