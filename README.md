```
 ██████╗ ██████╗ ███╗   ███╗██████╗  ██████╗ ███████╗███████╗██████╗ 
██╔════╝██╔═══██╗████╗ ████║██╔══██╗██╔═══██╗██╔════╝██╔════╝██╔══██╗
██║     ██║   ██║██╔████╔██║██████╔╝██║   ██║███████╗█████╗  ██████╔╝
██║     ██║   ██║██║╚██╔╝██║██╔═══╝ ██║   ██║╚════██║██╔══╝  ██╔══██╗
╚██████╗╚██████╔╝██║ ╚═╝ ██║██║     ╚██████╔╝███████║███████╗██║  ██║
 ╚═════╝ ╚═════╝ ╚═╝     ╚═╝╚═╝      ╚═════╝ ╚══════╝╚══════╝╚═╝  ╚═╝
              model-aware parallel orchestration for cc
```

A cc skill that fans out parallel-shaped tasks across Haiku, Sonnet, and Opus — routing each subagent to the cheapest viable model, with cost discipline and conflict-detection bail-outs to keep composer's own overhead from defeating the purpose.

## Installation

One command:

```bash
mkdir -p ~/.claude/skills/composer && curl -o ~/.claude/skills/composer/SKILL.md https://raw.githubusercontent.com/jordan-adew/cc-composer/main/SKILL.md
```

Then start a fresh cc session — `/composer` is now available as a skill.

*Manual alternative: download [`SKILL.md`](https://github.com/jordan-adew/cc-composer/blob/main/SKILL.md) and place it at `~/.claude/skills/composer/SKILL.md`.*

---

## Why composer exists

When a task has many independent subtasks, doing them all in your active model burns expensive tokens on work a cheaper model could handle correctly. Composer plans the decomposition, picks a model per subtask, shows you the plan with cost numbers, and dispatches subagents in a single parallel batch — only after you approve.

The bigger the spread between subtask difficulties, the bigger the win.

## When to use it

Invoke composer when ALL of these are true:
- The task decomposes into 3+ **independent** subtasks
- Subtasks vary in difficulty (some mechanical, some need reasoning)
- Files touched by each subtask are knowable in advance
- No file is touched by more than one subtask
- The task is engineering-shaped (not research, content, strategy)

Good fits:
- "Refactor these 6 React components to use the new hook"
- "Rename `getUserData` → `fetchUserProfile` across the codebase"
- "Write tests for these 4 modules"

## When NOT to use it

Composer will refuse and tell you to do it inline if:
- The task is a single investigation or has step-by-step dependencies
- The work is small enough for a few inline steps
- Files can't be enumerated in advance
- Multiple subtasks would touch the same file
- The task is non-coding
- Cost estimate exceeds doing it inline
- Parent model is Haiku (too weak to plan reliably)

Bail-outs are direct, not polite. The skill exists because the wrong tasks defeat its purpose.

## How it works

```
                  ┌─→ haiku  ──┐
                  ├─→ haiku  ──┤
/composer ────────┼─→ sonnet ──┼─→ done
                  ├─→ sonnet ──┤
                  └─→ opus   ──┘
```

1. You invoke `/composer` with a task description
2. Parent model (Sonnet or Opus) plans the decomposition
3. For each subtask: picks Haiku, Sonnet, or Opus per a strict rubric
4. Estimates total token cost vs. doing the work inline
5. Renders the plan with rationale per pick
6. Stops at an approval gate
7. On `y`, fans out subagents in a single parallel batch via the `Agent` tool with explicit `model` overrides
8. Reports results

## Example

Invocation:

```
/composer "add module-level docstrings to these 4 Python files: scripts/sanitize_email.py, scripts/check_draft_leak.py, scripts/validate_extraction.py, scripts/test_sanitize_email.py"
```

What composer renders:

```
Task: Add module-level docstrings to 4 Python scripts
Subtasks: 4

1. sanitize_email.py — add module docstring → Haiku
   Rationale: docstring generation is pure summarization, no logic changes
   Files touched: scripts/sanitize_email.py

2. check_draft_leak.py — add module docstring → Haiku
   Rationale: small utility, summarization only
   Files touched: scripts/check_draft_leak.py

3. validate_extraction.py — add module docstring → Haiku
   Rationale: summarization only, no invariants
   Files touched: scripts/validate_extraction.py

4. test_sanitize_email.py — add module docstring → Haiku
   Rationale: test file docstring = summarize what's being tested; mechanical
   Files touched: scripts/test_sanitize_email.py

Shared-file check: none
Cost estimate: ~11.5k input + ~2k output across 4 Haiku subagents (coarse)
Inline-equivalent estimate: ~13.5k tokens on Opus parent (coarse)
Verdict: proceed (~15× savings)

Dispatch on approval? (y/n)
```

## Requirements

- cc with skill support
- Parent session must be Sonnet or Opus. Haiku-as-parent isn't smart enough to plan reliably; composer will bail and tell you to switch up.

## Model selection rubric

Composer applies this rubric to pick a model per subtask. The parent model can override either direction with stated rationale visible in the plan.

| Model | Use when |
|---|---|
| **Haiku** | Pure mechanical work: renames with no logic change, formatting, comment generation, boilerplate test scaffolds. No invariants, no edge-case logic, no judgment. |
| **Sonnet** | Default. Anything with logic, refactors that preserve behavior, test writing that requires understanding the code, moderate code-gen. |
| **Opus** | Architecture decisions, cross-cutting changes, ambiguous specs, anything where a wrong choice has expensive downstream effects. |

## License

MIT
