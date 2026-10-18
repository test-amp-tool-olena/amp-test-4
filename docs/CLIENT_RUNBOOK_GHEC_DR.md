# Client install runbook: GHEC → GHEC-DR

Audience: the operator installing AMP into a fresh control repo to run a
GitHub Enterprise Cloud → GHEC with Data Residency (GHEC-DR) migration.

> **See also:** [AMP User Guide](AMP_User_Guide.md) for the full migration walkthrough. For a 2-page brief of what AMP adds on top of `gh gei` — including the GHEC-DR-specific delta — see [AMP_Features_One_Pager_GitHub.md](AMP_Features_One_Pager_GitHub.md) or the [master one-pager](AMP_Features_One_Pager.md).

GHEC-DR is a separate GHEC tenant on a customer-specific subdomain
(`https://<tenant>.ghe.com`, API at `https://api.<tenant>.ghe.com`).
Tenants are EMU-only — users and teams are SCIM-provisioned by the
client's IdP.

## 1. Bootstrap the control repo

Run `scripts/setup-instance.sh --org <client-control-org> --name amp-ops`
to create the AMP control repo on the **source** github.com org. (The
control repo doesn't have to live in the DR tenant — it just needs the
secrets/vars below to reach both sides.)

## 2. Required secrets

Set at `https://github.com/<client-control-org>/amp-ops/settings/secrets/actions`.

| Name | Where it comes from | Scopes |
|---|---|---|
| `GH_PAT` | Source github.com PAT | `repo`, `read:org`, `read:user`, `workflow` |
| `GH_DEST_PAT` | EMU PAT issued on the GHEC-DR tenant | `repo`, `admin:org`, `workflow` |
| `APP_ID` | GitHub App on the control repo's org | (App, not PAT) |
| `APP_PRIVATE_KEY` | Same App's private key | (App, not PAT) |
| `MODELS_TOKEN` | Optional — Claude API for AI failure diagnosis | `models:read` |
| `BITBUCKET_API_TOKEN` | Only if migrating from Bitbucket Cloud | Workspace + PR read |

The GitHub App is only used for IssueOps comments on the control repo so
that comments trigger downstream workflows. Migrations themselves use
`GH_PAT` + `GH_DEST_PAT`.

## 3. Required variables (the new bit for DR)

Set at `.../amp-ops/settings/variables/actions`.

| Name | Value |
|---|---|
| `GH_DEST_API_URL` | `https://api.<tenant>.ghe.com` ← **DR-specific** |
| `SANDBOX_ORG` | A dryrun sandbox org **on the DR tenant** |
| `DEFAULT_SOURCE_ORG` | Source github.com org slug |
| `BITBUCKET_EMAIL` | Only if migrating from Bitbucket Cloud — email associated with `BITBUCKET_API_TOKEN` |

`GH_DEST_API_URL` is what makes AMP route migrations and post-migrate API
calls to the DR tenant instead of github.com. Leaving it unset (the
default for vanilla GHEC installs) targets `api.github.com`.

The sandbox org **must** live on the DR tenant — dryruns need to exercise
the same auth + URL path as real migrations, otherwise a green dryrun is
not evidence the real migration will work.

## 4. SCIM teams must pre-exist

For every issue carrying `dst-team:<base>`, the client's IdP team must
provision these three teams on the DR tenant **before** `/upsert-teams`
runs:

- `<base>-read`
- `<base>-write`
- `<base>-admin`

AMP's `_ensure_team()` will try `org.create_team()` first, catch the 422
that EMU returns, and fall back to `org.get_team_by_slug()`. If the team
isn't there, the issue gets a clear failure comment. Provision teams in
SCIM, then re-trigger `/upsert-teams` via the `cmd:upsert-teams` label.

## 5. Smoke test before the first real migration

1. Open one analyze run against a small source github.com org
   (`gh workflow run analyze.yml`).
2. Pick one small issue, label it `cmd:dryrun`.
3. Watch the issue-ops workflow run. In the "Execute command" step's
   logs, confirm the executed line contains
   `--target-api-url https://api.<tenant>.ghe.com`. If it doesn't, the
   `GH_DEST_API_URL` variable isn't set on the control repo.
4. Verify the dryrun repo lands in `SANDBOX_ORG` on the DR tenant
   (browser: `https://<tenant>.ghe.com/<sandbox_org>/<dryrun-repo>`),
   not on github.com.
5. If the issue carries `dst-team:`, confirm `/upsert-teams` either
   succeeds (team pre-existed) or fails with a useful comment naming the
   missing team.

## 6. Known limitations on this first install

- **`/configure-autolink` is a no-op placeholder**
  ([commands/__init__.py:480-484](../amp/commands/__init__.py#L480-L484)).
  Configure ADO→GHEC-DR autolinks manually if needed. Not relevant for
  this engagement (source is GHEC, not ADO) but document for future
  installs.
- **No separate GitHub App on the destination DR tenant.** AMP uses the
  source-tenant App for IssueOps only. All DR-side writes go through
  `GH_DEST_PAT`. This is fine for v1 — revisit if the client requires
  audit-logged App-based migrations on the DR side.
- **Team creation in EMU mode falls back to lookup-only via the 422-catch
  path**, not an explicit `EMU_MODE` flag. Behavior is correct as long as
  SCIM teams pre-exist (see §4).
