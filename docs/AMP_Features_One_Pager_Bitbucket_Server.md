# AMP for Bitbucket Data Center / Server → GitHub Enterprise Cloud

**IssueOps-driven migration from Bitbucket Server (Data Center) to GitHub Enterprise Cloud.**
Built on top of `gh bbs2gh` — does everything that tool doesn't.

---

## What `gh bbs2gh` does vs. what AMP adds for Bitbucket Server → GitHub

*Estimates assume a moderately experienced GitHub admin doing the work manually. Real numbers vary by org size, repo complexity, and on-call seniority. **Time saved is conservative.***

| Action | `bbs2gh` | AMP | Time saved | Unit |
|---|:-:|:-:|---|---|
| Discover all repos across all Bitbucket Server projects | ✗ | ✓ | 1–2 hrs | per source instance |
| Bearer-header auth fallback when Basic auth is disabled | ✗ | ✓ | 30–60 min | per engagement |
| Detect oversized blobs (≥400 MB, where migration hard-fails) pre-flight | ✗ | ✓ | 30 min – 2 hrs | per repo that would have failed |
| Detect long ref names (>255 chars) pre-flight | ✗ | ✓ | 15–30 min | per repo that would have failed |
| Deep LFS health check across every ref — orphan pointers, pointer/raw inconsistency, malformed .gitattributes, large raw blobs, source-object existence (Batch API), pointer fsck, server reachability | ✗ | ✓ | 15–45 min | per LFS repo |
| Pre-flight LFS disk-headroom check on runner | ✗ | ✓ | 30–60 min | per runner OOM avoided |
| Discover repo + project-level group permissions (two-tier cascade) | ✗ | ✓ | 10–20 min | per repo with group grants |
| Scan tracked files (excluding `node_modules`, `vendor`, `.gitmodules`) for hardcoded Bitbucket Server URLs (opt-in `--scan-hardcoded-urls`) | ✗ | ✓ | 15–45 min | per repo with hits |
| Pre-migrate hard-block check (blob / LFS / empty repo) | ✗ | ✓ | 5–8 min | per failed dry-run avoided |
| Pre-flight PAT validation against source (Bitbucket Server) + destination tenants before claiming the issue, flipping `status:*-running`, cloning the source baseline, or invoking `gh bbs2gh` (names the exact env var to refresh on 401/403) | ✗ | ✓ | 1–2 min | per stale-PAT attempt avoided |
| Capacity-aware migration queue (FIFO, respects GEI concurrency cap) | ✗ | ✓ | 30 min | per batch of >100 repos (avoids burning Actions minutes on idle-blocked workflows) |
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
| Archive Bitbucket Server source on `/archive` (flips repo read-only — BBS has no native archive) | ✗ | ✓ | 5–10 min | per source repo |
| GHEC-DR (Data Residency) targeting (PyGithub + GraphQL + GEI calls) | partial | ✓ | 4–8 hrs | per engagement |
| Migration duration labels for forecasting (`duration:5-10m`, color-coded) | ✗ | ✓ | 30–60 min | per batch-window plan |

> *Partial:* `bbs2gh` accepts a GHEC-DR target URL but doesn't reroute post-migration API calls, validation, Projects v2 sync, or downstream label-ops to the DR tenant — AMP does.

**On a typical 500-repo Bitbucket Server engagement with ~10% LFS and ~20% requiring some remediation, AMP's automation conservatively saves the migration team 4–6 weeks of human effort versus running `bbs2gh` alone.**

---

## What AMP discovers in your Bitbucket Server instance

- **Paginated project/repo enumeration** with Bearer-header auth fallback for instances where Basic auth is disabled (`bbs2gh` requires you to know this in advance).
- **Two-tier group permission discovery** — group grants exist at both the repo and project level in Bitbucket Server; AMP captures both layers so destination team assignment is correct.
- **Provider-aware archive** — `cmd:archive` flips the Bitbucket Server repo read-only (BBS has no native archive); `cmd:unarchive` reverses it.
- **LFS server probing against the Bitbucket-hosted LFS endpoint** — catches silent failures before migration starts.

---

## Time-saving features by lifecycle stage

**Discover & analyze — *catch failures weeks before the migration window***

- One `amp analyze` command across the whole Bitbucket Server instance produces a complete pre-flight risk inventory: 50+ auto-applied labels covering size, PRs, LFS health, blob size, ref names, submodules, action refs, CODEOWNERS, group permissions, and hardcoded source URLs.
- Findings land on the AMP issue immediately, so repo owners and the migration team see what to remediate *before* the cutover window — not during it.
- Blockers (`blob:error`, `lfs:broken`, `status:empty`) prevent doomed migrations from being launched at all.
- Source-side group discovery emits one `src-team:<slug>` label per group attached to the repo (informational routing metadata; feeds the Kanban Team field). On by default; opt out with the repo variable `AMP_DISABLE_SOURCE_TEAMS=true` (or `--no-read-source-teams` on a one-off CLI run) when project-level groups would push per-issue label counts past GitHub's 100-label cap. Analyze logs a warning at ≥80 discovered groups per repo.

**Status tracking — *the single most-asked-for number, automated***

- "How many done, how many left, how many blocked, what's running right now?" — answered automatically and continuously. Labels are the source of truth; the Kanban board, daily report, and Excel dashboard all derive from them.
- Migration team and client management both pull from the same surface — no weekly status meeting to compile, no stale spreadsheet circulating.

**Route & schedule**

- Label-driven routing (`dst-org:`, `dst-team:`, `dst-prefix:`) drives destination assignment. Apply in bulk via Excel or `amp bulk-assign --csv` (concurrent, idempotent).
- Capacity-aware FIFO queue respects GitHub's per-org concurrency cap — queued issues consume zero Action minutes until claimed.

**Auto-fix & remediate post-migration — *bbs2gh's biggest blind spot***

- `amp/migration-fixes` branch automatically rewrites `.gitmodules` URLs, workflow `uses:` refs, and `CODEOWNERS` team mentions — all pointed at the new destination.
- LFS objects verified server-side post-migration; missing objects are pushed and re-verified.
- Branch / tag / commit counts validated against the analyze-time snapshot.
- Three role-based teams (`<base>-read` / `-write` / `-admin`) created and attached automatically (EMU/SCIM fallback for tenants where teams are pre-provisioned).

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

- Five distinct token domains (BBS source / GitHub destination / control-AMP / IssueOps / advisor) with explicit precedence — no surprise mix-ups. The control (AMP) repo is reachable independently of source and destination credentials.
- **Secure, revocable delivery** — AMP can ship as a Cython-compiled wheel from a private release repo instead of source; customers install with a read-scoped token you revoke at engagement end, and a built-in expiry self-destructs even copied binaries, so no AMP source lands in the customer environment and access ends when the engagement does.
- Per-workflow runner selection via `vars.AMP_RUNNER_<NAME>` — self-hosted or GitHub-hosted, no workflow forks. Useful when your Bitbucket Server lives inside a private network.
- GHEC-DR: `GH_DEST_API_URL` reroutes destination call paths; `GH_AMP_API_URL` is the matching knob for the control-repo tenant.
- Migration queue concurrency tunable via `AMP_MIGRATE_CONCURRENCY`.
- LFS storage volume + headroom factor tunable for huge-repo runs.

---

**Extensibility** — AMP is built to be extended. If you need a capability that isn't already in this list (additional Bitbucket Server quirks, custom validation steps, bespoke reports, additional auto-repair patterns), it can be added on top of the existing plug-points. Best raised before or early in the engagement so it can be scoped; work done against custom requirements is billed to the engagement.

*AI-generated content disclaimer — This document was assembled with AI assistance from the AMP codebase. Specific behaviors, defaults, and thresholds may have shifted since publication; the authoritative source is the code in this repository.*

*Last updated: 2026-05-23*
