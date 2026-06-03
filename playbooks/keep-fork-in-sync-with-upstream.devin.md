<!--
 Licensed to the Apache Software Foundation (ASF) under one
 or more contributor license agreements.  See the NOTICE file
 distributed with this work for additional information
 regarding copyright ownership.  The ASF licenses this file
 to you under the Apache License, Version 2.0 (the
 "License"); you may not use this file except in compliance
 with the License.  You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing,
 software distributed under the License is distributed on an
 "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 KIND, either express or implied.  See the License for the
 specific language governing permissions and limitations
 under the License.
-->
# Playbook: Keep a Fork in Sync with Its Upstream Repository

## Overview
Set up an automated GitHub Actions workflow that keeps a fork's default branch
(e.g. `master`) in sync with its upstream repository, so upstream security
patches, dependency bumps, and fixes flow in automatically. The workflow uses
GitHub's built-in fork-sync (`merge-upstream`) API — the same mechanism as the
"Sync fork" button — which fast-forwards the branch and fails loudly if the
branch has diverged, rather than forcing anything.

## What's Needed From User
- The fork repository (e.g. `annagibaeva/Superset--Devin`).
- The upstream repository it tracks (e.g. `apache/superset`). If not provided,
  infer it from the fork's "forked from" metadata or `git remote`.
- The branch to keep in sync (defaults to the repo's default branch, usually
  `master` or `main`).
- Desired sync cadence (defaults to daily).

## Procedure
1. Confirm the fork, upstream repo, and target branch. Verify the fork was
   actually created from the named upstream.
2. Verify the target branch carries no custom commits on top of upstream — the
   fork-sync API only fast-forwards, so it will fail on a diverged branch. Run
   `git log upstream/<branch>..origin/<branch>` (add upstream as a remote
   temporarily if needed) to confirm there are no fork-only commits. If there
   are, flag this to the user before proceeding.
3. Create a feature branch (e.g. `devin/<timestamp>-sync-upstream-workflow`).
4. Add `.github/workflows/sync-upstream.yml` with:
   - `on:` a `schedule:` cron at the requested cadence (default `"0 6 * * *"`,
     daily 06:00 UTC) plus `workflow_dispatch:` for manual runs.
   - `permissions: contents: write` (required to update the branch).
   - A `concurrency:` group (e.g. `sync-upstream`, `cancel-in-progress: false`)
     so two syncs never run at once.
   - A step that calls the fork-sync API with the repo's `GITHUB_TOKEN`:
     `gh api --method POST -H "Accept: application/vnd.github+json"
     "repos/${{ github.repository }}/merge-upstream" -f branch=<branch>`,
     wrapped with `set -euo pipefail` and an `::error::` annotation + non-zero
     exit on failure so divergence/permission issues fail the run loudly.
5. Add a clear header comment in the workflow explaining what it does and the
   fast-forward-only assumption.
6. Validate the workflow YAML locally (e.g. `python -c "import yaml;
   yaml.safe_load(open('.github/workflows/sync-upstream.yml'))"` or `actionlint`
   if available).
7. Commit, push the branch, and open a PR. Wait for CI to pass.
8. After merge, optionally trigger one manual run via the Actions tab
   (`workflow_dispatch`) to confirm the sync succeeds end-to-end, and tell the
   user where to find it.

## Specifications
- Deliverable: a single PR adding `.github/workflows/sync-upstream.yml`.
- The workflow must be both scheduled and manually dispatchable.
- It must use the `merge-upstream` fork-sync API (fast-forward only) — it must
  NOT force-push or rewrite history.
- It must fail the run (non-zero exit) when the branch has diverged or the token
  lacks write access, surfacing a clear error message.
- Validation: workflow YAML parses successfully and CI passes on the PR; ideally
  a manual `workflow_dispatch` run completes successfully after merge.

## Advice and Pointers
- Use the repo-provided `secrets.GITHUB_TOKEN`; no PAT is needed for syncing a
  branch within the same repo.
- Keep `cancel-in-progress: false` so an in-flight sync is never interrupted.
- If the fork intentionally carries custom commits on the synced branch, the
  fork-sync API is the wrong tool — surface this and discuss a rebase/merge
  strategy instead.

## Forbidden Actions
- Do not force-push or rewrite the synced branch's history.
- Do not add a hardcoded personal access token to the workflow.
- Do not silently swallow sync failures — the run must fail loudly on conflict.
