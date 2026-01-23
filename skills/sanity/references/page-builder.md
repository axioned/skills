# Page Builder Reference

## System Architecture

1. **Schema** – blocks in CMS schema
2. **Block registration** – `pageBuilderBlocks` array
3. **Data fetching** – GROQ `pageBuilder[]`
4. **Component mapping** – `BLOCK_COMPONENTS`
5. **Rendering** – `PageBuilder` with `data-sanity` for visual editing

## 1. Schema Layer

**Location**: `apps/cms/schemaTypes/definitions/pagebuilder.ts`

```typescript
import { pageBuilderBlocks } from "@/schemaTypes/blocks/index";
import { defineArrayMember, defineType } from "sanity";

export const pagebuilderBlockTypes = pageBuilderBlocks.map(({ name }) => ({ type: name }));

export const pageBuilder = defineType({
  name: "pageBuilder",
  type: "array",
  of: pagebuilderBlockTypes.map((block) => defineArrayMember(block)),
});
```

- All blocks come from `pageBuilderBlocks`.
- Add blocks via `apps/cms/schemaTypes/blocks/index.ts`.

## 2. Block Registration

**Location**: `apps/cms/schemaTypes/blocks/index.ts`

```typescript
import { hero } from "./hero";
import { cta } from "./cta";
import { featureCards } from "./feature-cards";

export const pageBuilderBlocks = [hero, cta, featureCards];
```

To add a block: create schema → import and push to `pageBuilderBlocks`.

## 3. Page Builder in Documents

**Location**: `apps/cms/schemaTypes/documents/[document-name].ts`

```typescript
import { pageBuilderField } from "@/schemaTypes/common";

export const page = defineType({
  name: "page",
  type: "document",
  fields: [, /* ... */ pageBuilderField],
});
```

`pageBuilderField` is from `apps/cms/schemaTypes/common.ts`.

## 4. Data Fetching (GROQ)

**Location**: `apps/web/src/lib/sanity/query.ts`

```typescript
const pageBuilderFragment = /* groq */ `
  pageBuilder[]{
    ...,
    _type,
    ${ctaBlock},
    ${heroBlock},
    ${faqAccordionBlock},
    ${featureCardsBlock}
  }
`;

export const queryPageData = defineQuery(`
  *[_type == "page" && slug.current == $slug][0]{ ..., ${pageBuilderFragment} }
`);
```

## 5. Component Mapping

**Location**: `apps/web/src/components/page-builder.tsx`

```typescript
const BLOCK_COMPONENTS = {
  cta: CTABlock,
  hero: HeroBlock,
  featureCards: FeatureCards,
} as const satisfies Record<PageBuilderBlockTypes, React.ComponentType<any>>;
```

The object key must match the schema `name`.

## 6. Rendering

`PageBuilder`:

1. Takes `pageBuilder`, `id`, `type`
2. Maps each block to a component
3. Renders in order
4. Sets `data-sanity` for visual editing

```typescript
export function PageBuilder({ pageBuilder: initialBlocks = [], id, type }: PageBuilderProps) {
  const blocks = useOptimisticPageBuilder(initialBlocks, id);
  const { renderBlock } = useBlockRenderer(id, type);

  return (
    <main className="flex flex-col" data-sanity={containerDataAttribute}>
      {blocks.map(renderBlock)}
    </main>
  );
}
```

## Adding Page Builder to a Document

### 1. Schema

```typescript
import { pageBuilderField } from "@/schemaTypes/common";

export const myDocument = defineType({
  name: "myDocument",
  type: "document",
  fields: [defineField({ name: "title", type: "string", title: "Title" }), pageBuilderField],
});
```

### 2. GROQ

```typescript
*[_type == "myDocument" && slug.current == $slug][0]{ ..., ${pageBuilderFragment} }
```

### 3. Page

```typescript
import { PageBuilder } from "@/components/page-builder";

export default async function Page({ params }: Props) {
  const data = await fetchMyDocument(params.slug);
  const { _id, _type, pageBuilder } = data ?? {};
  return <PageBuilder id={_id} pageBuilder={pageBuilder ?? []} type={_type} />;
}
```

## Adding a New Block

1. Schema: `apps/cms/schemaTypes/blocks/[block-name].ts`
2. Index: add to `pageBuilderBlocks`
3. GROQ: add block fragment to `pageBuilderFragment`
4. Component: `apps/web/src/components/sections/[block-name].tsx`
5. `BLOCK_COMPONENTS`: add entry (key = schema `name`)
6. `pnpm typegen`

## Visual Editing (Sanity Presentation)

### Data attributes

```typescript
<div data-sanity={createBlockDataAttribute(block._key)} key={`${block._type}-${block._key}`}>
  <Component {...block} />
</div>
```

### Configuration

```typescript
createDataAttribute({
  id: documentId,
  baseUrl: env.NEXT_PUBLIC_SANITY_STUDIO_URL,
  projectId: env.NEXT_PUBLIC_SANITY_PROJECT_ID,
  dataset: env.NEXT_PUBLIC_SANITY_DATASET,
  type: documentType,
  path: `pageBuilder[_key=="${blockKey}"]`,
});
```

## Optimistic Updates

```typescript
function useOptimisticPageBuilder(initialBlocks: PageBuilderBlock[], documentId: string) {
  return useOptimistic<PageBuilderBlock[], any>(initialBlocks, (currentBlocks, action) => {
    if (action.id === documentId && action.document?.pageBuilder) return action.document.pageBuilder;
    return currentBlocks;
  });
}
```

## Error Handling

Unknown block types can render an error component. Typical causes:

- Schema exists, component not in `BLOCK_COMPONENTS`
- Key ≠ schema `name`
- Block fragment missing from `pageBuilderFragment`

## Troubleshooting

### Block not in CMS

- In `pageBuilderBlocks`? Schema exported? `pnpm typegen` and Studio restart.

### Block not rendering

- In `BLOCK_COMPONENTS`? Key = schema `name`? Fragment in `pageBuilderFragment`? Console errors?

### Visual editing broken

- `data-sanity` on blocks? `NEXT_PUBLIC_SANITY_STUDIO_URL`? Presentation config? Draft mode?

### Type errors

- `pnpm typegen`; correct types; `PagebuilderType` name = schema `name`.

## Data Flow

```text
Sanity CMS → Block Schema → Block Index → Page Builder Schema → Document (pageBuilderField)
→ GROQ (pageBuilder) → Page → PageBuilder → Block components → HTML + data-sanity
```

## Advanced Patterns

### Conditional block rendering

```typescript
if (process.env.NODE_ENV === "production" && block._type === "debug") return null;
```

### Block grouping

```typescript
const groupedBlocks = blocks.reduce((acc, block) => {
  const last = acc[acc.length - 1];
  if (last?.[0]?._type === block._type) last.push(block);
  else acc.push([block]);
  return acc;
}, [] as PageBuilderBlock[][]);
```

### Custom block wrapper

```typescript
<BlockWrapper block={block}><Component {...block} /></BlockWrapper>
```
