# Automation Reference

## Deploy

**Path**: `.github/workflows/sanity-deploy.yml`

```yaml
name: Deploy Sanity (Production)
on:
  push:
    branches:
      - main
    paths:
      - "packages/cms/**"
      - ".github/workflows/sanity-deploy.yml"
  workflow_dispatch:
    inputs:
      presentation_url:
        description: "Override SANITY_STUDIO_PRESENTATION_URL"
        required: false
        type: string

permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    name: Build and Deploy Sanity Studio
    runs-on: ubuntu-latest
    environment: production
    timeout-minutes: 15
    env:
      SANITY_AUTH_TOKEN: ${{ secrets.SANITY_DEPLOY_TOKEN }}
      SANITY_STUDIO_PROJECT_ID: ${{ vars.SANITY_STUDIO_PROJECT_ID }}
      SANITY_STUDIO_DATASET: ${{ vars.SANITY_STUDIO_DATASET }}
      SANITY_STUDIO_TITLE: ${{ vars.SANITY_STUDIO_TITLE }}
      SANITY_STUDIO_URL: ${{ vars.SANITY_STUDIO_URL }}
      SANITY_STUDIO_PRESENTATION_URL: ${{ github.event.inputs.presentation_url || vars.SANITY_STUDIO_PRESENTATION_URL }}
      SANITY_STUDIO_HOST: ${{ vars.SANITY_STUDIO_HOST }}

    steps:
      - name: Checkout repository
        uses: actions/checkout@v6
        with:
          fetch-depth: 1

      - name: Install pnpm
        uses: pnpm/action-setup@v4
        with:
          run_install: false

      - uses: actions/setup-node@v6
        with:
          node-version: lts/*
          cache: "pnpm"

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Deploy Sanity Studio
        working-directory: ./packages/cms
        run: pnpm run deploy
```

### Required Secrets and Variables (Deploy)

- **Secrets**: `SANITY_DEPLOY_TOKEN`
- **Variables**: `SANITY_STUDIO_PROJECT_ID`, `SANITY_STUDIO_DATASET`, `SANITY_STUDIO_TITLE`, `SANITY_STUDIO_URL`, `SANITY_STUDIO_PRESENTATION_URL`, `SANITY_STUDIO_HOST`

---

## Backup

**Path**: `.github/workflows/sanity-backup.yml`

```yaml
on:
name: Deploy Sanity (Production)
on:
  push:
    branches:
      - main
    paths:
      - "packages/cms/**"
      - ".github/workflows/sanity-deploy.yml"
  workflow_dispatch:
    inputs:
      presentation_url:
        description: "Override SANITY_STUDIO_PRESENTATION_URL"
        required: false
        type: string

permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    name: Build and Deploy Sanity Studio
    runs-on: ubuntu-latest
    environment: production
    timeout-minutes: 15
    env:
      SANITY_AUTH_TOKEN: ${{ secrets.SANITY_DEPLOY_TOKEN }}
      SANITY_STUDIO_PROJECT_ID: ${{ vars.SANITY_STUDIO_PROJECT_ID }}
      SANITY_STUDIO_DATASET: ${{ vars.SANITY_STUDIO_DATASET }}
      SANITY_STUDIO_TITLE: ${{ vars.SANITY_STUDIO_TITLE }}
      SANITY_STUDIO_URL: ${{ vars.SANITY_STUDIO_URL }}
      SANITY_STUDIO_PRESENTATION_URL: ${{ github.event.inputs.presentation_url || vars.SANITY_STUDIO_PRESENTATION_URL }}
      SANITY_STUDIO_HOST: ${{ vars.SANITY_STUDIO_HOST }}

    steps:
      - name: Checkout repository
        uses: actions/checkout@v6
        with:
          fetch-depth: 1

      - name: Install pnpm
        uses: pnpm/action-setup@v4
        with:
          run_install: false

      - uses: actions/setup-node@v6
        with:
          node-version: lts/*
          cache: "pnpm"

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Deploy Sanity Studio
        working-directory: ./packages/cms
        run: pnpm run deploy
```

### Required Secrets and Variables (Backup)

- **Secrets**: `SANITY_BACKUP_TOKEN`
- **Variables**: `SANITY_STUDIO_PROJECT_ID`, `SANITY_STUDIO_DATASET`

### Artifact

- **Path**: `packages/cms/backup.tar.gz`
- **Retention**: 15 days

---

## Tokens

- Use **separate** tokens for deploy and backup.
- Create at [sanity.io/manage](https://sanity.io/manage) → project → API → Tokens.
- **Deploy**: deployment permissions.
- **Backup**: read permission for dataset export.

---

## Local checks

```bash
cd packages/cms && pnpm run deploy
pnpx @sanity/cli dataset export production backup.tar.gz
```

---

## Troubleshooting

### Deploy fails

1. Confirm `SANITY_DEPLOY_TOKEN` and variables are set.
2. Check token has deploy permission.
3. Verify `SANITY_STUDIO_PROJECT_ID`.
4. Run `pnpm run deploy` in `packages/cms` locally.

### Backup fails

1. Confirm `SANITY_BACKUP_TOKEN` has read access.
2. Verify `SANITY_STUDIO_DATASET`.
3. Run locally: `cd packages/cms && pnpx @sanity/cli dataset export production backup.tar.gz`.

### Workflow not triggering

1. Check `paths` and branch.
2. Validate YAML.
3. Ensure required permissions.

---

## Security

- Use GitHub Secrets (and Variables) only; never commit tokens.
- Rotate tokens regularly.
- Use minimal permissions.
- Review logs and access.
