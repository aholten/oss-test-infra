# MERGE ORDER — making the `ci-passed` gate live for kubeflow/pipelines

## Invariant

`master` (and no other protected branch of kubeflow/pipelines) may never be in
a state where branch protection **requires** the `ci-passed` commit status while
no publisher exists to produce it. The commit that declared the requirement
already states this:

> 8ca666ca — *"Merge order matters: kubeflow/pipelines#14111 must land first …
> otherwise master merges block on a check that does not exist yet."*

Two corollaries the ordering below also preserves:

1. Never drop the `ci-passed` **label** gate from the human Tide query while the
   `ci-passed` **status** gate is not yet enforced — otherwise human PRs merge
   with no gate at all.
2. Never set `github_merge_blocks_policy: block` before the publisher is live —
   otherwise Tide flips from ignoring GitHub's BLOCKED state to hard-blocking on
   a status nobody publishes.

## Current state (verified 2026-09-01)

| Fact | Evidence |
|---|---|
| 8ca666ca is merged to `upstream/master` and declares `ci-passed` as a required branch-protection status **repo-wide** (all protected branches of pipelines, not just master) | `git merge-base --is-ancestor 8ca666ca upstream/master` → true; `git show upstream/master:prow/oss/config.yaml` |
| kubeflow/pipelines#14111 (the publisher) is **OPEN**, `mergeable_state=blocked`, head `9d6c9a73` | `gh api repos/kubeflow/pipelines/pulls/14111` |
| #14111's head runs the **old label-based** workflow (`Add 'ci-passed' label`, `Reset stale 'ci-passed' label` — `pull_request_target` runs from base), and carries `needs-ok-to-test` (no `ci-passed`) | `gh api .../commits/9d6c9a73/check-runs`, `.../issues/14111/labels` |
| #14238 merged **2026-09-01T13:50:22Z** with only a `tide` status — no `ci-passed` — proving the gate is declared but not enforced | `gh api .../pulls/14238`, `.../commits/c3b0959/status` |
| `upstream/master` human Tide query still requires the `ci-passed` **label**; only #2674 drops it | `git show upstream/master:prow/oss/config.yaml` (line 101) |
| Live branch protection is **unverifiable**: `gh api repos/kubeflow/pipelines/branches/master/protection` → `403 Resource not accessible by integration` | direct call |

## Why #14111 is currently deadlocked (bootstrap problem)

- #14111 carries `needs-ok-to-test`; master's label-based `ci-checks.yml` refuses
  to add the `ci-passed` label while that label is present (this is the very bug
  #14111 fixes).
- Branch protection requires the `ci-passed` **status**, which only #14111's own
  (unmerged) rewrite publishes.
- So #14111 cannot auto-merge: it can't earn the label (needs-ok-to-test), and
  the status it would publish doesn't exist yet.

It must be landed **manually** (admin merge, or remove `needs-ok-to-test` + apply
`lgtm`/`approved` and let Tide `permit`-merge it) — while
`github_merge_blocks_policy` is still `permit`.

## Ordering (each step gates the next)

1. **Merge kubeflow/pipelines#14111 (the publisher) FIRST.** This is the literal
   requirement of 8ca666ca's warning. Bootstrap via manual/admin merge as above.
   Do **not** flip `merge_blocks_policy` to `block` before this lands.

2. **Verify the publisher is live.** Confirm `ci-passed` is published as a
   **status** on real PR head SHAs (verified head only, re-published on re-run),
   and the old `Add 'ci-passed' label` check-run is gone.

3. **Merge this change (`github_merge_blocks_policy: {kubeflow/pipelines: block}`).**
   This is the moment the gate becomes *enforced*: Tide now treats GitHub's
   BLOCKED state (where the required status is actually checked) as blocking.
   Until this point the `ci-passed` **label** is still required in the human
   query (upstream/master), giving belt-and-suspenders during the transition.

4. **Confirm branch protection applied, then merge #2674.** #2674 (head `001648cb`)
   scopes the required status to `master` only (shrinking 8ca666ca's repo-wide
   blast radius onto release branches), adds `strict: true` (re-merge + re-test
   on base change), drops the now-redundant `ci-passed` label from the human
   query, and adds `needs-ok-to-test` to all three overlapping queries (human
   `repos:`, dependabot `repos:`, and `orgs: kubeflow`). Dropping the label is
   safe only *after* Steps 1–3 made the status gate live.

5. **Verify end-to-end.** A failing PR is BLOCKED and Tide does not merge it; a
   passing PR earns `ci-passed` and merges; a `needs-ok-to-test` human PR is
   held (`needs-ok-to-test` is in every overlapping query's `missingLabels`).

## Residual gaps (flagged, NOT fixed by this change)

`needs-ok-to-test` in the `orgs: kubeflow` query and `strict: true` are **already
covered by #2674's head** (`001648cb`) and are *not* gaps. The genuine residuals:

- **Live branch protection unreadable (403).** An admin must confirm which
  contexts are actually required before Step 5; the repo-side declaration
  (8ca666ca) is assumed, not confirmed, to be applied.
- **`github_merge_blocks_policy` support.** Field verified against
  `kubernetes-sigs/prow/pkg/config/tide.go` (map: `*` | org | org/repo →
  `ignore`/`permit`/`block`; default `permit`). The configurator image is
  `v20260325` (recent). Confirm the deployed Tide honors the field — otherwise
  the gate stays dead.
- **Blast radius of `block`.** For pipelines, `block` makes Tide respect *all*
  GitHub BLOCKED conditions (required reviews / other required statuses set in
  the GitHub UI, invisible here via 403), not only `ci-passed`. Confirm no
  unexpected required checks exist before Step 3 lands.

## Option-b fork (uncommitted WIP in the main working tree — READ ONLY, not adopted)

The uncommitted edit replaces the aggregated `ci-passed` context with direct
required-check placeholders (`<REQUIRED-CHECK-1>`, `<REQUIRED-CHECK-2>`) and adds
`strict: true`.

Recommendation (grounded in the invariant): **keep the aggregate `ci-passed`
status** as the single required context. It is a stable contract — the
underlying check set can change inside `ci-checks.yml` without touching
branch-protection config, and it is what #14111 and 8ca666ca were designed
around. The direct-placeholder approach is (1) incomplete (placeholders),
(2) brittle (every check rename/reorder requires an oss-test-infra config change
to keep branch protection in sync), and (3) redundant — it re-implements what
the aggregate already encodes. Its `strict: true` idea is already adopted in
#2674 (`001648cb`), so option b adds nothing beyond the placeholder rewrite —
reject the rewrite.
