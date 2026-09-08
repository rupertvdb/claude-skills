---
name: adversarial
description: Use when the user types /adversarial, or asks to "poke holes in this", "red-team this plan", "adversarial review", "try to break this", "what's wrong with what we just did", or otherwise wants the code or plan JUST produced in this thread stress-tested by a fresh, hostile reviewer that has no stake in the work. Dispatches an isolated subagent (own context, never sees the thread's rationalizations) whose only job is to find the fatal flaw. Report-only, read-only. Args: a model override (e.g. opus/sonnet/haiku), a target override (code/diff/plan), -quick for a lighter pass.
---

# /adversarial: a fresh, hostile reviewer against what you just built

The move you reach for right after producing something you believe is right:
a subagent whose only job is to prove you wrong. It does not see this
conversation, so it cannot inherit the reasoning that talked us into believing
the work was fine. It starts cold, reads the artifact, and hunts for the flaw.

Two things make it worth more than "re-read your own diff":

1. **Fresh context means no anchoring.** A subagent dispatched via the Agent
   tool gets none of this thread's history. It has not been persuaded. It reads
   the code or plan as a stranger would, which is exactly who finds the bug the
   author is blind to.
2. **Pinned to a strong model.** The reviewer defaults to **Opus**, never
   inheriting whatever tier this thread happens to be on. A critic should not be
   less capable than the author, or it cannot see what the author missed.

## When to use

- `/adversarial` right after writing code, a migration, or a design/plan you
  are about to commit to.
- "poke holes in this", "red-team it", "try to break this", "what did we miss".
- Before a risky merge, a schema change, anything security or data touching.

**Don't use** when:

- Nothing substantive was produced this thread (a chat answer, a one-line doc
  fix). There is nothing to attack.
- You want a full close-out pipeline (review + capture lessons + commit +
  ship). `/adversarial` is the single focused pass, on its own, with no ship
  step attached.

## Args

| Arg | Effect |
|---|---|
| (none) | Auto-detect the target, review with **Opus**, report-only, read-only. |
| `code` / `diff` | Force the target to be the code / uncommitted changes. |
| `plan` | Force the target to be the last plan or design (no diff needed). |
| a model name (e.g. `sonnet` / `haiku`) | Override the reviewer model (default `opus`). See the warning below: a model weaker than the author's defeats the point. |
| `-quick` | Lighter pass: drop the `ultrathink` nudge, ask for only the top findings. For small diffs. |

Args combine, e.g. `/adversarial plan sonnet` or `/adversarial code -quick`. Any
token you do not recognise (for example `/adversarial the auth change`): treat
it as a scope hint and pass it through to the reviewer, do not drop it or guess.

## Model choice (why the default is what it is)

- **Default: Opus.** Finding a subtle logic gap, a race, an unhandled failure
  path, an authorization hole, is a hard reasoning task. Use the strongest tier
  for it.
- **Pinned, not inherited.** Set `model` explicitly on the Agent call so the
  reviewer's quality never silently drops because this thread switched tiers.
- **Effort is a nudge, not a dial.** The Agent tool has no reasoning-effort
  parameter; the real lever you control is the model pin. The dispatch prompt
  includes the word `ultrathink` to *encourage* deeper reasoning, but do not
  claim a guaranteed "maximum reasoning" budget. The pin is the guarantee; the
  keyword is best-effort.
- **A downgrade below the author's tier defeats the purpose.** If this thread
  wrote the work on Opus and you pass `haiku`, the critic is now weaker than
  the author and will miss what the author missed. Only downgrade deliberately,
  for a fast sanity pass on a small change, and say so.
- **Same family is fine.** The big diversity win is the fresh context plus the
  hostile framing, not a different model vendor.

## Procedure

### Step 1: decide the target

Look at what this thread actually produced. **The artifact is what THIS thread
changed, not whatever is dirty on disk.**

- **A git repo with uncommitted changes** (`git -C <dir> status --short`):
  target is the **diff**, but scope it. A shared or long-lived checkout can
  carry unrelated modified and untracked files from earlier work. A blanket
  `git diff` would feed that junk to the reviewer and aim it at the wrong code.
  So pass the reviewer the **explicit list of files this thread touched** (you
  know which ones you edited), or a diff against a known base ref, not a bare
  `git diff`.
- **Not a git repo, or the work landed outside the tree** (skills, config,
  scratch files, generated docs): target is those **specific files**. Auto-
  detect them from what you just wrote this thread and pass their paths. Do not
  conclude "nothing to attack" just because `git` reports no diff.
- **No files changed, but the last substantial output was a plan / design /
  approach**: target is the **plan**. Reconstruct the *current, consolidated*
  plan (fold in every later revision, do not paste a stale first draft) to hand
  to the reviewer. It will not see the conversation.
- **Both a diff and a plan exist**: prefer the diff unless the user passed
  `plan`.
- **Arg override** (`code` / `diff` / `plan`) always wins.
- **Genuinely nothing produced**: say so and stop.

### Step 2: assemble the artifact for a cold reader

The subagent starts with zero context. Give it what it needs and nothing that
would bias it toward our conclusion:

- **For code:** the file paths (scoped per Step 1) and the command to see the
  change. Let the reviewer read the real files itself, so it is not limited to
  our summary.
- **For a plan:** paste the consolidated plan text in full. The plan path is
  inherently weaker at escaping bias than the code path, because the reviewer
  sees only what you paste. Mitigate it: include the whole plan, not the parts
  you think matter, and where a source doc or spec exists, point the reviewer at
  it to read directly.
- **Intent line:** state the problem the work solves in one bare sentence. Do
  not defend the approach. The dispatch prompt already tells the reviewer to
  distrust this line, so keep it neutral.

### Step 3: dispatch the isolated reviewer

Use the **Agent tool** (subagent, e.g. `subagent_type: "general-purpose"`),
`model` per the arg (**default `opus`**), and run it in the foreground so the
findings come back before you respond. The prompt below is the contract. Keep
the word `ultrathink` unless `-quick` was passed.

```
You are an adversarial reviewer. You did NOT write the work below and you have
no stake in it. Your ONLY job is to find the flaw that makes it fail. Assume it
is wrong until the code or logic proves otherwise. A review that finds nothing
is a review that did not look hard enough. ultrathink.

READ ONLY. Do not use Edit, Write, or any command that modifies files, git
state, or anything else. You investigate and report. You do NOT fix. Returning
a fix instead of a finding is a failed review.

Distrust the intent line below: it is the author's framing, not proof the work
is correct. Verify behaviour against the actual code / logic.

INTENDED BEHAVIOUR (one line, do NOT treat as proof it works):
<the neutral one-sentence description>

THE ARTIFACT:
<for code: the scoped file paths + the command to see the change; read the real
 files, do not trust any summary>
<for a plan: the full consolidated plan text>
<any scope hint the user passed through>

Hunt specifically for:
- CODE: correctness bugs, off-by-one / boundary / empty-input cases, race
  conditions, unhandled errors and partial-failure states, data loss or
  corruption, resource leaks, and — wherever the project handles auth, multiple
  users, or tenants — authorization gaps, cross-user/cross-tenant leakage, and
  secret exposure. Verify invariants against the actual guard or migration,
  NOT against a sibling that may itself be stale.
- PLAN: hidden assumptions, failure modes it never mentions, ordering and
  dependency hazards, blast radius, the missing rollback story, and whether it
  actually solves the stated problem or just the easy part. Name a cheaper or
  safer alternative if one exists.

For every finding give:
1. Severity: BLOCKER (ships broken / loses data / opens a hole) / MAJOR (wrong
   in a real case) / MINOR (will bite later).
2. A concrete failure scenario: specific inputs or state -> the wrong outcome.
3. Where: file:line, or the plan step.
4. CONFIRMED (you read the code and verified) vs SUSPECTED (looks wrong,
   unverified). Never dress a suspicion up as confirmed.
5. A direction for the fix (not a full rewrite, and do not apply it).

Ignore pure style and naming unless it causes a bug. Rank findings most-severe
first. End with a one-line verdict: SHIP / FIX-FIRST / RETHINK. Your final
message is the report itself, returned to the caller, not a human-facing
message.
```

For `-quick`: drop `ultrathink`, and ask for only the top findings.

### Step 4: relay the findings

The subagent's report comes back to you as the tool result; the user has not
seen it. Relay it faithfully:

- Lead with the **verdict** (SHIP / FIX-FIRST / RETHINK) and the BLOCKER /
  MAJOR count.
- List findings most-severe first, each with its failure scenario and location.
- Keep CONFIRMED vs SUSPECTED honest. Do not upgrade a suspicion.
- **Do not fix anything.** `/adversarial` reports; the human decides what to act
  on. If they then want fixes, that is a separate, explicit step.
- If the reviewer found nothing real, say so plainly. Do not manufacture
  findings to look thorough.

## Common mistakes

| Mistake | Fix |
|---|---|
| Reviewing in this thread instead of a subagent | Defeats the whole point. The value is the cold, un-anchored context. Always dispatch. |
| Feeding it a bare `git diff` in a shared/dirty checkout | It grabs unrelated dirty files and reviews the wrong code. Scope to the files this thread touched (Step 1). |
| Concluding "nothing to attack" because there is no git diff | The work may be outside the tree (skills, config, scratch). Target the specific files you just wrote. |
| Letting the reviewer inherit the thread's model, or downgrading below the author | Pin `model`, default Opus. A critic weaker than the author is useless. |
| Not enforcing read-only | General-purpose subagents have Write/Edit. The prompt forbids mutation; keep that clause. Report-only means the tree is untouched. |
| Feeding it our reasoning for why the work is good | That is the anchoring you are trying to escape. Give the artifact and a neutral intent, and tell it to distrust the intent. |
| Padding with style nits | Instruct it to ignore style unless it causes a bug. Blockers and majors are the product. |
