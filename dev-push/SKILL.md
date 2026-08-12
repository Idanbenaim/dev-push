---
name: dev-push
description: Safely deploy selected, already-committed frontend and backend changes to the development environment by cherry-picking or, only when proven safe, merging them onto the configured frontend and backend development branches. Use when the user says "push to dev," "push to the dev server," "deploy this to dev," or explicitly invokes dev-push. Inspect conflicts, database dependencies, sensitive files, validation, remote races, CI, deployment status, and rollback information while preserving unrelated local work.
---

# Dev Push

Deploy selected committed changes to development branches without disturbing unrelated work.

## Non-negotiable boundaries

- Never deploy uncommitted changes.
- Never discard, overwrite, stash, commit, or include unrelated user changes.
- Never force-push, reset a remote deployment branch, or rewrite remote history.
- Never execute database migrations or data-changing SQL.
- Never start a local server unless the user explicitly requests runtime testing.
- Never treat a successful Git push as proof that the dev application deployed.
- Exclude the optional AI repository unless the user explicitly requests it and identifies its development target branch.
- Ask only when missing information materially affects deployment safety.

## Repository mapping

| Component | Repository | Development target | Normal feature base |
| --- | --- | --- | --- |
| Frontend | `<frontend-repo>` | `origin/<frontend-dev-branch>` | `<frontend-feature-base>` |
| Backend | `<backend-repo>` | `origin/<backend-dev-branch>` | `<backend-feature-base>` |

Feature and deployment branches can have different ancestry. Default to cherry-picking only the intended commits.

## 1. Identify the deployment

Determine exactly what the user intends to deploy.

- Prefer commit hashes explicitly supplied by the user.
- Otherwise inspect the relevant feature branches and recent commits.
- Identify frontend and backend commits separately and preserve dependency order.
- Never assume both repositories use the same branch or commit.
- Never infer that every commit on the current branch should deploy.
- If the intended commits remain ambiguous, present the proposed commit list and ask before changing deployment branches.
- Confirm that every selected change is committed and each selected commit exists.

## 2. Perform preflight checks

Before mutation, inspect and record for each affected repository:

- repository path and remote URL;
- current branch and worktree status;
- existing Git worktrees;
- source branch and selected source commits;
- target branch and current remote target SHA;
- proposed cherry-pick or merge strategy;
- expected commit order;
- whether frontend, backend, or both are involved.

Run `git fetch origin --prune`. Verify that `origin` is the intended repository, not a fork or unrelated remote.

If the current checkout has local changes, leave it and its branch untouched. Use an isolated temporary Git worktree created from the latest remote deployment target.

## 3. Choose the integration strategy

Default to cherry-pick.

Use cherry-pick when:

- only selected commits should deploy;
- the source branch has unrelated work;
- frontend is based on `<frontend-feature-base>` but targets `<frontend-dev-branch>`;
- backend is based on `<backend-feature-base>` but targets `<backend-dev-branch>`;
- source and target histories diverged;
- a merge could include unrequested commits.

Several commits alone do not justify a merge. Cherry-pick consecutive commits as an ordered range when safe; otherwise use explicit hashes in dependency order.

Use merge only when all of these are proven:

- the complete source branch is intended;
- it contains no unrelated work;
- `target..source` contains exactly the intended commits;
- the source base history already exists in the target;
- no unexpected merge commits exist;
- the resulting diff contains only requested changes;
- preserving ancestry provides value.

Perform any merge in an isolated worktree and inspect the complete diff before pushing. Never merge a `<frontend-feature-base>`-based branch into frontend `<frontend-dev-branch>`, or a `<backend-feature-base>`-based branch into backend `<backend-dev-branch>`, without proving the extra ancestry is already present. Use cherry-pick whenever merge safety is unproven.

## 4. Handle duplicate, empty, and merge commits

Before cherry-picking, determine whether each patch is already present. Do not rely only on commit hashes because cherry-picks create new hashes.

If a cherry-pick is empty:

1. Verify that the intended patch is already present.
2. Skip only after confirming equivalence.
3. Report it as already deployed.
4. Do not create an empty commit.

If only part of a patch exists, inspect the difference and safely apply only the missing intended behavior.

For a selected merge commit:

1. Inspect its parents and effective change.
2. Prefer the underlying implementation commits.
3. Use a mainline-parent cherry-pick only when the correct parent and intended patch are unambiguous.
4. Ask the user if choosing the parent could materially change the result.

Never mechanically cherry-pick a merge commit.

## 5. Prepare the deployment result

For each affected repository:

1. Start from the latest target: frontend `origin/<frontend-dev-branch>`; backend `origin/<backend-dev-branch>`.
2. Record the target SHA before applying changes.
3. Apply selected commits using the approved strategy.
4. Record original source hashes and resulting deployment hashes.
5. Do not push before reviewing conflicts, database dependencies, sensitive files, and validation.

## 6. Resolve conflicts

When a conflict occurs:

1. Inspect every conflicted file and surrounding code.
2. Understand both target behavior and source intent.
3. Preserve compatible behavior from both sides.
4. Avoid unrelated refactors.
5. Search for unresolved conflict markers.
6. Stage only resolved files, continue the operation, and inspect the complete resulting diff.
7. Repeat all relevant validation.

Never blindly choose “ours” or “theirs.” Resolve ordinary conflicts when code and commit intent support one correct outcome.

Abort and ask when business behavior is ambiguous, database migrations compete, resolution expands scope meaningfully, or the resolution could discard required behavior.

## 7. Detect database changes

Inspect selected backend commits and the final backend diff for:

- migrations, seeders, models, or schema changes;
- added, removed, or renamed tables or columns;
- index, constraint, relation, or enum changes;
- raw DDL, stored procedures, or triggers;
- required configuration records;
- backfills or data transformations;
- code that depends on database state absent from the deployment.

Never execute database migrations or data-changing SQL.

When database work is detected, report relevant files and migration IDs, affected objects or records, additive/destructive/uncertain risk, backfill needs, old/new code compatibility, recommended manual order, rollback concerns, and any repository-provided SQL or migration command.

Treat this as best-effort code analysis; never claim the database was updated. Pause before pushing until the user confirms that required manual database work and deployment order are understood or completed.

If none is found, explicitly report that no manual database work was detected.

## 8. Check sensitive and unintended files

Inspect source changes and final deployment diffs for:

- `.env` files, credentials, tokens, passwords, or private keys;
- local authentication overrides or machine configuration;
- debug-only or local-endpoint changes;
- database dumps, exports, customer data, or production data;
- unexpected generated files or large binaries.

Never print suspected secret values. Stop before pushing and safely report the filename and concern when a possible secret or unintended local file appears.

## 9. Validate the result

For each repository:

- search for unresolved conflict markers;
- run `git diff --check`;
- compare the complete final diff with the recorded target SHA;
- confirm only intended files changed;
- confirm every selected commit is represented;
- confirm no unrelated source history entered the result.

For backend changes, run syntax checks on changed JavaScript, focused tests when available, and API-contract review when frontend work depends on it.

For frontend changes, run practical targeted type checks, tests, lint, or builds and review backend API compatibility.

Do not start servers on `<backend-port>`, `<frontend-port>`, or elsewhere unless runtime testing was explicitly requested. If requested, first check for and reuse an applicable running server.

If validation fails, determine whether deployment introduced the failure. Fix an in-scope, unambiguous issue and repeat validation when safe. Do not push while required validation fails; ask if the fix expands scope or changes intended behavior.

## 10. Determine deployment order

- For a backward-compatible API addition, push backend first, then frontend.
- If frontend works with both backend versions, either order can be safe.
- For a single-repository change, push only that repository.
- For database dependencies, follow the confirmed database/application order.
- For a breaking API contract, stop and propose a staged compatibility deployment.

A typical breaking-change sequence is backward-compatible backend support, manual database work, frontend adoption, then later compatibility cleanup. Never describe a multi-repository deployment as atomic.

## 11. Protect against remote races

Immediately before each push:

1. Fetch the remote target again.
2. Compare its SHA with the preparation SHA.
3. If it changed, do not push the stale result.
4. Rebuild on the new remote tip.
5. Repeat conflict handling, database and sensitive-file inspection, and validation.

Never force-push past a remote update.

## 12. Push safely

After validation and required database confirmation:

1. Push backend to `origin/<backend-dev-branch>` when required by the approved order and verify the remote points to the expected commit.
2. Push frontend to `origin/<frontend-dev-branch>` when required and verify the remote points to the expected commit.
3. Use an explicit target refspec so no unrelated local branch is pushed.
4. If rejected, fetch and reassess; never force.

If one repository succeeds and the other fails, immediately report the partial deployment. Do not automatically revert, reset, or rewrite the successful remote.

## 13. Record rollback information

Record frontend and backend pre-deployment SHAs, deployed SHAs, source-to-deployment commit mapping, deployment order, and database assumptions.

If rollback is later requested, prefer explicit revert commits. Never reset or force-push a deployment branch, and never automatically roll back a successful push without explicit authorization.

## 14. Verify deployment status

After pushing:

1. Confirm remote branches contain the expected commits.
2. Determine whether the push triggered CI or a development deployment.
3. Report the CI or deployment URL when discoverable.
4. Inspect status when authenticated read-only tooling is available.
5. Distinguish among preparation completed, Git push accepted, CI completed, dev deployment completed, and runtime verification completed.

Do not claim that the dev server updated from a successful push alone. If status cannot be inspected, state that the branch was pushed but runtime deployment remains unverified.

## 15. Handle failures safely

Stop before mutation when commits are ambiguous; before push when database confirmation is required, a possible secret/local file exists, conflict resolution requires guessing, or required validation fails; and immediately after a partial multi-repository push. Rebuild when the remote target changes. Never conceal skipped, blocked, or failed validation or perform an automatic remote rollback.

Preserve isolated worktrees containing unresolved or useful diagnostic state until the user decides whether they are needed. Remove only temporary artifacts created by this workflow after successful completion and only when removal cannot affect user work.

## 16. Report the result

Finish with a concise deployment report containing:

- repositories deployed;
- source branches and commits;
- cherry-pick or merge strategy;
- source-to-deployment commit mapping;
- conflicts and resolutions;
- validation commands and results;
- database changes and manual actions;
- sensitive-file check result;
- deployment order;
- pre-deployment and final remote SHAs;
- push results;
- CI or deployment status;
- runtime verification status;
- partial-deployment warnings;
- remaining manual work or blockers.
