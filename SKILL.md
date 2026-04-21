---
name: composer
description: Orchestrate a parallel-shaped task by dispatching subagents on cost-appropriate models (Haiku/Sonnet/Opus). Use when a task splits into 3+ independent subtasks of varying difficulty and you want to avoid burning the parent model's tokens on work a cheaper model can handle. Triggers - "/composer", "compose this", "orchestrate this", "fan this out across models". Do NOT use for - sequential dependency chains, single-step tasks, exploratory work where files aren't knowable in advance, non-coding tasks, or anything that fits in a few inline steps. Requires Sonnet or Opus as parent model (Haiku-as-parent will bail).
---

# /composer

Orchestrates parallel-shaped work across cost-appropriate models. The parent model (you, Sonnet or Opus) plans the decomposition, picks the cheapest viable model per subtask, gets the user's approval on the plan, then dispatches subagents in a single parallel batch via the `Agent` tool with explicit `model` overrides.

The point is to stop burning expensive-model tokens on work a cheaper model can do correctly — without composer's own overhead defeating the purpose.

## When to use composer

Invoke composer ONLY when ALL of these are true:
- The task naturally decomposes into 3 or more **independent** subtasks
- Subtasks vary in difficulty (some are mechanical, others need real reasoning)
- Files touched by each subtask can be enumerated in advance
- No file is touched by more than one subtask
- The task is engineering-shaped (not research, content, or strategy)

Good fits:
- "Refactor these 6 React components to use the new hook"
- "Rename `getUserData` → `fetchUserProfile` across the codebase"
- "Write tests for these 4 modules"
- "Generate boilerplate scaffolds for these N entities"

## When NOT to use composer

Refuse and tell the user to do it inline if any of these hold:
- The task is a single investigation or has step-by-step dependencies (step B needs step A's output)
- The task is small enough to do in a few inline steps
- The user can't tell you which files will be touched
- Multiple subtasks would touch the same file
- The task is non-coding (research, strategy, writing, content)
- The parent model is Haiku (Haiku is too weak to plan reliably — bail and ask the user to switch up)

Bail-out messages are direct. Format: "Not a composer-shaped task because [specific reason]. Just do it inline."

Do not negotiate on bail-outs. The skill exists because the wrong tasks defeat its purpose.

## Protocol

When invoked, the parent model follows these steps in order:

1. **Parse the task.** Read the user's task description. Identify the deliverable.
2. **Check parent-model gate.** If parent is Haiku, bail immediately: "Composer requires Sonnet or Opus as the parent model. Switch up and try again."
3. **Check task category.** If the task is non-coding (writing, research, strategy, content, analysis), bail immediately: "Composer is engineering-shaped only. Not for non-coding tasks. Just do it inline."
4. **Decompose.** Identify independent subtasks. If you can't get to 3+ independent units, bail.
5. **Enumerate files per subtask.** For each subtask, list the files it will touch. If a subtask requires exploration to even know which files are involved, bail (see "When NOT to use").
   - **Verify each file exists** before proceeding. If any target file is missing, bail with: "Target file [path] does not exist. Either you're in the wrong directory, the path is wrong, or the file needs to be created first." Offer the user concrete next steps.
6. **Shared-file check.** Cross-reference all files. If any file appears in 2+ subtask scopes, bail with: "Subagents would conflict on [filename]. Not parallel-safe."
7. **Pick a model per subtask.** Apply the rubric (next section). Default to Sonnet. Drop to Haiku only if the work is genuinely mechanical with no invariants. Escalate to Opus only for cross-cutting / architectural work.
8. **Estimate cost.** Apply the cost estimation heuristic (later section). If estimate ≥ inline-equivalent, bail with: "Composer overhead exceeds inline cost. Just do it inline."
9. **Render the plan.** Use the exact plan format below. Show all decisions, including rationale for each model pick and the cost numbers.
10. **Wait for approval.** Stop. Do NOT dispatch. Wait for user response.
11. **On approval (`y`):** Dispatch all subagents in a single tool-call batch (multiple `Agent` tool uses in one assistant message), each with the chosen `model` override.
12. **On edit / rejection:** Re-render the plan with the user's changes, or stop if rejected.
13. **After dispatch:** Wait for all subagents to return. Render the result summary block. Do NOT auto-retry failures — surface them to the user.

## Plan format

The plan is rendered in-conversation in this exact structure:

```
Task: <restated task in one line>
Subtasks: <N>

1. <subtask description> → <Haiku|Sonnet|Opus>
   Rationale: <one sentence — why this model>
   Files touched: <comma-separated paths>

2. <subtask description> → <Haiku|Sonnet|Opus>
   Rationale: <one sentence>
   Files touched: <comma-separated paths>

...

Shared-file check: none | CONFLICT — <details>
Cost estimate: ~<X> input + <Y> output tokens across <N> subagents (coarse)
Inline-equivalent estimate: ~<Z> tokens on parent model (coarse)
Verdict: proceed | refuse — <reason>

Dispatch on approval? (y/n)
```

Rules for rendering:
- Always show the rationale per subtask. The user needs visible reasoning to catch bad model picks.
- Cost numbers are approximations (see Cost estimation section below). Show them anyway — they're a sanity check, not a measurement.
- If verdict is "refuse," the plan is informational only — do not ask for approval, just explain and stop.
- Keep it scannable. No prose paragraphs in the plan. Tables and structured lists only.

## Model selection rubric

Default to Sonnet. Use this rubric to override.

| Model | Use when |
|---|---|
| **Haiku** | Pure mechanical work: renames with no logic change, formatting, comment generation, boilerplate test scaffolds with explicit assertions, simple file-template instantiation. **No invariants, no edge-case logic, no judgment.** |
| **Sonnet** | Default. Anything with logic, refactors that preserve behavior, test writing that requires understanding the code, moderate code-gen, doc generation that requires reading multiple files. |
| **Opus** | Architecture decisions, cross-cutting changes, ambiguous specs, anything where wrong choice has expensive downstream effects. |

Override rules:
- You may override the rubric in either direction, but the rationale must appear in the plan.
- When uncertain between Haiku and Sonnet, pick Sonnet. The cost difference is 5x; the bug cost from a wrong Haiku call is much higher.
- When uncertain between Sonnet and Opus, pick Sonnet. Opus subagents should be rare — if every subagent is Opus, composer is providing no value and you should bail.

Anti-patterns to refuse:
- All subagents on Opus → no savings, just dispatch overhead. Bail.
- Haiku for anything with conditional logic, edge cases, or invariants. Use Sonnet.
- Sonnet for trivial mechanical work that Haiku can do. Use Haiku.

## Cost estimation

Estimates are coarse approximations, not measurements. Their purpose is to catch "obviously the math doesn't work" cases before dispatch.

Method:

1. For each subagent:
   - Estimate input tokens: subagent prompt (~500 tokens) + file context the subagent must read (count file sizes in lines × ~10 tokens/line, or pass file paths and let the subagent read selectively — note in the rationale which approach you used)
   - Estimate output tokens: rough guess of how much the subagent will produce (a refactor of one file ≈ 1-2k output tokens; a test suite ≈ 2-5k; etc.)
2. Sum across all subagents at the chosen model's rates:
   - Haiku: $1/M input, $5/M output (relative units: 1× / 1×)
   - Sonnet: $3/M input, $15/M output (3× / 3×)
   - Opus: $15/M input, $75/M output (15× / 15×)
3. Compare to inline-equivalent: the same total work on the parent model (Sonnet or Opus rates).

If composer's estimate is ≥ inline-equivalent, bail. The most common cause is over-fanning out: many small subagents whose multiplied input contexts erase the per-token savings.

Show both numbers in the plan. Be honest that they're approximations. If the user disputes them in the approval, accept their correction.

## Subagent prompts

Each subagent receives a focused prompt — only its slice of the work, not the full task. This keeps input context small.

Template:

```
Task: <one-line description of THIS subagent's slice>

Files in scope:
<list of file paths the subagent should read and/or modify>

Success criteria:
<2-3 bullet points of what "done" looks like for this slice>

NOT your responsibility:
<what other subagents are handling — so you don't drift into adjacent work>

Return: <what to report back — typically "diff summary" or "files modified + one-line description of changes">
```

Dispatch all subagents in a single assistant message containing multiple `Agent` tool uses, each with the chosen `model` parameter. Do not loop sequentially — the parallelism is the point.

## Dispatch & result handling

On approval:
- Dispatch ALL subagents in a single assistant message (multiple `Agent` tool uses in one batch).
- Each `Agent` call sets `model` explicitly to "haiku", "sonnet", or "opus" — do NOT rely on inheritance.
- Do not interleave dispatches with other work. The batch goes out together.

When all subagents return, render this summary block:

```
Composer dispatch results:
- Subagent 1 (<model>): <done | failed> — <one-line result>
- Subagent 2 (<model>): <done | failed> — <one-line result>
...

Total subagents: <N>
Failures: <count>
```

On failures:
- Do NOT auto-retry. A failure means the rubric or the decomposition was wrong; surface it.
- For each failure, note the failure reason and let the user decide whether to retry, re-decompose, or do that piece inline.

## Worked example

User invokes: `/composer "refactor these 6 React components to use the new useAuth hook: Header.tsx, Sidebar.tsx, Profile.tsx, Settings.tsx, Notifications.tsx, BillingPanel.tsx"`

Plan rendered:

```
Task: Refactor 6 React components to use useAuth hook
Subtasks: 6

1. Header.tsx — replace useContext(AuthContext) with useAuth() → Sonnet
   Rationale: behavior-preserving refactor with potential edge cases in conditional rendering
   Files touched: src/components/Header.tsx

2. Sidebar.tsx — same refactor → Sonnet
   Rationale: same as #1
   Files touched: src/components/Sidebar.tsx

3. Profile.tsx — same refactor → Sonnet
   Rationale: same as #1
   Files touched: src/components/Profile.tsx

4. Settings.tsx — same refactor → Sonnet
   Rationale: same as #1
   Files touched: src/components/Settings.tsx

5. Notifications.tsx — same refactor → Sonnet
   Rationale: same as #1
   Files touched: src/components/Notifications.tsx

6. BillingPanel.tsx — same refactor → Sonnet
   Rationale: same as #1
   Files touched: src/components/BillingPanel.tsx

Shared-file check: none
Cost estimate: ~6k input + ~9k output tokens across 6 subagents (coarse, Sonnet rates)
Inline-equivalent estimate: ~25k tokens on Opus parent (coarse)
Verdict: proceed (~5x savings)

Dispatch on approval? (y/n)
```

User: `y`

Composer dispatches 6 Sonnet subagents in parallel. After all return:

```
Composer dispatch results:
- Subagent 1 (sonnet): done — Header.tsx refactored, conditional rendering preserved
- Subagent 2 (sonnet): done — Sidebar.tsx refactored
- Subagent 3 (sonnet): done — Profile.tsx refactored
- Subagent 4 (sonnet): done — Settings.tsx refactored
- Subagent 5 (sonnet): done — Notifications.tsx refactored
- Subagent 6 (sonnet): done — BillingPanel.tsx refactored

Total subagents: 6
Failures: 0
```
