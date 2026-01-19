---
name: sanity-page-builder
description: Understand and work with the Sanity page builder system from schema definition to React rendering
---

# Sanity Page Builder System

## When to use this skill

Use this skill when you need to understand how the page builder system works, add page builder to documents, or troubleshoot page builder rendering issues.

## System Architecture

The page builder is a flexible content system that allows content editors to compose pages using reusable blocks. The flow is:

1. **Schema Definition** - Blocks defined in CMS schema
2. **Block Registration** - Blocks registered in page builder array
3. **Data Fetching** - GROQ queries fetch page builder data
4. **Component Mapping** - React components mapped to block types
5. **Rendering** - PageBuilder component renders blocks dynamically

## How It Works

### 1. Schema Layer (CMS)

**Location**: `apps/cms/schemaTypes/definitions/pagebuilder.ts`

The page builder schema automatically includes all blocks from the blocks index:

```typescript
import { pageBuilderBlocks } from "@/schemaTypes/blocks/index";
import { defineArrayMember, defineType } from "sanity";

export const pagebuilderBlockTypes = pageBuilderBlocks.map(({ name }) => ({
  type: name,
}));

export const pageBuilder = defineType({
  name: "pageBuilder",
  type: "array",
  of: pagebuilderBlockTypes.map((block) => defineArrayMember(block)),
});
```

**Key Points**:

- Automatically includes all blocks from `pageBuilderBlocks` array
- No manual block registration needed here
- Blocks are added by including them in `apps/cms/schemaTypes/blocks/index.ts`

### 2. Block Registration

**Location**: `apps/cms/schemaTypes/blocks/index.ts`

All blocks are exported in an array:

```typescript
import { hero } from "./hero";
import { cta } from "./cta";
import { featureCards } from "./feature-cards";
// ... other blocks

export const pageBuilderBlocks = [
  hero,
  cta,
  featureCards,
  // ... other blocks
];
```

**Adding a new block**:

1. Create block schema in `apps/cms/schemaTypes/blocks/[block-name].ts`
2. Import and add to `pageBuilderBlocks` array
3. The page builder schema automatically picks it up

### 3. Using Page Builder in Documents

**Location**: `apps/cms/schemaTypes/documents/[document-name].ts`

Add the page builder field to any document:

```typescript
import { pageBuilderField } from "@/schemaTypes/common";

export const page = defineType({
  name: "page",
  type: "document",
  fields: [
    // ... other fields
    pageBuilderField, // Add page builder
  ],
});
```

The `pageBuilderField` helper is defined in `apps/cms/schemaTypes/common.ts`:

```typescript
export const pageBuilderField = defineField({
  name: "pageBuilder",
  group: GROUP.MAIN_CONTENT,
  type: "pageBuilder",
  description: "Build your page by adding different sections",
});
```

### 4. Data Fetching (GROQ)

**Location**: `apps/web/src/lib/sanity/query.ts`

The page builder fragment includes all block fragments:

```typescript
const pageBuilderFragment = /* groq */ `
  pageBuilder[]{
    ...,
    _type,
    ${ctaBlock},
    ${heroBlock},
    ${faqAccordionBlock},
    ${featureCardsBlock},
    // ... all block fragments
  }
`;
```

Use in document queries:

```typescript
export const queryPageData = defineQuery(`
  *[_type == "page" && slug.current == $slug][0]{
    ...,
    ${pageBuilderFragment}
  }
`);
```

### 5. Component Mapping

**Location**: `apps/web/src/components/page-builder.tsx`

Blocks are mapped to React components:

```typescript
const BLOCK_COMPONENTS = {
  cta: CTABlock,
  hero: HeroBlock,
  featureCards: FeatureCards,
  // ... other blocks
} as const satisfies Record<PageBuilderBlockTypes, React.ComponentType<any>>;
```

**Important**: The key must match the block's `name` in the schema.

### 6. Rendering

The `PageBuilder` component:

1. Receives `pageBuilder` array from Sanity
2. Maps each block to its React component
3. Renders blocks in order
4. Adds Sanity data attributes for visual editing

```typescript
export function PageBuilder({
  pageBuilder: initialBlocks = [],
  id,
  type,
}: PageBuilderProps) {
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

### Step 1: Add Field to Schema

```typescript
import { pageBuilderField } from "@/schemaTypes/common";

export const myDocument = defineType({
  name: "myDocument",
  type: "document",
  fields: [
    defineField({
      name: "title",
      type: "string",
      title: "Title",
    }),
    pageBuilderField, // Add page builder
  ],
});
```

### Step 2: Add to GROQ Query

```typescript
export const queryMyDocument = defineQuery(`
  *[_type == "myDocument" && slug.current == $slug][0]{
    ...,
    ${pageBuilderFragment}
  }
`);
```

### Step 3: Use in Page Component

```typescript
import { PageBuilder } from "@/components/page-builder";

export default async function Page({ params }: Props) {
  const data = await fetchMyDocument(params.slug);
  const { _id, _type, pageBuilder } = data ?? {};

  return (
    <PageBuilder id={_id} pageBuilder={pageBuilder ?? []} type={_type} />
  );
}
```

## Adding a New Block to Page Builder

### Complete Workflow

1. **Create block schema** - `apps/cms/schemaTypes/blocks/[block-name].ts`
2. **Register in index** - Add to `pageBuilderBlocks` array
3. **Create GROQ fragment** - Add block fragment to `pageBuilderFragment`
4. **Create React component** - `apps/web/src/components/sections/[block-name].tsx`
5. **Register component** - Add to `BLOCK_COMPONENTS` in `page-builder.tsx`
6. **Generate types** - Run `pnpm typegen`

See the `sanity-blocks` skill for detailed steps.

## Visual Editing (Sanity Presentation)

The page builder supports visual editing through Sanity Presentation tool:

### Data Attributes

Each block gets a `data-sanity` attribute:

```typescript
<div
  data-sanity={createBlockDataAttribute(block._key)}
  key={`${block._type}-${block._key}`}
>
  <Component {...block} />
</div>
```

### Configuration

Data attributes are created with:

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

This enables:

- Click-to-edit in Sanity Studio
- Real-time preview updates
- Visual content editing

## Optimistic Updates

The page builder uses optimistic updates for better UX:

```typescript
function useOptimisticPageBuilder(initialBlocks: PageBuilderBlock[], documentId: string) {
  return useOptimistic<PageBuilderBlock[], any>(initialBlocks, (currentBlocks, action) => {
    if (action.id === documentId && action.document?.pageBuilder) {
      return action.document.pageBuilder;
    }
    return currentBlocks;
  });
}
```

This provides instant UI updates when content changes in Sanity Studio.

## Error Handling

Unknown block types show an error component:

```typescript
function UnknownBlockError({ blockType, blockKey }: Props) {
  return (
    <div role="alert" className="...">
      <p>Component not found for block type:</p>
      <code>{blockType}</code>
    </div>
  );
}
```

**Common causes**:

- Block schema exists but component not registered
- Component key doesn't match schema name
- Block fragment missing from GROQ query

## Best Practices

For comprehensive best practices, see the `sanity-best-practices` skill. Key points for page builder:

- **Block naming**: Schema name must match component key in `BLOCK_COMPONENTS`
- **Component registration**: Key must exactly match schema `name`
- **GROQ fragment**: Always add block fragment to `pageBuilderFragment`
- **Type safety**: Use `PagebuilderType<"blockName">` for component props

## Troubleshooting

### Block Not Appearing in CMS

1. **Check block registration**: Verify block is in `pageBuilderBlocks` array
2. **Check schema**: Ensure block schema is properly exported
3. **Regenerate types**: Run `pnpm typegen`
4. **Restart CMS**: Restart Sanity Studio dev server

### Block Not Rendering

1. **Check component registration**: Verify component is in `BLOCK_COMPONENTS`
2. **Check key match**: Ensure component key matches schema `name`
3. **Check GROQ fragment**: Verify block fragment is in `pageBuilderFragment`
4. **Check console**: Look for errors in browser console

### Visual Editing Not Working

1. **Check data attributes**: Verify `data-sanity` attributes are present
2. **Check environment variables**: Verify `NEXT_PUBLIC_SANITY_STUDIO_URL` is set
3. **Check Sanity config**: Verify Presentation tool is configured
4. **Check draft mode**: Ensure draft mode is enabled for preview

### Type Errors

1. **Regenerate types**: Run `pnpm typegen`
2. **Check imports**: Verify using correct type imports
3. **Check block name**: Ensure type name matches schema name

## Data Flow Diagram

```
Sanity CMS
    ↓
Block Schema (apps/cms/schemaTypes/blocks/)
    ↓
Block Index (apps/cms/schemaTypes/blocks/index.ts)
    ↓
Page Builder Schema (apps/cms/schemaTypes/definitions/pagebuilder.ts)
    ↓
Document Schema (includes pageBuilderField)
    ↓
GROQ Query (fetches pageBuilder array)
    ↓
Page Component (receives pageBuilder data)
    ↓
PageBuilder Component (maps blocks to components)
    ↓
Block Components (render individual blocks)
    ↓
Browser (rendered HTML with data-sanity attributes)
```

## Example: Complete Page Builder Setup

### Document Schema

```typescript
import { pageBuilderField } from "@/schemaTypes/common";

export const page = defineType({
  name: "page",
  type: "document",
  fields: [
    defineField({
      name: "title",
      type: "string",
      title: "Title",
    }),
    pageBuilderField,
  ],
});
```

### GROQ Query

```typescript
export const queryPageData = defineQuery(`
  *[_type == "page" && slug.current == $slug][0]{
    ...,
    ${pageBuilderFragment}
  }
`);
```

### Page Component

```typescript
import { PageBuilder } from "@/components/page-builder";
import { client } from "@/lib/sanity/client";
import { queryPageData } from "@/lib/sanity/query";

export default async function Page({ params }: Props) {
  const data = await client.fetch(queryPageData, { slug: params.slug });
  const { _id, _type, pageBuilder } = data ?? {};

  return (
    <PageBuilder id={_id} pageBuilder={pageBuilder ?? []} type={_type} />
  );
}
```

## Advanced Patterns

### Conditional Block Rendering

```typescript
const renderBlock = useCallback((block: PageBuilderBlock) => {
  // Skip rendering certain blocks in production
  if (process.env.NODE_ENV === "production" && block._type === "debug") {
    return null;
  }
  // ... normal rendering
}, []);
```

### Block Grouping

```typescript
// Group consecutive blocks of same type
const groupedBlocks = blocks.reduce((acc, block) => {
  const lastGroup = acc[acc.length - 1];
  if (lastGroup?.[0]?._type === block._type) {
    lastGroup.push(block);
  } else {
    acc.push([block]);
  }
  return acc;
}, [] as PageBuilderBlock[][]);
```

### Custom Block Wrappers

```typescript
const renderBlock = useCallback((block: PageBuilderBlock) => {
  const Component = BLOCK_COMPONENTS[block._type];

  return (
    <BlockWrapper block={block}>
      <Component {...block} />
    </BlockWrapper>
  );
}, []);
```
