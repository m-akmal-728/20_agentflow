---
name: stamp
description: "Use to commit a session to a single topic — project (e.g. 50_Excelbridge) or US shortcode (e.g. EBX-S2). Once stamped, /resume-handover, /handover, /goal, /lightsout and /kickoff skip all 'what next?' prompts and operate only on items matching the stamp. No unstamp — close the session to clear."
argument-hint: "[project-or-shortcode]"
---

# /stamp — Session Topic Commitment

A commitment device. Once stamped, downstream skills (`resume-handover`, `handover`, `goal`, `lightsout`, `kickoff`) silently consult the stamp and refuse to ask "what should we do next?" — they execute against the stamped topic only.

## State file

```
~/.claude/guards/stamp-${SID}.txt   # SID resolved via ~/.claude/scripts/stamp-context.sh
```

One line. Contents = the stamp value as a literal string (project dir name, US shortcode, or free-form tag — Stamp does not interpret it; consumers do glob/grep).

Session-keyed so parallel CC sessions cannot collide. Old session files become orphan no-ops when the session closes.

**Critical:** Do NOT use `${CLAUDE_SESSION_ID:-default}` directly — that env var is unset inside the Bash tool subprocess, so the `:-default` fallback collapses every session to a shared `stamp-default.txt` and leaks stamps across sessions. Always source the shared resolver:

```bash
source "$HOME/.claude/scripts/stamp-context.sh"
# Now $SID, $STAMP_FILE, $STAMP, $GOAL_STOP are set.
# If $STAMP_FILE is empty, the session id could not be resolved — treat as unstamped.
```

## Behavior

### `/stamp <value>`

Set stamp directly to `<value>`. No prompt. Overwrites any existing stamp.

### `/stamp` (no arg, no current stamp)

1. Inspect `pwd`. If it resolves under `~/projects/<X>/...` or is itself `~/projects/<X>`, extract `<X>` as the inferred topic.
2. Ask the user (one yes/no): **"Stamp this session as `<X>`?"**
3. On yes → write the file and confirm. On no → do not write; report "session left unstamped."

If `pwd` is not under `~/projects/`, do not guess — emit: `Cannot infer stamp from cwd. Pass an explicit value: /stamp <project-or-shortcode>` and stop.

### `/stamp` (no arg, stamp already exists)

Read the file, report:
```
Stamped: <value>
Funnel: <N Ready + M in-progress items matching the stamp>   (counted from 00_Governance/BACKLOG.md)
```
No prompt, no change.

If a mission resolves for the stamp (`stamp-context.sh` sets `$MISSION_FILE`), add a line to the report:
`Mission: <mission_id> (goal G/N)` — read G/N from `python3 ~/.claude/scripts/mission_parse.py status --path "$MISSION_FILE"`.

### Unstamp

There is none. Close the CC session to clear. If mid-session topic change is needed, open a fresh session (cache-economic anyway: a long session on a new topic loses the cache TTL benefit).

## Implementation

```bash
source "$HOME/.claude/scripts/stamp-context.sh"

# Fail closed: if session id cannot be resolved, refuse to write a shared file.
if [ -z "$STAMP_FILE" ]; then
    echo "Cannot resolve CLAUDE session id (env vars unset and persist file missing). Stamp refused — would otherwise leak across sessions." >&2
    exit 1
fi
mkdir -p "$(dirname "$STAMP_FILE")"

if [ -n "$ARGUMENTS" ]; then
    echo "$ARGUMENTS" > "$STAMP_FILE"
    # Report: stamped as $ARGUMENTS
elif [ -n "$STAMP" ]; then
    CURRENT="$STAMP"
    # Report current stamp + funnel size (see Funnel counting below)
else
    # Infer from pwd:
    PROJ=$(pwd | sed -nE 's|.*/projects/([^/]+).*|\1|p')
    if [ -z "$PROJ" ]; then
        # Report: cannot infer, pass explicit value
        exit 0
    fi
    # ASK USER (one yes/no): "Stamp this session as $PROJ?"
    # On yes → echo "$PROJ" > "$STAMP_FILE" and confirm
    # On no  → report unstamped
fi
```

### Funnel counting (when reporting current stamp)

```bash
BACKLOG="$HOME/projects/00_Governance/BACKLOG.md"
# Count lines containing the stamp value AND a Ready/in-progress status marker.
# Conservative grep — consumers do the precise filtering.
grep -ci "$CURRENT" "$BACKLOG" 2>/dev/null || echo 0
```

## Consumer protocol (how other skills consult the stamp)

Every stamp-aware skill MUST start with this block:

```bash
source "$HOME/.claude/scripts/stamp-context.sh"
# Provides: $SID, $STAMP_FILE, $STAMP, $GOAL_STOP
# $STAMP_FILE empty => session id unresolvable => treat as unstamped.
```

### Critical-path resolution (in stamped consumers)

Resolve in this order — stop at first match:

1. **Goal-stop active and matches stamp** — `goal-stop/active.json` exists AND its `project` field matches `$STAMP` (or `$STAMP` is a US-ID present in `scope_uss[].id`):
   - Critical path = `scope_uss[]` filtered by `status != "complete"`, sorted by `order` ASC.
   - Next item = first entry.
   - Funnel-empty when `exit_condition.all_of` predicates all evaluate true (or all `scope_uss[]` reach `status == "complete"`).
2. **Stamp only, no matching goal-stop** — agent reads BACKLOG.md, filters lines matching `$STAMP`, infers order from in-progress status + recent HANDOVER history. Funnel-empty when no Ready/in-progress items match.
3. **No stamp** — pre-existing unstamped behavior.

Skips: `workflow` stays agnostic and operates on the full backlog regardless of stamp — outer loop is portfolio-wide by design.

### Per-skill behavior matrix

| Skill | Unstamped | Stamped |
|---|---|---|
| `resume-handover` | Newest HANDOVER anywhere; report and STOP | Newest `HANDOVER-*${STAMP}*.md`; **execute the next-task immediately**, no prompt |
| `kickoff` | Ask which project | Skip selection. Run critical-path resolution above. Execute next item. |
| `handover` (`sh:handoff`) | Write `HANDOVER-<project>-<timestamp>.md` | Write `HANDOVER-${STAMP}-<timestamp>.md`. After write, run funnel-empty check; if empty → final line recommends `/lightsout` |
| `lightsout` | Full review | Restrict to items matching `${STAMP}`. If invoked from stamped empty-funnel state, proceed without confirmation. |
| `workflow` | Operates on full backlog | **No change — workflow stays agnostic.** Outer-loop population is portfolio-wide. |

**Rule:** when `$STAMP` is set, a consumer skill MUST NOT ask the user which project/story/topic to act on. It either acts via the resolution order above, or reports "stamp funnel empty — `/lightsout` recommended" and stops.

## Anti-goals

- Stamp does NOT interpret its value. It is a literal string. Consumers do matching.
- Stamp does NOT track sequencing or compute critical path. When a `goal-stop/active.json` exists for the stamp, consumers READ its `scope_uss[].order` — they do not re-derive it. Without a goal-stop, the agent infers order from the backlog at call time.
- Stamp does NOT affect `workflow`. The outer loop (BACKLOG → TODO queue) stays portfolio-wide and topic-agnostic.
- Stamp does NOT provide an unstamp command. Closing the session is the unstamp.
- Stamp does NOT persist across sessions. The file is session-keyed.

## Why this skill exists

Across the existing handover/resume loop, the highest-friction moment is the "what to work on?" question that fires on every cold start — even when the answer is obvious from context. Stamp removes the question by making the answer a one-line file written once per session. The loop becomes `execute → handover → resume → execute → ... → funnel empty → lightsout` with zero user prompts in between.
