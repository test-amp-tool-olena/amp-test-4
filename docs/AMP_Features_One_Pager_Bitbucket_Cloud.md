# AMP for Bitbucket Cloud → GitHub Enterprise Cloud

**IssueOps-driven migration from Bitbucket Cloud (bitbucket.org) to GitHub Enterprise Cloud.**
GitHub's Enterprise Importer does not natively support Bitbucket Cloud as a source — without AMP, Bitbucket Cloud migrations rely on per-repo scripts, manual exports, and bespoke remediation. AMP closes that gap end-to-end.

---

## What native GEI tooling does vs. what AMP adds for Bitbucket Cloud → GitHub

*Estimates assume a moderately experienced GitHub admin doing the work manually. Real numbers vary by org size, repo complexity, and on-call seniority. **Time saved is conservative.***

| Action | Native GEI | AMP | Time saved | Unit |
|---|:-:|:-:|---|---|
| First-class Bitbucket Cloud source support — `gh bbc-exporter migrate` exports the repo and imports it to GitHub in one pass | ✗ (no `bbc2gh`) | ✓ | — | — |
| Discover all repos across a Bitbucket Cloud workspace | ✗ | ✓ | 1–2 hrs | per workspace |
| Atlassian scoped-API-token support via `BITBUCKET_API_TOKEN` + `BITBUCKET_EMAIL` (app passwords retire 2026-06-09 and are intentionally not wired up) | ✗ | ✓ | 30–60 min | per engagement |
| UUID-stable repo identity (survives renames) vs. slug-based API churn | ✗ | ✓ | 30–60 min | per engagement |
| Detect oversized blobs (≥400 MB, where migration hard-fails) pre-flight | ✗ | ✓ | 30 min – 2 hrs | per repo that would have failed |
| Detect long ref names (>255 chars) pre-flight | ✗ | ✓ | 15–30 min | per repo that would have failed |
| Deep LFS health check across every ref — orphan pointers, pointer/raw inconsistency, malformed .gitattributes, large raw blobs, source-object existence (Batch API), pointer fsck, server reachability | ✗ | ✓ | 15–45 min | per LFS repo |
| Pre-flight LFS disk-headroom check on runner | ✗ | ✓ | 30–60 min | per runner OOM avoided |
| Discover workspace + repo-tier group permissions (cascade) | ✗ | ✓ | 10–20 min | per repo with group grants |
| Scan tracked files (excluding `node_modules`, `vendor`, `.gitmodules`) for hardcoded `bitbucket.org` URLs (opt-in `--scan-hardcoded-urls`) | ✗ | ✓ | 15–45 min | per repo with hits |
| Rate-limit-paced API access (`BBC_REQUEST_INTERVAL`) | ✗ | ✓ | 30–90 min | per engagement (avoids throttling stalls) |
| Pre-migrate hard-block check (blob / LFS / empty repo) | ✗ | ✓ | 5–8 min | per failed dry-run avoided |
| Pre-flight token validation against source (Bitbucket Cloud) + destination tenants before claiming the issue, flipping `status:*-running`, cloning the source baseline, or invoking `gh bbc` (names the exact env var to refresh on 401/403; flags missing `BITBUCKET_EMAIL`) | ✗ | ✓ | 1–2 min | per stale-token attempt avoided |
| Capacity-aware migration queue (FIFO, respects GitHub concurrency cap) | ✗ | ✓ | 30 min | per batch of >100 repos (avoids burning Actions minutes on idle-blocked workflows) |
| One-shot migrate of a repo and every AMP-managed direct submodule via `/migrate-with-submodules`, with a single aggregated status comment on the parent issue that auto-updates as each child finishes (categorised skip reasons; already-migrated siblings reported as no-action-needed with destination repo age) | ✗ | ✓ | 5–15 min | per parent repo with submodules |
| One-shot migrate of a repo and its **full recursive submodule tree** — children, grandchildren and deeper — via `/migrate-submodules-deep`; cycle-safe and de-duplicated, with the same aggregated parent status comment as the direct-submodule command | ✗ | ✓ | 5–15 min | per parent repo with a multi-level submodule tree |
| Reconcile stuck-running migrations (>2 h, no live worker → re-queued) | ✗ | ✓ | 15–30 min | per stuck workflow |
| Operator-workstation fallback: same migrate / dryrun / remigrate / migrate-with-submodules / migrate-submodules-deep logic runnable from a Mac / Linux / Windows laptop via `amp run` when a hosted runner can't finish (LFS disk, 6 h timeout, queue saturation); `migrate-with-submodules` migrates a parent and its direct submodules and `migrate-submodules-deep` its full recursive tree, locally in one pass | ✗ | ✓ | 1–4 hrs | per stuck migration recovered without CI churn |
| Post-migration LFS verification — object existence + auto-push, plus .gitattributes / pointer / raw-blob structure re-checks | ✗ | ✓ | 30 min | per LFS repo (silent gap caught) |
| Post-migration branch / tag / commit count validation | ✗ | ✓ | 5–15 min | per repo |
| Auto-attach `<base>-read` / `-write` / `-admin` teams (EMU/SCIM fallback) | ✗ | ✓ | 5–10 min | per migrated repo |
| Detect and rewrite submodule URLs on `amp/migration-fixes` branch (points at new destination) | ✗ | ✓ | 10–30 min | per repo with submodules |
| Detect and rewrite cross-repo workflow `uses:` action refs (Bitbucket Pipelines / GitHub Actions targets) on `amp/migration-fixes` branch | ✗ | ✓ | 10–20 min | per workflow file |
| Detect and rewrite `CODEOWNERS` team mentions on `amp/migration-fixes` branch | ✗ | ✓ | 10–20 min | per CODEOWNERS file |
| Classify GEI run warnings into plain-language explanations in the destination handoff issue; flag warned runs with a queryable `status:has-warnings` label + export column | ✗ | ✓ | 10–30 min | per repo with warnings |
| Treat best-effort post-migration write failures (branch protection, secret/variable stubs, custom properties, environments, migration-fixes) as warnings, not migration failures — the verified migration is never marked failed for them | ✗ | ✓ | 15–30 min | per repo with a partial post-migration step |
| Auto-diagnose failures (rules + LLM + heuristic, 72 h cache) | ✗ | ✓ | 15–45 min | per failure |
| Agentic root-cause analysis with tool-use loop (`search_logs`, `check_lfs_status`, …) | ✗ | ✓ | 30 min | per non-trivial failure |
| Auto-repair long ref names | ✗ | ✓ | 30 min | per affected repo |
| Auto-repair LFS push gaps (classified: source-missing / dest-auth / dest-saml / dest-quota) | ✗ | ✓ | 30 min – 5 hrs | per affected repo |
| Real-time migration status across all repos (labels → Kanban → daily report) | ✗ | ✓ | 1–2 hrs | per week (eliminates status meetings) |
| Daily progress report posted automatically to a tracking discussion post | ✗ | ✓ | 15–30 min | per day |
| Blockers / warnings rollups at a glance (Excel + Project "Blocked" view) | ✗ | ✓ | 30–60 min | per planning session |
| Bulk routing for thousands of repos via CSV (concurrent + idempotent); enables many-to-many migrations from many source orgs into one or more destination GitHub orgs | ✗ | ✓ | 2–4 hrs | per 1,000 repos |
| Excel export with risk rollups + command generation | ✗ | ✓ | 1–2 hrs | per planning cycle |
| Analytics workbook: native Excel pivots (status × LFS/provider/size, blockers, age, org) + in-table filter-then-run command builder | ✗ | ✓ | 2–4 hrs | per planning cycle |
| Per-repo state-change history (one row per status/cmd transition, with actor + elapsed) | ✗ | ✓ | 1–2 hrs | per audit / retrospective |
| Kanban Projects v2 sync (forward — label → board column) | ✗ | ✓ | 30–60 min | per week |
| Archive Bitbucket Cloud source on `/archive` (flips repo read-only) | ✗ | ✓ | 5–10 min | per source repo |
| GHEC-DR (Data Residency) targeting (PyGithub + GraphQL + GEI calls) | ✗ | ✓ | 4–8 hrs | per engagement |
| Migration duration labels for forecasting (`duration:5-10m`, color-coded) | ✗ | ✓ | 30–60 min | per batch-window plan |

**Because GEI provides no first-class Bitbucket Cloud path, AMP's value on these engagements is even larger than the per-row numbers suggest — on a typical 500-repo Bitbucket Cloud engagement with ~10% LFS and ~20% requiring some remediation, AMP conservatively saves 5–7 weeks of human effort versus a hand-rolled migration toolkit.**

---

## What AMP discovers in your Bitbucket Cloud workspace

- **Workspace-wide enumeration** using Atlassian scoped API tokens (`BITBUCKET_API_TOKEN` + `BITBUCKET_EMAIL`). App passwords retire 2026-06-09 and are intentionally not wired up; on free workspaces, the legacy unscoped "Create API token" button at id.atlassian.com produces a token that 401s against Bitbucket — only the "Create API token **with scopes**" path works (select `read:account`, `read:workspace:bitbucket`, `read:repository:bitbucket`, `read:pullrequest:bitbucket`, `read:webhook:bitbucket` at minimum).
- **UUID-stable repo identity** — repos that get renamed in Bitbucket Cloud during the engagement still match against the same AMP issue. Slug-only tooling loses the link on rename.
- **Two-tier group permission discovery** (workspace + repo) — Bitbucket Cloud permissions cascade from workspace to repo; AMP captures both layers.
- **Rate-limit-aware API access** — Bitbucket Cloud throttles aggressively; AMP paces requests via `BBC_REQUEST_INTERVAL` so long enumeration passes don't stall.
- **Provider-aware archive** — `cmd:archive` flips the Bitbucket Cloud repo read-only; `cmd:unarchive` reverses it.

---

## Time-saving features by lifecycle stage

**Discover & analyze — *catch failures weeks before the migration window***

- One `amp analyze` command across the whole Bitbucket Cloud workspace produces a complete pre-flight risk inventory: 50+ auto-applied labels covering size, PRs, LFS health, blob size, ref names, submodules, action refs, CODEOWNERS, group permissions, and hardcoded source URLs.
- Findings land on the AMP issue immediately, so repo owners and the migration team see what to remediate *before* the cutover window — not during it.
- Blockers (`blob:error`, `lfs:broken`, `status:empty`) prevent doomed migrations from being launched at all.
- Source-side workspace / repo group discovery emits one `src-team:<slug>` label per group attached to the repo (informational routing metadata; feeds the Kanban Team field). On by default; opt out with the repo variable `AMP_DISABLE_SOURCE_TEAMS=true` (or `--no-read-source-teams` on a one-off CLI run) when workspace-level groups would push per-issue label counts past GitHub's 100-label cap. Analyze logs a warning at ≥80 discovered groups per repo.

**Status tracking — *the single most-asked-for number, automated***

- "How many done, how many left, how many blocked, what's running right now?" — answered automatically and continuously. Labels are the source of truth; the Kanban board, daily report, and Excel dashboard all derive from them.
- Migration team and client management both pull from the same surface — no weekly status meeting to compile, no stale spreadsheet circulating.

**Route & schedule**

- Label-driven routing (`dst-org:`, `dst-team:`, `dst-prefix:`) drives destination assignment. Apply in bulk via Excel or `amp bulk-assign --csv` (concurrent, idempotent).
- Capacity-aware FIFO queue respects GitHub's per-org concurrency cap — queued issues consume zero Action minutes until claimed.

**Auto-fix & remediate post-migration — *the gap GEI never even addressed***

- `amp/migration-fixes` branch automatically rewrites `.gitmodules` URLs, workflow `uses:` refs, and `CODEOWNERS` team mentions — all pointed at the new destination.
- LFS objects verified server-side post-migration; missing objects are pushed and re-verified.
- Branch / tag / commit counts validated against the analyze-time snapshot.
- Three role-based teams (`<base>-read` / `-write` / `-admin`) created and attached automatically.

**Agentic recovery — *failures self-diagnose and (where safe) self-heal***

- 3-tier advisor on every failed operation: rules → LLM agent with tool-use loop → heuristic fallback. 72 h cache prevents duplicate spend.
- Root cause labeled directly on the issue (`cp-failreason-<slug>`).
- Auto-repair handlers fix long ref names and LFS push gaps. A `repo-already-exists` failure is diagnosed but not auto-repaired: AMP never deletes an existing destination repo on its own — resolve it with `/remigrate` or a different target name.

**Observe & report**

- Daily activity report posted to a tracking discussion post weekday afternoons.
- Terminal dashboard (`amp dashboard`) for at-a-glance health.
- Excel export with conditional formatting — Blockers in red, Warnings in yellow.
- Analytics workbook (`amp export-analytics`) — filterable Data table, native Excel PivotTables (status × LFS/provider/size, primary blocker, LFS, age, source org × namespace), a State Changes sheet (one row per status/cmd transition with actor + elapsed), and an in-table command builder. State history also standalone via `amp state-history`.

---

## Operations & scale

- Five distinct token domains (Bitbucket Cloud source / GitHub destination / control-AMP / IssueOps / advisor) with explicit precedence — no surprise mix-ups. Bitbucket Cloud auth uses `BITBUCKET_API_TOKEN` + `BITBUCKET_EMAIL`. The control (AMP) repo is reachable independently of source and destination credentials.
- **Secure, revocable delivery** — AMP can ship as a Cython-compiled wheel from a private release repo instead of source; customers install with a read-scoped token you revoke at engagement end, and a built-in expiry self-destructs even copied binaries, so no AMP source lands in the customer environment and access ends when the engagement does.
- Per-workflow runner selection via `vars.AMP_RUNNER_<NAME>` — self-hosted or GitHub-hosted, no workflow forks.
- GHEC-DR: `GH_DEST_API_URL` reroutes destination call paths; `GH_AMP_API_URL` is the matching knob for the control-repo tenant. Bitbucket Cloud migrations support GHEC and GHEC-DR destinations only — GHES targets are rejected before the export starts.
- Destination repo visibility for Bitbucket Cloud migrations defaults to `private`; override per engagement via `BBC_TARGET_REPO_VISIBILITY` (`public` / `private` / `internal`).
- Migration queue concurrency tunable via `AMP_MIGRATE_CONCURRENCY`; the per-migration subprocess kill-timeout (default 1 h) is tunable via `AMP_MIGRATION_TIMEOUT_SECS` for large export+import runs.
- LFS storage volume + headroom factor tunable for huge-repo runs.

---

**Extensibility** — AMP is built to be extended. If you need a capability that isn't already in this list (additional Bitbucket Cloud quirks, custom validation steps, bespoke reports, additional auto-repair patterns), it can be added on top of the existing plug-points. Best raised before or early in the engagement so it can be scoped; work done against custom requirements is billed to the engagement.

*AI-generated content disclaimer — This document was assembled with AI assistance from the AMP codebase. Specific behaviors, defaults, and thresholds may have shifted since publication; the authoritative source is the code in this repository.*

*Last updated: 2026-05-23*
