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

## Branch protection (apply in repo Settings — not enforceable via PR)

Branch protection is a repository **setting**, so it cannot be merged as code. Apply the
following to `master` so every change goes through review and CI (this directly addresses
"large additions with no incremental review"):

- **No direct pushes** — all changes must go through a pull request.
- Require **at least 1 approving review** before merging.
- **Dismiss stale approvals** when new commits are pushed.
- Require **conversation resolution** before merging.
- Require **status checks to pass** before merging (see table below).
- Require branches to be **up to date** before merging.
- **Block force pushes** and **deletions** of `master`.

### Required status checks

| Check | What it catches |
|---|---|
| `pre-commit (current)` | Formatting (ruff-format), linting (ruff, pylint), type checking (mypy) |
| `unit-tests (current)` | Backend Python unit + integration tests |
| `lint-check` | PR title format (Conventional Commits) |
| `License Check` | Apache license compliance on new files |
| `frontend-build` | Frontend TypeScript compiles without errors |

### Apply via the GitHub UI

1. Go to **Settings → Rules → Rulesets** → edit the existing disabled ruleset
   (or create a new one).
2. Set **Enforcement status** to **Active**.
3. Under **Target branches**, add `master`.
4. Add the rules: Restrict deletions ✅, Block force pushes ✅,
   Require a pull request (1 approval, dismiss stale, conversation resolution) ✅,
   Require status checks (add the five checks above, require up-to-date) ✅.

### Apply via the API (single command)

This repo already has a disabled ruleset (ID `17145019`). Run this to update
and activate it:

```bash
gh api -X PUT repos/annagibaeva/Superset--Devin/rulesets/17145019 \
  --input - <<'JSON'
{
  "name": "Protect master",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["refs/heads/master"],
      "exclude": []
    }
  },
  "rules": [
    { "type": "deletion" },
    { "type": "non_fast_forward" },
    {
      "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 1,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": false,
        "require_last_push_approval": false,
        "required_review_thread_resolution": true
      }
    },
    {
      "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": true,
        "required_status_checks": [
          { "context": "pre-commit (current)" },
          { "context": "unit-tests (current)" },
          { "context": "lint-check" },
          { "context": "License Check" },
          { "context": "frontend-build" }
        ]
      }
    }
  ],
  "bypass_actors": []
}
JSON
```

### Verify after applying

```bash
gh api repos/annagibaeva/Superset--Devin/rulesets \
  --jq '.[] | "\(.name): \(.enforcement)"'
# Expected: Protect master: active
```

## Defense-in-depth: Direct Push Guard

The [`direct-push-guard.yml`](workflows/direct-push-guard.yml) workflow runs
on every push to `master`. If a push did **not** come through a merged pull
request (and isn't the daily upstream sync), it:

1. **Fails the run** (red in Actions → GitHub emails the owner)
2. **Opens a GitHub issue** (deduped by `direct-push-alert` label)
3. **Posts to Slack** (optional — add `SLACK_WEBHOOK_URL` secret)

With the ruleset active, GitHub blocks direct pushes at the API level, so
this workflow should never fire. It exists as a backstop in case the ruleset
is temporarily disabled or bypassed.

## Note on the sync workflow and direct-push allowlist

The direct-push-guard allowlists bot actors (e.g. `github-actions[bot]`) and
pushes that match a successful `sync-upstream.yml` run, so the daily
upstream fast-forward does not trigger false alerts.

Because `master` is fast-forwarded/merged from upstream daily, prefer **additive, isolated**
fork-specific files (like this one and the two workflows above) over edits to shared upstream
files, to minimize future merge conflicts.
