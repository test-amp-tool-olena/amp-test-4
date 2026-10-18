% AMP — Agentic Migration Platform
% User Guide

> **See also — feature briefs:** for a 2-page sales / overview brief of what AMP adds on top of `gh gei`, see [AMP_Features_One_Pager.md](AMP_Features_One_Pager.md) (master) or the per-SCM variants: [ADO](AMP_Features_One_Pager_ADO.md) · [Bitbucket Server](AMP_Features_One_Pager_Bitbucket_Server.md) · [Bitbucket Cloud](AMP_Features_One_Pager_Bitbucket_Cloud.md) · [GitHub](AMP_Features_One_Pager_GitHub.md) · [GitLab](AMP_Features_One_Pager_GitLab.md). Run `make` at the repo root to regenerate the `.docx` versions.

> **Known limitations:** for a short list of what's flaky, opt-in, less-tested, or best-effort (GitLab, GitHub Projects grouping, wheel expiry, etc.), see [AMP_Known_Limitations.md](AMP_Known_Limitations.md).

# Overview

AMP (Agentic Migration Platform) is an IssueOps platform for migrating
repositories from Azure DevOps, Bitbucket Server, GitHub, and GitLab to
GitHub. Every source repository becomes a GitHub issue with automatically
computed labels that describe its size, risk, and characteristics. You drive
migrations by applying labels or posting slash commands on those issues —
individually or in bulk.

## Sources and Targets

![Sources AMP can pull from and the GitHub Enterprise Cloud variants it migrates to](diagrams/sources_and_targets.png)

Each source has its own namespace concept — Azure DevOps *projects*,
Bitbucket Server *projects*, GitHub *orgs*, and GitLab *groups* — and AMP
flattens those into target GitHub orgs. The mapping from source namespace
to destination org is per-project and lives in `config/mappings/` (or is
applied at runtime via `dst-org` labels and `amp bulk-assign --csv`).

> Diagram source: [`docs/diagrams/sources_and_targets.mmd`](diagrams/sources_and_targets.mmd).
> Re-render with: `npx -y -p @mermaid-js/mermaid-cli mmdc -i docs/diagrams/sources_and_targets.mmd -o docs/diagrams/sources_and_targets.png -b transparent -w 1400`

# How It Works

The migration lifecycle is tracked entirely through GitHub issues and labels.
The typical flow is:

1. **Analyze** — Run `amp analyze` to scan a source org. One issue is created
   per repo with analysis labels (size, PR count, blob risk, LFS health) plus
   source labels (`namespace`, `src-org`, `src-team`) and default destination
   labels (`dst-org`, `dst-prefix`).
2. **Route** — Adjust where each repo lands by editing `dst-org`, `dst-team`,
   and `dst-prefix` labels — single-issue in the GitHub UI, 25 at a time via
   the native bulk-edit, or thousands at a time via `amp bulk-assign --csv`.
3. **Dry Run** — Apply the `cmd:dryrun` label (or post `/dryrun`) to test
   migration to a sandbox org. Failures trigger automatic AI diagnosis.
4. **Migrate** — Apply `cmd:migrate` to run the production migration via
   GitHub Enterprise Importer (GEI). The `dst-org` label is required; the
   target repo name is built as `<dst-prefix>-<source_repo>`.
5. **Post-Migration** — Automatic: if `dst-team` is set, the read/write/admin
   teams are ensured in the destination org and attached to the migrated repo
   with pull/push/admin permissions.
6. **Archive** — Apply `cmd:archive` to lock and archive the source repo.

The destination can be either GitHub.com (the default) or a tenant-specific
GitHub Enterprise Cloud with Data Residency (GHEC-DR) instance such as
`https://<tenant>.ghe.com`. See **GHEC-DR Support** below.

# Runtime Architecture

This view is for the GitHub admin / architect who needs to know what gets
installed where, what each component calls outbound, and what tokens, vars,
and network egress to allow before the first migration runs.

![AMP runtime architecture: operator surfaces, the AMP control repo (issues, workflows, secrets) installed in a customer GHEC org, and the external systems each leg touches](diagrams/runtime_architecture.png)

**Key things for an admin to notice:**

- **Where AMP runs.** The AMP control repo lives in a customer-owned GHEC
  org. It is *not* required to be the source or destination org — pick one
  org and treat it as the migration hub. Issues represent migrations; labels
  and comments drive the workflows.
- **GitHub Actions, hosted or self-hosted.** Workflows default to
  GitHub-hosted `ubuntu-latest` runners, but every `runs-on:` is wired
  through an `AMP_RUNNER` variable so you can point individual workflows
  (or the whole platform) at a self-hosted runner label without editing
  YAML. Set `AMP_RUNNER` to override globally, or
  `AMP_RUNNER_<WORKFLOW>` (e.g. `AMP_RUNNER_ANALYZE`,
  `AMP_RUNNER_SLOT_RUNNER`) to override one workflow at a time. Setup
  (Python + `amp` CLI + `gh` + extensions) is provided by the
  [`setup-amp`](../.github/actions/setup-amp/action.yml) composite
  action — it probes for `gh` and only downloads it when missing, so
  the same workflow YAML works on bare self-hosted nodes and on
  pre-provisioned images. The runner needs outbound egress to source
  SCMs, the destination GHEC tenant, and (for ADO/BBS) the GEI transit
  blob storage.
- **Migration archive storage.** Bitbucket Server migrations transit a git
  archive through blob storage. For **Bitbucket
  Server** AMP passes `--use-github-storage` so GitHub-owned storage is
  used by default (no external account). For **Azure DevOps** AMP does
  not require archive storage configuration; `gh ado2gh` uses GitHub's
  managed migration path. **GitLab** migration likewise uploads its `gl-exporter`
  archive to GitHub-owned blob storage by default, or to Azure Blob when
  `AZURE_STORAGE_CONNECTION_STRING` is set. GitHub→GitHub uses GEI's own
  transfer and needs no archive storage.
- **Two operator entry paths.** Most work is IssueOps (label or comment on
  an issue → workflow runs). The `amp` CLI is also available for local-only
  work (`analyze`, `export`, `dashboard`, `advise`, `setup-project`, etc.)
  and reads the same source SCM APIs from the operator's workstation.
- **Tokens to provision.** Always: `GH_PAT`, `GH_DEST_PAT`. Plus one of
  `ADO_PAT` / `BBS_PAT` / `GITLAB_TOKEN` per source in use. Optional but
  recommended: `APP_ID` + `APP_PRIVATE_KEY` (raises rate limits, avoids PAT
  rotation), `MODELS_TOKEN` (AI failure advisor), `GH_COM_TOKEN` (no-scope
  github.com PAT, lifts the runner bootstrap rate limit from 60/hr anon to
  5000/hr — add if `setup-amp` ever fails with "API rate limit exceeded").
  For cross-tenant moves like GHEC → GHEC-DR, also `DEST_APP_ID` +
  `DEST_APP_PRIVATE_KEY`. For installs where the control (AMP) repo lives
  on a tenant separate from both source and destination, set `GH_AMP_PAT`
  and `AMP_APP_*`; when the control repo lives on the destination tenant
  (the common case) leave `GH_AMP_*` unset and the control client falls
  back to `GH_DEST_PAT` automatically.
- **Tenant URLs are configurable.** `GH_DEST_API_URL` lets the
  destination be `https://api.<tenant>.ghe.com` instead of
  `api.github.com` — relevant for data-residency customers. `GH_API_URL`
  is the symmetric source-side switch: set it when the source GitHub
  tenant is itself a GHEC-DR / GHES instance (e.g. analyzing repos
  hosted on `https://api.<tenant>.ghe.com` and migrating them to
  github.com or another tenant). `GH_AMP_API_URL` is the third axis —
  set it only when the control (AMP) repo lives on a tenant different
  from where workflows run; defaults to the host tenant. Any of the three
  can be set independently. See **GHEC-DR Support**.

> Diagram source: [`docs/diagrams/runtime_architecture.mmd`](diagrams/runtime_architecture.mmd).
> Re-render with: `npx -y -p @mermaid-js/mermaid-cli mmdc -i docs/diagrams/runtime_architecture.mmd -o docs/diagrams/runtime_architecture.png -b transparent -w 1800`

# Getting Started

## Prerequisites

- Python 3.10 or later
- GitHub CLI (`gh`) with extensions: `gh-gei`, `gh-ado2gh`, `gh-bbs2gh`,
  and (for Bitbucket Cloud) `katiem0/gh-bbc-exporter`
- `git` and `git-lfs` installed
- Personal access tokens for source and destination platforms

## Local test environment (optional)

For exercising the analyzer/migration paths against varied repo shapes
without burning a paid SaaS account, [`test-env/`](../test-env/README.md)
spins up GitLab CE and Bitbucket Server locally on Colima and seeds a
6-repo fixture set per platform. One command brings it up, one command
tears it down:

```bash
python3 test-env/run.py up      # idempotent; first GitLab boot ~5–8 min
python3 test-env/run.py status
python3 test-env/run.py down    # stop containers, keep data
```

Bitbucket Cloud is hosted-only — for cloud testing use a free workspace
at bitbucket.org. See [`test-env/README.md`](../test-env/README.md) for
the BBS license step (one-time, free 30-day eval) and full details.

## Installation

There are two ways to install the `amp` CLI.

### From source (development)

Clone the repository and install in editable mode:

```bash
pip install -e .
```

### From the private distribution (recommended for engagements)

AMP can be delivered as a **Cython-compiled, platform-tagged wheel** (native `.so`/`.pyd`, no readable source) published to a private distribution repo (e.g. `xebia/amp-dist`), so no AMP source is placed in the customer environment. You install it with a **read-scoped token** that is **revoked at engagement end**, and each build carries a **runtime expiry** (default 14 days) so even a copied binary stops working after the window. A weekly cron refreshes the published build, so pinned CI keeps getting a non-expired wheel; run `amp update` to refresh a local install. Xebia consultants set `AMP_INTERNAL_LICENSE` (a signed token) to bypass the expiry. See [AMP_Client_Distribution_Setup.md](AMP_Client_Distribution_Setup.md) for the full onboarding/offboarding runbook.

```bash
# Requires a fine-grained PAT with Contents: read on the dist repo, and gh CLI.
AMP_VERSION=0.1.0 AMP_DIST_REPO=xebia/amp-dist AMP_DIST_TOKEN=ghp_xxx \
  scripts/install-amp.sh
```

`pip` automatically selects the wheel matching your platform (Linux x86_64, macOS arm64, …). The wheel carries only the Python CLI — you still install `gh` + the migration extensions, `git`, and `git-lfs` separately (see Prerequisites). In CI, the same wheel is installed by `.github/actions/setup-amp` when `secrets.AMP_DIST_TOKEN`, `vars.AMP_VERSION`, and `vars.AMP_DIST_REPO` are set; otherwise it falls back to `pip install -e .`.

> **Tuning config without rebuilding the wheel:** the wheel bundles the default `labels.yml` / `failure_rules.yml` / `project_views.yml`. To use a tuned copy (e.g. one checked into your control repo), set `AMP_CONFIG_DIR` to that `config/` directory and AMP will read from there instead.
>
> **Maintainer build/publish:** `python scripts/build_wheel.py --expiry-days 14` builds a Cython-compiled wheel for the current platform (needs Cython + a C compiler; no license); `.github/workflows/publish-amp-wheel.yml` matrix-builds linux-x64 + macOS-arm64 + Windows-x64 × Python 3.11–3.13 on a `v*` tag, uploads to the dist repo release, and a weekly cron refreshes the build so pinned installs stay non-expired.

This registers the `amp` command. Verify with:

```bash
amp --help
```

## Authentication

AMP resolves tokens from command-line flags or environment variables. Create
a file (e.g. `~/.ghpats`) and source it before running commands:

```bash
export GH_PAT=ghp_...           # Source GitHub PAT
export GH_DEST_PAT=$GH_PAT      # Destination GitHub PAT
export ADO_PAT=...              # Azure DevOps PAT (if needed)
export GITLAB_TOKEN=...         # GitLab token (if needed)
export BBS_PAT=...              # Bitbucket Server PAT (if needed)
export BITBUCKET_API_TOKEN=...  # Bitbucket Cloud scoped API token (if needed; see below)
export BITBUCKET_EMAIL=...      # Email associated with BITBUCKET_API_TOKEN
export MODELS_TOKEN=$GH_PAT     # Token for AI advisor
```

```bash
source ~/.ghpats
```

| Context | Resolution Order |
|---|---|
| Source platform | `--token` flag → `GH_PAT` / `ADO_PAT` / `GITLAB_TOKEN` / `BBS_PAT` / `BITBUCKET_API_TOKEN` + `BITBUCKET_EMAIL` |
| Destination GitHub | `--dest-token` → GitHub App (`DEST_APP_*` → `APP_*` same-tenant) → `GH_DEST_PAT` → `GITHUB_TOKEN` |
| Control (AMP) repo | GitHub App (`AMP_APP_*` → `APP_*` same-tenant) → `GH_AMP_PAT` → `GH_DEST_PAT` → `GITHUB_TOKEN` → `GH_PAT` |
| IssueOps (CI) | GitHub App (`AMP_APP_*` → `APP_*`) → `GH_AMP_PAT` → `GH_DEST_PAT` → `GITHUB_TOKEN` → `GH_PAT` |
| AI advisor | `MODELS_TOKEN` → `GITHUB_TOKEN` → Copilot SDK |
| Runner bootstrap (gh CLI release lookup + `gh extension install`) | `GH_COM_TOKEN` (no-scope github.com PAT) → anonymous (60/hr) |

### Three tenants: source, destination, control

AMP treats *source* (repos to migrate), *destination* (where they go), and
*control* (the AMP repo with the issues and workflows) as three independent
GitHub-side credential domains. They often overlap — the control repo
typically lives on either the source or destination tenant — but they don't
have to.

- **Control repo on destination tenant** (the most common GHEC-DR pattern):
  set `GH_DEST_PAT` and leave `GH_AMP_*` unset. The control client picks up
  `GH_DEST_PAT` via the back-compat fallback chain.
- **Control repo on source tenant**: set `GH_AMP_PAT` (or rely on `GH_PAT`
  fallback if the source PAT also has issue read/write on the control repo).
  Set `GH_AMP_API_URL=https://api.github.com` when source is github.com.
- **Control repo on its own third tenant**: set both `GH_AMP_PAT` and
  `GH_AMP_API_URL` to that tenant's credentials and API URL.

### GitHub App authentication (optional)

Three App pairs map to the three GitHub-side domains; each App is
single-tenant, so don't reuse one App's credentials for another tenant's API:

- `APP_*` (`APP_ID` + `APP_PRIVATE_KEY`) — host-tenant App. When set, AMP
  prefers App-installation tokens over PATs for destination API calls
  (same-tenant) and control-repo ops (host-tenant). The App gets its own REST
  budget per installation — starting at 5,000/hr and scaling with org size
  up to 12,500/hr (15,000/hr for GHEC-owned Apps) — separate from the PAT
  the migration CLIs share. Useful when high-volume IssueOps + scheduled
  jobs (daily report, exports, cleanup) start crowding migrations off the
  PAT budget.
- `DEST_APP_*` (`DEST_APP_ID` + `DEST_APP_PRIVATE_KEY`) — destination-tenant
  App, for cross-tenant migrations (e.g. AMP installed in source GHEC,
  migrating to GHEC-DR).
- `AMP_APP_*` (`AMP_APP_ID` + `AMP_APP_PRIVATE_KEY`) — control-repo-tenant
  App, for installs where the control repo lives on a tenant separate from
  both the host runner and the migration destination.

App auth is **fully optional**: with the secrets unset (or left as the
`REPLACE_ME` placeholder), AMP falls back to PAT and behaves exactly as
before. Migration subprocesses (`gh-gei`, `gh-ado2gh`, `gh-bbs2gh`) and
the local CLI continue using PATs unchanged.

See [`docs/GITHUB_APP_SETUP.md`](GITHUB_APP_SETUP.md) for the one-time setup steps.

### Runner bootstrap rate limit (optional)

The runner setup composite (`.github/actions/setup-amp/action.yml`) calls
`api.github.com` twice during job startup: once to resolve the latest `gh` CLI
release tag (only when `gh` isn't already on the runner) and once per `gh
extension install` for `gei` / `ado2gh` / `bbs2gh` / `bbc`. These calls go to
**github.com**, not your tenant — the workflow's tenant token can't
authenticate them (it 401s against `api.github.com`), so they run anonymously
and share the **60/hr-per-IP** anonymous rate limit with every other runner on
the same egress IP. On shared self-hosted runners this saturates quickly and
surfaces as `HTTP 403: API rate limit exceeded`.

To raise the limit to **5000/hr authenticated**, set the `GH_COM_TOKEN`
repository secret to a github.com PAT with no scopes selected (the bearer
itself is what authenticates against api.github.com; no permissions are
needed). The composite exports it inside the install step only, after first
unsetting any inherited tenant token so it can't leak the wrong direction.

Skip this for personal sandboxes — anonymous works fine when extensions are
installed once per runner lifetime. Add it when `setup-amp` failures start
appearing across multiple jobs in CI. **`GH_COM_TOKEN` must be a github.com
PAT** — never a GHEC-DR / tenant token.

### Bitbucket Cloud — source prereqs

A few Atlassian-side constraints surface only at migration time; surface them
before starting:

- **Use a scoped Atlassian API token, not the legacy unscoped one.** At
  https://id.atlassian.com/manage-profile/security/api-tokens, click
  **"Create API token with scopes"** (not the older "Create API token"
  button — that one produces a Jira/Confluence-only token that 401s against
  every Bitbucket endpoint). Pick **Bitbucket Cloud** as the app and select
  at minimum `read:account`, `read:workspace:bitbucket`,
  `read:repository:bitbucket`, `read:pullrequest:bitbucket`, and
  `read:webhook:bitbucket`. App passwords retire 2026-06-09 and AMP does
  not support them.
- **Free workspaces block writes to private repos.** Atlassian's free
  Bitbucket Cloud tier enforces a workspace-wide read-only rule on private
  repos (the error message reads "user limit exceeded"). For private-repo
  migrations the client must be on Standard ($3.30/seat/mo) or higher
  before AMP can pull the source.
- **Workspace Access Tokens (Bearer auth) are a Premium feature** and are
  not available on Free or Standard. AMP supports them via
  `BITBUCKET_ACCESS_TOKEN` if a client already uses them, but doesn't
  require them.
- **Custom user groups can't be created via the public API.** Add them via
  the Bitbucket UI (Workspace settings → User groups). The auto-created
  system groups (`bitbucket-admins-*`, `bitbucket-users-*`, `org-admins`)
  are visible to the analyzer if granted per-repo permission and become
  `src-team:*` labels.
- **Bitbucket Cloud "projects" are not analyzed.** AMP enumerates repos
  flat across the workspace. If the client uses BBC projects as a logical
  grouping, that grouping does not carry forward — destination orgs and
  team assignments are driven by `dst-org:` / `dst-team:` labels on the
  AMP issue instead.
- **`ref:long` (>255-char ref names) cannot trigger on BBC sources.** The
  BBC API caps ref names at 128 characters and rejects longer ones at
  push time, so AMP's `ref:long` analyzer signal is effectively dead code
  for BBC. (BBS, GitHub, and GitLab sources can still trigger it.)

### Bitbucket Cloud — destination prereqs

- **Destination must be GHEC, GHEC-DR, or GHEC EMU.** Standard (free / Pro /
  Team) github.com orgs cannot be migration destinations — GitHub's
  `createMigrationSource` mutation returns "Resource not accessible by
  integration" against them.
- **Migrator role grant required.** Whether AMP authenticates to the
  destination via PAT or GitHub App, the *actor* (user or App) needs the
  Migrator org role. Grant it via the
  [`grantMigratorRole`](https://docs.github.com/en/graphql/reference/mutations#grantmigratorrole)
  GraphQL mutation against the destination org. Without it, dryrun and
  migrate fail at the import step with "Resource not accessible by
  integration."

# Step-by-Step Usage

## Step 1: Setup Labels and Project Board

Before your first analysis, create the 50+ labels AMP uses to categorize
repos, and optionally set up a GitHub Project board for visual tracking.

```bash
# Create labels in the IssueOps repo
amp setup-labels --repo <owner>/<issueops-repo>

# Create a Kanban project board
amp setup-project --repo <owner>/<issueops-repo> --name "Migration: ClientName"
```

## Step 2: Analyze Source Repositories

Run `amp analyze` to scan every repository in a source organization. One
GitHub issue is created per repo, with three categories of labels: analysis
labels (size, PR count, blob risk, LFS health, ref validity, submodules,
webhooks, age), source labels (`namespace`, `src-org`, `src-team`), and
default destination labels (`dst-org`, `dst-team`, `dst-prefix`). Routing
(where a repo goes, which team owns it, what prefix it gets) flows through
these destination labels — the issue body is display-only for those fields.

> **Two exceptions the analyzer manages, don't hand-edit:** the
> `<!-- amp-id: ... -->` marker and (for ADO) the project name it encodes
> are the canonical identity for the source repo. Refresh, re-analyze, and
> every ADO API call route through them (see the "Key things to know"
> section at the end). Editing the marker to point at a different repo or
> ADO project silently redirects subsequent operations. The
> `<!-- amp:source-baseline -->` and `<!-- amp:env-recreate-data -->`
> markers are also analyzer-managed — leave them alone.

`analyze` is **idempotent**: re-running it on the same source matches each
analyzed repo to its existing issue by `amp_id` and refreshes that issue
(title, body, labels) in place. New repos appearing on the source get a
fresh issue. No duplicates are created. Legacy issues without an `amp_id`
are matched by provider+namespace+repo name and backfilled. If two issues
both claim the same source repo, both are flagged
`status:duplicate-candidate` and skipped — pick a survivor manually.

`analyze` never marks issues obsolete. If a source repo disappears, its
issue stays untouched. Use `amp refresh` (a separate command) when you
also want missing-repo detection and obsolete-marking with safety guards.

```bash
# GitHub source
amp analyze --source github --org my-source-org \
    --default-dst-org my-target-org --repo <owner>/<issueops-repo>

# Azure DevOps source
amp analyze --source ado --org my-ado-org --project MyProject \
    --default-dst-org my-target-org --repo <owner>/<issueops-repo>

# Bitbucket Server
amp analyze --source bitbucket --org PROJECT_KEY \
    --default-dst-org my-target-org --repo <owner>/<issueops-repo>

# Bitbucket Cloud (BITBUCKET_API_TOKEN + BITBUCKET_EMAIL required)
amp analyze --source bitbucket-cloud --org my-workspace \
    --default-dst-org my-target-org --repo <owner>/<issueops-repo>

# GitLab
amp analyze --source gitlab --org my-group \
    --default-dst-org my-target-org --repo <owner>/<issueops-repo>
```

### Analyzing every group/project on a server

For `gitlab` and `bitbucket` (server) sources, omit `--org` to auto-discover
every group/project the token can see and analyze each in turn. Issues all
land in the single `--repo` IssueOps repo; the per-issue `namespace:<slug>`
label tells you which source group/project each repo came from.

```bash
# All visible GitLab top-level groups (subgroups included automatically)
amp analyze --source gitlab    --repo <owner>/<issueops-repo>

# All visible Bitbucket Server projects
amp analyze --source bitbucket --repo <owner>/<issueops-repo>
```

`--org` remains required for `github`, `ado`, and `bitbucket-cloud` — those
providers don't have a meaningful "every namespace on the server" concept
that maps to a single migration scope.

Useful flags:

- `--default-dst-org` / `--default-dst-team` — applied as `dst-org:<value>` /
  `dst-team:<value>` to every issue created. Both optional; you can leave
  them off and assign destinations later via the UI or `amp bulk-assign`.
- `--read-source-teams` / `--no-read-source-teams` — discover teams/groups
  attached to each source repo and emit one `src-team:<slug>` label per team
  (default on). Disable for very large runs where the extra API calls per
  repo would slow analysis materially, or when the source (typically ADO)
  has enough project-level teams to push the per-issue label count past
  GitHub's cap of 100. The repo variable `AMP_DISABLE_SOURCE_TEAMS=true`
  (strict literal, case-insensitive) is the workflow-level equivalent and
  wins over the CLI flag when both are set — use it on `analyze.yml`,
  `refresh.yml`, and `issue-ops.yml` so the opt-out sticks across every
  run without editing YAML. When source-teams sync is on and the discovered
  team count crosses 80 for any repo, `amp analyze` logs a warning naming
  the env var so operators can flip it before a full run fails on the
  100-label API cap.

You can also trigger analysis from the GitHub Actions UI using the
"Analyze Source" workflow.

### Empty source repositories

A source repo with no commits has nothing to migrate. The analyzer
recognises this in two places:

- **Pre-clone probe** — a cheap check that, for most providers, reads a
  null default branch straight from the repo listing. When it fires, the
  clone, LFS deep check, blob scan, ref hygiene, CODEOWNERS, and workflow
  inspection are all skipped — none of them have anything to measure.
- **Post-clone ground truth** — after cloning, a repo with zero commits,
  branches, and tags is empty regardless of what the API reported. This is
  what catches GitHub, whose REST API returns a configured `default_branch`
  (e.g. `main`) even when no such ref exists. The clone still runs but is
  instant — there is nothing to download.

Either way the issue ends up carrying only identity labels plus
`status:empty` and a minimal body that names the source and explains that
migration is not applicable.

Detection signal per provider:

| Provider | Signal |
|---|---|
| GitHub | Post-clone: zero commits, branches, and tags (the REST API reports a `default_branch` even on an empty repo) |
| GitLab | `project.default_branch` is null |
| Azure DevOps | `repository.defaultBranch` is null |
| Bitbucket Cloud | `repository.mainbranch` is null |
| Bitbucket Server | `/branches/default` returns 404 |

The post-clone count check is a backstop for every provider — if a
pre-clone probe ever misses, the zero-count check still catches it.

`status:empty` is mutually exclusive with every other `status:*` label —
re-analysis of a repo found empty clears any stale `status:analyzed`,
`status:dryrun-*`, or `status:migrate-*` it had previously accumulated.
Empty repos appear in their own **Empty** column on the project board
and as their own row in the daily digest — they are NOT counted under
`analyzed`, `dryrun-*`, or `migrate-*`.

`/dryrun`, `/migrate`, and `/remigrate` refuse to operate on a
`status:empty` issue. The handler posts a blocker comment and exits
before any queue claim or destination work; no `status:*-failed` is
stamped, no Action minutes are consumed.

To clear the state, either push a commit to the source and run
`/update-issue` (the analyzer flips the issue back to `status:analyzed`),
or remove the `status:empty` label by hand if it was a misdetection.

### Analysis Labels

| Category | Labels | Meaning |
|---|---|---|
| Provider | `provider:ado`, `provider:bitbucket`, `provider:bitbucket-cloud`, `provider:github`, `provider:gitlab` | Source platform |
| Size | `size:small` (<100 MB), `size:medium`, `size:large`, `size:huge` (>5 GB) | Repository disk size |
| PRs | `prs:small` (0–100), `prs:medium`, `prs:large`, `prs:xlarge` (5000+) | Pull request count |
| Blobs | `blob:ok`, `blob:warning` (>100 MB), `blob:error` (>375 MB — blocker) | Large file risk |
| LFS | `lfs:none`, `lfs:active`, `lfs:historical`, `lfs:broken` (blocker), `lfs:oversized` | Git LFS health |
| LFS size | `lfs:large` (orthogonal — rides alongside the LFS state label when the historical LFS payload exceeds `AMP_LFS_LARGE_THRESHOLD_BYTES`, default 5 GiB; counts every unique OID across all refs, not just HEAD) | Routes the migration worker to `vars.AMP_RUNNER_MIGRATION_WORKER_LARGE` to avoid disk-OOM on the default hosted runner |
| LFS hygiene | `lfs:orphan-pointers`, `lfs:inconsistent`, `lfs:attributes-invalid` (orthogonal warnings — set by analyze across every branch and tag) | `orphan-pointers`: a file is committed as an LFS pointer but no `.gitattributes` rule (root or nested) covers its path — a later edit silently re-stores it as a raw blob. `inconsistent`: the same path is an LFS pointer on some refs and a raw blob on others. `attributes-invalid`: a `.gitattributes` LFS glob is malformed (e.g. brace expansion `{*.tgz}`) and git ignores it. Migration runs; the `amp/migration-fixes` branch forward-fixes `.gitattributes`. |
| LFS source gap | `lfs:source-missing` (blocker — set by analyze or post-push verify) | The source LFS server cannot serve OIDs the repo's pointers reference. There is nothing for the migration to transfer — a data-loss risk. Repair the source LFS store (e.g. re-push the objects) before migrating. |
| LFS destination gap | `lfs:dest-missing` (orthogonal — set by the post-migration verifier when AMP's `git lfs push --all` exits 0 but the destination LFS Batch API still reports objects missing) | The advisor's `lfs-objects-missing-after-push` rule diagnoses this and recommends checking destination LFS quota, PAT scopes, or SAML re-auth |
| LFS repair diagnosis | `lfs:dest-auth`, `lfs:dest-saml`, `lfs:dest-quota` (orthogonal — set by the `lfs-objects-missing-after-push` auto-repair handler) | The repair handler probes the source LFS server, then the destination's upload acceptance via Batch API (no PUT — no bytes leave). When that's inconclusive it retries the push serially (`lfs.concurrenttransfers=1`, `lfs.batch=false` + `AMP_LFS_DEBUG=1`) so per-object errors surface. The label records the determined root cause; the auto-repair comment includes the diagnostic detail. On `dest-auth` / `dest-saml` the PAT needs re-scoping or SSO authorization. On `dest-quota` contact the destination org's GitHub admin. |
| Volume | `refs:high` (>5k branches), `tags:high` (>5k tags), `commits:huge` (>250k commits), `releases:oversized` (>30 GB of release assets — GitHub-only, predicts GEI's 40 GB metadata-archive cap) | Metadata-archive bloat risk |
| Age | `age:<1mo`, `age:1–6mo`, `age:6–12mo`, `age:>1yr`, `stale` | Last commit age |
| Attributes | `submodules`, `webhooks`, `ref:long` | Special characteristics |
| Duration | `duration:0-5m`, `duration:5-10m`, … `duration:55-60m`, `duration:60m+` | Wall-clock time of the most recent GEI run (set post-run by dryrun/migrate). Green ≤10 min, yellow 10–30 min, salmon 30–45 min, red 45–60 min, dark red `60m+` (near the GEI kill-timer). Useful for spotting outliers and forecasting batch windows. |
| Readiness — submodules | `submodules:rewrite-needed` | ≥1 `.gitmodules` URL points at the source SCM host. AMP rewrites these on the post-migration fixes branch when the sibling repo is also AMP-managed. Only the host/org/repo coordinates change — the URL's original ssh or https scheme is preserved. |
| Readiness — workflows | `actions:rewrite-needed`, `actions:ok`, `actions:none` | Cross-repo `uses:` references in `.github/workflows/*.yml`. Marketplace owners (`actions/`, `github/`, `docker/`) are whitelisted and never flagged. |
| Readiness — CODEOWNERS | `codeowners:rewrite-needed`, `codeowners:none` | `@org/team-slug` mentions that may not resolve at the destination if team slugs differ. |
| Readiness — recreate | `secrets:present` | Workflow files reference secrets, variables, or environments. AMP auto-creates stubs at the destination on `/dryrun` and `/migrate` (see `secrets-recreate:*` below). |
| Readiness — hardcoded URLs | `hardcoded-urls:present` | Tracked files contain references to the source SCM host. Only emitted when `amp analyze --scan-hardcoded-urls` is passed (off by default to keep analysis fast). |
| Readiness — branch protection | `branch-protection:present` | Classic branch protection rules detected on the source (GitHub source only). AMP attempts to recreate them at the destination on `/dryrun` and `/migrate` (see `branch-protection:dest-*` below). |
| Readiness — custom properties | `custom-properties:present` | Source repo has custom repository properties set (GitHub source only). AMP carries these through on `/migrate`. |
| Readiness — environments | `environments:source-present`, `environments:source-none` | Source repo has GitHub Actions Environments. AMP recreates each environment + its protection rules at the destination on `/dryrun` and `/migrate` (see `environments:*` below). |
| Readiness — org secrets FYI | `org-secrets:source-present` | Source org has Actions secrets or variables. FYI only — AMP does not write at org scope; coordinate centrally. |
| Post-migration — fixes branch | `migration-fixes:none`, `migration-fixes:pushed`, `migration-fixes:failed` | Outcome of pushing `amp/migration-fixes` to the destination after `/dryrun` or `/migrate`. |
| Post-migration — run warnings | `status:has-warnings` | Additive flag — the last `/dryrun` or `/migrate` completed but GEI reported warnings (PR-review threads / commit comments that GitHub preserved as plain comments) **or a best-effort post-migration step (branch protection, secret/variable stubs, custom properties, environments, `amp/migration-fixes`) didn't fully land** (🟧 warning rows in the verification block — the migration still succeeded). Coexists with `status:*-success` and is re-evaluated on every run, so a later clean run clears it. Surfaced in the **Run Warnings** export column and explained in plain language in the destination-repo handoff issue. |
| Post-migration — validation | `status:validate-passed`, `status:validate-failed`, `status:validate-history-rewritten` | Source-vs-destination count comparison after `/migrate`. SHA / commit-count assertion is skipped only when `lfs:oversized` is set (operator-run pre-migration rewrite required). `lfs:active` and `lfs:historical` do not skip — AMP's LFS path preserves commit SHAs. |
| Post-migration — properties | `custom-properties:applied`, `custom-properties:created+applied`, `custom-properties:dest-not-set` | Custom-properties carry-through outcome (GitHub source). `created+applied` means AMP created the property schema at the destination org before setting the value. |
| Post-migration — secrets stubs | `secrets-recreate:created`, `:partial`, `:failed`, `:skipped`, `:none`, `:rate-limit-deferred` | Auto-recreate of workflow secret/variable stubs at the destination. Existing destination values are never overwritten (GET-first idempotency). Placeholder value is `AMP_ PLACEHOLDER_ REPLACE_ ME` — scrubbed in workflow logs as `***`; cross-reference the post-migrate comment's stub list to identify unset secrets. Re-trigger via `cmd:upsert-secrets`. |
| Post-migration — environments | `environments:created`, `:partial`, `:failed`, `:none`, `:rate-limit-deferred` | Auto-recreate of source GitHub Actions Environments + protection rules at the destination. Required reviewers attempted first; on 422 retried without reviewers and flagged per env for manual re-add (cross-tenant ID translation is not solved in v1). Re-trigger via `cmd:upsert-envs`. |
| Post-migration — branch protection | `branch-protection:dest-applied`, `:dest-partial`, `:dest-failed`, `:dest-skipped` | Auto-recreate of classic branch protection rules at the destination on `/dryrun` and `/migrate`. Per-branch PUT to GitHub's protection API using the structured config captured at analyze time (re-fetched from source if the issue was analyzed before this feature shipped). Cross-tenant push-restriction actors (users/teams/apps that don't resolve at the destination) are stripped on retry and flagged in the consolidated comment. Rulesets and actor bypass lists are out of scope. Best-effort — failures never roll back the migration. |

### Source & Destination Labels

Source and destination metadata live entirely in labels (not the issue
body), so they're filterable, bulk-editable, and visible in the GitHub UI
dropdown.

| Label | Role | Example values |
|---|---|---|
| `namespace:<slug>` | Unified source grouping (ADO project / BB project / GL group / GH org) | `namespace:platform-team`, `namespace:proj1` |
| `src-org:<slug>` | Top-level source identifier (matters for ADO where org ≠ project) | `src-org:rschwarz0884` |
| `src-team:<slug>` | Source teams/groups attached to the repo (multi-valued) | `src-team:backend`, `src-team:platform` |
| `dst-org:<slug>` | Target GitHub org — REQUIRED for `/migrate` | `dst-org:ts-dest` |
| `dst-team:<base>` | Base team name; `/upsert-teams` creates `<base>-read`/`-write`/`-admin` | `dst-team:platform` |
| `dst-prefix:<slug>` | Prepended to source repo name to form target repo name | `dst-prefix:ado`, `dst-prefix:legacy` |

Default `dst-prefix` is derived per provider: Bitbucket uses the project key;
ADO uses the project name (slugified); GitLab uses the top-level group;
GitHub uses the source org. The trailing dash is added when the target name
is constructed, so `dst-prefix:ado` on source repo `MyApp` produces target
repo `ado-MyApp`.

**Disabling the default prefix (1:1 org migrations).** For migrations where a
single source org maps to a single destination org, prefixing is unnecessary —
there's no collision risk. Set repo variable `AMP_DISABLE_DEFAULT_REPO_PREFIX=true`
(seeded by `setup-instance.sh` and visible in *Settings → Secrets and variables
→ Actions → Variables*) and `amp analyze` will stop injecting the per-provider
default `dst-prefix:` label. Target repo names then match source repo names
exactly (`MyApp` → `MyApp`, not `ado-MyApp`). The flag affects only the
analyzer-computed default; operators can still hand-apply a `dst-prefix:<name>`
label on individual issues to override, and `/update-issue` will preserve any
manually-set prefix. Flip the variable to `false` (or delete it) to restore
default prefixing.

**Conflict semantics:** if an issue ends up with two or more `dst-org:`,
`dst-team:`, or `dst-prefix:` labels, `/migrate` and `/upsert-teams` refuse
to run and post a comment listing the conflicts. Remove the duplicates and
re-trigger the command.

## Step 3: Assign Destinations (Routing)

Decide which target org each repo lands in, which team owns it, and what
its renamed repo name should be. This is done by setting `dst-org:`,
`dst-team:`, and `dst-prefix:` labels. Choose the workflow that matches
your batch size:

**Single issue or up to ~25 at a time — GitHub UI:** Select issues with
the checkboxes on the issue list, click "Edit labels", and pick the
`dst-org`/`dst-team`/`dst-prefix` values from the dropdown. The native
bulk-edit handles 25 at a time (one screen of issues).

**Hundreds to tens of thousands — `amp bulk-assign --csv`:** Author a CSV
with columns `issue_url`, `dst_org`, `dst_team`, `dst_prefix` (any subset
of trailing columns is optional) and run the command. Labels are created
on the fly if needed; failed rows after retries land in
`<csv>.failures.csv` so you can re-run only those.

```bash
# Preview the planned label changes
amp bulk-assign --csv routing.csv --repo <owner>/<issueops-repo> --dry-run

# Apply them (concurrent, idempotent, rate-limit-aware)
amp bulk-assign --csv routing.csv --repo <owner>/<issueops-repo>

# Re-run only the failures
amp bulk-assign --csv routing.csv.failures.csv --repo <owner>/<issueops-repo>
```

### CSV cell semantics

| Cell value | Behavior |
|---|---|
| empty / blank | Leave that label family alone for this row |
| `<remove>` or `none` | Strip any existing `dst-X:*` label of that family |
| any other value | Replace any existing `dst-X:*` label with `dst-X:<slugified value>` |

### Excel export (review-only)

Export all issues to an Excel workbook for offline review:

```bash
amp export --repo <owner>/<issueops-repo> -o migrations.xlsx
```

| Sheet | Purpose |
|---|---|
| Repos | One row per repo — source/`dst-*` labels, resolved target repo, risk levels. Two rollup columns aggregate everything needing attention: **Blockers** (hard — migration won't run) and **Warnings** (soft — manual remediation). Cells with values are highlighted (Blockers red, Warnings yellow). A separate **Run Warnings** column shows `Yes` when the last `/dryrun` or `/migrate` produced GEI runtime warnings (the `status:has-warnings` flag) — distinct from the readiness **Warnings** rollup. |
| Commands | Type a label in the "Apply Label" column → formula generates a `gh` CLI command |
| Summary | Pivot-style counts by status, provider, size, and risk |

**Blockers** (hard — migration cannot run / has failed):

| Label | Meaning |
|---|---|
| `blob:error` | Files larger than 375 MB detected — hard GEI limit |
| `lfs:broken` | LFS pointers exist but objects are missing/corrupt at source |
| `meta:oversized` | Metadata archive exceeded GEI's 40 GB cap |
| `status:analysis-failed` | Initial clone/analysis failed |
| `status:dryrun-failed` | Dry-run migration failed |
| `status:migrate-failed` | Production migration failed |
| `status:duplicate-candidate` | Multiple issues map to the same source repo |
| `migration-fixes:failed` | Post-migration branch push failed |
| `status:validate-failed` | Destination drift vs. source after migration |
| `ref:long` / `ref:invalid-chars` / `ref:case-collision` / `ref:path-conflict` | Ref names that break under GitHub's import rules |

**Warnings** (soft — needs manual remediation but doesn't block migration):

| Label | Meaning |
|---|---|
| `blob:warning` | Files between 100–375 MB |
| `lfs:oversized` | Files > 100 MB not LFS-tracked |
| `lfs:orphan-pointers` | LFS pointer blobs at paths no `.gitattributes` rule covers — a later edit re-stores the file as a raw blob |
| `lfs:inconsistent` | The same path is an LFS pointer on some refs and a raw blob on others |
| `lfs:attributes-invalid` | A `.gitattributes` LFS glob is malformed (e.g. brace expansion `{*.tgz}`) and silently does nothing |
| `lfs:dest-missing` | AMP's post-migration LFS push exited 0 but the destination LFS server still reports objects missing (quota, SAML, or PAT scope) |
| `lfs:source-missing` | Source LFS server cannot serve OIDs the destination needs — AMP cannot push what source doesn't have; repair source history |
| `lfs:dest-auth` | Destination LFS server rejected AMP's PAT/App on the upload probe (HTTP 401/403) — verify Contents: write scope |
| `lfs:dest-saml` | Destination org is SAML-gated and the PAT is not authorized — re-authorize the PAT for that org |
| `lfs:dest-quota` | Destination LFS storage quota exceeded (surfaced by serial-mode re-push) — contact GitHub admin to raise the org quota |
| `submodules:rewrite-needed` | Submodule URLs point at source SCM |
| `actions:rewrite-needed` | Workflow `uses:` references need updating |
| `codeowners:rewrite-needed` | CODEOWNERS @org/team mentions may not resolve at destination |
| `secrets:present` | Workflow refers to secrets/vars/environments that must be recreated |
| `hardcoded-urls:present` | Tracked files contain hardcoded SCM URLs |
| `branch-protection:present` | Classic branch protection rules detected — AMP attempts to recreate them at the destination on `/dryrun` and `/migrate` |
| `refs:high` / `tags:high` / `commits:huge` / `releases:oversized` | Metadata-archive bloat risks |

The full classification lives in [amp/blockers.py](../amp/blockers.py). To promote a warning to a blocker (or vice versa), edit the two tuples there — no label rename needed.

### Analytics workbook (pivots, state history, command builder)

For fleet-level analysis the analytics workbook goes beyond the flat review export:

```bash
amp export-analytics --repo <owner>/<issueops-repo> -o analytics.xlsx
```

| Sheet | Purpose |
|---|---|
| Data | One row per repo (the **Repos** columns plus a single-valued **Primary Blocker** column) as a filterable Excel Table (`tblData`). |
| State Changes | One row per `status:*` / `cmd:*` transition — added and removed events — with the actor and hours since the previous change. |
| Status / Blockers / LFS / Age / Orgs | Native Excel **PivotTables** counting issues: status × LFS/provider/size, primary-blocker breakdowns, LFS, age, and source org × namespace. |
| Legend | Explains each sheet and the command-builder flow. |

**Pivots are real, interactive PivotTables** (generated by injecting OOXML pivot parts — openpyxl cannot author pivots itself). They **refresh on open**; after editing the Data sheet, right-click any pivot → *Refresh* to recompute. Because Excel refuses to let one PivotTable expand over another, the pivots on each tab are stacked with vertical gaps. If your Excel/LibreOffice build ever flags the file, regenerate with `--countifs-pivots` to emit the workbook without native pivots.

**In-table command builder.** The Data table appends three columns:

- **Action** — a dropdown: `add` or `remove`.
- **Command** — a dropdown of every `cmd:*` label (kept in sync with `config/labels.yml`).
- **gh Command** — a formula that assembles `gh issue edit <#> -R <repo> --add-label/--remove-label "<cmd>"`.

Because the builder lives inside `tblData`, you can **filter/sort the repos first**, fill the dropdowns on the rows that match, and then:

```bash
amp import-commands --file analytics.xlsx --repo <owner>/<repo>        # preview
amp import-commands --file analytics.xlsx --repo <owner>/<repo> | bash # execute
```

`import-commands` emits commands for **only the rows left visible by the filter**, and reads either workbook layout (the analytics **Data** sheet or the classic **Commands** sheet).

**State-change history on its own.** The same per-transition data is available as CSV without the workbook:

```bash
amp state-history --repo <owner>/<repo> -o state-changes.csv
# from a fresh checkout (no install needed):
python3 scripts/state-changes.py <owner> <repo> --output-dir ./output
```

Each row is `Issue # · Source Org · Source Repo · AMP ID · Label · Stage · Event · Timestamp · Date · Actor · Elapsed (h) · Seq · Issue URL`, sorted by issue then time — so a single repo reads as its migration narrative (dryrun-success → migrate-failed → migrate-success → archived). The timestamps come from GitHub's issue timeline (AMP keeps no local history); the fetch pages past the 100-event limit so heavily re-run repos aren't truncated.

## Step 4: Dry Run

Test a migration by running a dry run to a sandbox org. There are two ways
to trigger it:

**Via labels (bulk-friendly):** Apply the `cmd:dryrun` label to one or
more issues in the GitHub UI. The label-ops workflow converts this to a
`/dryrun` comment, which the issue-ops workflow executes. The command
label is auto-removed after dispatch.

**Via comments (single issue):** Post `/dryrun` as a comment on an issue.

## Step 5: Production Migration

Once the dry run succeeds, trigger the real migration the same way:

- Apply the `cmd:migrate` label, or post `/migrate` as a comment.

Before migration, AMP automatically checks for blockers:

- `blob:error` label — files >375 MB block GEI migration
- `lfs:broken` label — LFS integrity must be repaired first
- Missing dry run — warns if no successful dry run on record

**Re-running `/migrate` is safe.** If the issue already reached
`status:migrate-success` and the destination repo is still there, GEI
refuses with "No operation will be performed". AMP recognises this as an
idempotent re-run, preserves the prior `status:migrate-success` label,
and posts an explanatory note pointing you at `/remigrate` if you
actually want to replace the destination. A `/migrate` that hits the
same error without a prior success on the issue is still treated as a
genuine failure — the destination conflict was not produced by AMP and
the operator needs to see it.

### GitLab migration (`gl-exporter`)

`gh gei` has no GitLab source, so a GitLab `/migrate` (and `/dryrun`)
runs a three-step flow instead of a single GEI subprocess — export →
upload → import — but is triggered identically (`cmd:migrate` /
`/migrate`) and produces the same status labels, verification, and
post-migration steps:

1. **Export** — `gl-exporter` (a Ruby tool, run as a Docker image)
   produces a `ghe-migrator`-format archive. Needs Docker on the runner
   and an **admin** `GITLAB_TOKEN`. Point `GL_EXPORTER_IMAGE` at a
   pullable image ref, or pre-build the `gl-exporter:1.7.14` tag.
2. **Upload** — the archive goes to GitHub-owned blob storage by default
  (no external account is required),
  or to Azure Blob when `AZURE_STORAGE_CONNECTION_STRING` is set.
3. **Import** — the GitHub migration GraphQL API ingests the archive
   (`GL_EXPORTER_ARCHIVE` source type) and AMP polls it to completion.

**Two prerequisites are specific to GitLab:**

- The destination org must be **authorized by GitHub for
  `GL_EXPORTER_ARCHIVE` migrations** — a gated feature arranged through a
  GitHub migration engagement. An unauthorized org fails fast with a
  clear remediation message.
- `GITLAB_URL` must match the URL the GitLab instance reports for its own
  projects — the importer validates the source URL against the archive.

### Migrating a repo and its submodules together

For a parent repo whose `.gitmodules` references sibling repos that are
*also* AMP-managed, applying the `cmd:migrate-with-submodules` label (or
posting `/migrate-with-submodules` as a comment) queues `/migrate` for
the parent **and** for each AMP-managed direct submodule in one action.
The handler:

- Enqueues the parent through the normal `/migrate` path (all preflight
  checks apply).
- Reads the submodules table from the parent issue body, resolves each
  URL to the sibling AMP issue via the same index used by the
  `amp/migration-fixes` rewriter.
- For each resolved sibling, checks whether the destination repo already
  exists at `<dst-org>/<dst-prefix>-<source-repo>`:
  - **Not yet at destination** → applies `cmd:migrate` to the sibling
    (the dispatcher claims it FIFO, same as any other `/migrate`).
  - **Already at destination** → skips queueing; reports the sibling as
    "already at destination — no action needed". This is *not* a
    failure. The destination repo's `created_at` / `pushed_at` are
    included so an operator can tell at a glance whether the existing
    repo is freshly migrated or years old (and therefore unlikely to be
    a recent re-creation).

Submodules that don't resolve to a single AMP issue are skipped with a
**categorised reason** so "why did only 3 of 5 submodules migrate?" is
answered without clicking into anything:

- **`no matching AMP issue`** — no sibling registered in the control repo
  for `<owner>/<repo>`.
- **`ambiguous`** — more than one AMP issue matches the same owner+repo;
  the operator must disambiguate (the offending issue numbers are
  listed).
- **`self-reference`** — the submodule URL resolves back to the parent
  issue itself.
- **`unparseable URL`** — the submodule URL is not in a form AMP knows
  how to parse (raw exception text is included).
- **`failed to apply cmd:migrate label`** — the GitHub API rejected the
  label apply (e.g. rate limit, permission). The exception class and
  message are surfaced.

Each queued child issue receives a small tracker comment pointing back
at the parent. When the child's `/migrate` finishes (success or
failure), the worker re-renders the parent's aggregated status comment
in place — so the parent issue ends up with **one rolling table** of
per-submodule status (✅ success / ❌ failed / ⏳ running / ⏭ skipped)
rather than N separate comments scattered across N child issues.
`/migrate-with-submodules` traverses **direct** submodules only (one
level deep).

#### Migrating the whole submodule tree: `/migrate-submodules-deep`

When a submodule has submodules of its own, applying the
`cmd:migrate-submodules-deep` label (or posting `/migrate-submodules-deep`
as a comment) does everything `/migrate-with-submodules` does, but
resolves the **full recursive descendant tree** — children, grandchildren,
and deeper — and queues `/migrate` for every AMP-managed descendant in one
action. The traversal is **cycle-safe and de-duplicated**: a submodule
cycle (A → B → A) terminates instead of looping, and a repo referenced by
two different parents (a "diamond") is queued exactly once. It reads each
descendant's submodule table from the issue snapshot the control repo
already loaded, so the recursion adds no extra API calls beyond the single
index scan. The aggregated status comment, skip-reason categories, and
already-migrated reporting are identical to the one-level command. Use
`/migrate-with-submodules` when you want only the immediate children, and
`/migrate-submodules-deep` when you want the entire dependency tree.

Re-running either command on the same parent posts a *new* aggregated
comment (each invocation is uniquely timestamped); older status comments
stay as an audit trail and are not updated by future completions.

## Migration Queue

GitHub caps concurrent repo migrations at 10 per destination org, and
dryruns count against the same cap. Without a queue, applying
`cmd:migrate` to 50 issues at once spawns 50 GitHub Actions workflows
that each block for the full migration duration — even when GitHub itself
has the migration parked. You pay Action minutes for runs that aren't
actually progressing. The queue prevents that.

**Default: queue is ON.** A fresh install gets capacity-aware dispatch
without any extra setup. Tune the cap (default 10) if you want a
different ceiling:

```sh
gh variable set AMP_MIGRATE_CONCURRENCY --body "10" --repo <owner/repo>
```

**How it works.** `/migrate`, `/dryrun`, and `/remigrate` don't run
inline. Instead they stamp a queue label (`status:migrate-queued` /
`status:dryrun-queued`) and exit in seconds. A separate dispatcher
(`migration-dispatcher.yml`) runs whenever:

- An issue gets a queue label (immediate drain)
- A `Migration Worker` workflow completes (free a slot)
- Every 15 min on a safety cron
- You manually run `gh workflow run migration-dispatcher.yml`

The dispatcher counts how many issues are currently `*-running`
(migrate + dryrun share the cap), claims `cap − running` queued issues
oldest-first by swapping their label to `*-running`, and dispatches one
`Migration Worker` workflow per claimed issue. Queued issues consume zero
Action minutes until they're claimed.

**Watch the queue:**

```sh
gh issue list --label "status:migrate-queued"   # waiting to start
gh issue list --label "status:dryrun-queued"
gh issue list --label "status:migrate-running"  # currently running
gh issue list --label "status:dryrun-running"
```

**Reconciler.** Every dispatcher tick scans for issues stuck in
`*-running` for >2 h with no live Migration Worker workflow and re-queues
them. Threshold is `MIGRATE_STUCK_THRESHOLD_MIN` (default 120). When the
issue carries an `<!-- amp-local-run -->` marker comment (see
*Running migrations locally*), the reconciler holds off until
`AMP_LOCAL_RUN_TIMEOUT_HOURS` (default 24) elapses so a long LFS push
isn't reclaimed mid-flight.

**Opt out.** If you'd rather run every command inline (pre-queue
behavior):

```sh
gh variable set AMP_QUEUE_ENABLED --body "false" --repo <owner/repo>
```

In-flight Migration Workers complete normally. New `cmd:migrate` labels run
inline as before. Any issues already `status:*-queued` are stranded —
either remove the variable (or set it back to anything other than
`false`) to drain them, or strip the queue label manually and re-add
`cmd:migrate` to run inline:

```sh
gh issue edit <N> --remove-label "status:migrate-queued"
gh issue edit <N> --add-label "cmd:migrate"
```

## Running Migrations Locally

When a GitHub-hosted runner can't finish a migration — LFS disk pressure
on shared runners, the 6 h job timeout on very large repos, queue
saturation during a launch, transient API rate limits — `amp run` lets
the operator execute the same migration from their workstation. The
migration code is identical to what `migration-worker.yml` runs; labels
and comments on the AMP issue still update the same way, so the
Kanban / dispatcher / reconciler all see consistent state.

### One-time setup

Each script is idempotent: it probes for every dependency and installs
only what's missing. Running it twice on the same machine is a no-op
the second time.

```sh
# macOS / Linux
scripts/setup-local.sh

# Windows (best-effort; tested on PowerShell 7 + winget)
pwsh -File scripts/setup-local.ps1
```

The script checks Python 3.11+, installs `amp` editable, ensures the
GitHub CLI is on PATH, probes for `gh-gei` / `gh-ado2gh` / `gh-bbs2gh` /
`gh-bbc` (installs only what's missing), and verifies `git` plus
`git-lfs`. It finishes by printing a copy-paste template of the env
vars you need to export.

Tokens follow the same precedence as CI ([CLAUDE.md](../CLAUDE.md)
*Token resolution*). There are **three distinct token domains** —
do not reuse the same PAT for all three unless your source and
destination are the same tenant:

| Domain | Purpose | Set one of |
| --- | --- | --- |
| **Source** | Reach the repo being migrated *from* | `GH_PAT` (GitHub source); `ADO_PAT`; `GITLAB_TOKEN`; `BBS_PAT` + `BBS_URL`; `BITBUCKET_API_TOKEN` + `BITBUCKET_EMAIL` |
| **Destination** | Create / write the repo at the GHEC / GHEC-DR tenant | `APP_ID` + `APP_PRIVATE_KEY`, *or* `GH_DEST_PAT` |
| **Control** | Read / label the AMP issue | Single-tenant: same as Source. GHEC-DR: reused from Destination automatically when `GH_DEST_API_URL` is set. |
| GHEC-DR routing | Route both Destination and Control ops to the DR tenant | `GH_DEST_API_URL=https://api.<tenant>.ghe.com` |
| LFS staging dir (optional, off default `/mnt/amp-lfs`) | macOS-friendly LFS path | `AMP_LFS_STORAGE_DIR=$HOME/.amp/lfs` |

For a GHEC-DR install, `GH_PAT` stays pointed at the *source* platform
(often a github.com PAT). `amp run` detects DR mode from
`GH_DEST_API_URL` and reuses `GH_DEST_PAT` (or App auth) to reach the
control repo on the DR tenant — there is no separate "control PAT" to
configure.

### Auth diagnostic on errors

When any `amp` command fails on an auth-related condition (missing
token, insufficient scopes, or a 401 / 403 from the GitHub API), the
CLI automatically prints an **auth inventory** of every credential env
var grouped by domain (Source / Destination / Control), each with a
redacted preview (`ghp_…1234`) so the operator can confirm the right
PAT is in the right slot. GitHub-style PATs that are set also get a
live probe (`GET /user` against the appropriate API URL, 5 s timeout)
labelled `✓ valid (…)`, `✗ invalid (401 on …)`, or
`? unreachable (…)` — so a wrong PAT-to-tenant pairing or a typo'd
`GH_DEST_API_URL` is visible immediately rather than after the fourth
failed migration. The diagnostic only runs on the error path; non-auth
commands and successful invocations don't pay the probe cost.

### Pre-flight PAT validation

`/migrate`, `/dryrun`, and `/remigrate` each whoami-probe the resolved
source and destination tokens *before* claiming the issue, flipping
`status:*-running`, posting the claim marker, cloning the source
baseline, deleting the sandbox/target repo, or invoking `gh gei`. A
`401` or `403` from either probe returns a blocker comment naming the
exact env var to refresh — `GH_PAT`, `ADO_PAT`, `GITLAB_TOKEN`,
`BBS_PAT`, `BITBUCKET_API_TOKEN` + `BITBUCKET_EMAIL`, or `GH_DEST_PAT`
/ App credentials. Network errors, 5xx, and non-auth 4xx are treated
as inconclusive (logged, not blocked). Set `AMP_SKIP_TOKEN_PREFLIGHT=true`
to bypass — useful during incidents where the probe endpoints are
themselves degraded. With the check in place, an expired PAT fails the
command in a few seconds rather than after a minute of source-baseline
cloning and a `gh gei` `PENDING_VALIDATION` poll cycle that ends in
"Your PAT is invalid".

### Running

```sh
# Dry-run an analyzed issue locally instead of waiting on a runner.
amp run --issue 42 --repo myorg/amp-control --command dryrun

# Cut over for real.
amp run --issue 42 --repo myorg/amp-control --command migrate

# Re-migrate (force re-creation at destination, e.g. after a failed first attempt).
amp run --issue 42 --repo myorg/amp-control --command remigrate

# Migrate a parent repo AND each AMP-managed direct submodule, locally, in one pass.
amp run --issue 42 --repo myorg/amp-control --command migrate-with-submodules

# Migrate a parent repo AND its full recursive submodule tree (deepest-first).
amp run --issue 42 --repo myorg/amp-control --command migrate-submodules-deep

# Validate environment + tokens without doing anything destructive.
amp run --issue 42 --repo myorg/amp-control --command migrate --dry-run
```

When invoked, `amp run`:

1. Runs preflight: probes binaries, resolves tokens, fetches the issue.
   The only state preflight refuses is one already `status:*-running` —
   a CI worker or another local run may be migrating it right now; pass
   `--force` to override. Every other state (`analyzed`, no status,
   `dryrun-*`, `*-failed`, `*-queued`, `*-success`) runs without `--force`.
2. Posts an `<!-- amp-local-run -->` marker comment with the operator
   name, hostname, and start time.
3. Flips the status label to `*-running`.
4. Calls `amp.commands.dispatch("run-<verb>", ...)` — byte-for-byte the
   same call `migration-worker.yml` makes.
5. The handler stamps `*-success` or `*-failed` itself on completion.

#### Migrating a submodule tree locally

`--command migrate-with-submodules` is the laptop equivalent of the
`/migrate-with-submodules` IssueOps command. Where the IssueOps version
*enqueues* the parent + each AMP-managed submodule for CI workers to run,
the `amp run` version **executes every migration on the laptop**, in
sequence:

- **Plan + confirmation first.** Before any migration starts, `amp run`
  prints the full resolved run order — every submodule child plus the
  parent — and prompts `Proceed — migrate all of the above? [y/N]`. If
  something looks wrong (a submodule missing, an unexpected target),
  answer `n` and nothing is touched. Pass `--yes` to skip the prompt
  (the plan is still printed); `--dry-run` prints the plan and exits. If
  every issue in the tree is already migrated there is nothing to run, so
  the plan is printed and the command exits without prompting.
- **Children first, parent last.** Submodules migrate before the parent so
  the parent's `amp/migration-fixes` branch can rewrite submodule URLs to
  point at already-migrated siblings.
- **Continue-on-failure.** A failed submodule is logged; the run proceeds
  to the remaining submodules and still attempts the parent. The command
  exits non-zero if anything failed.
- **Skips are not failures.** A submodule with no AMP issue, an ambiguous
  match, a self-reference, or an unparseable URL is reported as *skipped*.
  A submodule — or the parent — whose destination repo **already exists**
  is reported as *already-migrated* and is not run: this is a `migrate`,
  not a `remigrate`, so an existing destination is never overwritten. Use
  `/remigrate` to replace a destination. `--force` does **not** change
  this — it only overrides the `status:*-running` claim guard (so you can
  re-run after a prior local attempt left an issue claimed); it never
  re-runs an already-migrated destination.
- **Summary.** A per-issue table (migrated / failed / skipped /
  already-migrated) prints at the end. Each issue also gets its own
  per-issue comment exactly as a single `amp run --command migrate` would.
- `--dry-run` resolves the tree and prints the planned run order without
  claiming or migrating anything.

A parent with no submodules behaves exactly like `amp run --command
migrate`.

`--command migrate-submodules-deep` is the laptop equivalent of the
`/migrate-submodules-deep` IssueOps command: it resolves the **full
recursive descendant tree** rather than just the direct children, and
migrates **deepest-first** (a grandchild before its parent, that parent
before the top repo) so every repo's `amp/migration-fixes` branch can
rewrite submodule URLs to siblings that are already live. The traversal is
cycle-safe and de-duplicated. Everything else — the plan/confirm prompt,
continue-on-failure, skip vs. already-migrated handling, `--yes`,
`--dry-run`, and `--force` semantics — is identical to
`migrate-with-submodules`.

### Coordination with CI

`amp run` and the CI workers can't both work the same issue at once.
Once the local-run marker is posted and the status flips to `*-running`,
the dispatcher won't claim it. The reconciler ignores `*-running` issues
that carry the marker until `AMP_LOCAL_RUN_TIMEOUT_HOURS` (default 24)
elapses — after that it assumes the laptop is forgotten and reclaims the
issue. Set the variable longer if you expect very long pushes.

### Aborting

Ctrl-C in the middle of a local run posts an
`<!-- amp-local-run-aborted -->` marker and rolls the status back to
`*-queued`. A CI worker can then pick it up again, or you can re-run
locally with `amp run`. If `amp run` itself crashes hard (process
killed, OS reboot), the issue may stay in `*-running` until the
reconciler reclaims it — clear the label manually with `gh issue edit`
to recover sooner.

### Flags

- `--force` — proceed even if the issue is already `*-running`. Use only
  when you're certain no CI worker is active (check the Actions tab).
- `--skip-claim` — don't touch labels or post the marker. Useful for
  resuming after a partial run that already left the issue `*-running`.
- `--dry-run` — preflight only, no side effects.

### Windows caveats

`scripts/setup-local.ps1` is a best-effort port of the bash script. It's
been smoked on PowerShell 7 + winget on Windows 11; older Windows or
PowerShell 5.1 may require manual installs. The `amp run` command itself
is pure Python and works identically on Windows. The bash helpers in
`scripts/` (other than `setup-local.sh`) are not ported — they're CI-only
tooling.

## Step 6: Post-Migration (Automatic)

After every successful `/dryrun` and every successful `/migrate`, AMP runs
several best-effort post-migration steps (migration-fixes branch, custom
properties, environments, secret/variable stubs, branch protection). Each
can fail independently without affecting the underlying migration's
`migrate-success` status — a failure surfaces as a 🟧 **warning** row in the
verification block and raises the additive `status:has-warnings` flag, never
a rollback or a `migrate-failed`. Re-run the matching `/upsert-*` command to
finish a warned step. Only a content/structural verification mismatch
(branch / tag / commit-count divergence, an unreachable destination, a
source-freeze violation) fails the migration. When `gh gei` migrated the
repo with the source's **archived** flag preserved, AMP temporarily
unarchives the destination for the duration of these writes and restores
the archived state afterwards.

Outcomes from every check (migration-fixes, validation, LFS,
custom-properties) are aggregated into a **single consolidated comment**
on the AMP issue, with bullets grouped and ordered ✅ pass → 🟨 skip →
🟧 warn → ❌ fail so the operator can scan the headline state at a glance. The
long-form output (server probe excerpts, file lists, validation deltas)
is tucked behind a collapsible "Verification details" section in the
same comment.

### Migration-fixes branch

AMP pushes a branch named `amp/migration-fixes` to the destination repo
containing rewrites for cross-repo references that GEI does not handle:

- `.gitmodules` URLs pointing at the source SCM host are rewritten to the
  predicted destination location (only when the sibling repo is also
  AMP-managed — `dst-org:` and `dst-prefix:` on its AMP issue drive the new
  path).
- `.github/workflows/*.yml` cross-repo `uses:` references (reusable
  workflows, composite actions) are rewritten the same way.
- `CODEOWNERS` `@<src-org>/<team>` mentions are rewritten to
  `@<dst-org>/<team>`. AMP first looks up the destination org from any
  AMP issue on the same source org, then probes the destination org for a
  team with that exact slug; only when both checks pass is the line
  rewritten. Mentions whose source org isn't AMP-managed, or whose team
  doesn't exist at the destination, are left alone and surfaced in the
  destination-repo handoff issue (next section) for manual follow-up.

The branch is created but **no PR is opened** — repo owners review the diff
against the default branch and merge if it looks right. On dryrun, the
branch lands on the sandbox destination (with sandbox-prefixed sibling
names so the preview is faithful); on `/migrate`, it lands on the real
destination.

Outcomes:

- `migration-fixes:pushed` — branch pushed; comment includes the URL and
  the rewrite summary.
- `migration-fixes:none` — nothing needed rewriting; any stale branch from
  a previous run is deleted.
- `migration-fixes:failed` — push failed; comment includes the error.
  Migration's own success is preserved.

To disable: set `AMP_DISABLE_MIGRATION_FIXES=true` (case-sensitive) on the
runner environment.

### Post-migration validation (`/dryrun` and `/migrate`)

AMP compares the destination repo against the source on branches, tags,
default-branch commit count, and the first/last-commit SHAs on the
default branch. The `amp/migration-fixes` branch is excluded from the
destination branch count so AMP's own post-migration branch doesn't fail
the assertion.

**Source baseline snapshot.** Migrations can run for hours on large
repos, and an actively-developed source can gain branches or commits in
that window. To keep verification correct in those cases, AMP captures
a refs-only **source baseline** at the moment GEI starts (a
`git clone --bare --filter=blob:none` run in parallel with GEI so its
cost is hidden behind the migration itself). The verifier compares the
destination against that baseline, not against whatever source looks like
when verification runs. The baseline is persisted to a hidden marker on
the AMP issue body (`<!-- amp:source-baseline: {...} -->`) for audit and
re-runs.

**Source freeze advisory.** AMP also re-reads source at verification
time and emits a separate **Source freeze** check that shows up as 🟨
when source moved during the migration (branches added or deleted, tip
moved). This does not fail the run — the dest is verified against the
baseline regardless — but it surfaces source-freeze violations so the
operator can flag them with the team. When the team honored the freeze,
the check shows ✅.

Outcomes:

- `status:validate-passed` — all counts match the baseline.
- `status:validate-failed` — comment shows source vs destination delta.
- `status:validate-history-rewritten` — `lfs:oversized` is set, meaning the
  operator had to run a pre-migration rewrite (`git lfs migrate import`
  / `git filter-repo`) to bring files >100MB into LFS or remove them
  before GitHub would accept the push. SHA / commit-count assertions
  are skipped by design in that case, but the structural checks
  (branch/tag counts) still run. `lfs:active` and `lfs:historical` do
  not trigger this — AMP's LFS migration path transfers objects via
  the Batch API and preserves commit SHAs.

### LFS server-side verification (`/dryrun` and `/migrate`)

For every repo labeled `lfs:active`, AMP runs a deeper LFS check than the
pre-migration analyzer's pointer count. After the migration finishes (or
the dryrun's sandbox repo is created), AMP:

1. Lists every LFS pointer in the source's git tree across all refs
   (OID + size, via `git lfs ls-files --json --all`).
2. Calls the **destination** LFS server's Batch API for those OIDs and
   classifies each as present (server returned a `download` action) or
   missing (server returned a 404 or omitted the action block).
3. If any are missing, fetches the objects from the source and pushes
   them to the destination LFS server (`git lfs fetch --all` then
   `git lfs push --all`), then re-probes the Batch API.

This catches a class of silent failure that pointer-count comparisons
cannot: `gh gei` migrates the git tree but does **not** copy LFS objects
between separate LFS servers on cross-tenant migrations, so the source
and destination will report the same pointer count even when zero
binaries transferred. Running this in dryrun mode also exercises the
push path before the real migration, so an LFS-server quota or auth
failure surfaces against the sandbox repo instead of the production cut.

Outcome appears in the consolidated `/dryrun` or `/migrate` success
comment as one of (`N` = source LFS object count, `D` = destination LFS
object count — the values are spelled out side-by-side so it's obvious
they match):

- `✅ LFS files — source: N · destination: N — all objects verified on
  destination LFS server`
- `✅ LFS files — source: N · destination: N — AMP pushed M missing
  object(s)` (GEI delivered some; AMP closed the gap)
- `❌ LFS files — source: N · destination: D — M object(s) still missing
  after push` (objects could not be delivered)
- `❌ LFS files — source: N · destination: unknown — LFS server probe
  failed: …` (the Batch API itself was unreachable; AMP refuses to
  silently pass)

The fetch step uses the **source token** (e.g. `GH_PAT` for github.com /
GHEC source) — it needs `repo` scope (classic PAT) or `Contents: read`
(fine-grained PAT). Source LFS auth uses the same path as `git fetch`, so
anything that can clone the repo can also read its LFS objects. Transient
fetch failures (network/5xx) are retried once; auth/permission failures
(`git-lfs` exit 1) fail fast so the operator can fix the token instead of
waiting through a retry. The summary line in the workflow log shows the
most diagnostic `git-lfs` error line; the full multi-line stderr is
included in the AMP issue comment's check details.

#### Disk headroom

`git lfs fetch --all` on a multi-GB LFS repo would normally fill the
runner's `/tmp` and abort partway through with hundreds of `no space left
on device` errors. AMP avoids this by passing `-c lfs.storage=<dir>` to
git-lfs so the LFS object store lives outside the clone — by default on
`/mnt/amp-lfs` (the Azure ephemeral disk on Linux GHA runners has tens of
GB free, vs. `/tmp` on the OS volume).

Two complementary guards protect against the disk-out case:

1. **Workflow pre-flight** — `migration-worker.yml` runs a step before
   any `amp` code that reads the analyzed `LFS storage` value from the
   issue body, `df`'s the storage volume, and fails the run early (with
   a comment on the issue) if free space is below `payload × 1.5`. So a
   too-small volume is caught in under a minute, not after an hour of
   half-downloaded objects.
2. **Python pre-flight in `_fetch_then_push_lfs`** — the same check runs
   inside Python so local `amp` invocations and any caller outside the
   migration-worker workflow also fail fast.

Tuning knobs (all repo variables / environment variables):

- `AMP_LFS_STORAGE_DIR` — absolute path for the LFS object store. Defaults
  to `/mnt/amp-lfs` in the worker workflow. Override for self-hosted
  runners whose large volume is mounted elsewhere.
- `AMP_LFS_DISK_HEADROOM_FACTOR` — multiplier on the analyzed payload for
  the free-space comparison. Default `1.5` (covers the `incomplete/` ↔
  `objects/` overlap during fetch plus the push buffer).
- `AMP_LFS_DISK_PRECHECK=false` — escape hatch that disables the Python
  pre-flight only. The workflow step still runs.
- `AMP_LFS_DEBUG=1` — enables `GIT_TRACE=1` + `GIT_CURL_VERBOSE=1` on the
  LFS fetch/push subprocesses. Heavy output (token bytes are scrubbed)
  but the only way to see per-object TLS / SAML / 5xx detail when the
  destination LFS server silently drops objects. Use when a `/dryrun` or
  `/migrate` produces an `lfs:dest-missing` label and you need to know
  why the per-object pushes failed. Off by default.

If the pre-flight fails, point `AMP_LFS_STORAGE_DIR` at a larger volume
(or move the workflow to a runner with more disk) and re-queue the
command.

### Secrets / variables stubs (`/dryrun` and `/migrate`, GitHub source only)

For every `secrets.X` / `vars.X` reference detected in source workflow YAML at
analyze time, AMP creates a placeholder stub at the destination so workflows
have something to bind to once owners fill in real values:

- **Repo-level vs. environment-level scope** is inferred from the surrounding
  job's `environment:` declaration. Env-scoped stubs are written under the
  matching destination environment (recreated first — see below).
- **Idempotent**: GET-first on every name. If a secret/variable is already
  present at the destination, AMP **does not overwrite it** — operators who
  pre-populated real values are safe.
- **Placeholder value**: `AMP_PLACEHOLDER_REPLACE_ME` (referenced in this
  doc as `AMP_ PLACEHOLDER_ REPLACE_ ME` with broken-up formatting so the
  literal token remains greppable). Secrets are libsodium sealed-box
  encrypted with the repo (or env) public key; variables go in plaintext.

**Diagnosing unset secrets in failed workflow runs:** GitHub Actions' log
scrubber redacts secret values, so the placeholder string appears as `***`
in any job log — operators can't grep for the literal string in the log.
Instead, the post-migrate comment lists **every secret AMP stubbed by
name**; cross-reference that list against the secret name in the failure
to identify "still has placeholder" vs "real value but wrong."

Outcomes:

- `secrets-recreate:created` — every stub was missing; AMP created them all.
- `secrets-recreate:skipped` — every name already existed at the destination
  (nothing to do; existing values preserved).
- `secrets-recreate:partial` — mix of created + skipped, possibly with
  per-name failures. Comment lists each bucket.
- `secrets-recreate:failed` — could not create any stubs (typically the
  destination token lacks `repo:actions` scope). Comment lists per-name
  errors.
- `secrets-recreate:rate-limit-deferred` — destination API rate limit too
  low to safely run; re-trigger later via `cmd:upsert-secrets`.

**Org-level secrets/vars** on the source org are surfaced as FYI only —
AMP does **not** write at org scope (admin write is rarely held by the
migration runner). They're listed in the AMP issue body and the destination
migration-report issue with a "coordinate centrally" note.

Re-trigger after manually creating any missing environments at the
destination by applying the `cmd:upsert-secrets` label or posting
`/upsert-secrets`. Safe to spam (idempotent).

### Environments recreate (`/dryrun` and `/migrate`, GitHub source only)

For each GitHub Actions Environment on the source repo at analyze time, AMP
recreates the environment at the destination with its full protection
configuration:

- Wait timer
- Prevent self-review toggle
- Required reviewers (users + teams)
- Deployment branch policy (protected-branches-only OR custom patterns)
- Per-environment custom branch policy patterns

The AMP issue body also surfaces (display-only — not migrated by the env
recreate step itself):

- Configured env-scoped **secret names** (values are never readable; recreate
  manually with real values — empty placeholders are created by the separate
  `cmd:upsert-secrets` step).
- Configured env-scoped **variable name=value pairs** (variables are not
  secret; values come through cleanly).
- **Custom deployment protection rules** (third-party app gates such as
  Datadog / ServiceNow / Slack approvals). AMP **cannot** recreate these
  cross-tenant — operators must re-install the relevant app at the
  destination and re-attach the rule manually.
- **Workflow refs** — cross-link of the env-scoped secrets/vars referenced
  inside workflow YAML, so the env block shows actual usage in context.
- **`Open env in GitHub UI`** link — clickable link straight to the source
  env settings page.

When an env has no protection rules at all (no wait timer, no reviewers, no
branch policy), the block renders an explicit `_No protection rules
configured — open to all branches, no reviewers._` line so the section never
looks empty.

**Reviewers — try, fall back, surface broadly.** Reviewer user/team IDs are
tenant-scoped. AMP attempts the source reviewers as-is on the first PUT; on
422 ("invalid reviewer") it retries without `reviewers` and flags the env
as `reviewers_stripped` so the post-migrate comment lists which envs need
manual reviewer re-add. Other protections (wait timer, branch policies) are
still applied. This is the common path for cross-tenant migrations (GHEC →
GHEC-EMU / GHEC-DR).

**Idempotent**: GET-first per env. If the env already exists at the
destination, AMP leaves it alone — operators may have intentionally pruned
some protections, and an unconditional PUT would clobber that. Deployment
branch policies replay POST-only-missing (by name).

Outcomes:

- `environments:created` — every env was missing; AMP created them all.
- `environments:partial` — some envs were skipped (already existed) or had
  reviewers stripped or failed. Comment lists each.
- `environments:failed` — could not create any environments. Comment lists
  per-env errors.
- `environments:none` — source had no environments.
- `environments:rate-limit-deferred` — re-trigger later via
  `cmd:upsert-envs`.

Re-trigger via `cmd:upsert-envs` (or `/upsert-envs`).
Safe to spam.

### Custom-properties carry-through (`/migrate`, GitHub source only)

AMP reads custom repository properties off the AMP issue body and applies
the values to the destination repo. If the destination org doesn't yet
have a property schema defined (HTTP 422 from GitHub), AMP attempts to
**create the schema** at the destination org level — using the type,
allowed values, and description captured at analyze time — and then
retries setting the value.

Outcomes:

- `custom-properties:applied` — values set against an existing schema.
- `custom-properties:created+applied` — AMP created the schema at the
  destination org, then set the values.
- `custom-properties:dest-not-set` — schema creation failed (typically a
  permissions issue — the dest token lacks `admin:org`). Migration's own
  `migrate-success` is preserved. Comment lists the failed property names.

### Destination-repo migration report issue (`/dryrun` and `/migrate`)

After a successful `/migrate` AMP opens an issue **on the destination
repo** titled `AMP migration report — YYYY-MM-DD` (today's date, ISO
format). After `/dryrun` it opens the equivalent issue on the sandbox
repo titled `AMP dry-run preview — YYYY-MM-DD` so the operator can
preview exactly what the repo owner will see post-migration.

The body is a single consolidated handoff document scoped for the repo
owner — who typically has no access to the AMP control plane:

- **Verification block** at the top: pass / skip / warn / fail counts plus a
  per-check table with explanations. 🟧 warn rows mark best-effort
  post-migration steps that didn't fully land — the migration still
  succeeded. The structural count rows
  (Releases, Tag count, Branch count, Commit count) render their result
  as a click-through link to the matching destination-repo page, so the
  reader can confirm the number without leaving the issue.
- **What was auto-fixed** summary: bulleted list of submodule URL,
  workflow `uses:`, and CODEOWNERS team rewrites that landed on the
  `amp/migration-fixes` branch. AMP opens a pull request for that
  branch; the summary (and each auto-corrected section's banner) links
  the PR and flags that it is awaiting review and ready to merge once it
  looks right. If the PR could not be created, the summary links the
  branch itself instead.
- **What you need to do** checklist: one bullet per analysis section
  that still requires manual attention (secrets to recreate, branch
  protection, ref hygiene, hardcoded URLs, etc.).
- **Migration warnings to review** section (only when the GEI run
  produced warnings): rendered directly under the checklist. It
  explains, in plain language, what each common warning category means
  — pull-request review threads and commit comments that GitHub could
  not recreate in their exact form and preserved as plain comments —
  and reassures that they are almost always cosmetic (code, branches,
  tags and history are unaffected). The raw warning log is included in
  a nested collapsed block, capped to keep the issue scannable, with a
  link to GitHub's migration troubleshooting guide.
- **Per-section detail** mirrored from the AMP control issue: every
  analysis section (Source, Repository Analysis, Submodules, Workflow
  Cross-Repo References, CODEOWNERS, Recreate-at-destination, Branch
  Protection, Hardcoded URLs, Ref Hygiene, LFS Analysis, Custom
  Properties, Recent committers) wrapped in a `<details>` collapsed by
  default. Each carries a status banner: ✅ auto-corrected, 🟨
  partially auto-corrected, ⚠️ manual action required, or ℹ️
  informational.

Operator-only AMP content (the routing-labels italics, the commands
reference, the hidden `<!-- amp-id -->` marker) is stripped — none of it
is useful to the repo owner. If the AMP issue body has been hand-edited
and the `<!-- amp:begin -->` / `<!-- amp:end -->` sentinels are missing,
AMP falls back to the verification-only block and notes the gap inline.

The issue is tagged with an `amp-migration` label, which AMP auto-creates
on the destination repo the first time it's needed (green, description
"Auto-created by the AMP platform"). On `/dryrun` the issue lives until
the next dryrun (when the sandbox repo and everything in it is wiped);
on `/migrate` it stays as a durable handoff record.

Best-effort: if the dest-repo issue cannot be created (permissions,
network, etc.), AMP logs a warning and the migration's success state is
preserved. The AMP-issue comment still has the full report. Re-running
`/migrate` against the same destination creates a new dated issue each
time; duplicates accumulate quietly and can be closed
manually.

### `/upsert-teams`

After the three steps above, the post-migrate workflow runs `/upsert-teams`. If the issue carries a `dst-team:<base>`
label, AMP ensures three teams exist in the destination org and attaches
the migrated repo to each at the appropriate permission tier. If there's
no `dst-team:`, this step no-ops cleanly.

| Team | Permission attached to repo |
|---|---|
| `<base>-read` | pull |
| `<base>-write` | push |
| `<base>-admin` | admin |

Membership in the three teams determines who can read, write, or
administer the repo, so permissions can be adjusted simply by moving
people between teams. Teams that don't yet exist are created as private
(closed) teams. For ADO sources, the workflow also posts a checklist for
autolink references and pipeline rewiring.

**SCIM/EMU customers** (where teams are provisioned by the IdP and can't
be created via API — including all GHEC-DR tenants) rely on a fallback
path: AMP attempts `create_team`, catches the 422 that EMU returns, and
falls back to `get_team_by_slug`. As long as SCIM has provisioned
`<base>-read`, `<base>-write`, and `<base>-admin` before `/upsert-teams`
runs, this works correctly. A future `team_management: scim` mode will
make this explicit and surface missing teams as a
`status:waiting-on-scim-team` label.

## Step 7: Archive Source

Apply the `cmd:archive` label or post `/archive` to lock and archive the
source repository. Behavior is provider-specific:

- **GitHub** — repo is archived (`archived=true`).
- **GitLab** — project is archived.
- **Azure DevOps** — repo is disabled (ADO has no archive concept).
- **Bitbucket Server** — repo is set to read-only.
- **Bitbucket Cloud** — no per-repo equivalent; AMP marks the issue archived
  and logs a warning. Use workspace access controls if write-blocking is
  required.

If you need to undo an archive (e.g. premature archive, or a stakeholder
asks for write access back on the source), apply `cmd:unarchive` or post
`/unarchive`. This reverses the provider-specific operation above and
flips the issue's status back to `status:migrate-success` so dashboards
stop counting the repo as archived. On failure (e.g. token lacks admin),
the status is left untouched.

# GHEC-DR Support

AMP supports targeting a GitHub Enterprise Cloud with Data Residency tenant
(`https://<tenant>.ghe.com`, API at `https://api.<tenant>.ghe.com`) as the
migration destination. Set the `GH_DEST_API_URL` environment variable (or
the corresponding GitHub Actions repo variable) to the tenant's API URL,
and AMP will:

- Pass `--target-api-url` to `gh gei`/`gh ado2gh`/`gh bbs2gh` so migrations
  land on the DR tenant rather than github.com.
- Route post-migration PyGithub calls (upsert-teams, set-properties) and
  GraphQL projects calls to the DR tenant.
- Generate `target_repo_url` links in success comments that point at the
  DR host instead of github.com.

When `GH_DEST_API_URL` is unset (or empty) the destination defaults to
`https://api.github.com` — vanilla GHEC migrations require zero new
configuration.

For a step-by-step install runbook covering secrets, variables, and the
SCIM team prerequisite, see
[`CLIENT_RUNBOOK_GHEC_DR.md`](CLIENT_RUNBOOK_GHEC_DR.md).

# Bitbucket Cloud Source

Bitbucket Cloud (bitbucket.org) is supported via the third-party
[`gh bbc-exporter`](https://github.com/katiem0/gh-bbc-exporter) extension,
which exports a workspace repo and imports it to a GitHub destination using
the GEI archive format. The IssueOps workflow auto-installs this extension
in CI; for local use install it once with
`gh extension install katiem0/gh-bbc-exporter`.

**Required credentials**

| Type | Name | Purpose |
|---|---|---|
| Secret | `BITBUCKET_API_TOKEN` | Bitbucket Cloud API token with workspace + PR read scope. App passwords are deprecated by Atlassian on 2026-06-09; use API tokens. |
| Variable | `BITBUCKET_EMAIL` | Email associated with the API token (used as the HTTP Basic auth username). |

**Optional: throttling for large workspaces.** Bitbucket Cloud enforces a
1,000 req/hour baseline limit per authenticated user (scaling to up to
10,000/hr based on paid workspace seats). The analyzer handles 429
responses with exponential backoff, but a workspace with several hundred
repos can stretch a refresh across the rate-limit window. Set
`BBC_REQUEST_INTERVAL` (seconds, decimal allowed) to space requests out
proactively rather than reactively. Example: `BBC_REQUEST_INTERVAL=2`
keeps a 500-repo refresh comfortably under the baseline limit.

**Migration.** `/dryrun` and `/migrate` run `gh bbc-exporter migrate`, which
exports the workspace repo, uploads the archive to GitHub-owned storage, and
imports it — all in a single run. AMP passes the resolved destination token to
the exporter via the `GITHUB_PAT` environment variable (never on the command
line, so it can't leak into process listings). The destination repo is created
`private` unless `BBC_TARGET_REPO_VISIBILITY` (`public` / `private` /
`internal`) overrides it. Because export, upload, and import all happen in one
process, a large repo can run long — raise `AMP_MIGRATION_TIMEOUT_SECS`
(default `3600`) if a migration is killed mid-run. Post-migration verification
(branch / tag / commit counts against a pre-migration source baseline) runs for
Bitbucket Cloud the same as for GitHub sources.

**Known MVP gaps**

- `/archive` and `/unarchive` are no-ops for Bitbucket Cloud (no per-repo
  equivalent). Use workspace access controls if write-blocking is needed.
- Only GitHub Cloud destinations supported (api.github.com or
  `*.ghe.com`). GHES targets are rejected — `gh bbc-exporter` does not
  support GHES.
- 40 GiB GEI archive ceiling — the analyzer warns above ~30 GiB.
- Issues, wikis, and labels are not exported (Bitbucket data model
  diverges from GitHub's).
- Mannequin reclamation is a manual post-migration step.

# AI Failure Analysis & Auto-Repair

When any operation fails (dry run or migration), the AI advisor runs
automatically using a 3-tier diagnosis system:

**Tier 1 — Rules Engine:** Fast, deterministic pattern matching against
17 known failure patterns (expired PAT, repo already exists, large file
blocked, etc.). If confidence is 90% or higher, the diagnosis is accepted
immediately.

**Tier 2 — Agentic AI:** If rules don't match, an AI agent uses the
Copilot SDK with 6 diagnostic tools (search logs, check repo, check LFS,
check refs, check rate limits, get issue details) to investigate the
failure across up to 8 reasoning turns.

**Tier 3 — Heuristic Fallback:** Basic pattern detection when the AI
service is unavailable.

The advisor never modifies the source repo. When a repair is possible, it
clones the source, applies the fix, and pushes to a new repo with a
`-fixed` suffix. The operation is then retried against the fixed copy.

**Repair labels are scoped to the most recent failure.** When you trigger a
new `/dryrun`, `/migrate`, or `/remigrate` (whether the issue is queued or
runs immediately), AMP first clears any existing `cp-repair-available` or
`cp-repair-applied` label. This prevents stale repair proposals from a prior
failed attempt from lingering on the issue or the Kanban board while a
fresh attempt is in flight. If the new attempt fails, the advisor stamps
the appropriate `cp-repair-*` label as usual.

You can also run the advisor manually:

```bash
# Post analysis as a comment on the issue
amp advise --issue 42 --repo <owner>/<issueops-repo>

# Print to terminal only
amp advise --issue 42 --repo <owner>/<issueops-repo> --format terminal
```

# IssueOps Commands Reference

These commands can be posted as comments on any migration issue, or
triggered by applying the corresponding label.

| Command | Label | Description |
|---|---|---|
| `/dryrun` | `cmd:dryrun` | Dry run migration to sandbox org |
| `/migrate` | `cmd:migrate` | Production migration via GEI (requires `dst-org:` label) |
| `/remigrate` | `cmd:remigrate` | Delete target repo and re-migrate |
| `/archive` | `cmd:archive` | Archive/lock source repo |
| `/unarchive` | `cmd:unarchive` | Reverse `/archive`; reverts status to `migrate-success` on success |
| `/upsert-teams` | `cmd:upsert-teams` | Ensure `dst-team:<base>-read/-write/-admin` exist and attach repo |
| `/set-properties --props '{}'` | `cmd:set-properties` | Set custom properties on target |
| `/configure-autolink` | — | Set up autolink references (ADO) |
| `/update-issue` | `cmd:update-issue` | Re-run analysis, refresh issue |
| `/upsert-secrets` | `cmd:upsert-secrets` | Re-run secret/variable stub creation at destination (idempotent; safe to spam) |
| `/upsert-envs` | `cmd:upsert-envs` | Re-run GitHub Actions environment recreation at destination (idempotent; safe to spam) |
| `/upsert-protection` | `cmd:upsert-protection` | Re-run classic branch protection rule recreation at destination (GitHub source only; idempotent) |
| `/upsert-properties` | `cmd:upsert-properties` | Re-run custom-properties carry-through to destination (idempotent) |
| `/upsert-fixes` | `cmd:upsert-fixes` | Re-push `amp/migration-fixes` branch (submodule / actions / CODEOWNERS rewrites; idempotent) |
| `/upsert-validate` | `cmd:upsert-validate` | Re-run branch/tag/commit count validation against destination |

## Re-running individual post-migrate steps

Each `/upsert-*` command above re-runs one of the post-migrate steps that `/migrate` and `/dryrun` already run automatically. They are individually retriggerable so an operator can repeat a step that failed, was rate-limit-deferred, or needs to be re-run after fixing source data — without re-doing the underlying `gh gei` migration. All are idempotent: existing destination state is left alone (GET-first) and re-runs against an already-completed step are no-ops.

On success each step stamps a short `upserted:<name>` label on the issue (`upserted:teams`, `upserted:secrets`, `upserted:envs`, `upserted:protection`, `upserted:properties`, `upserted:validate`). `/upsert-fixes` decomposes into three sub-labels (`upserted:submodules`, `upserted:workflows`, `upserted:codeowners`) so an operator can see at a glance which of the three rewrite types actually applied. The longer detailed families (`migration-fixes:*`, `status:validate-*`, `secrets-recreate:*`, `environments:*`, `branch-protection:dest-*`, `custom-properties:*`) remain the source of truth for partial / failure / rate-limit-deferred / none states.

# CLI Commands Reference

All commands accept `--verbose` / `-v` for debug logging.

| Command | Description |
|---|---|
| `amp analyze` | Analyze source org, create one issue per repo |
| `amp refresh` | Re-analyze an existing IssueOps repo with missing-repo detection and obsolete-marking (analyze + safety guards) |
| `amp bulk-assign` | Apply `dst-org`/`dst-team`/`dst-prefix` labels to many issues from a CSV |
| `amp setup-labels` | Create or update all 50+ labels in the IssueOps repo |
| `amp setup-project` | Create a GitHub Project with Kanban views |
| `amp add-to-project` | Add migration issues to a GitHub Project |
| `amp configure-project` | Replace a project's Status options with the AMP migration set (destructive; requires `--force`) |
| `amp create-project-view` | Add a board / table / roadmap view to an existing project |
| `amp backfill-project-status` | Sync every open issue's `status:*` label to the project Status field (one-time) |
| `amp sync-project-status` | Sync one issue's `status:*` to the project Status field (invoked by `sync-project-status.yml`) |
| `amp export` | Export issues to an Excel workbook (3 sheets) |
| `amp export-analytics` | Export the analytics workbook — Data table, State Changes, native pivot views, in-table command builder |
| `amp state-history` | Dump one row per `status:`/`cmd:` state transition to CSV |
| `amp import-commands` | Generate `gh` CLI commands from an edited Excel file (classic or analytics layout) |
| `amp advise` | Run AI failure analysis on a specific issue |
| `amp generate-mannequin-csv` | Enumerate unclaimed mannequins for a destination org and write them to a CSV (org-wide) |
| `amp reclaim-mannequins` | Reclaim mannequins org-wide from a filled-in CSV |
| `amp daily` | Generate a daily activity report (terminal, markdown, or Excel) |
| `amp dashboard` | Show a migration summary in the terminal |

# Org-wide Post-migration Operations

Some post-migration cleanup steps operate on the **destination org as a
whole**, not a single migrated repo. These are deliberately exposed as
manually triggered workflows + CLI commands rather than IssueOps
commands — running them from a single repo's issue would create the
false impression that the effect is per-repo when it actually spans
every repo in the org.

## Mannequin reclamation

GEI creates a placeholder "mannequin" user at the destination for every
source commit author / PR actor that doesn't map to a known
destination GitHub identity. Mannequin records live at the **org**
level — one record per source identity, referenced across every
imported repo in that org. Reclaiming a mannequin rewrites
attributions everywhere the mannequin appears, in a single pass.

The reclaim flow is two steps:

1. **Generate the unmapped CSV** — `gh gei generate-mannequin-csv`
   wrapped by AMP. Produces a row per mannequin with an empty `target`
   column.
2. **Reclaim** — operator fills in the `target` GitHub login for every
   mannequin to be claimed, then `gh gei reclaim-mannequin` (wrapped
   by AMP) does the org-wide rewrite.

Both steps are exposed two ways — pick whichever fits your habit:

| Step | Workflow (GitHub UI) | CLI (laptop) |
|---|---|---|
| Generate | **Generate Mannequin CSV** workflow → downloads as artifact | `amp generate-mannequin-csv --org <dst> --output mannequins/<dst>.csv` |
| Reclaim | **Reclaim Mannequins** workflow (reads `mannequins/<org>.csv` from the repo) | `amp reclaim-mannequins --org <dst> --csv mannequins/<dst>.csv` |

Both surfaces call the shared backend in `amp/mannequins.py`, so
behavior cannot diverge.

Typical operator flow with the workflows: trigger **Generate Mannequin
CSV** with `target_org=<dst>` → download the
`mannequins-<dst>-<run-id>` artifact → fill in the GitHub-login column
→ commit it to `mannequins/<dst>.csv` on this repo → trigger **Reclaim
Mannequins** with `target_org=<dst>` and `csv_path=mannequins/<dst>.csv`.

Posting `/reclaim-mannequins` on a per-repo issue is deprecated: the
command now posts a one-shot pointer comment to this section and
takes no other action.

# Status Labels (Migration Lifecycle)

Status labels are mutually exclusive and track where each repo is in the
migration lifecycle.

| Label | Meaning |
|---|---|
| `status:analyzing` | Analysis in progress |
| `status:analyzed` | Analysis complete — ready for action |
| `status:empty` | Source repo has no commits / no default branch — migration not applicable. `/dryrun`, `/migrate`, `/remigrate` refuse to run. |
| `status:dryrun-queued` | Dry run queued |
| `status:dryrun-running` | Dry run in progress |
| `status:dryrun-success` | Dry run succeeded |
| `status:dryrun-failed` | Dry run failed — check AI advisor comment |
| `status:migrate-queued` | Migration queued |
| `status:migrate-running` | Migration in progress |
| `status:migrate-success` | Migration succeeded — post-migration runs automatically |
| `status:migrate-failed` | Migration failed — check AI advisor comment |
| `status:repairing` | Auto-repair in progress |
| `status:archiving` | Source repo archiving in progress |
| `status:archived` | Source repo archived |

# Daily Reports & Dashboard

AMP generates daily activity reports automatically (weekdays at 6 PM UTC)
and posts them to a pinned tracking issue. Activity counts cover the
US-Eastern calendar day and are a point-in-time snapshot — the report
header carries an `As of … ET` timestamp, since the scheduled run lands
mid-afternoon Eastern. You can also run reports manually:

```bash
# Terminal summary
amp daily --repo <owner>/<issueops-repo>

# Markdown file
amp daily --repo <owner>/<issueops-repo> --format md -o report.md

# Excel with pivot data
amp daily --repo <owner>/<issueops-repo> --format xlsx -o daily.xlsx

# Quick dashboard
amp dashboard --repo <owner>/<issueops-repo>
```

# GitHub Projects Integration

AMP has two complementary Projects v2 integrations: a one-time project
**creation** path (`setup-project` / `add-to-project`) and an ongoing
**status sync** that keeps a board's Status column in lock-step with each
issue's `status:*` label.

## Creating a board

```bash
amp setup-project --repo <owner>/<issueops-repo> \
    --name "Migration: ClientName" --views status,provider,risk,team

amp add-to-project --repo <owner>/<issueops-repo> \
    --project "Migration: ClientName"
```

The board columns follow the migration path: Analyzed → Dry Run →
Migrate → Re-migrate → Archived.

## Status sync (labels → Kanban column)

When you wire AMP to a project, every `issues:labeled` /
`issues:unlabeled` event triggers
[`sync-project-status.yml`](../.github/workflows/sync-project-status.yml),
which reads the issue's current labels, picks the highest-priority
`status:*` label, and writes the mapped value into the project's Status
single-select field. Labels remain the source of truth — the project's
Status is derived and never read back.

The highest-priority resolver keeps the board convergent when GitHub
webhooks arrive out of order (e.g. the dispatcher's `migrate-running`
event and the migration-worker's `migrate-success` event), so the card
settles on the terminal state regardless of delivery order.

### One-time operator setup

Configure two **repo variables** on the IssueOps repo, then provision
the project's Status options and backfill existing issues:

```bash
# 1. Tell AMP which project to sync into.
gh variable set AMP_PROJECT_OWNER  --body "<org-login>" --repo <owner>/<issueops-repo>
gh variable set AMP_PROJECT_NUMBER --body "<N>"         --repo <owner>/<issueops-repo>

# 2. Replace the project's Status options with the AMP migration set.
#    DESTRUCTIVE — wipes Todo/In Progress/Done and any custom options.
amp configure-project --owner <org-login> --number <N> --force

# 3. Populate the board with the current state of existing issues.
amp backfill-project-status --repo <owner>/<issueops-repo> \
    --owner <org-login> --number <N>
```

If either variable is unset, the sync workflow short-circuits in its
`if:` guard before allocating a runner, so the integration is fully
opt-in. On GHEC-DR, `GH_DEST_API_URL` must also point at the destination
tenant — GraphQL routes through the same destination client.

The token needs `project` scope (classic PAT) or `Projects:
read+write` (fine-grained PAT / GitHub App).

`configure-project` is destructive because GitHub's GraphQL API has no
non-destructive option-add for single-select fields —
`updateProjectV2Field` total-replaces. `--force` is required to
acknowledge this.

### Status column map

| `status:*` label | Project Status option |
|---|---|
| `status:analyzing` | Analyzing |
| `status:analyzed` | Analyzed |
| `status:analysis-failed` | Analysis Failed |
| `status:empty` | Empty |
| `status:dryrun-queued`, `status:dryrun-running` | Dry Run |
| `status:dryrun-success` | Dry Run Passed |
| `status:dryrun-failed` | Dry Run Failed |
| `status:migrate-queued`, `status:migrate-running` | Migrate |
| `status:migrate-success` | Migrated |
| `status:migrate-failed` | Migration Failed |
| `status:archiving`, `status:archived` | Archive |
| `status:obsolete`, `status:duplicate-candidate` | Needs Review |

`status:repairing` is intentionally a no-op — transient auto-repair
runs preserve the previous column.

## Custom views

`amp create-project-view` adds extra views to an existing project via
the REST API (board / table / roadmap layout, filter expression, visible
fields). `group-by` is set in the GitHub UI afterwards — the REST API
does not yet honor it.

```bash
amp create-project-view --owner <org-login> --number <N> \
    --name "By age" --layout board \
    --visible-fields "Status,Age,Title" --filter "is:open"
```

## Blocked view

`amp install` (and any `amp install` re-run against an existing project)
creates a **Blocked** table view that surfaces every repo with at least
one hard blocker — the same list that drives the **Blockers** column in
the Excel export.

Three project fields back the view, auto-populated by the existing
status-sync flow:

| Field | Type | Source |
|---|---|---|
| **Blockers** | Text | Comma-joined hard blockers (mirrors the export column) |
| **Warnings** | Text | Comma-joined soft items (`secrets:present`, `*:rewrite-needed`, …) |
| **Primary Blocker** | Single-select | Highest-severity hard blocker — `group_by` target |

The Blocked view filters on `has:"Primary Blocker"` (any repo with a
hard blocker set) and shows Title, Status, Primary Blocker, Blockers,
Warnings, Provider, Size. To group by blocker type, open the view and
click **⋯ → Group by → Primary Blocker** (one-time UI step; same REST
limitation as the Age view).

**Adopting on an existing project** (board predates this feature):

```bash
# 1) Add Blockers/Warnings/Primary Blocker fields (destructive on Status — required flag)
amp configure-project --owner <org-login> --number <N> --force

# 2) Backfill the new fields on every existing project item
amp backfill-project-status --repo <owner>/<issueops-repo> \
    --owner <org-login> --number <N>

# 3) Create the view itself
amp create-project-view --owner <org-login> --number <N> \
    --name "Blocked" --layout table \
    --visible-fields "Title,Status,Primary Blocker,Blockers,Warnings,Provider,Size" \
    --filter "has:\"Primary Blocker\""
```

To promote a warning to a blocker (or demote one), edit the tuples in
[amp/blockers.py](../amp/blockers.py) — no label rename or re-analysis
needed. Existing items pick up the new classification on the next
project sync (any label change on the issue triggers a re-sync).

# GitHub Actions Workflows

AMP uses GitHub Actions for automation. These workflows run in the
IssueOps repository:

| Workflow | Trigger | Purpose |
|---|---|---|
| `analyze.yml` | Manual (`workflow_dispatch`) | Analyze source org, create issues |
| `refresh.yml` | Manual (`workflow_dispatch`) | Re-analyze with missing-repo detection and obsolete-marking |
| `issue-ops.yml` | Issue comment created | Execute `/command` from comments |
| `label-ops.yml` | Issue labeled | Convert `cmd:*` labels into `/command` comments |
| `post-migrate.yml` | `status:migrate-success` label | Invoke `/upsert-teams` (creates+attaches read/write/admin teams if `dst-team` set) |
| `sync-project-status.yml` | `issues:labeled`, `issues:unlabeled` | Mirror the issue's highest-priority `status:*` label into the Projects v2 Status field (gated on `AMP_PROJECT_OWNER`/`AMP_PROJECT_NUMBER`) |
| `migration-dispatcher.yml` | Queue label applied, `Migration Worker` completion, opt-in hourly weekday cron (`AMP_DISPATCHER_CRON_ENABLED=true`), manual | Claim `cap − running` queued issues FIFO and fire one `Migration Worker` per claim; also reconciles stuck `*-running` issues older than 2 h |
| `migration-worker.yml` | Invoked by `migration-dispatcher.yml` | Execute one queued `/dryrun`, `/migrate`, or `/remigrate` against a single issue, then stamp the duration label |
| `export-issues.yml` | Manual + schedule | Export issues to an Excel workbook artifact (`amp export`) |
| `daily-report.yml` | Schedule (weekdays 6 PM UTC) + manual | Generate and post daily activity report |
| `cleanup-skipped-runs.yml` | Schedule (weekdays 5 AM & 7 PM Eastern; cron `0 9 * * 1-5` and `0 23 * * 1-5` UTC) + manual | Delete skipped workflow runs to keep the Actions UI tidy |
| `diagnose-secret-masking.yml` | Manual | Bisect log-masking issues caused by `GH_PAT` overlapping an org slug — replays setup steps individually instead of going through the `setup-amp` composite |

**About `cleanup-skipped-runs`:** every issue comment that doesn't start
with `/` and every label-add other than `status:migrate-success` creates
a skipped workflow run. Those add up. This workflow deletes them on a
twice-daily schedule (outside US business hours). Run it ad-hoc anytime
via `gh workflow run cleanup-skipped-runs.yml`. It's intentionally cron-only
— event-driven cleanup caused parallel invocations to drain the shared
PAT's API budget and break concurrent migrations.

# Required Secrets & Variables (CI)

Configure these on your IssueOps GitHub repository for the workflows to
function:

| Type | Name | Purpose |
|---|---|---|
| Secret | `GH_PAT` | Source GitHub personal access token |
| Secret | `GH_DEST_PAT` | Destination GitHub personal access token |
| Secret | `ADO_PAT` | Azure DevOps PAT (if migrating from ADO) |
| Secret | `AZURE_STORAGE_CONNECTION_STRING` | Optional Azure Blob connection string for archive uploads. Used by Bitbucket Server when external Azure storage is selected, and honored by GitLab to route the `gl-exporter` archive to Azure instead of GitHub-owned storage. **Not required for ADO or GitLab migrations when GitHub-owned storage is used**. |
| Secret | `MODELS_TOKEN` | GitHub token for AI analysis |
| Secret | `APP_PRIVATE_KEY` | GitHub App private key (optional, higher rate limits) |
| Variable | `APP_ID` | GitHub App ID (optional) |
| Variable | `GH_API_URL` | Source GitHub API URL — set to `https://api.<tenant>.ghe.com` when the source is a GHEC-DR / GHES tenant; leave unset for github.com |
| Variable | `GH_DEST_API_URL` | Destination API URL — set to `https://api.<tenant>.ghe.com` for GHEC-DR; leave unset for github.com |
| Variable | `SANDBOX_ORG` | GitHub org for dry-run migrations |
| Variable | `AMP_QUEUE_ENABLED` | Set to `false` to opt out of the migration queue (runs `/migrate` / `/dryrun` / `/remigrate` inline). Default on. |
| Variable | `AMP_DISPATCHER_CRON_ENABLED` | Set to `true` to enable the dispatcher's hourly weekday safety cron (reconciles workers stuck >2 h with no live run). Default off — event-driven triggers (queue label, worker completion, manual dispatch) fire regardless. Strict literal: only `"true"` enables. |
| Variable | `AMP_MIGRATE_CONCURRENCY` | Max concurrent `*-running` issues per destination org when the queue is on (default `10`) |
| Variable | `AMP_DISABLE_DEFAULT_REPO_PREFIX` | Set to `true` to stop `amp analyze` from injecting the per-provider default `dst-prefix:` label. Use for 1:1 source-org → destination-org migrations where target repo names should match source names. Default `false` (prefix on). |
| Variable | `AMP_PROJECT_OWNER` | Org login that owns the Projects v2 board used for status sync (unset → sync workflow is a no-op) |
| Variable | `AMP_PROJECT_NUMBER` | Project number from the URL `/orgs/<org>/projects/<N>` |
| Variable | `AMP_RUNNER` | Global override for `runs-on:` (e.g. `self-hosted` or a label group). Default `ubuntu-latest` |
| Variable | `AMP_RUNNER_<WORKFLOW>` | Per-workflow override (e.g. `AMP_RUNNER_ANALYZE`, `AMP_RUNNER_SLOT_RUNNER`); takes precedence over `AMP_RUNNER` |
| Variable | `AMP_LFS_STORAGE_DIR` | Absolute path where the LFS verify step writes its object store (passed to `git -c lfs.storage=…`). Defaults to `/mnt/amp-lfs` in `migration-worker.yml` so the heavy bytes use the Azure ephemeral disk on Linux GHA runners instead of `/tmp`. Override for self-hosted runners that mount a large volume elsewhere. |
| Variable | `AMP_LFS_DISK_HEADROOM_FACTOR` | Multiplier on the analyzed LFS payload for the pre-flight free-space check (default `1.5`). |
| Variable | `AMP_LFS_DEBUG` | Set to `1` to enable `GIT_TRACE=1` + `GIT_CURL_VERBOSE=1` on the LFS `fetch --all` / `push --all` subprocesses. Heavy output, scrubbed for token bytes — use only when diagnosing an `lfs:dest-missing` failure. Off by default. |
| Variable | `AMP_HEARTBEAT_INTERVAL` | Seconds between progress heartbeat log lines while a long step runs silently — verification clones, `git lfs fetch/push --all`, and the deep LFS analysis scans. Default `30`; set `0` to disable the heartbeat. |

# Operator Scripts

Standalone helpers in [`scripts/`](../scripts/) for ad-hoc operator tasks
that fall outside the IssueOps flow. They're invoked directly (not through
the CLI or workflows) and assume `gh` is authenticated.

## GitHub

### `print-releases-for-gh-repo.sh`

Dump a markdown table of every release and its assets for a GitHub repo —
useful for sizing release-asset exports before a migration, since `gh gei`
does not carry release binaries across.

```bash
# One row per asset (default)
scripts/print-releases-for-gh-repo.sh <owner>/<repo>

# One row per release (file count + total MB)
scripts/print-releases-for-gh-repo.sh <owner>/<repo> --summary

# Query a non-github.com host (GHES / GHEC-DR)
scripts/print-releases-for-gh-repo.sh <owner>/<repo> --host apex-space.ghe.com
```

Output is GitHub-flavored markdown; pipe to a file or paste into an issue.

# Key Things to Know

**Routing lives in labels:** Source location and migration destination are
stored as labels (`namespace`, `src-org`, `src-team`, `dst-org`, `dst-team`,
`dst-prefix`), not the issue body. The visible body text (tables, sections,
committer lists) is display-only — hand-editing it has no effect on routing.

**But the hidden analyzer markers ARE authoritative — don't hand-edit them.**
The `<!-- amp-id: ... -->` marker is the canonical identity of the source
repo and, on ADO, encodes the exact project name the API needs (the
slugified `namespace:` label is unusable there — a project like
`SITA Baggage Management` becomes `sita-baggage-management`, which the ADO
API rejects). `amp refresh` and `/update-issue` read the marker first, the
` · <amp_id>` title suffix only as a fallback for legacy issues. Rewriting
the marker to a different provider, project scope, or raw ID silently
redirects every subsequent `refresh` / `/update-issue` / re-analyze call to
that new target. Same warning applies to `<!-- amp:source-baseline -->`
(post-migration validation) and `<!-- amp:env-recreate-data -->`
(post-migration secret/env recreate). If you need to fix an identity
mistake, re-run `amp analyze` on the source — don't edit the marker.

**Pick the batch size for the workflow:** 1–25 repos: native GitHub
bulk-edit (checkboxes + Edit labels dropdown). 50+ repos:
`amp bulk-assign --csv`. Use `--dry-run` first; failed rows land in
`<csv>.failures.csv` for re-runs.

**Always one `dst-org`, `dst-team`, `dst-prefix` per issue:** Multiple
labels of the same family cause `/migrate` and `/upsert-teams` to refuse.
Remove duplicates and re-trigger.

**3-tier teams from one label:** `dst-team:platform` attaches three teams
in the destination org: `platform-read` (pull), `platform-write` (push),
`platform-admin` (admin). Permissions are managed by team membership.

**Labels drive everything:** Applying a `cmd:*` label is equivalent to
posting the corresponding `/command`. Labels are auto-removed after
dispatch.

**AI advisor is automatic:** You don't need to manually invoke the advisor
— it runs on every failure. Use `amp advise` only for re-diagnosis or
local debugging.

**Source repos are never modified:** Auto-repair pushes fixes to a `-fixed`
copy. The original source is only touched by `/archive`.

**`blob:error` and `lfs:broken` are blockers:** These labels prevent
migration. You must resolve oversized files or LFS integrity issues before
proceeding.

**Verbose mode:** Add `--verbose` to any command for detailed debug output.

**GHEC-DR opt-in:** Setting `GH_DEST_API_URL` is the only switch needed to
target a Data Residency tenant. Leaving it unset preserves vanilla GHEC
behavior — existing installs need no change.
