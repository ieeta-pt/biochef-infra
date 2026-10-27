# Contributing

These rules apply to every BioChef repo: [biochef-app](https://github.com/ieeta-pt/biochef-app), [biochef-agent](https://github.com/ieeta-pt/biochef-agent), [biochef-coordinator](https://github.com/ieeta-pt/biochef-coordinator), [biochef-hub](https://github.com/ieeta-pt/biochef-hub), [biochef-recipes](https://github.com/ieeta-pt/biochef-recipes), [biochef-infra](https://github.com/ieeta-pt/biochef-infra).

## The flow

1. Branch from `master`.
2. Open a pull request to `master`.
3. Get one approval from someone other than the author, and resolve every comment.
4. Squash-merge. GitHub deletes the branch.
5. To release, tag `master` with `vX.Y.Z`.

## Branches and releases

| Topic | Rule |
|---|---|
| `master` | The only long-lived branch in every repo. It always works, and nobody pushes to it directly. |
| Branch names | `feat/…`, `fix/…`, `docs/…`, `chore/…`, `hotfix/…`, or `recipe/<tool>` in biochef-recipes. Short-lived. |
| Release | A [SemVer](https://semver.org) tag `vX.Y.Z` on `master`, made through GitHub Releases. Anyone with write access can release. A tag is never moved or deleted; only the manager's team can, to fix a mistake. |
| Hotfix | Branch `hotfix/…` from the release tag and fix it there. Tag the patch on that branch, then bring the fix to `master` with a normal PR. |

## Pull requests

| Rule | Why |
|---|---|
| One topic per PR, describing what changed and how it was tested | Easy to review and to revert |
| Approved by someone other than the author | A second pair of eyes on every change |
| An interface change updates its description (OpenAPI, README) in the same PR | Other repos depend on it |
| Squash-merge; the PR title becomes the commit message | One commit per change on `master` |

## Submodule pins

| Rule | Detail |
|---|---|
| Pins point at release tags | Until a repo has releases, at commits on `master`. Never at a PR branch |
| Moving the pins is its own PR to `biochef-infra` | Prepare it with `git submodule update --remote` |
| To try a PR branch, check it out inside the submodule | `git submodule update` returns to the pin; never commit that pointer |

## Never

| Don't | Because |
|---|---|
| Push to `master`, or force-push a branch someone else uses | It loses or rewrites other people's work |
| Move or delete a release tag | A tag must always mean the same code |
| Commit `.env` files, keys or credentials | A pushed secret stays in the history even after it is deleted |
| Commit real data | Tests and examples use synthetic data only |
