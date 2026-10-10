---
name: ar-taskflow-v2
description: Version 2 of ar-taskflow, opt-in. Same Raw Prompt - Understand - Plan - Code - Document flow, plus intent probes, environment assumptions with evidence tiers, sealed acceptance tests, an intent-defeat review, risk-scaled verification, and failure classification that revises the artifact that let a defect through. Use when user says "ar-taskflow-v2", "ar-taskflow v2", or the project sets flow.version 2.
disable-model-invocation: true
---

# Task Flow v2

v2 is ar-taskflow plus the deltas below. It is not a separate workflow.

## Load the base

1. Read `ar-taskflow/SKILL.md` in full, from the sibling directory of this skill's base directory.
   Follow it exactly, with three exceptions:
   - Its **🔀 Version Select** section does not apply: you are already in v2.
   - Where a delta below names a v1 step, the delta extends or replaces that step.
   - Every file a delta adds lives in `{task_folder}` beside the v1 files.
2. Write `- **Flow version:** 2` under `## Current State` in `execution-summary.md` when you create
   it, and keep it on every rewrite. `ar-taskflow-resume` reads this line to resume in v2.

**Configuration (all optional, read from `.claude/config_hints.json` beside the v1 keys):**

```bash
risk_surfaces=$(jq -c '.flow.risk_surfaces // []' .claude/config_hints.json 2>/dev/null)
target_command=$(jq -r '.verify.target_command // empty' .claude/config_hints.json 2>/dev/null)
```

- `flow.risk_surfaces`: path globs whose change is high risk by declaration (D6).
- `verify.target_command`: the command that runs verification in the real target environment the
  project ships to (D1). Absent means `target` tier checks are manual steps.

---

## D1. Environment assumptions and evidence tiers

**Phase 2 step 5, add to `execution_plan.md`:**

```markdown
## Environment Assumptions
| # | Assumption about where this runs | ACs relying on it | How it is checked |
|---|---|---|---|
| EA-1 | {inputs, data shape and volume, platform or target, external services, timing, resources, permissions} | AC-2, AC-4 | {check, or "unchecked"} |
```

List only assumptions the verification depends on and the real environment could break. A test
double, fixture, emulator, or local config is an assumption about the real thing it stands in for:
name it.

**Phase 2 step 6, two extra fields per `acceptance_criteria.json` row:**

```json
{ "id": "AC-1", "description": "...", "verification": "...", "passes": false,
  "tier": "unit", "required_tier": "target" }
```

Tiers, lowest to highest evidence:

| Tier | Evidence comes from |
|---|---|
| `static` | reading, type-check, lint |
| `unit` | an isolated test with doubles for its collaborators |
| `integration` | real collaborators inside a local or CI environment |
| `target` | the real environment the project ships to: `verify.target_command`, or a named manual step there |
| `human` | a stakeholder judges the observed behaviour |

`tier` is what `verification` produces. `required_tier` is the lowest tier at which the evidence is
about the real world rather than a stand-in for it. `tier` and `required_tier` are immutable after
plan approval, like the other fields. A row may not flip to `passes: true` on a lower-tier check
than its own `tier`.

**Phase 4 pre-condition, after the v1 AC gate passes, compute the evidence ceiling:**

```bash
order='{"static":0,"unit":1,"integration":2,"target":3,"human":4}'
jq -r --argjson o "$order" '.criteria[]
  | select(.required_tier and ($o[.tier] < $o[.required_tier]))
  | "\(.id): verified at \(.tier), needs \(.required_tier). \(.description)"' \
  "{task_folder}/acceptance_criteria.json"
```

Every printed row, and every `unchecked` Environment Assumption, goes into `pr-description.md`
under `## Not yet verified in the target environment`, and into `execution-summary.md`. It does
not block. It is the list of what a human still has to observe before trusting the change.

## D2. Sealed acceptance tests and the intent-defeat review

**Sealed tests, a new step between plan approval and Phase 3:**

Spawn a bounded general-purpose subagent in a fresh context. It must not be the context that wrote
the plan. Pass it `prompt-understanding.md`, `acceptance_criteria.json`, the public interfaces
`execution_plan.md` names (signatures and entry points only, not the approach), and the project's
test conventions from `<standards_location>`. Prompt:

> For each criterion whose `verification` can be automated at `unit` or `integration` tier, write
> one test that fails when the described behaviour is absent and passes only when it is present.
> Test the observable behaviour in `description`, never the approach. Prefer inputs a hardcoded
> answer could not satisfy: vary values, include a boundary and a rejection case where the
> criterion implies one. Follow the nearest existing tests' location and naming. Return the file
> paths and, per file, the AC ids it covers. Write no implementation code.

Then:

1. Record the paths in `acceptance_criteria.json` as `"sealed_tests": {"AC-1": ["path"], ...}` and
   point each covered row's `verification` at its sealed test. This is the one permitted edit to
   `verification` after plan approval, and only before Phase 3 starts.
2. Fingerprint them:
   ```bash
   cat {sealed test paths, sorted} | git hash-object --stdin > "{task_folder}/sealed-tests.sha"
   ```
3. Run them once and confirm they fail for the expected reason (behaviour absent), not for an
   unrelated error. A sealed test that passes before any code is written proves nothing: send it
   back to the subagent.

**In Phase 3 the implementing context must not edit a sealed test.** A sealed test that is wrong is
a criterion problem: stop, record it in the Change Log, get approval (autonomous mode: escalate),
re-run the sealing subagent for that row, and rewrite `sealed-tests.sha`.

**Phase 4 pre-condition, after the AC gate:** recompute the fingerprint. A mismatch with no Change
Log entry blocks Phase 4, exactly like a `passes: false` row.

**Intent-defeat review, Phase 4, before 4h:**

Spawn a bounded general-purpose subagent in a fresh context. Pass `raw_prompt.md`,
`prompt-understanding.md`, `acceptance_criteria.json`, and the task diff, wrapped in nonce-stamped
evidence delimiters as the v1 Phase 1 gate does. Prompt:

> Assume this diff was produced by an optimiser that only had to make these checks pass. Find every
> way it satisfies the criteria while defeating the evident intent: values or branches that match
> test inputs, behaviour that differs under test or CI, assertions that were weakened, skipped, or
> deleted, errors swallowed so a check passes, stubs or placeholders left on a production path,
> a criterion met only on its happy path, a requirement or dependency the diff invents that the
> prompt does not support. Cite file:line for each. Refute; never bless. Return the JSON verdict
> shape from ar-taskflow's Autonomous Mode.

Interactive mode: show the findings with the 4h review. Autonomous mode: it is a gate, `repair` and
`escalate` work as at every other gate.

## D3. Intent probes

Skip when `## Change Class` is `BEHAVIOR_PRESERVING`: unchanged behaviour is the intent.

**Phase 1, a new step after step 2 (code read) and before step 3 (clarifying questions):**

Build two to four probes. A probe is a concrete situation where two plausible readings of the
prompt produce different behaviour. Also list the requirements an expert would assume without being
told, which the prompt never states.

Present them before any recommendation of your own, so the answer is the user's judgement and not
agreement with yours:

```
Before I write the understanding, a few concrete cases where the prompt could go two ways:

P1. {situation, with concrete values}
    Reading A: {what happens}
    Reading B: {what happens}
    Which should it be?

Assumed without being stated, correct me if any is wrong:
  - {e.g. data written before a restart is still there after it}
```

Ask the smallest question that settles each probe. Then the v1 step 3 clarifying questions follow
as usual. In `prompt-understanding.md`, add:

```markdown
## Intent Probes
| # | Situation | Decided behaviour | Decided by |
|---|---|---|---|
| P1 | ... | ... | user, {date} |

## Out of Scope
- {behaviour a probe ruled out}
```

Every decided probe and every confirmed unstated requirement becomes an AC row in Phase 2, or an
`Out of Scope` line. None stays only in chat.

**Autonomous mode:** a probe settled by the ticket or the code goes under `## Assumptions` with its
citation. A probe that is not derivable and changes the outcome is an escalation blocker. Add to
the Phase 1 gate rubric: "Is every probe either decided with a cited source or escalated? Is any
plausible divergent reading of the prompt not probed?"

## D4. Failure classification

**Applies whenever a defect surfaces after a check had accepted the work:** a real-bug review
comment or failed check in 4l or `ar-taskflow-fix-comments`, CI red after local green, a resumed
task whose previously-green criterion fails, or a defect reported against the change after merge.

Before fixing, classify it:

| Class | Meaning | Repair, in this order |
|---|---|---|
| `implementation` | the code breaks a stated criterion under a stated assumption | add a check that fails on it, then fix the code |
| `requirement` | the code meets the criteria, the criteria missed what the stakeholder wanted | Change Log + approved AC row, its check, then the code |
| `environment` | the real environment differs from an Environment Assumption, or one was missing | add or correct the EA row, raise the affected rows' `required_tier`, add a check that exercises it, then the code |
| `evaluator` | a check that should have caught it did not (too weak, wrong tier, skipped, or not run) | strengthen that check so it fails on the defect, then the code |

In 4l, classify the actionable findings before invoking `ar-taskflow-fix-comments`: make the
non-code repairs first, then let it fix the code and resolve the threads. A defect can be more than
one class: apply every row. Record each in
`{task_folder}/failure-log.md`:

```markdown
| Date | Source | What happened | Class | Artifact revised | Evidence |
|---|---|---|---|---|---|
```

A fix that only changes code for a `requirement`, `environment`, or `evaluator` defect is
incomplete: the same gap admits the next defect.

**After merge (Phase 5 or later):** when the user reports how the change behaved in use, with the
task folder, read the task files, classify, append to `failure-log.md`, and if code must change,
start a new v2 task whose `raw_prompt.md` cites the original task folder and the failure-log row.

## D5. Lessons with scope and retirement

Extends `learning-routing.md` destination 1 (project rule).

A rule written from a v2 run carries one metadata line directly under it:

```markdown
<!-- lesson: scope={paths, modules, or feature area}; evidence={task folder or PR}; retire-when={observable condition} -->
```

- `scope` limits where the rule applies. A rule with no stated scope is project-wide; say so.
- `retire-when` is something a later run can check, such as "the upstream API supports pagination"
  or "module X is removed". "Never" is allowed only for a stable project convention.

**Propose a lesson** when `failure-log.md` shows the same class in the same area twice, in this
task or across archived tasks in `{done_folder}`.

**Phase 2 step 3 addition:** while reading applicable rules, check each in-scope lesson's
`retire-when`. If it is observably met, propose retiring the rule. Retire on user approval only.
If evidence in this task contradicts a lesson, say so and propose narrowing or replacing it; never
silently follow or silently ignore it.

## D6. Risk-scaled verification

**Phase 2 step 5, add to `execution_plan.md`:**

```markdown
## Risk
- **Reversibility:** {can it be rolled back without users or data noticing}
- **Exposure:** {who sees it: developers only, internal, a subset, everyone}
- **Blast radius:** {persisted data, money, credentials or permissions, external parties, none}
- **Level:** low | medium | high
```

The level is `high` if any of: not cleanly reversible, persisted data or money or permissions in
the blast radius, or the diff touches a `flow.risk_surfaces` glob. `medium` if exposure is
everyone. Otherwise `low`. Recompute against `flow.risk_surfaces` at each Change Scope evaluation:
the diff can grow into a risk surface.

| Level | Extra over v1 |
|---|---|
| low | none. v1 Change Scope decides. |
| medium | the intent-defeat review is required in interactive mode too |
| high | the above, plus: every AC row touching the risky surface has `required_tier` of `target` or `human`; the Change Scope answer is at least the full gate; the code review runs even when the 4h fingerprint matches; the PR description opens with the Risk block |

Risk only raises v1's verification, never lowers it.
