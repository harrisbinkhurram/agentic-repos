# Changelog

## v1.4.0, 2026-10-11

**Summary:** PR review converges in fewer rounds. The fixer reviews its own fix diff before pushing,
and both reviewer and fixer sweep a finding's shape across the whole PR, so a class is closed in
one pass instead of one sibling per round.

**Cost note:** `ar-taskflow-fix-comments` adds one fresh-context `a_sag_code_reviewer` run per fix
round, scoped to that round's diff, plus at most one repair commit and one re-review. On by
default; `review.fix_delta_review: false` in `config_hints.json` turns it off. Both skills add a
search per finding for other instances of its shape.

**Changed:**
- `skills/ar-taskflow-fix-comments/SKILL.md`: Phase 2 sweeps each finding's shape across the PR's
  changed files and fixes every in-PR instance in the same commit. Phase 4 splits commit from push
  and adds "Review the fix delta": `a_sag_code_reviewer` reviews `{fix_base}..HEAD` before push.
  Bug / Security / Missing findings get one repair commit; anything still flagged or needing a
  decision is surfaced, not looped: a "Fix-delta review" block in the Phase 4 output and every
  Phase 6 ending, approval before the push when interactive, and a `Needs human review` reply on
  the affected finding. The check posts nothing and writes no round marker, so it does not count
  against the round budget. Phase 3 gains an "Outside this PR" section. Phase 5 reverts the whole
  round (`{fix_base}..HEAD`), since a round can now hold a repair commit. Every fence that calls
  the GitHub API or pushes re-derives the PR number, repository and bot login itself, because each
  Bash call is a fresh shell.
- `skills/ar-global-pr-reviewer/checks/classifier.md`: new step 4, sibling sweep. Name the shape,
  search the PR's changed files and the paths they call, report all in-PR instances in one comment.
  It runs for PR-INTRODUCED and PR-EXPOSED findings. Three or more in-PR instances become one
  Approach comment. Instances outside the PR keep the PRE-EXISTING downgrade.
- `skills/ar-global-pr-reviewer/SKILL.md`: Phase 6.5 summary and the Phase 7 reviewer prompt carry
  the sweep; the comment format gains a "Same defect at:" line. New "Run state": Phase 0 writes the
  run's values to `~/ar-global-pr-reviewer/state/<owner>-<repo>-<pr>.env` and every later fence,
  including those in `checks/classifier.md` and `checks/author-intent.md`, loads it, since each
  Bash call is a fresh shell. Loading it after Phase 2 enters the review worktree. Fences stop with
  `exit` instead of a top-level `return`, and PR metadata is piped with `printf '%s\n'` instead of
  `echo`, which rewrites backslash escapes in zsh. Lists of files and reviewer logins are read line
  by line into arguments, since zsh does not word-split an unquoted variable.

## v1.3.1, 2026-10-11

**Summary:** `ar-taskflow` checks, after creating a worktree, that the machine's worktree root is
listed in the user's Claude settings, and offers to add it, so new worktrees stop asking for file
access one by one.

**Cost note:** One script call per created worktree. A question is asked only when the root is
missing, and at most once per run.

**Changed:**
- `skills/ar-taskflow/SKILL.md`: after the post-worktree command, run agentic-devkit's
  `scripts/a_s_worktree_access.sh check` on the new worktree. When the root is missing, ask before
  running its `add`. The helper resolves the root (`<repos>/WorkTrees` for the worktree helper's
  layout, else the worktree's parent) and edits only the user's settings, never the project's.
  Needs an agentic-devkit that ships the helper; with an older one the step is skipped.

## v1.3.0, 2026-10-10

**Summary:** `ar-taskflow-v2`, an opt-in second version of the task flow. It reads v1 and layers six
deltas on named steps, so v1 fixes reach v2 without a copy. v1 stays the default. Two of the deltas,
failure classification and scoped lessons, ship as shared rules that v1 applies too.

**Cost note:** The v2 deltas are inert until a project runs `ar-taskflow v2` or sets
`flow.version: 2`. A v2 run adds two fresh-context subagents per task (test sealing, intent-defeat
review) and up to four probe questions in Phase 1. In v1, `ar-taskflow-fix-comments` adds one
classification line per fixed defect; rules files gain a metadata line only when a rule is written.

**Added:**
- `skills/ar-taskflow-v2/SKILL.md`: D1 environment assumptions and per-criterion evidence tiers
  (`tier`, `required_tier`) with an evidence-ceiling report in the PR; D2 acceptance tests sealed by
  an independent subagent before Phase 3, fingerprinted and gated, plus an intent-defeat review
  before 4h; D3 intent probes in Phase 1; D4 failure classification (implementation, requirement,
  environment, evaluator) and D5 scoped lessons, both extending the shared rules below with
  v2's tiers and failure-log recurrence counting; D6 a Risk block that raises verification for high-risk changes.
- `rules/universal/failure-classification.md`: classify a defect that got past a check as
  implementation, requirement, environment, or evaluator; repair the artifact that admitted it
  before the code; record it in the task's `failure-log.md`; recurrence is a lesson candidate.
- Optional config keys `flow.version`, `flow.risk_surfaces`, `verify.target_command`.

**Changed:**
- `skills/ar-taskflow/SKILL.md`: a Version Select section routes to v2 on a `v2` argument or
  `flow.version: 2`. Phase 2 and Phase 3 apply lesson lines when reading rules; 4g and 4h classify
  a defect found after an earlier check had passed.
- `skills/ar-taskflow-resume/SKILL.md`: resumes under v2 when `execution-summary.md` records
  `Flow version: 2`; classifies a previously-green criterion that now fails.
- `skills/ar-taskflow-fix-comments/SKILL.md`: classifies each real defect before fixing it (Phase 2),
  reports the classes (Phase 3), writes lesson metadata and counts recurring classes (Phase 7).
- `rules/universal/learning-routing.md`: lesson metadata (scope, evidence, retire-when) on rules it
  writes, and scope, retirement, and contradiction checks when rules are read.
- `settings.json`, `templates/settings.template.json`: allow `Skill(ar-taskflow-v2)`.
- `GUIDE.md`: lists `ar-taskflow-v2`.

## v1.2.0, 2026-09-17

**Summary:** Codex support from the same files Claude Code reads, a git safety guard that covers what
the permission list never could, and the review/task-flow improvements that had accumulated upstream.
Nothing here is a second copy of anything: both harnesses symlink one `skills/` directory and run one
guard script.

**Cost note:** Autonomous mode is opt-in and defaults to OFF, so an existing project's flow is
unchanged until `flow.autonomous` is set. The review round budget adds no step on a first review; it
only stops an agent-to-agent exchange that would otherwise not stop. `install-codex.sh` is a separate
command, never run implicitly.

**Added:**
- `install-codex.sh`, the Codex half of the global install. Symlinks the same `skills/` into
  `~/.agents/skills/`, renders agentic-devkit agents into `~/.codex/agents/*.toml` (haiku ->
  gpt-5.6-luna, sonnet -> gpt-5.6-terra, opus -> gpt-5.6; sandbox derived from declared tools),
  installs scripts and hints on the Codex path, and writes a marker-guarded block into
  `~/.codex/AGENTS.md`. `--target` adds a repo's `.codex/config.toml` and the git-guard hook;
  `--target-only` leaves the global layer alone; `--check-only` runs every contract suite.
- `.codex/config.toml` and `.codex/hooks.json`, this repo's own Codex layer.
- `docs/CODEX.md`, how the two harnesses share one set of files, and the `/hooks` trust step that a
  Codex install is unprotected without.
- `AGENTS.md`, now the cross-harness source of truth. `CLAUDE.md` is a two-line import of it.
- `scripts/ar-lint/git-guard-cases.sh`, 43 cases covering every refusal and every command the guard
  must NOT block.
- `scripts/ar-lint/install-cases.sh`, 23 cases for `install.sh` against a disposable HOME.
- `scripts/ar-lint/codex-install-cases.sh`, 33 cases for `install-codex.sh`.
- Skills `ar-optimize-docs`, `ar-optimize-doc-comments`, `ar-optimize-inline-comments`,
  `ar-optimize-tests`, `ar-sonar-sweep`.
- Rules `lean-docs.md`, `lean-doc-comments.md`, `lean-inline-comments.md`, `test-layer-policy.md`,
  `delegation-and-cost.md`, `diagram-output-medium.md`, `seed-vs-migration-jobs.md`,
  `task-flow-startup.md`.

**Changed:**
- `scripts/ar-session/guard-default-branch.sh`, extended from three refusals to nine. It now also
  refuses `git push --all/--mirror`, `--force-with-lease` or a rebase onto a shared branch
  (main/master/develop/staging/`release/*`/`story/*`) or from a detached HEAD, a whole-tree
  `git checkout .`/`git restore .`, `git reset --hard/--merge/--keep`, `git clean` without a dry run,
  a commit whose branch was switched earlier in the same command, and a git command whose command
  word is an unresolvable expansion. The in-progress rebase verbs always pass. When the hook payload
  cannot be read it now degrades by scope, refusing git and allowing the rest with a warning, rather
  than failing open on everything.
- `settings.json` and `templates/settings.template.json`, rebuilt on one rule: `deny` holds only what
  must never run, `ask` holds the operator decisions where a human's answer changes the outcome, and
  `allow` holds everything else, stack-neutral and never pruned per project. `git rebase`,
  `git reset` and `git clean` left the deny list, where a prefix match blocked legitimate uses such
  as `git rebase --continue` and stopped nothing a rephrasing could not evade; the guard handles them
  properly instead.
- `rules/universal/code-review.md`, 145 -> 244 lines. New **Round Budget for Agent-to-Agent
  Exchanges** (marker-counted rounds, `review.max_agent_rounds` default 3, class-is-a-shape dedup,
  and the explicit statement that green CI and the two agents agreeing do not extend the budget), and
  new **Test Layer**, **Documentation**, **Doc Comments**, **Inline Comments** and **Dual-Harness
  Surfaces** criteria, with matching severity bullets.
- `skills/ar-global-pr-reviewer`, pins the PR head by `headRefOid` and aborts if the ref moves
  mid-run, instead of reading the mutable `origin/$HEAD_BRANCH`; counts the round before classifying
  and posts one summary at the cap; posts ONE batched review instead of a call per comment, because
  each `pulls/{n}/comments` call creates its own review object and five comments read as five rounds;
  optional unattended `review.auto_post`.
- `skills/ar-taskflow-fix-comments`, 423 -> 1024 lines. Pre-flight gate (PR state, head OID match,
  clean tree, push-URL validation), post-merge fix-forward, marker-based round detection with the
  `ROUND CAP` stop, a five-state bot-review staleness model, scope-classified verification that
  defers to an existing pre-push hook and reads pipeline exit codes correctly, and a phase that feeds
  durable learnings back into the rules.
- `skills/ar-taskflow`, 2367 -> 2851 lines. Default branch is now resolved, not assumed (it was
  hardcoded to `main`, which was wrong on every master-based repo); new **Change Scope** section so
  verification matches what the diff earns; a real permission predicate in place of a one-line manual
  check; branch-scoped force-push policy matching the guard; opt-in autonomous mode with verifier
  gates and an escalation ledger; mode-gated push.
- `skills/ar-taskflow-resume`, resolves the default branch, keeps the post-resume smoke test off
  `verify.full_command`, and no longer records a resume the user then declines.
- `skills/ar-taskflow-review`, resolves a project-specific `*-code-reviewer` for real instead of
  naming it in a comment, and applies every `alwaysApply: true` rule even to a docs-only diff.
- `install.sh`, links the new `ar-lint` suites, and no longer grows a blank line in the shell rc and
  global `CLAUDE.md` on every run.

**Documentation:**
- `README.md`, `GUIDE.md`, `docs/ARCHITECTURE.md` and `CONTRIBUTING.md` rewritten for the two-harness
  model, the widened guard, the allow/ask/deny rule, the round budget, the new skills and rules, and
  the contract suites. `GUIDE.md` no longer claims there is deliberately no `ask` list, and no doc
  still describes the guard as covering only the default branch and force-push.
- `setup.md`, the settings step now writes and merges `ask` (the merge silently dropped it, so an
  upgraded project lost every operator gate), the dangling-rule-reference check now covers the new
  rule filenames, and there is a new optional **Step: Add Codex Support**. `ar-install` and
  `ar-upgrade` offer that step and say the `/hooks` trust requirement out loud.

**Fixed:**
- Every reference to the devkit review skill named `a_sk_l_review_pr`, which does not exist. The
  skill is `a_sk_review_pr`. 20 occurrences across 9 files, so the delegation was broken wherever it
  was invoked.
- `install.sh` and `install-codex.sh` were not idempotent: the managed blocks they write gained a
  blank line per run. Both installers now have contract suites that assert a second run is
  byte-identical.
- `ar-optimizer` referred to `aa-init-*`, a skill name from the upstream framework that does not
  exist here.

## v1.2.0, 2026-08-16 (fork release, superseded by the v1.2.0 above)

**Summary:** ar-taskflow keeps the zellij pane name showing which task it is running and what it is doing right now, so a developer with several panes open can tell them apart at a glance.

**Cost note:** One `zellij action rename-pane` call per phase boundary, plus one per execution-plan step in Phase 3. Each is a local IPC call to the running zellij session, no network and no measurable wall-clock cost. Nothing gates on it: every call is `2>/dev/null || true` and the flow continues if it fails. Off entirely outside zellij, and switchable off inside it with `flow.zellij_pane_status: "off"`.

**Added:**
- `skills/ar-taskflow/SKILL.md`, new **🪟 Pane Status (zellij)** section: environment detection (`$ZELLIJ` plus `zellij` on PATH, never a question to the user), the rename command, the `<Task Short Name> : <current action>` name format, a phase-to-action table, and the `undo-rename-pane` restore. Renames target `-p "$ZELLIJ_PANE_ID"` rather than the focused pane, so clicking into another pane cannot cause the flow to rename the user's pane instead of its own.
- New optional config key `flow.zellij_pane_status` (`auto` default, `off` to disable), read in the Configuration block alongside the other `flow.*` settings.

**Files changed:**
- `skills/ar-taskflow/SKILL.md`, added the Pane Status section and the config read; added a one-line 🪟 cue at the head of Phases 0-5 pointing at the action to set there.
- `skills/ar-taskflow-resume/SKILL.md`, step 1 runs the same detection and adopts the naming of whichever phase is resumed into; added to the References list.
- `config_hints.json`, `CLAUDE.md`, `.claude-plugin/plugin.json`, version bumped to 1.2.0. `plugin.json` had drifted at 1.1.0 (v1.1.1 did not update it) and is now back in sync.

## v1.1.1, 2026-08-05

**Summary:** Two bug fixes, a devkit skill name that does not exist, and a project-name normalisation that leaked whitespace into a directory name.

**Cost note:** None. Both are corrections to existing behaviour; no new steps, round-trips, or wall-clock time on any path.

**Fixed:**
- **`a_sk_l_review_pr` is not a real skill.** The installed devkit skill is `a_sk_review_pr`. The bad name appeared 18 times across 9 files and did not resolve to anything. It also contradicted devkit's own naming scheme, where the `l_` modifier means "local-only" and applies to routines only (`a_r_l_*`); devkit's `CLAUDE.md` states "Never write `a_sk_l_*`". Because `rules/universal/code-review.md` is part of the installable rules layer and `templates/ar-framework-hints.md` is installed to `~/.claude/`, the wrong name was being written into every adopted repo and read by every session on the machine.
- **Project names containing a space produced a malformed improvements directory.** `ar-record-improvement` Step 7 pascal-cased only snake/kebab/lower-case names; a name already starting with a capital passed through unchanged, whitespace included, so a `project.name` of "My Service" produced a directory literally named `My Service_AgenticRepos`. Whitespace is now a word separator alongside `_` and `-`, giving `My_Service_AgenticRepos`. Names already pascal or snake-cased are unaffected (`Example_Project` stays `Example_Project`). The same derivation in `ar-add-improvement` is fixed identically, the two skills resolve this directory independently and must agree on it.

**Files changed:**
- `rules/universal/code-review.md`, `templates/ar-framework-hints.md`, `CLAUDE.md`, `GUIDE.md`, `docs/ARCHITECTURE.md`, `skills/ar-taskflow-review/SKILL.md`, `skills/ar-global-pr-reviewer/SKILL.md`, `.claude/commands/ar-self-reviewer/SKILL.md`, `.claude/commands/ar-add-improvement/SKILL.md`, corrected the skill name; no logic changes.
- `skills/ar-record-improvement/SKILL.md`, replaced the case-guarded pascal-case block with a single normalisation that treats whitespace as a separator.
- `.claude/commands/ar-add-improvement/SKILL.md`, matching normalisation so both skills derive the same directory.
- `config_hints.json`, `CLAUDE.md`, version bumped to 1.1.1.

## v1.1.0, 2026-07-12

**Summary:** Optional, opt-in per-repo "global-layer precheck" hook so a teammate who clones an adopted repo without the machine-global layer gets an in-session install nudge instead of silent degradation.

**Cost note:** Opt-in only, the default install writes nothing and adds a single yes/no question to the `ar-install` flow. When enabled, the advisory mode (choice 2) only prints a SessionStart banner; the enforce mode (choice 3) hard-blocks edits until the layer is installed.

**Added:**
- `templates/agentic-repos-precheck.sh`, self-silencing detector for the global layer. Detects both install paths (the `install.sh` skills symlink or a plugin install) plus the devkit agents; advisory by default (exit 0), `--block` mode exits 2 for a PreToolUse wall. Points a plugin-only install at `./install.sh` for the complete layer (the plugin does not ship agentic-devkit).

**Files changed:**
- `setup.md`, added **Step: Write Global-Layer Precheck Hook (Optional, opt-in)**; carved out the narrow precheck exception in the "no per-project hooks" rule (it fires only when the global layer is absent and self-silences when present, so it never duplicates the global hooks); added a **Prerequisites** subsection to the generated `AGENTS.md`.
- `.claude/commands/ar-install/SKILL.md`, Phase 4 now offers the optional precheck step; intro reflects the opt-in exception.
- `.claude/commands/ar-upgrade/SKILL.md`, can add the opt-in precheck to an already-adopted repo on request.
- `config_hints.json`, `CLAUDE.md`, `.claude-plugin/plugin.json`, version bumped to 1.1.0.

## v1.0.0

First release of Agentic Repos, the AI-readiness layer that builds on agentic-devkit.

- **Global procedure, local config.** Driver `ar-*` skills, scripts, and hooks install once per machine. Each repo gets only its config + extracted rules via `/ar-install`. No per-repo skill/agent copies.
- **Two install paths.** A Claude Code plugin (`/plugin marketplace add mahsanamin/agentic-repos` then `/plugin install agentic-repos`, hooks fire globally at user scope), or `./install.sh` (symlinks + shell helpers a plugin cannot provide; `--no-hooks` for plugin users).
- **Depends on agentic-devkit** for reusable agents (`a_sag_*`), atomic skills (`a_sk_*`), and worktree helpers (`a_g_worktree_*`). `install.sh` bootstraps it if missing.
- **Rule extraction is first-class.** `ar-agent-ready` and `/ar-install` read the actual codebase and extract its conventions into rules, so every session follows the team's real way of working.
- **Session governance.** `install.sh` wires a global SessionStart hook that steers every session in an agent-ready repo onto the workflow, plus a default-branch / force-push guard.
- **Feedback loop.** `ar-record-improvement` (from any repo) feeds `ar-add-improvement` (in this repo) to improve the framework from real use.
- **Tracker-agnostic.** GitHub Issues by default; Jira, Linear, or none by config.
