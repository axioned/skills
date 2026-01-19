---
name: sanity-automation
description: GitHub Actions workflows for Sanity CMS deployment and backup automation
---

# Sanity Automation

## When to use this skill

Use this skill when you need to set up, modify, or troubleshoot GitHub Actions workflows for Sanity CMS deployment and backup automation.

## Workflow Files

- **Deploy**: `.github/workflows/deploy-sanity.yml` - Deploys Sanity Studio on push to main
- **Backup**: `.github/workflows/backup-sanity.yml` - Backs up Sanity dataset on schedule

## Deploy Workflow

### Location

`.github/workflows/deploy-sanity.yml`

### Triggers

```yaml
on:
  push:
    branches:
      - main
    paths:
      - "apps/cms/**"
      - ".github/workflows/deploy-sanity.yml"
  workflow_dispatch:
    inputs:
      presentation_url:
        description: "Override SANITY_STUDIO_PRESENTATION_URL"
        required: false
        type: string
```

**Triggers when**:

- Code is pushed to `main` branch AND changes are in `apps/cms/**` or the workflow file
- Manual trigger via GitHub Actions UI with optional `presentation_url` override

### Required Secrets

Set these in GitHub repository settings → Secrets:

- `SANITY_DEPLOY_TOKEN` - Sanity authentication token for deployment
- `SANITY_STUDIO_PROJECT_ID` - Your Sanity project ID
- `SANITY_STUDIO_DATASET` - Dataset name (e.g., "production")
- `SANITY_STUDIO_TITLE` - Studio title
- `SANITY_STUDIO_PRESENTATION_URL` - URL for Sanity Presentation tool
- `SANITY_STUDIO_PRODUCTION_HOSTNAME` - Production hostname for Studio

### Workflow Steps

1. **Checkout repository**
2. **Cache Bun dependencies** - Speeds up subsequent runs
3. **Setup Bun** - Installs latest Bun version
4. **Install dependencies** - Runs `bun install --frozen-lockfile`
5. **Deploy Sanity Studio** - Runs `bun run deploy` in `apps/cms` directory

### Environment Variables

```yaml
env:
  SANITY_AUTH_TOKEN: ${{ secrets.SANITY_DEPLOY_TOKEN }}
  SANITY_STUDIO_PROJECT_ID: ${{ secrets.SANITY_STUDIO_PROJECT_ID }}
  SANITY_STUDIO_DATASET: ${{ secrets.SANITY_STUDIO_DATASET }}
  SANITY_STUDIO_TITLE: ${{ secrets.SANITY_STUDIO_TITLE }}
  SANITY_STUDIO_PRESENTATION_URL: ${{ github.event.inputs.presentation_url || secrets.SANITY_STUDIO_PRESENTATION_URL }}
  SANITY_STUDIO_PRODUCTION_HOSTNAME: ${{ secrets.SANITY_STUDIO_PRODUCTION_HOSTNAME }}
```

### Permissions

```yaml
permissions:
  contents: read
  id-token: write
```

- `contents: read` - Read repository contents
- `id-token: write` - Required for OIDC authentication

## Backup Workflow

### Location

`.github/workflows/backup-sanity.yml`

### Triggers

```yaml
on:
  schedule:
    - cron: "0 10 */5 * *" # Every 5 days at 10:00 UTC
  workflow_dispatch: # Manual trigger
```

**Triggers when**:

- Scheduled: Every 5 days at 10:00 UTC
- Manual: Via GitHub Actions UI

### Required Secrets

- `SANITY_STUDIO_PROJECT_ID` - Your Sanity project ID
- `SANITY_BACKUP_TOKEN` - Sanity authentication token for backup
- `SANITY_STUDIO_DATASET` - Dataset name to backup

### Workflow Steps

1. **Checkout repository**
2. **Cache Bun dependencies**
3. **Setup Bun**
4. **Install dependencies**
5. **Export Sanity dataset** - Creates `backup.tar.gz` file
6. **Set timestamp** - For unique artifact naming
7. **Upload backup artifact** - Stores backup for 15 days

### Backup Command

```bash
bunx @sanity/cli dataset export ${{ secrets.SANITY_STUDIO_DATASET }} backup.tar.gz
```

### Artifact Storage

- **Name**: `sanity-backup-{dataset}-{timestamp}`
- **Path**: `apps/cms/backup.tar.gz`
- **Retention**: 15 days

## Setting Up Secrets

### In GitHub Repository

1. Go to repository → Settings → Secrets and variables → Actions
2. Click "New repository secret"
3. Add each required secret:
   - `SANITY_DEPLOY_TOKEN`
   - `SANITY_BACKUP_TOKEN`
   - `SANITY_STUDIO_PROJECT_ID`
   - `SANITY_STUDIO_DATASET`
   - `SANITY_STUDIO_TITLE`
   - `SANITY_STUDIO_PRESENTATION_URL`
   - `SANITY_STUDIO_PRODUCTION_HOSTNAME`

### Getting Sanity Tokens

1. Go to [sanity.io/manage](https://sanity.io/manage)
2. Select your project
3. Go to API → Tokens
4. Create tokens with appropriate permissions:
   - **Deploy token**: Needs deployment permissions
   - **Backup token**: Needs read permissions for dataset export

## Modifying Workflows

### Adding New Trigger

```yaml
on:
  push:
    branches:
      - main
      - develop # Add new branch
  pull_request:
    types: [opened, synchronize] # Add PR trigger
```

### Adding New Step

```yaml
steps:
  # ... existing steps
  - name: Run custom script
    working-directory: ./apps/cms
    run: |
      echo "Custom step"
      bun run custom-script
```

### Changing Schedule

Modify the cron expression in backup workflow:

```yaml
schedule:
  - cron: "0 10 * * *" # Daily at 10:00 UTC
  - cron: "0 10 */7 * *" # Weekly
  - cron: "0 10 1 * *" # Monthly on 1st
```

Cron format: `minute hour day month day-of-week`

### Adding Environment Variables

```yaml
env:
  EXISTING_VAR: ${{ secrets.EXISTING }}
  NEW_VAR: ${{ secrets.NEW_VAR }} # Add here
```

## Troubleshooting

### Deployment Fails

1. **Check secrets**: Verify all required secrets are set
2. **Check token permissions**: Ensure deploy token has correct permissions
3. **Check project ID**: Verify `SANITY_STUDIO_PROJECT_ID` is correct
4. **Check logs**: Review GitHub Actions logs for specific errors
5. **Test locally**: Run `bun run deploy` in `apps/cms` directory locally

### Backup Fails

1. **Check backup token**: Ensure token has read permissions
2. **Check dataset name**: Verify `SANITY_STUDIO_DATASET` matches your dataset
3. **Check storage**: Ensure GitHub Actions has storage quota available
4. **Test export locally**:
   ```bash
   cd apps/cms
   bunx @sanity/cli dataset export production backup.tar.gz
   ```

### Workflow Not Triggering

1. **Check paths filter**: Ensure changes are in specified paths
2. **Check branch**: Verify pushing to correct branch
3. **Check workflow file**: Ensure YAML syntax is valid
4. **Check permissions**: Verify workflow has necessary permissions

### Cache Issues

Clear cache by modifying cache key or using `--force`:

```yaml
- uses: actions/cache@v5
  with:
    path: ~/.bun/install/cache
    key: ${{ runner.os }}-bun-${{ hashFiles('**/bun.lockb') }}-v2 # Add version
```

## Best Practices

For comprehensive best practices, see the `sanity-best-practices` skill. Key points for automation:

- **Separate tokens**: Use different tokens for deploy and backup
- **Minimal permissions**: Grant only necessary permissions to tokens
- **Regular backups**: Schedule backups at appropriate intervals
- **Security**: Never commit secrets, use GitHub Secrets
- **Testing**: Test commands locally before committing

## Example: Custom Deployment Workflow

```yaml
name: Deploy Sanity (Staging)
on:
  push:
    branches:
      - develop
    paths:
      - "apps/cms/**"

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    env:
      SANITY_AUTH_TOKEN: ${{ secrets.SANITY_STAGING_DEPLOY_TOKEN }}
      SANITY_STUDIO_PROJECT_ID: ${{ secrets.SANITY_STUDIO_PROJECT_ID }}
      SANITY_STUDIO_DATASET: staging
    steps:
      - uses: actions/checkout@v6
      - uses: oven-sh/setup-bun@v2
      - run: bun install --frozen-lockfile
      - name: Deploy to Staging
        working-directory: ./apps/cms
        run: bun run deploy
```

## Monitoring Workflows

### GitHub Actions Dashboard

- View workflow runs: Repository → Actions tab
- Check status: Green (success), Red (failure), Yellow (in progress)
- View logs: Click on workflow run → Click on job → View logs

### Setting Up Notifications

1. Go to repository → Settings → Notifications
2. Enable "Actions" notifications
3. Choose notification method (email, web, etc.)

## Security Considerations

1. **Never commit secrets**: Always use GitHub Secrets
2. **Rotate tokens**: Regularly rotate authentication tokens
3. **Limit permissions**: Grant minimal required permissions
4. **Review access**: Regularly review who has access to secrets
5. **Audit logs**: Review workflow execution logs for suspicious activity
