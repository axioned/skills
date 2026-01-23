# Best Practices Reference

## GROQ Query Best Practices

### Fragment composition

1. Start with base fragments, then compose up.
2. Reuse existing fragments; use `${fragmentName}`.
3. One concern per fragment.

### Query structure

1. Explicit `_type == "x"`.
2. Project only what you need.
3. Use `order()`.
4. `defined(field)` before use.
5. `array::compact()` for arrays.
6. `coalesce()` for defaults.

### Image handling

- Do not expand images inline. Use `imageFragment` or `imageFields`.

### Code style

- Template literals, clear indentation, comments for complex logic, consistent spacing.

### Naming

- Queries: camelCase + `Query` (e.g. `queryHomePageData`).
- Fragments: `xxxFragment` or `xxxBlock`.
- Internal: `_richText`, `_buttons`.

---

## Schema Best Practices

### File naming

- kebab-case (e.g. `feature-cards.ts` → `featureCards`).
- Same naming in schema, components, and GROQ.

### Fields

- Descriptions (plain language), validation for required, icons (lucide-react or sanity/icons), preview, groups.

### Helpers

- `documentSlugField()`, `pageBuilderField`, `imageWithAltField()`, `buttonsField`, `customRichText()`.

---

## Document Best Practices

- **Slug**: `documentSlugField()`, unique per type.
- **SEO**: `seoFields`, `ogFields`, `seoHideFromLists`, fallbacks.
- **Draft**: `perspective: "drafts"`, `useCdn: false`, `stega: true` when draft.
- **Errors**: `notFound()`, null/undefined handling, fallbacks.
- **Static**: `generateStaticParams`, slug arrays for nested routes.
- **Sitemap**: all public pages, `lastModified`, priorities.

---

## Block Best Practices

- **`BLOCK_COMPONENTS`**: key = schema `name`.
- **GROQ**: always add block to `pageBuilderFragment`; reuse fragments; `array::compact()`.
- **Types**: `PagebuilderType<"blockName">`; regenerate after schema changes.
- **Fragments**: `richTextFragment`, `buttonsFragment`, `imageFragment`, `baseUrlFragment`.
- **Arrays**: `array::compact()`, `_key`/`_id` in maps, handle empty.
- **Optional**: conditional render, defaults, avoid empty sections.

### Component registration

```typescript
// ✅
const BLOCK_COMPONENTS = { featureCards: FeatureCardsBlock } as const;

// ❌
const BLOCK_COMPONENTS = { featureCardsBlock: FeatureCardsBlock } as const;
```

---

## Page Builder Best Practices

- Schema `name` = `BLOCK_COMPONENTS` key; kebab-case files.
- `data-sanity` for visual editing; Presentation config; env for Studio URL.
- Unknown block type: error component and clear message.

---

## Automation Best Practices

- **Tokens**: separate deploy/backup; minimal permissions; rotate; review access.
- **Workflows**: path filters; cache deps; error handling; notifications.
- **Backup**: regular schedule; retention; test restore; document steps.
- **Security**: only GitHub Secrets/Variables; limit permissions; audit logs; separate envs.
- **Local**: test `pnpm run deploy` and `pnpx @sanity/cli dataset export` before commits.

---

## Type Generation Best Practices

### When to run

- New document or block types, schema edits, new GROQ.

### Commands

```bash
pnpm typegen
```

Or from CMS package:

```bash
cd apps/cms && sanity schema extract && sanity typegen generate --enforce-required-fields
```

### Usage

- Prefer `@/lib/sanity/types` and `PagebuilderType<"blockName">`; avoid hand-written types where generated ones exist.

---

## Code Organization

- Schemas: `apps/cms/schemaTypes/`
- Queries: `apps/web/src/lib/sanity/query.ts`
- Block components: `apps/web/src/components/sections/`

---

## Performance Best Practices

- **Queries**: only needed fields, projections, limit, pagination.
- **Images**: Sanity CDN, sizes, responsive, lazy load.
- **Caching**: CDN, query cache, static generation, revalidation.

---

## Testing Best Practices

- **Local**: queries, component + sample data, errors, draft.
- **Schema**: in dev, previews, validation, relations.
- **Integration**: full flows, page builder, routes, sitemap.

---

## Common Patterns

### Error handling

```typescript
const data = await client.fetch(query);
if (!data) notFound();
```

### Optional field

```typescript
{ field && <Component field={field} /> }
```

### Array map

```typescript
{ items?.map((item) => <Item key={item._key || item._id} {...item} />) }
```

### Typed block

```typescript
type Props = PagebuilderType<"blockName">;
export function BlockName({ ...props }: Props) {
  /* ... */
}
```
