# Classifier, PR-INTRODUCED / PR-EXPOSED / PRE-EXISTING

Mandatory step before a finding is added to the draft. Silently flagging pre-existing issues as if the PR caused them is the #1 cause of low-confidence noise. This file describes the full procedure.

## The three classes

- **PR-INTRODUCED**, the issue exists because of this PR's changes. The diff added or modified the offending code.
- **PR-EXPOSED**, the PR makes a latent bug reachable. The PR didn't change the buggy code itself, but added a new caller / new code path that now exercises it.
- **PRE-EXISTING**, the same issue is present in unchanged sibling code. This PR neither introduces nor exposes it. The PR is at most a "good moment" to fix it, not the cause.

## Procedure

For each potential issue you've identified, before drafting a comment. Every fence below starts by loading the run state (`SKILL.md` → Phase 0, Run state), which sets `WORKTREE_DIR` and `BASE_BRANCH`; the two lines are shown once, in step 2.

1. **Name the relevant symbol**, the function, variable, pattern, or filename the issue centers on. Be specific. "The validation logic" is not specific enough; `OrderItemValidator.validateItem` is.

2. **Did THIS PR introduce or modify the offending lines?**

   ```bash
   AR_STATE="$HOME/ar-global-pr-reviewer/state/{OWNER}-{REPO}-{PR_NUMBER}.env"   # see Run state
   . "$AR_STATE" || { echo "❌ No run state at $AR_STATE; start from Phase 0." >&2; exit 1; }
   git -C "$WORKTREE_DIR" log -p --first-parent \
     -S "<symbol>" \
     "origin/$BASE_BRANCH...HEAD"
   ```

   - If the diff added or modified those exact lines: **PR-INTRODUCED**. Go to step 4.
   - If no hit (the symbol existed before, untouched by this PR): go to step 3.

3. **Does the same pattern appear in unchanged sibling files?**

   List the files the PR touched:
   ```bash
   touched=$(git -C "$WORKTREE_DIR" diff --name-only "origin/$BASE_BRANCH...HEAD")
   ```

   Grep for the pattern in everything EXCEPT those files:
   ```bash
   git -C "$WORKTREE_DIR" grep -n "<pattern>" -- \
     $(printf ':!%s ' $touched)
   ```

   - If matches appear in unchanged code: **PRE-EXISTING**. The PR didn't cause this; it's a pre-existing condition.
   - If no matches in unchanged code, AND step 2 didn't tag PR-INTRODUCED: check whether the PR added a new caller/path that now reaches the buggy code (a new `controller` method that calls an existing `service` method, etc.). If yes: **PR-EXPOSED**, go to step 4.
   - A PRE-EXISTING finding skips step 4: its comment already lists the locations.

4. **Sweep the PR for every instance of the same shape** (PR-INTRODUCED and PR-EXPOSED findings). Name the finding's shape as the installed code-review rule defines a class ("X is missing from list Y", "the number in A disagrees with the number in B", "case Z is not handled", "the same function or block"), then search the PR's changed files and the code paths they call for every other instance of it: the sibling branch of the same guard, the parallel copy of the same logic, the same claim restated in another file.

   ```bash
   # Recomputed here: step 3 is skipped on the PR-INTRODUCED path, and an empty list would grep the whole repo.
   touched=$(git -C "$WORKTREE_DIR" diff --name-only "origin/$BASE_BRANCH...HEAD")
   [ -n "$touched" ] || { echo "No changed files resolved; sweep not run." >&2; exit 1; }
   git -C "$WORKTREE_DIR" grep -n "<pattern>" -- $touched
   ```

   A grep finds the literal copies; read the callers and parallel paths of the changed code for the rest. Every in-PR instance joins the ONE comment for this finding ("same defect at A, B, C"), anchored on the first, so the fixer closes the shape in one pass instead of one round per instance. Instances outside the PR keep step 3's PRE-EXISTING treatment and go to Notes.

   Three or more in-PR instances of one shape are a class, not a list: per the installed code-review rule's class definition, report them as ONE Approach comment that names the shape, lists the instances, and asks for one structural fix.

5. **Record classification** alongside the finding before drafting any comment text. Comments without classification are rejected from the draft.

## How classification affects the comment

| Class | Comment phrasing | Where to post |
|---|---|---|
| PR-INTRODUCED | "X is wrong / missing / unsafe. Suggested fix: …" | Inline on the changed line |
| PR-EXPOSED | "This PR newly exercises code path X, which has issue Y. Worth handling before this lands." | Inline on the new call site |
| PRE-EXISTING | "Optional: same pattern at file:line and N other locations. This PR is a good moment to fix the family, not just here." | "Notes" section of the draft (NOT inline). Do not phrase as if the PR caused it. |

## Edge case, refactor PRs

A refactor that moves code without changing it: a "PRE-EXISTING" classification still applies to the issue itself, but the refactor is a legitimately good moment to fix it. Mention this in the draft Notes; let the human decide whether to scope-creep the PR or file a follow-up.

## Skip the classifier when

Only one case: the PR is a pure delete (removes a file or function entirely). There's no "exists in sibling code" check needed, the PR removed the code; any reachability concerns are about callers of the now-deleted symbol.
