% AMP — Label Guide

> Companion to [README.md](../README.md) (user-facing flow + CLI reference) and [AMP_User_Guide.md](AMP_User_Guide.md) (long-form guide). This file is the **complete** label reference — every label defined in [`config/labels.yml`](../config/labels.yml), grouped by area, with meaning, who applies it, and when.

# Why labels matter

AMP has **no separate database**. Each source repo becomes one GitHub issue, and the issue's labels *are* the state. The CLI, the workflows, the dispatcher, the AI advisor, and the Projects v2 Kanban all read and write the same label set. This means:

- **You drive AMP by applying labels.** Apply `cmd:dryrun` to a row in the GitHub UI (or 5,000 rows from a CSV via `amp bulk-assign`) and the IssueOps bridge fires the workflow.
- **You read AMP's state by reading labels.** `status:migrate-success` + `migration-fixes:pushed` + `upserted:teams` says everything you need to know about an issue's progress.
- **Labels are not live until you push them.** `config/labels.yml` is a manifest. Run `amp setup-labels --repo <owner>/<control-repo>` against the control repo to create them via the GitHub API. Idempotent; safe to re-run after editing the manifest.

# How labels get applied

| Source | What it sets | Where the logic lives |
|---|---|---|
| **`amp analyze`** | Analysis labels: `provider:*`, `size:*`, `prs:*`, `blob:*`, `lfs:*`, `age:*`, `submodules*`, `webhooks`, `actions:*`, `codeowners:*`, `secrets:present`, `hardcoded-urls:present`, `branch-protection:present`, `custom-properties:present`, `ref:*`, `refs:high`, `tags:high`, `commits:huge`, `releases:oversized`, plus `namespace:`, `src-org:`, `src-team:` and the analyze-time `status:analyzing` / `status:analyzed` / `status:analysis-failed` / `status:empty` / `status:duplicate-candidate`. | [amp/analyze/](../amp/analyze/) |
| **Status machine** | `status:*` lifecycle (mutually exclusive within stage), `status:has-warnings`, `status:validate-*`, `migration-fixes:*`, `custom-properties:*`, `secrets-recreate:*`, `environments:*`, `branch-protection:dest-*`, `upserted:*`. Stage isolation prevents a later failure from clobbering an earlier success. | [amp/labels.py](../amp/labels.py) — `set_status_label`, `set_post_migrate_label`, `stamp_upserted` |
| **IssueOps workflows** | Stamp `cmd:*` → run the command → stamp the resulting status + post-migrate labels. The `cmd:*` label triggers the workflow; the workflow itself clears the trigger label and applies outcome labels. | [.github/workflows/label-ops.yml](../.github/workflows/label-ops.yml) → [.github/workflows/issue-ops.yml](../.github/workflows/issue-ops.yml) → [amp/commands/](../amp/commands/) |
| **AI advisor** | `cp-failreason-<slug>`, `cp-repair-available`, `cp-repair-applied`, `lfs:dest-auth`, `lfs:dest-saml`, `lfs:dest-quota`, `meta:oversized` (when diagnosed as the cause). | [amp/ai/advisor.py](../amp/ai/advisor.py), [amp/ai/repair.py](../amp/ai/repair.py) |
| **Operator** (you) | `cmd:*` triggers, `cfg:skip-releases`, routing labels `dst-org:*`, `dst-team:*`, `dst-prefix:*`, and corrections to any analyzer-set label. Applied via GitHub UI, `gh issue edit`, `amp bulk-assign --csv`, or `amp import-commands` (Excel). | n/a — manual |

> **Re-analysis preserves operator intent.** When `cmd:update-issue` fires, AMP re-applies analysis labels but keeps `status:`, `cmd:`, `cfg:`, `dst-`, `cp-repair-`, `provider:`, `duration:`, `migration-fixes:`, `custom-properties:applied`, `secrets-recreate:`, `environments:created/partial/failed/rate-limit-deferred`, `branch-protection:dest-*`, and `upserted:*`. Set in `PRESERVE_PREFIXES_DEFAULT` in [amp/labels.py](../amp/labels.py).

# Label reference by area

## Source provider (`provider:*`)

One per issue — recorded when the analyzer creates the issue.

| Label | Meaning | Applied by | When |
|---|---|---|---|
| `provider:ado` | Source: Azure DevOps | analyzer | issue creation |
| `provider:bitbucket` | Source: Bitbucket Server | analyzer | issue creation |
| `provider:bitbucket-cloud` | Source: Bitbucket Cloud | analyzer | issue creation |
| `provider:github` | Source: GitHub | analyzer | issue creation |
| `provider:gitlab` | Source: GitLab | analyzer | issue creation |

## Lifecycle status (`status:*`)

Mutually exclusive **within a stage** (analyze, dryrun, migrate, archive, repair, validate). Setting a new label in a stage clears its same-stage peers but leaves earlier stages alone — so `status:analyzed` + `status:dryrun-success` + `status:migrate-failed` is a valid combination (failed migrate after a clean dryrun). Stage mapping: [`STATUS_STAGES`](../amp/labels.py).

| Label | Meaning | Applied by | When |
|---|---|---|---|
| `status:analyzing` | Analysis in progress | analyzer | start of `amp analyze` |
| `status:analyzed` | Repo analyzed, ready for action | analyzer | end of successful analyze |
| `status:analysis-failed` | Analysis failed (e.g. clone refused) — fix and re-run analyze | analyzer | analyze hit an unrecoverable error |
| `status:empty` | Source repo has no commits / no default branch — migration not applicable | analyzer | clone showed zero commits or no default branch |
| `status:duplicate-candidate` | Multiple issues match the same source repo; needs human review | analyzer | duplicate AMP-ID detected during refresh |
| `status:obsolete` | Source repo no longer exists; flagged by refresh | `amp refresh` | source returns 404 on re-check |
| `status:dryrun-queued` | Dry run queued | dispatcher | `cmd:dryrun` applied while queue is enabled |
| `status:dryrun-running` | Dry run in progress | migration-worker | worker claimed the queued item |
| `status:dryrun-success` | Dry run completed successfully | migration-worker | GEI exited 0 |
| `status:dryrun-failed` | Dry run failed | migration-worker | GEI exited non-zero |
| `status:migrate-queued` | Migration queued | dispatcher | `cmd:migrate` applied while queue is enabled |
| `status:migrate-running` | Migration in progress | migration-worker | worker claimed the queued item |
| `status:migrate-success` | Migration completed successfully | migration-worker | GEI exited 0 |
| `status:migrate-failed` | Migration failed | migration-worker | GEI exited non-zero |
| `status:has-warnings` | Last `/dryrun` or `/migrate` completed but GEI reported warnings, **or** a best-effort post-migration step didn't fully land. Coexists with `status:*-success`; re-evaluated each run, so a later clean run clears it. | migration-worker / post-migrate handlers | warning surface from GEI or a `partial` / `failed` post-migrate outcome |
| `status:repairing` | Auto-repair in progress | advisor / repair handler | tier-2 repair plan accepted |
| `status:archiving` | Archiving in progress | archive handler | `cmd:archive` claimed |
| `status:archived` | Source repo archived | archive handler | archive succeeded |
| `status:validate-passed` | Post-migration source-vs-destination comparison passed | validate handler | branch/tag/commit counts match (and SHA where applicable) |
| `status:validate-failed` | Post-migration source-vs-destination comparison found a mismatch — see issue comment | validate handler | counts diverged |
| `status:validate-history-rewritten` | History rewrite (e.g. LFS conversion) makes SHA comparison non-applicable; non-SHA checks still run | validate handler | `lfs:*` indicates the repo's history was rewritten |

## Size, PRs, and volume

Rough capacity buckets, set at analyze time. Thresholds live at the top of [amp/analyze/base.py](../amp/analyze/base.py).

| Label | Meaning | Applied by | When |
|---|---|---|---|
| `size:small` | Repo < 100 MB | analyzer | always (one of four) |
| `size:medium` | Repo 100 MB – 1 GB | analyzer | always |
| `size:large` | Repo 1 GB – 5 GB | analyzer | always |
| `size:huge` | Repo > 5 GB | analyzer | always |
| `prs:small` | 0–100 pull requests | analyzer | always (one of four) |
| `prs:medium` | 101–500 pull requests | analyzer | always |
| `prs:large` | 501–5000 pull requests | analyzer | always |
| `prs:xlarge` | 5000+ pull requests | analyzer | always |
| `refs:high` | More than 5000 branches — metadata import may time out | analyzer | branch count > 5000 |
| `tags:high` | More than 5000 tags — metadata import may time out | analyzer | tag count > 5000 |
| `commits:huge` | More than 250k commits — archive generation may exceed export window | analyzer | commit count > 250000 |
| `releases:oversized` | Releases exceed 30 GB — likely to hit GEI's 40 GB metadata-archive cap; consider `cfg:skip-releases` | analyzer | GitHub source only, releases payload > 30 GB |
| `meta:oversized` | Metadata archive exceeded 40 GB cap during migration — retry with `cfg:skip-releases` | advisor | observed mid-migration, not analyze-time |

## Age (`age:*`, `stale`)

Last commit age. One `age:*` label is always present after analyze; `stale` is an additional flag.

| Label | Meaning | Applied by | When |
|---|---|---|---|
| `age:<1mo` | Last commit < 1 month ago | analyzer | always (one of four) |
| `age:1-6mo` | Last commit 1–6 months ago | analyzer | always |
| `age:6-12mo` | Last commit 6–12 months ago | analyzer | always |
| `age:>1yr` | Last commit > 1 year ago | analyzer | always |
| `stale` | No commits in > 1 year | analyzer | additive — sits alongside `age:>1yr` |

## Blob risk (`blob:*`)

Files in git history. `blob:error` is a **migration blocker**.

| Label | Meaning | Applied by | When |
|---|---|---|---|
| `blob:ok` | No large blob issues | analyzer | no blob > 100 MB found |
| `blob:warning` | Blobs > 100 MB detected | analyzer | files between 100–375 MB exist in history |
| `blob:error` | Blobs > 375 MB — **migration blocker** | analyzer | files > 375 MB exist in history; `/migrate` pre-flight refuses |

## LFS health, hygiene, and routing

Set by [amp/analyze/lfs.py](../amp/analyze/lfs.py). The deep LFS check spans every branch and tag, not just HEAD.

### Health (mutually-exclusive state)

| Label | Meaning | Applied by | When |
|---|---|---|---|
| `lfs:none` | No LFS involvement ever | analyzer | no `.gitattributes` LFS rules and no pointer blobs |
| `lfs:active` | LFS currently in use and healthy | analyzer | active LFS at HEAD and Batch API serves all OIDs |
| `lfs:historical` | LFS was configured but no longer active | analyzer | LFS pointers in history but not at HEAD |
| `lfs:broken` | LFS pointers exist but objects missing/corrupt — **blocker** | analyzer | Batch API can't resolve pointer OIDs |
| `lfs:oversized` | Files should be LFS-tracked but aren't | analyzer | raw blobs above the LFS threshold for which no `.gitattributes` rule matches |

### Hygiene (orthogonal warning flags)

| Label | Meaning | Applied by | When |
|---|---|---|---|
| `lfs:orphan-pointers` | LFS pointers for paths no `.gitattributes` rule covers; a later edit can re-store the file raw | analyzer | pointer blob at a path with no matching rule |
| `lfs:inconsistent` | Same path is an LFS pointer on some refs and a raw blob on others; split LFS state across refs | analyzer | path differs between refs |
| `lfs:attributes-invalid` | `.gitattributes` LFS rule is malformed (e.g. brace expansion `{*.tgz}`) and silently does nothing | analyzer | `.gitattributes` parser rejected a rule git would silently ignore |

### Size routing

| Label | Meaning | Applied by | When |
|---|---|---|---|
| `lfs:large` | LFS payload > `AMP_LFS_LARGE_THRESHOLD_BYTES` (5 GiB default); routes to `MIGRATION_WORKER_LARGE` runner | analyzer | total LFS payload across history exceeds threshold |

### Destination-side gap (post-push verifier + advisor)

| Label | Meaning | Applied by | When |
|---|---|---|---|
| `lfs:dest-missing` | Destination LFS missing source objects after push | verifier | post-push Batch API confirms gap despite `git lfs push` exit 0 |
| `lfs:source-missing` | Source LFS server cannot serve referenced OIDs; **data-loss risk**, blocker | analyzer / verifier | source Batch API returns 404 for referenced OIDs |
| `lfs:dest-auth` | Destination LFS rejected PAT/App (HTTP 401/403) on upload probe — verify `Contents: write` scope | advisor | repair probe got 401/403 |
| `lfs:dest-saml` | Destination org is SAML-gated and the PAT is not authorized — re-authorize the PAT for that org | advisor | repair probe surfaced SAML gate |
| `lfs:dest-quota` | Destination LFS quota exceeded (HTTP 507 / insufficient_storage) — contact admin to raise org quota | advisor | repair probe returned 507 |

## Repo attributes

Facts about the source repo, surfaced at analyze time so operators know what won't migrate as-is.

| Label | Meaning | Applied by | When |
|---|---|---|---|
| `submodules` | Repo has git submodules | analyzer | `.gitmodules` present |
| `submodules:rewrite-needed` | ≥1 submodule URL points at the source SCM host and will need updating post-migration | analyzer | at least one submodule URL matches the source host |
| `webhooks` | Repo has active webhooks | analyzer | source API returned ≥1 webhook |
| `actions:ok` | `.github/workflows` present; no cross-org `uses:` references found | analyzer | workflow files exist, all `uses:` are marketplace |
| `actions:rewrite-needed` | `.github/workflows` references reusable workflows/actions in source-org repos that need path rewriting | analyzer | cross-org `uses:` reference detected |
| `actions:none` | No `.github/workflows` directory | analyzer | no workflow files |
| `codeowners:rewrite-needed` | CODEOWNERS contains `@org/team` mentions that may not resolve on the destination | analyzer | `@org/team-slug` references found |
| `codeowners:none` | No CODEOWNERS file (or empty/no team mentions) | analyzer | no CODEOWNERS or empty |
| `secrets:present` | Workflow files reference secrets/vars/environments — these do **not** migrate | analyzer | `secrets.X` / `vars.X` / `environment:` referenced in workflows |
| `hardcoded-urls:present` | Tracked files contain hardcoded references to the source SCM host | analyzer | only when `amp analyze --scan-hardcoded-urls` is passed (opt-in, +1–30s per repo) |
| `branch-protection:present` | Source has classic branch protection rules; AMP recreates them on `/dryrun` and `/migrate` | analyzer | GitHub source only — protection rules detected |
| `custom-properties:present` | Source repo has custom repository properties set (carried through on `/migrate`) | analyzer | GitHub source only — properties exist |
| `ref:long` | Ref names exceeding GitHub limits | analyzer | any ref name > limit |
| `ref:invalid-chars` | Ref names containing characters rejected by `git check-ref-format` | analyzer | unsafe ref name detected |
| `ref:case-collision` | Ref names differing only by case (break on case-sensitive filesystems) | analyzer | two refs differ only in case |
| `ref:path-conflict` | Ref names where a parent ref collides with a child path | analyzer | refs/heads/foo + refs/heads/foo/bar |

## Duration (`duration:*`)

GEI wall-clock time, in 5-minute buckets. Useful for dashboard / planning future runs.

| Label | Meaning | Applied by | When |
|---|---|---|---|
| `duration:0-5m` … `duration:55-60m` | GEI run took N–N+5 minutes | migration-worker | end of each migrate / dryrun run |
| `duration:60m+` | GEI run exceeded 60 minutes (near kill-timer) | migration-worker | escalate or split the migration |

## Command triggers (`cmd:*`)

**Applying a `cmd:*` label is how you tell AMP to do something.** The IssueOps bridge ([.github/workflows/label-ops.yml](../.github/workflows/label-ops.yml)) translates the label to the equivalent `/command` comment, which fires the handler in [.github/workflows/issue-ops.yml](../.github/workflows/issue-ops.yml). Labels are bulk-editable (UI / CSV / Excel); comments are not.

| Label | Equivalent command | What it does |
|---|---|---|
| `cmd:dryrun` | `/dryrun` | Dry run migration to sandbox org |
| `cmd:migrate` | `/migrate` | Production migration via GEI (requires `dst-org:`) |
| `cmd:migrate-with-submodules` | `/migrate-with-submodules` | Queue `/migrate` for this repo and each AMP-managed direct submodule |
| `cmd:remigrate` | `/remigrate` | Delete target and re-migrate |
| `cmd:archive` | `/archive` | Archive/lock source repo |
| `cmd:unarchive` | `/unarchive` | Reverse archive: unlock source repo |
| `cmd:set-properties` | `/set-properties --props '{}'` | Set custom properties on target |
| `cmd:update-issue` | `/update-issue` | Re-run analysis and refresh issue |
| `cmd:upsert-teams` | `/upsert-teams` | Upsert `dst-team:*` read/write/admin teams and attach repo at destination (idempotent) |
| `cmd:upsert-secrets` | `/upsert-secrets` | Upsert placeholder stubs for workflow secrets/variables at destination |
| `cmd:upsert-envs` | `/upsert-envs` | Upsert source-side Actions environments at destination |
| `cmd:upsert-protection` | `/upsert-protection` | Upsert source classic branch protection rules at destination |
| `cmd:upsert-properties` | `/upsert-properties` | Upsert source custom properties at destination |
| `cmd:upsert-fixes` | `/upsert-fixes` | Re-push `amp/migration-fixes` branch (submodule / actions / codeowners rewrites) |
| `cmd:upsert-validate` | `/upsert-validate` | Re-run branch/tag/commit count validation against destination |
| `cmd-pending:remigrate` | *(internal)* | Dispatcher should run `/remigrate` (not `/migrate`) when claiming this queued issue |

## Config flags (`cfg:*`)

Operator-set knobs that change how the next migration runs.

| Label | Meaning | Applied by | When |
|---|---|---|---|
| `cfg:skip-releases` | Pass `--skip-releases` to `gh gei` (avoid 40 GB metadata-archive cap from bulky release binaries) | operator | apply when `releases:oversized` or after `meta:oversized` failure |

## AI advisor (`cp-*`)

Tied to the most recent failed attempt. Starting a new `/dryrun`, `/migrate`, or `/remigrate` clears existing `cp-repair-*` so the project board never shows a stale repair while a fresh attempt is in flight.

| Label | Meaning | Applied by | When |
|---|---|---|---|
| `cp-failreason-<slug>` | AI-derived root cause (e.g. `cp-failreason-expired-pat`) | advisor | after any failed dryrun/migrate, tier-1 or tier-2 |
| `cp-repair-available` | AI repair proposal available for review | advisor | tier-2 plan produced but not auto-applicable |
| `cp-repair-applied` | AI repair auto-applied to a `-fixed` companion repo, ready for retry | repair handler | auto-applicable fix pushed |

## Post-migration: fixes branch (`migration-fixes:*`)

Outcome of pushing `amp/migration-fixes` to the destination after `/dryrun` or `/migrate`. Mutually exclusive within itself.

| Label | Meaning | Applied by | When |
|---|---|---|---|
| `migration-fixes:none` | No rewrites needed | post-migrate handler | analyzer flagged nothing in this category |
| `migration-fixes:pushed` | `amp/migration-fixes` branch pushed to destination | post-migrate handler | branch + PR (best-effort) succeeded |
| `migration-fixes:failed` | Branch push failed — see issue comment | post-migrate handler | Git Data API write failed |

## Post-migration: custom properties (`custom-properties:*` outcomes)

The analyze-time `custom-properties:present` flag is separate (it's an *attribute*, not an outcome). Outcomes below are mutually exclusive within themselves.

| Label | Meaning | Applied by | When |
|---|---|---|---|
| `custom-properties:applied` | Property values applied on destination repo | post-migrate handler | dest org already had the schema |
| `custom-properties:created+applied` | Schema created at destination org level, then values applied on the repo | post-migrate handler | 422 retry path succeeded |
| `custom-properties:dest-not-set` | Custom properties not applied at destination (permissions/schema); migration still succeeded | post-migrate handler | permissions/API error |

## Post-migration: branch protection at destination (`branch-protection:dest-*`)

Outcome of recreating classic rules at the destination. Best-effort — failures never roll back the migration. Cross-tenant push restriction actors are stripped on retry.

| Label | Meaning | Applied by | When |
|---|---|---|---|
| `branch-protection:dest-applied` | All classic branch protection rules recreated at the destination | post-migrate handler | every rule replayed cleanly |
| `branch-protection:dest-partial` | Some rules applied, others skipped or failed | post-migrate handler | mixed outcome — see issue comment |
| `branch-protection:dest-failed` | No rules applied at destination (permissions/API error); migration still succeeded | post-migrate handler | total failure |
| `branch-protection:dest-skipped` | Recreate skipped (no matching branches, or source rules unavailable) | post-migrate handler | nothing to do |

## Post-migration: secrets / variables recreate (`secrets-recreate:*`)

Outcome of upserting **placeholder stubs** for workflow secrets/variables at the destination. Values are not migrated — the owner fills them in.

| Label | Meaning | Applied by | When |
|---|---|---|---|
| `secrets-recreate:none` | No repo/env secrets or variables to recreate at destination | post-migrate handler | source had none |
| `secrets-recreate:created` | All stubs created at destination (placeholders) | post-migrate handler | full success |
| `secrets-recreate:partial` | Some stubs created; others skipped (already existed) or failed | post-migrate handler | mixed outcome |
| `secrets-recreate:failed` | Could not create stubs at destination (permissions/auth) | post-migrate handler | total failure |
| `secrets-recreate:skipped` | All secrets/variables already existed at destination — nothing created (idempotent) | post-migrate handler | full skip |
| `secrets-recreate:rate-limit-deferred` | GitHub API rate-limit too low to safely create stubs; re-trigger via `cmd:upsert-secrets` later | post-migrate handler | rate-limit guard |

## Post-migration: environments (`environments:*`)

| Label | Meaning | Applied by | When |
|---|---|---|---|
| `environments:source-present` | Source repo has GitHub Actions Environments — recreated at destination on `/dryrun` and `/migrate` | analyzer | GitHub source only — environments detected |
| `environments:source-none` | Source repo has no Actions Environments | analyzer | GitHub source, none found |
| `environments:none` | No environments to recreate at destination | post-migrate handler | source had none |
| `environments:created` | All environments + branch policies created at destination (reviewers stripped if unresolvable) | post-migrate handler | full success |
| `environments:partial` | Some environments created; others skipped (already existed) or failed | post-migrate handler | mixed outcome |
| `environments:failed` | Could not create environments at destination (permissions/auth) | post-migrate handler | total failure |
| `environments:rate-limit-deferred` | GitHub API rate-limit too low to safely create environments; re-trigger via `cmd:upsert-envs` later | post-migrate handler | rate-limit guard |

## Post-migration: "done" flags (`upserted:*`)

Short-form green flags stamped on success by each `/upsert-*` step. Scan the `upserted:*` set on an issue to see at a glance which post-migrate steps completed. **The detailed families above remain the source of truth for failure / partial / rate-limited states.**

| Label | Means the named step completed |
|---|---|
| `upserted:teams` | `/upsert-teams` — `dst-team:*` teams created/attached at destination |
| `upserted:secrets` | `/upsert-secrets` — at least one secret/variable stub created at destination |
| `upserted:envs` | `/upsert-envs` — at least one environment created at destination |
| `upserted:protection` | `/upsert-protection` — at least one branch protection rule applied at destination |
| `upserted:properties` | `/upsert-properties` — source custom properties applied at destination |
| `upserted:validate` | `/upsert-validate` — destination branch/tag/commit counts match the source |
| `upserted:submodules` | `/upsert-fixes` — submodule URL rewrites pushed in `amp/migration-fixes` |
| `upserted:workflows` | `/upsert-fixes` — workflow `uses:` rewrites pushed in `amp/migration-fixes` |
| `upserted:codeowners` | `/upsert-fixes` — CODEOWNERS team rewrites pushed in `amp/migration-fixes` |

## Org-wide FYI

| Label | Meaning | Applied by | When |
|---|---|---|---|
| `org-secrets:source-present` | Source org has Actions secrets/variables — FYI only, AMP does not write at org scope | analyzer | recreate manually post-migration |

## Dynamic families (per-repo values)

These aren't enumerated in `labels.yml` — they're created on demand by `amp analyze` (defaults) and `amp bulk-assign` (auto-create). Slug rules below.

| Family | Format | Meaning | Applied by |
|---|---|---|---|
| `namespace:<slug>` | one per issue | Unified source grouping (ADO project / BB project / GL group / GH org) | analyzer |
| `src-org:<slug>` | one per issue | Top-level source identifier (matters for ADO where org ≠ project) | analyzer |
| `src-team:<slug>` | multi-valued | Source-side team/group attached to the repo | analyzer |
| `dst-org:<slug>` | one per issue | Target GitHub org — **required** for `/migrate` | operator (defaults from mapping file) |
| `dst-team:<base>` | multi-valued | Base team name; `/upsert-teams` creates `<base>-read`/`-write`/`-admin` | operator |
| `dst-prefix:<slug>` | optional | Prepended to source repo name to form target repo name (`<dst-prefix>-<source_repo>`) | operator |

# Migration pre-flight blockers

`/migrate` (and `/dryrun`, `/remigrate`) refuse to run if any of these labels are present:

- `blob:error` — file > 375 MB in history. Fix the source (BFG / `git filter-repo`) or split the repo.
- `lfs:broken` — pointers exist but the LFS server can't serve the objects. Restore the LFS store.
- `lfs:source-missing` — source LFS server cannot serve referenced OIDs. Data-loss risk; cannot migrate what doesn't exist.
- `status:empty` — repo has no commits / no default branch. Nothing to migrate. Set manually if you disagree with the analyzer.

Warnings (won't block, but flagged in the pre-flight comment):
- Missing `dst-org:` label → `/migrate` refuses.
- No prior `status:dryrun-success` → `/migrate` warns but proceeds.

# Cross-references

- [README.md — Label Reference](../README.md#label-reference) — same data in a different shape, plus the IssueOps command table.
- [config/labels.yml](../config/labels.yml) — source of truth for label name, color, description. Run `amp setup-labels --repo <owner>/<control-repo>` after editing.
- [amp/labels.py](../amp/labels.py) — `STATUS_STAGES`, `POST_MIGRATE_STAGES`, `PRESERVE_PREFIXES_DEFAULT`, `set_status_label`, `set_post_migrate_label`, `stamp_upserted`.
- [amp/analyze/base.py](../amp/analyze/base.py) — size / PR / commit thresholds.
- [AMP_User_Guide.md](AMP_User_Guide.md) — long-form operator guide.
