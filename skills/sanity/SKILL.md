---
name: sanity
description: Work with Sanity CMS—GROQ queries, documents, page builder blocks, and automation. Use when creating or modifying Sanity schemas, GROQ queries, documents, blocks, page builder, deployment, or backups.
---

# Sanity

Unified skill for working with Sanity CMS: schemas, GROQ queries, documents, page builder blocks, and automation.

## When to Use

Load this skill when:

- Creating or modifying GROQ queries and fragments
- Creating document types (schema → routes → sitemap)
- Creating page builder blocks (schema → GROQ → component)
- Working with the page builder system
- Reviewing or applying Sanity best practices
- Setting up or changing deploy/backup GitHub Actions

## Topic Groups and References

Use the reference that matches the task:

| Task | Reference |
|------|-----------|
| GROQ queries, fragments, fetch patterns | [references/queries.md](references/queries.md) |
| Document types, routes, sitemap | [references/documents.md](references/documents.md) |
| Page builder blocks (schema → component) | [references/blocks.md](references/blocks.md) |
| Page builder system, component mapping | [references/page-builder.md](references/page-builder.md) |
| Best practices, naming, patterns | [references/best-practices.md](references/best-practices.md) |
| Deploy and backup workflows | [references/automation.md](references/automation.md) |

## Instructions

1. **Identify the topic** from the user’s request (queries, documents, blocks, page builder, best practices, or automation).
2. **Load the matching reference**: open the file from the table above (e.g. `references/queries.md`).
3. **Follow the reference** for steps, patterns, and examples.
4. **Apply best practices** from `references/best-practices.md` when implementing.

## Key Locations

- **GROQ**: `apps/web/src/lib/sanity/query.ts`
- **Document schemas**: `apps/cms/schemaTypes/documents/`
- **Block schemas**: `apps/cms/schemaTypes/blocks/`
- **Block index**: `apps/cms/schemaTypes/blocks/index.ts`
- **Page builder schema**: `apps/cms/schemaTypes/definitions/pagebuilder.ts`
- **Block components**: `apps/web/src/components/sections/`
- **Page builder component**: `apps/web/src/components/page-builder.tsx`
- **Deploy workflow**: `.github/workflows/sanity-deploy.yml`
- **Backup workflow**: `.github/workflows/sanity-backup.yml`

## Type Generation

After schema or query changes:

```bash
pnpm typegen
```

Or from the CMS package directory:

```bash
sanity schema extract && sanity typegen generate --enforce-required-fields
```
