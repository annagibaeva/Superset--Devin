# Repository Governance (fork-specific)

This is a fork of [`apache/superset`](https://github.com/apache/superset) kept in sync
with upstream by the scheduled [`sync-upstream.yml`](workflows/sync-upstream.yml) workflow.
This document captures the **fork-specific** conventions that keep changes reviewable and
the history healthy. It is intentionally additive so it does not conflict with upstream files
during sync.

## Pull request size

Large, multi-purpose PRs cannot be reviewed incrementally and bury real changes among
mechanical ones. To keep PRs reviewable:

- **Keep PRs focused** — one logical change per PR. Split unrelated changes.
- PRs are auto-labeled `size/xs … size/xl` by the
  [`pr-size-labeler.yml`](workflows/pr-size-labeler.yml) workflow. A `size/xl` label
  (> ~1000 lines of reviewable change) is a prompt to ask "should this be split?" — it
  is a **warning, not a hard block**.
- **Generated/vendored files do not count** toward the size label and are collapsed in
  diffs (see below), so the label reflects *human-written* change.

## Generated / vendored files

Generated artifacts (translation catalogs, dependency lockfiles) make diffs look enormous
and obscure real review surface. They are marked `linguist-generated=true` in
[`.gitattributes`](../.gitattributes) so GitHub collapses them in PR diffs and excludes
them from language statistics:

- `superset/translations/**/*.po`, `*.pot`
- `**/package-lock.json`, `**/yarn.lock`

When a change touches both code and generated files, **keep the generated output in a
separate commit** from the hand-written change so reviewers can skip it cleanly.

## Recommended branch protection (apply in repo Settings — not enforceable via PR)

Branch protection is a repository **setting**, so it cannot be merged as code. Apply the
following to `master` so every change goes through review and CI (this directly addresses
"large additions with no incremental review"):

- Require a pull request before merging, with **at least 1 approving review**.
- **Dismiss stale approvals** when new commits are pushed.
- Require **status checks to pass** before merging (select the CI checks this repo runs,
  e.g. the lint / unit-test / `Validate All GitHub Actions` jobs).
- Require branches to be **up to date** before merging.
- **Block force pushes** and **deletions** of `master`.
- (Recommended) Require **signed commits** and **conversation resolution**.

Apply via the GitHub UI (Settings → Branches → Add branch ruleset) or the API:

```bash
# Requires admin on the repo. Adjust the required check contexts to match this repo's CI.
gh api -X PUT repos/annagibaeva/Superset--Devin/branches/master/protection \
  --input - <<'JSON'
{
  "required_pull_request_reviews": {
    "required_approving_review_count": 1,
    "dismiss_stale_reviews": true
  },
  "required_status_checks": {
    "strict": true,
    "contexts": []
  },
  "enforce_admins": false,
  "restrictions": null,
  "allow_force_pushes": false,
  "allow_deletions": false
}
JSON
```

> Populate `"contexts"` with the exact check names you want to require (copy them from a
> recent PR's checks list). Leaving it empty requires no specific checks.

## Note on the sync workflow

Because `master` is fast-forwarded/merged from upstream daily, prefer **additive, isolated**
fork-specific files (like this one and the two workflows above) over edits to shared upstream
files, to minimize future merge conflicts.
