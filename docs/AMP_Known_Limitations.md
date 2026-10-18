% AMP — Known Limitations & Caveats

Things in AMP that are known to be **flaky, partial, opt-in, less-tested, or
best-effort**. If you hit one of these, it's expected behavior — not a bug you've
discovered. Reported as of 2026-05-31.

This is a quick "so you aren't surprised" reference. Where a topic is covered in
depth elsewhere, this page just flags it and links out.

## Source providers

- **GitLab is the least battle-tested provider.** Migration works and is exercised
  against a test bed, but it has had far less real-world mileage than ADO /
  Bitbucket / GitHub, and unit coverage is thin. A couple of analyzer checks fail
  *soft* rather than loud: `repo_exists` returns "inconclusive" (not true/false)
  on any non-404 API error, and webhook discovery swallows errors and returns an
  empty list (`amp/analyze/gitlab.py`). **Recommendation:** after a GitLab
  migration, sanity-check the destination branch/tag/commit counts yourself rather
  than trusting a clean run.

- **Bitbucket Cloud auth & limits** (`amp/analyze/bitbucket_cloud.py`):
  - App Passwords are **not** supported (Atlassian is retiring them 2026-06-09).
    Use an API token (`BITBUCKET_API_TOKEN` + `BITBUCKET_EMAIL`) or
    `BITBUCKET_ACCESS_TOKEN`.
  - The API allows ~1000 requests/hour. For large workspaces, set
    `BBC_REQUEST_INTERVAL` (seconds) to throttle and avoid 429 storms.
  - Reported PR counts are **bucketed, capped at 5500** — enough to place a repo
    in a size bucket, not an exact count.

## GitHub Projects v2

- **The board can't be "grouped by Status" programmatically — you set that once by
  hand.** AMP keeps each issue's `status:*` label in sync with the project's
  *Status field value*. But the board's **group-by / view layout** is not something
  the API will set for us: Projects v2 GraphQL has no `createProjectV2View`
  mutation, and the REST view endpoint AMP uses accepts `name`, `layout`, `filter`,
  and `visible_fields` but **not `group_by`** (`amp/projects/views.py`). Project
  *templates* don't reliably carry grouping either. So after setup, open the
  project in the UI and set **Group by → Status** once; it sticks from there.

- **`amp configure-project` is destructive — run it once, at setup.** GitHub's
  GraphQL `updateProjectV2Field` total-replaces the options on a single-select
  field (there's no non-destructive "add option"), so configuring the Status field
  wipes existing options. That's why the command requires `--force`
  (`amp/projects/manager.py`).

- **No drag-to-trigger.** Dragging a card to a new Status column does **not** fire
  a migration command. `projects_v2_item.edited` isn't supported on GitHub
  Enterprise tenants, so that reverse path was removed. Labels are the source of
  truth; the project's Status is derived from them and never read back.

## Post-migration fixes are best-effort (they never roll back a migration)

After a successful `/dryrun` or `/migrate`, AMP runs several follow-up steps. Each
is **best-effort**: on failure it stamps a warning label and comments on the issue,
but the migration itself still stands. If one of these is marked failed, the repo
*did* migrate — only the follow-up needs a look.

- **`amp/migration-fixes` branch + PR** (submodule/Actions/CODEOWNERS rewrites) —
  on failure stamps `migration-fixes:failed`. PR creation is separately
  best-effort; the branch can land even if the PR doesn't (`amp/migration_branch/`).
- **Validation** (destination vs. analyze-time branch/tag/commit counts) — stamps
  `status:validate-failed` on mismatch. The exact-SHA assertion is **skipped** when
  `lfs:*` labels indicate history was rewritten (`amp/validate.py`).
- **Custom-properties carry-through** — partial failures are recorded per-property
  and don't block the others; stamps `custom-properties:dest-not-set`
  (`amp/validate.py`).

## Teams (destination team creation is largely unverified)

- **The label/routing half works; the destination-side create-and-attach half has
  only been smoke-tested.** `src-team:` labels are recorded as informational source
  metadata, and `dst-team:<base>` labels are parsed, auto-appliable at analyze time
  (`--default-dst-team`), and bulk-settable via the `dst_team` CSV column
  (`amp/issue_parser.py`, `amp/analyze/base.py`, `amp/bulk_assign.py`). That path is
  covered by tests.
- **Actual team creation and repo attachment is not.** `/upsert-teams` creates the
  `<base>-read` / `<base>-write` / `<base>-admin` team suite and attaches the
  migrated repo at pull/push/admin (`_ensure_team` in `amp/commands/__init__.py`,
  fired after `status:migrate-success` by `post-migrate.yml`). This has **not been
  exercised against a live destination org** — the only automated coverage is a
  dispatch-wiring smoke test. Team creation, repo attachment, the tier→permission
  mapping, and idempotency on re-run are all **unverified in practice**.
  **Recommendation:** after `/upsert-teams`, confirm in the destination org UI that
  the three teams exist and have the migrated repo attached at the right permission.
- **SCIM / EMU orgs are not yet supported.** Where teams are provisioned by an IdP,
  `org.create_team` returns 422/403; the code falls back to fetch-by-slug, but the
  intended behavior (refuse to create, require a pre-existing team, stamp
  `status:waiting-on-scim-team`) is an open `TODO` and not implemented.
- **Sources with many project-scoped teams can exceed GitHub's 100-label cap.**
  Analyze emits one `src-team:<slug>` label per team attached to the source
  repo, and on ADO those teams are project-scoped (identical set on every
  repo in the project). GitHub caps labels per issue at 100 — a project with
  ~90+ teams leaves no room for `size:*` / `blob:*` / `status:*` / `cmd:*`
  labels and the issue create/update fails. Set the repo variable
  `AMP_DISABLE_SOURCE_TEAMS=true` on the control repo to skip `src-team:*`
  discovery entirely (workflow-scope; matches `analyze.yml`, `refresh.yml`,
  and `/update-issue`). AMP does **not** truncate partial team lists —
  incomplete `src-team:*` data would mislead any downstream consumer that
  treats the label set as complete. When discovery is on and any repo
  returns ≥80 teams, `amp analyze` logs a warning naming the env var so
  operators can flip it before a full run wastes runner minutes.

## Opt-in and slow paths

- **`amp analyze --scan-hardcoded-urls` is off by default** and adds roughly
  1–30s per repo (greps tracked files for source-SCM hostnames). Turn it on only
  when you need it (`amp/cli.py`, `amp/analyze/base.py`).
- **The deep LFS check is the slow part of analysis.** It runs locally against a
  mirror clone. Override its wall-clock budget with `AMP_LFS_SCAN_TIMEOUT` if it's
  timing out on a huge repo (`amp/analyze/lfs.py`).

## AI failure advisor

- **Tier 2 (the LLM step) needs a token.** Resolution is `MODELS_TOKEN` →
  `GITHUB_TOKEN` → Copilot SDK (interactive login, often **unavailable in CI**).
  With no usable token, AMP falls back to Tier 3 heuristics, which are a best-guess
  on novel failures (`amp/ai/`).
- Advisor responses are **cached 72h** in `~/.cache/amp/advisor/`; a stale-looking
  suggestion may be a cache hit.

## Distribution — compiled wheel (operators)

- **Compiled wheels are time-bombed.** Each build bakes in an `EXPIRY` (default 14
  days); past it the CLI refuses to run unless a valid Xebia-signed internal
  license is present (`amp/_guard.py`). A weekly rebuild cron re-stamps the pinned
  release so active installs keep working, and CI re-pulls each run — so this bites
  mainly a *copied* binary left sitting after the engagement. **Source / editable
  (`pip install -e .`) installs are not affected.**
- **Cython compilation is defense-in-depth, not cryptography.** It removes readable
  source from the artifact; it is not an encryption guarantee
  (`scripts/build_wheel.py`).
- **Wheels are built natively per platform/Python** (linux-x64, macOS-arm64,
  windows-x64 × Python 3.11–3.13) and never cross-built; pip selects the matching
  tag. A platform/Python combo that wasn't built simply has no wheel.
- **Known:** a local system-clock rollback can defer the expiry (also noted in
  `AMP_Distribution_Plan.md` / `AMP_IP_Protection_One_Pager.md`).

## Platform

- **Windows** has a few path/encoding caveats — see the "Windows caveats" section
  of [AMP_User_Guide.md](AMP_User_Guide.md) rather than duplicating them here.

## See also

- [AMP_User_Guide.md](AMP_User_Guide.md) — full setup, env vars, Windows caveats.
- [AMP_Distribution_Plan.md](AMP_Distribution_Plan.md) — wheel/expiry/license deep dive.
- [CLIENT_RUNBOOK_GHEC_DR.md](CLIENT_RUNBOOK_GHEC_DR.md) — install runbook with its own per-install known-limitations section.
