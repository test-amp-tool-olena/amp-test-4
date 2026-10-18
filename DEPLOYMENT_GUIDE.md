# AMP Customer Deployment Guide

This guide covers deploying AMP to your GitHub organization.

## Prerequisites

- GitHub organization on GitHub Enterprise Cloud (GHEC)
- GitHub PAT(s) with appropriate access to source and destination orgs
- Distribution credentials (AMP_DIST_TOKEN, AMP_VERSION, AMP_DIST_REPO) — provided by Xebia

## Quick Start

### 1. Create the Control Repository

Create a new private repository: `<your-org>/amp-control`

Then copy the entire contents of this distribution to that repository (your Xebia contact will provide a script, or do this manually via git).

### 2. AMP CLI is Installed Automatically

The GitHub Actions workflows handle AMP installation automatically via the `setup-amp` composite action. You do **not** need to install AMP locally — it runs entirely within GitHub Actions workflows.

### 3. Configure GitHub App (Optional)

The platform works with personal access tokens (PATs), but GitHub Apps provide better security and audit trails.

See the main [README.md](README.md) for App configuration guidance.

### 4. Set Up Labels

Labels are configured in `config/labels.yml`. To sync them to your GitHub repos:

1. Edit `config/labels.yml` if needed (optional)
2. Run the `setup-labels` workflow manually or via the CLI:

```bash
amp setup-labels --repo <your-org>/amp-control
```

This creates/updates all migration labels in your control repo.

### 5. (Optional) Create a Projects V2 Board

For visual tracking of migrations:

```bash
amp configure-project --repo <your-org>/amp-control \
    --owner <your-org> --number <project-N> --force
```

Then manually configure views and group-by fields in GitHub UI.

### 6. Configure Workflow Secrets and Variables

Each workflow needs certain secrets and variables. Edit the control repository settings:

**Minimum required secrets:**
- `GH_DEST_PAT` — GitHub token for destination org (for migrations)
- `GH_PAT` — GitHub token for source (for analysis)
- `AMP_DIST_TOKEN` — Read token for wheel distribution (required, provided by Xebia)

**Minimum required variables:**
- `AMP_VERSION` — e.g., `0.1.0` (provided by Xebia)
- `AMP_DIST_REPO` — e.g., `xebia/amp-dist` (provided by Xebia)

### 7. Set Up Source/Target Mappings

Copy and customize the example files in `config/mappings/`:

```bash
cp config/mappings/EXAMPLE_ADO.yml config/mappings/my-org-ado.yml
# Edit my-org-ado.yml with your Azure DevOps org details
```

### 8. Analyze Your First Batch

Trigger the `analyze` workflow in the control repository:

1. Go to Actions > Analyze
2. Click "Run workflow"
3. Provide your source platform details (org, project, repos, token)

This creates one issue per repository with full analysis. See [AMP_User_Guide.md](docs/AMP_User_Guide.md) for next steps.

### 9. Continue with Dry-Run and Migration

After analysis, use GitHub labels to control the flow:

- Apply `cmd:dryrun` label to test migrations
- Apply `cmd:migrate` label to perform actual migrations

The workflows respond to label changes automatically.

## Configuration

### Label Customization

Edit `config/labels.yml` and re-run the `setup-labels` command to sync changes.

### Failure Rules

Edit `config/failure_rules.yml` to customize the tier-1 failure advisor patterns.

### Projects V2 Sync

Edit `config/project_views.yml` to define custom views, then sync to GitHub:

```bash
amp configure-project --owner <your-org> --number <project-N> --force
```

## Troubleshooting

### Token Resolution

AMP tries multiple token sources in priority order. See the main [README.md](README.md) for the full fallback chain and how to debug which token is being used.

### Workflow Failures

Check the workflow logs in your control repository's Actions tab. Common issues:
- Missing secrets/variables
- Expired or revoked tokens
- Network/API rate limits
- Source platform connectivity

### Queue Issues

If migrations get stuck in queued state, check the `migration-dispatcher` workflow logs. The queue reconciler runs periodically and can be triggered manually.

### Local CLI Usage (Advanced)

If your Xebia contact provides source access, you can run AMP commands locally:

```bash
# First, install the wheel from Xebia's distribution repo
pip install --find-links https://github.com/xebia/amp-dist/releases amp

# Then run commands
amp analyze --source github --org my-source-org \
    --repo <your-org>/amp-control --token <source-pat>
```

## Support

For issues or questions, contact your Xebia engagement contact or open an issue in the control repository.

See [AMP_Known_Limitations.md](docs/AMP_Known_Limitations.md) for known issues and workarounds.

## Next Steps

- [AMP User Guide](docs/AMP_User_Guide.md) — Full walkthrough of the migration flow
- [Label Guide](docs/AMP_Label_Guide.md) — Detailed label taxonomy and state machine
- [GHEC-DR Runbook](docs/CLIENT_RUNBOOK_GHEC_DR.md) — If deploying to GHEC-DR
- [Known Limitations](docs/AMP_Known_Limitations.md) — Before you start
