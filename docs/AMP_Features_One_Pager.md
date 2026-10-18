# AMP — Agentic Migration Platform

**IssueOps-driven, multi-SCM migration to GitHub Enterprise Cloud.**
Built on top of `gh gei` / `ado2gh` / `bbs2gh` — does everything those tools don't.

---

## What `gh gei` does vs. what AMP adds

*Estimates assume a moderately experienced GitHub admin doing the work manually. Real numbers vary by org size, repo complexity, and on-call seniority. **Time saved is conservative.***

| Action | GEI | AMP | Time saved | Unit |
|---|:-:|:-:|---|---|
| Discover all repos across an org / project / group | ✗ | ✓ | 1–2 hrs | per source org |
| Detect oversized blobs (≥400 MB, where migration hard-fails) pre-flight | ✗ | ✓ | 30 min | per repo that would have failed |
| Detect long ref names (>255 chars) pre-flight | ✗ | ✓ | 15 min | per repo that would have failed |
| Deep LFS health check across every ref — orphan pointers, pointer/raw inconsistency, malformed .gitattributes, large raw blobs, source-object existence (Batch API), pointer fsck, server reachability | ✗ | ✓ | 15–45 min | per LFS repo |
| Pre-flight LFS disk-headroom check on runner | ✗ | ✓ | 30 min | per runner OOM avoided |
| List org-level Actions secrets / vars for operator awareness | ✗ | ✓ | 5–10 min | per source org |
| Scan tracked files (excluding `node_modules`, `vendor`, `.gitmodules`) for hardcoded source-SCM URLs (opt-in `--scan-hardcoded-urls`) | ✗ | ✓ | 15–45 min | per repo with hits |
| Pre-migrate hard-block check (blob / LFS / empty repo) | ✗ | ✓ | 5–8 min | per failed dry-run avoided |
| Pre-flight PAT validation against source + destination tenants before claiming the issue, flipping `status:*-running`, cloning the source baseline, or invoking `gh gei` (names the exact env var to refresh on 401/403) | ✗ | ✓ | 1–2 min | per stale-PAT attempt avoided |
| Capacity-aware migration queue (FIFO, respects GEI concurrency cap) | ✗ | ✓ | 30 min | per batch of >100 repos (avoids burning Actions minutes on idle-blocked workflows) |
| One-shot migrate of a repo and every AMP-managed direct submodule via `/migrate-with-submodules`, with a single aggregated status comment on the parent issue that auto-updates as each child finishes (categorised skip reasons; already-migrated siblings reported as no-action-needed with destination repo age) | ✗ | ✓ | 5–15 min | per parent repo with submodules |
| One-shot migrate of a repo and its **full recursive submodule tree** — children, grandchildren and deeper — via `/migrate-submodules-deep`; cycle-safe and de-duplicated, with the same aggregated parent status comment as the direct-submodule command | ✗ | ✓ | 5–15 min | per parent repo with a multi-level submodule tree |
| Reconcile stuck-running migrations (>2 h, no live worker → re-queued) | ✗ | ✓ | 15–30 min | per stuck workflow |
| Operator-workstation fallback: same migrate / dryrun / remigrate / migrate-with-submodules / migrate-submodules-deep logic runnable from a Mac / Linux / Windows laptop via `amp run` when a hosted runner can't finish (LFS disk, 6 h timeout, queue saturation); `migrate-with-submodules` migrates a parent and its direct submodules and `migrate-submodules-deep` its full recursive tree, locally in one pass | ✗ | ✓ | 1–4 hrs | per stuck migration recovered without CI churn |
| Post-migration LFS verification — object existence + auto-push, plus .gitattributes / pointer / raw-blob structure re-checks | ✗ | ✓ | 30 min | per LFS repo (silent gap caught) |
| Post-migration branch / tag / commit / first+last SHA validation against pre-GEI source baseline (immune to source drift during migration) | ✗ | ✓ | 5–15 min | per repo |
| Source freeze advisory — flag commits / branches added to source during the migration window | ✗ | ✓ | 5–15 min | per repo where team didn't honor freeze |
| Discover and carry custom repository properties to destination (creates schema if missing) | ✗ | ✓ | 5–10 min | per property per repo |
| Auto-attach `<base>-read` / `-write` / `-admin` teams (EMU/SCIM fallback) | ✗ | ✓ | 5–10 min | per migrated repo |
| Inventory workflow secrets / env vars / environments (scope-aware: repo vs env) and auto-create placeholder stubs at destination (encrypted, idempotent — never overwrites existing values) | ✗ | ✓ | 5–10 min | per secret / variable per repo |
| Discover and recreate source GitHub Actions Environments at destination (wait timer, reviewers, branch policies, prevent self-review, env-scoped secret/variable names, third-party deployment protection rule app slugs; reviewer-strip fallback on 422) | ✗ | ✓ | 10–30 min | per environment |
| Discover and recreate classic branch protection rules at destination on `/dryrun` and `/migrate` (cross-tenant push-restriction strip-and-retry; best-effort, never blocks) | ✗ | ✓ | 10–20 min | per protected branch |
| Manual re-trigger every post-migrate step individually via `/upsert-teams` / `/upsert-secrets` / `/upsert-envs` / `/upsert-protection` / `/upsert-properties` / `/upsert-fixes` / `/upsert-validate` (idempotent — safe to spam after fixing transient failures or sibling-repo gaps) | ✗ | ✓ | 10–20 min | per affected repo |
| Detect and rewrite submodule URLs on `amp/migration-fixes` branch (points at new destination) | ✗ | ✓ | 10–30 min | per repo with submodules |
| Detect and rewrite cross-repo workflow `uses:` action refs on `amp/migration-fixes` branch | ✗ | ✓ | 10–20 min | per workflow file |
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
| Provider-aware archive (ADO disable, BBS read-only, GitLab archive) | partial | ✓ | 5–10 min | per source repo |
| GHEC-DR (Data Residency) targeting (PyGithub + GraphQL + GEI calls) | partial | ✓ | 4–8 hrs | per engagement |
| Migration duration labels for forecasting (`duration:5-10m`, color-coded) | ✗ | ✓ | 30–60 min | per batch-window plan |

> *Partial:* GEI archives GitHub source repos but can't disable ADO repos, flip Bitbucket Server repos read-only, or archive GitLab projects. GEI supports GHEC-DR as a destination but doesn't reroute post-migration API calls, validation, or Projects v2 sync — AMP does.

**On a typical 500-repo engagement with ~10% LFS and ~20% requiring some remediation, AMP's automation conservatively saves the migration team 4–6 weeks of human effort versus running GEI alone.**

---

## Time-saving features by lifecycle stage

**Discover & analyze — *catch failures weeks before the migration window***

- One `amp analyze` command across ADO / Bitbucket / GitHub / GitLab produces a complete pre-flight risk inventory: 50+ auto-applied labels covering size, PRs, LFS health, blob size, ref names, secrets (scope-aware: repo vs environment), workflow variables, GitHub Actions Environments (with full protection rules **plus env-scoped secret/variable names and any third-party deployment protection rule app slugs**), org-level secrets/vars, CODEOWNERS, submodules, action refs, branch protection, and custom properties.
- Findings land on the AMP issue immediately, so repo owners and the migration team see what to remediate *before* the cutover window — not during it.
- Blockers (`blob:error`, `lfs:broken`, `status:empty`) prevent doomed migrations from being launched at all.
- Idempotent refresh keeps the backlog clean; obsolete repos get marked when they disappear from the source.
- Source-side team / group discovery emits one `src-team:<slug>` label per team attached to each repo (informational routing metadata; feeds the Kanban Team field). On by default; opt out with the repo variable `AMP_DISABLE_SOURCE_TEAMS=true` (or `--no-read-source-teams` on a one-off CLI run) when a project-scoped source stacks enough teams to push past GitHub's 100-label-per-issue cap. Analyze logs a warning at ≥80 discovered teams so you can flip the switch before the next full run wastes runner minutes.

**Status tracking — *the single most-asked-for number, automated***

- "How many done, how many left, how many blocked, what's running right now?" — answered automatically and continuously. Labels are the source of truth; the Kanban board, the daily report, and the Excel dashboard all derive from them.
- Migration team and client management both pull from the same surface — no weekly status meeting to compile, no stale spreadsheet circulating.
- Per-stage labels (`status:analyzed` → `status:dryrun-*` → `status:migrate-*` → `status:archived`) make every transition observable and reportable.
- Duration labels make batch-window forecasting concrete instead of guesswork.

**Route & schedule**

- Label-driven routing (`dst-org:`, `dst-team:`, `dst-prefix:`) drives destination assignment. Apply individually in the GitHub UI, in bulk via Excel, or at scale via `amp bulk-assign --csv` (concurrent, idempotent, with a `.failures.csv` for targeted retry).
- Capacity-aware FIFO queue respects GitHub's per-org concurrency cap — queued issues consume zero Action minutes until claimed.
- Drag a card on the Kanban into Dry Run / Migrate / Archive to trigger the operation.

**Migrate**

- IssueOps commands (`/dryrun`, `/migrate`, `/remigrate`) work via comment (single issue) *or* label (bulk-friendly).
- `/migrate-with-submodules` queues `/migrate` for a parent and each AMP-managed direct submodule in one action; the parent issue ends up with one rolling status comment showing per-submodule outcome (✅ / ❌ / ⏳ / ⏭) that each child worker updates in place as it finishes. Skipped siblings get a categorised reason (`no matching AMP issue`, `ambiguous`, `self-reference`, `unparseable URL`, `label-error`) and already-migrated siblings show the destination repo's age, so operators can answer "why did only 3 of 5 migrate?" without clicking into anything.
- `/migrate-submodules-deep` does the same but walks the **full recursive submodule tree** (children, grandchildren, and deeper), queuing `/migrate` for every AMP-managed descendant in one action. The traversal is cycle-safe and de-duplicated (a submodule cycle, or a repo shared by two parents, is queued once) and adds no extra API calls beyond the single index scan.
- Pre-flight hard-block check refuses migrations against unsound source repos.
- Dry-run exercises the LFS push path before production, surfacing quota / auth failures early.

**Auto-fix & remediate post-migration — *GEI's biggest blind spot***

- `amp/migration-fixes` branch automatically rewrites three things GEI leaves broken: `.gitmodules` URLs, workflow `uses:` refs, and `CODEOWNERS` team mentions — all pointed at the new destination. No PR opened; owners review and merge.
- LFS objects verified server-side post-migration; missing objects are pushed and re-verified.
- Branch / tag / commit counts and first/last-commit SHAs validated against a refs-only **source baseline** captured at GEI start time (so a team commit mid-migration doesn't false-fail the run). A separate **Source freeze** advisory check (🟨) calls out when source moved during the migration window without blocking the result.
- Custom repository properties carried through, with schema auto-creation at the destination org if missing.
- Three role-based teams (`<base>-read` / `-write` / `-admin`) created and attached automatically. EMU / SCIM fallback for tenants where teams are pre-provisioned.
- GitHub Actions Environments recreated at the destination (wait timer, branch policies, prevent self-review, reviewers). 422 on missing reviewers? AMP retries without them and flags the env so operators can re-assign after SCIM provisions the principal.
- Classic branch protection rules recreated at the destination on `/dryrun` and `/migrate` from the analyze-time snapshot (required reviews, code-owner reviews, dismiss-stale, status checks, enforce admins, push restrictions, linear history, signed commits, lock branch). 422 on push-restriction actors that don't resolve at the destination tenant? AMP strips `restrictions` and retries — the rest of the rule still lands. Best-effort: failures stamp a `branch-protection:dest-*` label and surface in the consolidated comment but never roll back the migration.
- Placeholder secret / variable stubs created at the destination (encrypted with libsodium sealed-box, value `AMP_PLACEHOLDER_REPLACE_ME`). Idempotent: existing destination values are never overwritten. Manual re-trigger via the `/upsert-*` post-migrate command family (`/upsert-teams`, `/upsert-secrets`, `/upsert-envs`, `/upsert-protection`, `/upsert-properties`, `/upsert-fixes`, `/upsert-validate`) makes recovery from any single step safe to spam. Each step that completes stamps a short `upserted:<name>` label so an issue's done state is scannable at a glance.
- These post-migration write steps are **best-effort**: a step that doesn't fully land surfaces as a 🟧 warning and raises `status:has-warnings` — it never marks a verified migration `migrate-failed`. Only a content/structural mismatch (counts, SHAs, an unreachable destination) does. When `gh gei` carried the source's archived flag across, AMP temporarily unarchives the destination for these writes and restores it afterwards.
- A clean handoff issue opens on the new repo with an operator-stripped checklist for the human owner. When the GEI run produced warnings, the issue adds a "Migration warnings to review" section that explains each warning category in plain language and reassures owners they are almost always cosmetic.

**Agentic recovery — *failures self-diagnose and (where safe) self-heal***

- 3-tier advisor on every failed operation: deterministic rules first, then an LLM agent with a real tool-use loop (`search_logs`, `get_issue_details`, `check_repo_exists`, `check_ref_names`, `check_lfs_status`, `get_api_rate_limit`), then a heuristic fallback. 72 h cache prevents duplicate spend.
- Root cause labeled directly on the issue (`cp-failreason-<slug>`) — operators see *why* not just *that*.
- Auto-repair handlers fix the most common classes of failure: long ref names and LFS push gaps with full classification. A `repo-already-exists` failure is diagnosed but **not** auto-repaired — AMP never deletes an existing destination repo on its own; the operator resolves it with `/remigrate` or a different target name.

**Observe & report**

- Daily activity report posted to a tracking discussion post weekday afternoons (per-provider pivot, today's activity, completed migrations, failures requiring attention).
- Terminal dashboard (`amp dashboard`) for at-a-glance health.
- Excel export with conditional formatting — Blockers in red, Warnings in yellow — and a Summary sheet with pivot-style counts.
- Analytics workbook (`amp export-analytics`) — a filterable Data table, **native Excel PivotTables** (status × LFS/provider/size, primary blocker, LFS, age, source org × namespace), a **State Changes** sheet (one row per status/cmd transition with actor + elapsed), and an in-table command builder (Action/Command dropdowns → pasteable `gh` lines for just the filtered rows). State history is also available standalone via `amp state-history` / `scripts/state-changes.py`.
- Projects v2 Kanban with bidirectional sync; the "Blocked" view automatically surfaces every repo with a hard blocker.

---

## SCM coverage

| Capability | ADO | Bitbucket Server | Bitbucket Cloud | GitHub | GitLab |
|---|:-:|:-:|:-:|:-:|:-:|
| Org / group / project enumeration | ✓ | ✓ | ✓ | ✓ | ✓ |
| Repo migration engine | ado2gh | bbs2gh | bbc-exporter | gh gei | gl-exporter + GraphQL import |
| LFS deep analysis | ✓ | ✓ | ✓ | ✓ | ✓ |
| Migration-fixes branch (submodules / actions / CODEOWNERS) | ✓ | ✓ | ✓ | ✓ | ✓ |
| Post-migration validation | ✓ | ✓ | ✓ | ✓ | ✓ |
| AI failure advisor + auto-repair | ✓ | ✓ | ✓ | ✓ | ✓ |
| Branch protection discovery | — | — | — | ✓ | — |
| Branch protection recreate at destination (`/dryrun` + `/migrate`) | — | — | — | ✓ | — |
| Custom properties discovery + carry-through | — | — | — | ✓ | — |
| Secret / variable stub creation at destination (scope-aware) | — | — | — | ✓ | — |
| Source GitHub Actions Environments recreate at destination | — | — | — | ✓ | — |
| Provider-specific archive | disable | read-only | read-only | archive | archive |
| GHEC-DR destination | ✓ | ✓ | ✓ | ✓ | ✓ |
| Projects v2 sync (control plane lives on GitHub) | ✓ | ✓ | ✓ | ✓ | ✓ |

---

## Operations & scale

- Seven distinct token domains (source / destination / control-AMP / IssueOps / advisor / runner-bootstrap / distribution), each with a clear precedence chain — no surprise source-vs-destination token mix-ups.
- **Secure, revocable delivery** — AMP can ship as a Cython-compiled, platform-tagged wheel from a private release repo instead of source. Customers install it with a read-scoped token you revoke at engagement end, and each build carries an expiry that self-destructs even copied binaries; no AMP source lands in the customer environment, and access ends when the engagement does.
- Three independent GitHub tenants supported: source, destination, and control (where the AMP repo lives). The control repo typically aliases to one of the other two but can be on a third tenant entirely.
- Per-workflow runner selection via `vars.AMP_RUNNER_<NAME>` — self-hosted or GitHub-hosted, no workflow forks.
- GHEC-DR: one variable (`GH_DEST_API_URL`) reroutes destination call paths (GEI, PyGithub, GraphQL); `GH_AMP_API_URL` is the matching knob for the control-repo tenant.
- Migration queue concurrency tunable via `AMP_MIGRATE_CONCURRENCY`; opt-out only via the explicit literal `AMP_QUEUE_ENABLED=false`. The per-migration subprocess kill-timeout (default 1 h) is tunable via `AMP_MIGRATION_TIMEOUT_SECS` for large export+import runs.
- LFS storage volume + headroom factor tunable for huge-repo runs (`AMP_LFS_STORAGE_DIR`, `AMP_LFS_DISK_HEADROOM_FACTOR`).
- Cross-tenant destination auth via `DEST_APP_ID` / `DEST_APP_PRIVATE_KEY` GitHub App; matching `AMP_APP_ID` / `AMP_APP_PRIVATE_KEY` for cross-tenant control-repo auth.

---

**Extensibility** — AMP is built to be extended. If a customer needs a capability that isn't already in this list (additional SCM quirks, custom validation steps, bespoke reports, additional auto-repair patterns), it can be added on top of the existing plug-points. Best raised before or early in the engagement so it can be scoped; work done against custom requirements is billed to the engagement.

*AI-generated content disclaimer — This document was assembled with AI assistance from the AMP codebase. Specific behaviors, defaults, and thresholds may have shifted since publication; the authoritative source is the code in this repository.*

*Last updated: 2026-05-23*
