---
name: contentful-integration
description: Guide for integrating Contentful CMS with Next.js applications. Use when asked to "integrate Contentful with Next.js", "setup Contentful for Next.js" or "work on the Contentful query".
metadata:
  author: axioned
  version: "1.0.0"
  argument-hint: <file-or-pattern>
---

# Contentful Integration

Best practices for integrating Contentful CMS with Next.js applications.

## When to Use

- Use this skill when integrating Contentful CMS with Next.js applications
- This skill is helpful for setting up Contentful clients, configuring environments, and querying content
- Use when working with Contentful API queries, preview mode, or type generation

## Instructions

### Dependencies

**Required packages:**

```json
{
  "dependencies": {
    "@contentful/live-preview": "^4.6.38",
    "@contentful/rich-text-plain-text-renderer": "^17.1.0",
    "@contentful/rich-text-react-renderer": "^16.1.0",
    "@contentful/rich-text-types": "^17.1.0",
    "@contentful/vercel-nextjs-toolkit": "1.6.0",
    "contentful": "^11.7.15",
    "cf-content-types-generator": "^2.16.0"
  }
}
```

### Environment Variables

**Required variables:**

```env
CONTENTFUL_SPACE_ID=""
CONTENTFUL_ACCESS_TOKEN=""
CONTENTFUL_PREVIEW_ACCESS_TOKEN=""
CONTENTFUL_REVALIDATE_SECRET=""
CONTENTFUL_MANAGEMENT_TOKEN=""
CONTENTFUL_ENVIRONMENT=""
```

### Setup Scripts

**Add to package.json:**

```json
{
  "scripts": {
    "cf:export": "contentful space export --config ./contentful-cli-config.json",
    "cf:typegen": "pnpm run cf:export && cf-content-types-generator --response ./types/contentful/export.json --out ./types/contentful --typeguard --v10"
  }
}
```

**contentful-cli-config.json (add to .gitignore):**

```json
{
  "spaceId": "",
  "environmentId": "development",
  "managementToken": "",
  "contentFile": "./types/contentful/export.json"
}
```

**Important:**

- Add `contentful-cli-config.json` to `.gitignore`
- Always use `development` environment for local development
- Never share management token

### Environments

**Environment Types:**

- **Environment** - Container entities within a space for isolated content type versions
- **Environment Alias** - Static ID that points to a target environment (can be switched)
- **Custom Alias** - Custom environment aliases (Premium tier only)

**Best Practices:**

- **Production:** Use `master` environment or `master` alias
- **Everything else:** Sandbox environments for development/testing
- **Never use sandbox environments for production**

**Local Development:**

- Set environment ID in `.env` or `.env.local`
- **Do not use `master` environment for local development**
- Clone from production environment for local testing
- Connect to sandbox environment for iteration

**Staging/QA:**

- Set environment ID in CI/CD pipeline
- **Do not use `master` environment for staging/QA**
- Create separate staging and QA environments

**Workflow:**

1. Developers work on development environment (cloned from `master`)
2. Changes applied to QA environment for testing
3. QA engineers perform manual/automated tests
4. If tests pass, apply to staging for final testing
5. Finally, apply to `master` environment

**Editorial Workflows:**

- Editors work with content in `master` environment or `master` alias
- Developers work on new features in sandbox environments
- Changes applied from sandbox to `master` when ready

**Custom Aliases:**

- Can create custom environment aliases (e.g., `development`, `staging`)
- **Limitations:**
  - Must not be used for production content
  - Requests to non-`master` environments are not cached or covered by SLAs
  - Available to Premium tier customers only

### Type Generation

**Generate TypeScript types:**

```bash
pnpm run cf:typegen
```

This:

1. Exports content model from Contentful
2. Generates TypeScript types in `./types/contentful`
3. Includes type guards for runtime validation

**Use generated types:**

```typescript
import type { BlogPost } from "./types/contentful";

const entry: BlogPost = {
  // Type-safe content
};
```

### Contentful Client Setup

**Create client:**

```typescript
import { createClient } from "contentful";

const client = createClient({
  space: process.env.CONTENTFUL_SPACE_ID!,
  accessToken: process.env.CONTENTFUL_ACCESS_TOKEN!,
  environment: process.env.CONTENTFUL_ENVIRONMENT || "master",
});

// Preview client
const previewClient = createClient({
  space: process.env.CONTENTFUL_SPACE_ID!,
  accessToken: process.env.CONTENTFUL_PREVIEW_ACCESS_TOKEN!,
  host: "preview.contentful.com",
  environment: process.env.CONTENTFUL_ENVIRONMENT || "master",
});
```

### Common Patterns

**Fetch entries:**

```typescript
const entries = await client.getEntries({
  content_type: "blogPost",
  "fields.slug": "my-post",
  limit: 10,
  skip: 0,
});
```

**Fetch single entry:**

```typescript
const entry = await client.getEntry("entry-id");
```

**Fetch by slug:**

```typescript
const entries = await client.getEntries({
  content_type: "blogPost",
  "fields.slug": "my-post",
  limit: 1,
});
const entry = entries.items[0];
```

### Next.js Integration

**Using Vercel Next.js Toolkit:**

```typescript
import { draftMode } from "@contentful/vercel-nextjs-toolkit";

// In API route
export async function GET(request: Request) {
  const { isEnabled } = await draftMode();
  // Handle preview mode
}
```

**Live Preview:**

```typescript
import { ContentfulLivePreview } from "@contentful/live-preview";

ContentfulLivePreview.init({
  locale: "en-US",
  enableInspectorMode: true,
  enableLiveUpdates: true,
});
```

## Resources

- [Contentful Documentation](https://www.contentful.com/developers/docs/)
