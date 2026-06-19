# Wireframe → Hi-Fi Handoff Protocol

This document defines the graduation gate and handoff process from `/sh:ux-design` to `/frontend-design`.

## Core Principle

**Build on top, not over.** The wireframe files stay in place. Hi-fi replaces styles and placeholders with production components. Flow structure and navigation are PRESERVED — not rebuilt.

## Graduation Checklist

ALL items must pass before invoking `/frontend-design`:

```
□ Flow complete — all steps clickable, no dead ends
  Evidence: Playwright test report (screenshot per step, all green)

□ Single concept — variant branches cleaned up
  Evidence: `git branch` shows only wireframe/<feature>, no concept-b/c

□ Step count frozen — no structural changes after graduation
  Agreement: user confirms "step count is final"

□ Interaction model locked — click-through / functional / chat-based
  Agreement: user confirms interaction pattern

□ Component mapping — every wireframe placeholder mapped to target component
  Artifact: docs/wireframe-<feature>-handoff.md (see template below)

□ JTBD validated — user can complete their job in the wireframe
  Evidence: walkthrough confirms job completion (or business-panel brief check)
```

**Use `/sh:verify`** to collect evidence for each item.

## Component Mapping Template

Create `docs/wireframe-<feature>-handoff.md`:

```markdown
# Wireframe Handoff: <feature>

## Flow Summary
- Screens: N
- Transitions: N
- Interaction model: <click-through|functional|chat-based>

## Component Mapping

| Wireframe element | Target component | Library | Notes |
|-------------------|-----------------|---------|-------|
| Primary button (dark rect) | `<Button variant="primary">` | Shadcn/Radix/etc. | |
| Input field (bordered rect) | `<Input>` | | |
| [Image 16:9] placeholder | `<Image>` or `<AspectRatio>` | | Needs real asset |
| [Avatar] placeholder | `<Avatar>` | | Pull from user profile |
| Step counter | `<Stepper>` or `<Progress>` | | |
| Card (bordered rect) | `<Card>` | | |
| Skip link | `<Button variant="ghost">` | | |

## Design Tokens to Apply

Source: Figma MCP `get_variable_defs` or project design system

| Token | Wireframe value | Production value |
|-------|----------------|-----------------|
| Background | #fff | var(--background) |
| Text primary | #333 | var(--foreground) |
| Text secondary | #666 | var(--muted-foreground) |
| Border | #ccc | var(--border) |
| Accent | #333 (or brand tint) | var(--primary) |

## Files

| Wireframe file | Action during hi-fi |
|---------------|-------------------|
| `path/to/screen-1.tsx` | Replace styles, keep structure |
| `path/to/screen-2.tsx` | Replace styles, keep structure |
| `path/to/layout.tsx` | Replace placeholder nav with real nav |

## What `/frontend-design` MUST preserve

- Screen count and order
- Navigation flow (which button goes where)
- Skip/back paths
- Progress indicator position
- Primary action per screen (1 CTA rule)

## What `/frontend-design` SHOULD change

- All grayscale values → design system tokens
- Placeholder boxes → real components
- System fonts → brand typography
- Bordered rectangles → styled cards with shadows/radius
- Labeled icon boxes → real icons from icon library
- Step counter → styled progress component
```

## MCP Tools for Handoff

| Tool | Purpose |
|------|---------|
| Figma MCP `get_code_connect_map` | Map wireframe elements to design system components |
| Figma MCP `create_design_system_rules` | Enforce mapping during hi-fi build |
| Figma MCP `get_variable_defs` | Pull production design tokens |
| Context7 MCP `resolve-library-id` + `query-docs` | Component library documentation |

## After Handoff

Once `/frontend-design` completes:
- `/sh:verify` — visual + functional verification
- `superpowers:finishing-a-development-branch` — merge, cleanup worktree
- Delete `wireframe/<feature>` branch (structure now lives in hi-fi)
