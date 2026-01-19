---
name: sanity-blocks
description: Complete workflow for creating custom Sanity page builder blocks from CMS schema to React component usage
---

# Create Sanity Page Builder Blocks

## When to use this skill

Use this skill when you need to create a new page builder block that can be added to pages in the Sanity CMS. This covers the complete workflow from schema definition to component rendering.

## Complete Workflow

1. Create block schema in CMS
2. Register block in blocks index
3. Add GROQ fragment for data fetching
4. Create React component
5. Register component in page-builder

## Step 1: Create Block Schema

**Location**: `apps/cms/schemaTypes/blocks/[block-name].ts`

**File naming**: Use kebab-case (e.g., `feature-cards.ts`, `testimonials-carousel.ts`)

### Template (Schema)

```typescript
import { buttonsField, imageWithAltField } from "@/schemaTypes/common";
import { customRichText } from "@/schemaTypes/definitions/rich-text";
import { IconName } from "lucide-react"; // or from "sanity/icons"
import { defineField, defineType } from "sanity";

export const blockName = defineType({
  name: "blockName", // kebab-case, matches filename
  title: "Block Display Name",
  icon: IconName,
  type: "object",
  fields: [
    defineField({
      name: "title",
      title: "Title",
      description: "The large text that is the primary focus of the block",
      type: "string",
    }),
    // Use common field helpers
    customRichText(["block"]),
    imageWithAltField(),
    buttonsField,
    // Add custom fields
    defineField({
      name: "customField",
      title: "Custom Field",
      description: "Description in simple terms",
      type: "string",
    }),
  ],
  preview: {
    select: {
      title: "title",
    },
    prepare: ({ title }) => ({
      title,
      subtitle: "Block Display Name",
    }),
  },
});
```

### Common Field Helpers

From `apps/cms/schemaTypes/common.ts`:

- `buttonsField` - Array of buttons
- `imageWithAltField()` - Image with alt text
- `richTextField` - Rich text editor
- `iconField` - Icon picker
- `customRichText(["block"])` - Rich text with specific block types

## Step 2: Register Block in Index

**Location**: `apps/cms/schemaTypes/blocks/index.ts`

Add your block to the exports:

```typescript
import { blockName } from "./block-name";

export const pageBuilderBlocks = [
  hero,
  cta,
  featureCards,
  blockName, // Add here
  // ... other blocks
];
```

The `pageBuilderBlocks` array is automatically used in `apps/cms/schemaTypes/definitions/pagebuilder.ts` to register blocks in the page builder.

## Step 3: Add GROQ Fragment

**Location**: `apps/web/src/lib/sanity/query.ts`

### Create Block Fragment

Add a fragment for your block using existing fragments when available:

```typescript
const blockNameBlock = /* groq */ `
  _type == "blockName" => {
    ...,
    ${richTextFragment},      // If block has rich text
    ${buttonsFragment},        // If block has buttons
    ${imageFragment},          // If block has image
    // Add custom fields
    "customField": customField,
    // For arrays with references
    "items": array::compact(items[]->{
      _id,
      _type,
      title,
      ${imageFragment}
    })
  }
`;
```

### Add to Page Builder Fragment

Add your block fragment to `pageBuilderFragment`:

```typescript
const pageBuilderFragment = /* groq */ `
  pageBuilder[]{
    ...,
    _type,
    ${ctaBlock},
    ${heroBlock},
    ${blockNameBlock},  // Add here
    // ... other blocks
  }
`;
```

### Available Fragments

- `imageFields` - Basic image fields
- `imageFragment` - Full image object
- `richTextFragment` - Rich text with markDefs
- `buttonsFragment` - Button array
- `baseUrlFragment` - URL handling
- `customLinkFragment` - Custom links

## Step 4: Create React Component

**Location**: `apps/web/src/components/sections/[block-name].tsx`

### Template (UI)

```typescript
import type { PagebuilderType } from "@/lib/sanity/types";
import { RichText } from "@/components/elements/rich-text";
import { SanityButtons } from "@/components/elements/sanity-buttons";
import { SanityImage } from "@/components/elements/sanity-image";
import { Badge } from "@axioned/ui/components/badge";

type BlockNameProps = PagebuilderType<"blockName">;

export function BlockNameBlock({
  title,
  buttons,
  badge,
  image,
  richText,
  items,
}: BlockNameProps) {
  return (
    <section className="mt-4 md:my-16" id="block-name">
      <div className="container mx-auto px-4 md:px-6">
        {badge && <Badge variant="secondary">{badge}</Badge>}
        {title && <h2>{title}</h2>}
        {richText && <RichText richText={richText} />}
        {buttons && (
          <SanityButtons
            buttons={buttons}
            className="grid gap-2 sm:grid-flow-col"
          />
        )}
        {image && (
          <SanityImage
            image={image}
            className="rounded-3xl object-cover"
            width={800}
            height={800}
          />
        )}
        {items && (
          <div className="grid gap-6 md:grid-cols-2 lg:grid-cols-3">
            {items.map((item) => (
              <div key={item._id}>
                {/* Item content */}
              </div>
            ))}
          </div>
        )}
      </div>
    </section>
  );
}
```

### Component Patterns

**Simple Content Block**:

```typescript
export function SimpleBlock({ title, richText }: SimpleBlockProps) {
  return (
    <section className="mt-4 md:my-16">
      <div className="container mx-auto px-4 md:px-6">
        {title && <h2>{title}</h2>}
        <RichText richText={richText} />
      </div>
    </section>
  );
}
```

**Card Grid**:

```typescript
export function CardsBlock({ title, cards }: CardsBlockProps) {
  return (
    <section className="mt-4 md:my-16">
      <div className="container mx-auto px-4 md:px-6">
        {title && <h2>{title}</h2>}
        <div className="grid gap-6 md:grid-cols-2 lg:grid-cols-3">
          {cards?.map((card) => (
            <Card key={card._key}>
              {card.image && <SanityImage image={card.image} />}
              {card.title && <h3>{card.title}</h3>}
              {card.richText && <RichText richText={card.richText} />}
            </Card>
          ))}
        </div>
      </div>
    </section>
  );
}
```

## Step 5: Register Component

**Location**: `apps/web/src/components/page-builder.tsx`

Add your component to `BLOCK_COMPONENTS`:

```typescript
import { BlockNameBlock } from "./sections/block-name";

const BLOCK_COMPONENTS = {
  cta: CTABlock,
  hero: HeroBlock,
  blockName: BlockNameBlock, // Add here - key matches schema name
  // ... other blocks
} as const satisfies Record<PageBuilderBlockTypes, React.ComponentType<any>>;
```

**Important**: The key in `BLOCK_COMPONENTS` must match the `name` in your schema definition.

## Step 6: Generate Types

After completing all steps, regenerate Sanity types:

```bash
cd apps/cms && sanity schema extract && sanity typegen generate --enforce-required-fields
```

Or from root:

```bash
pnpm typegen
```

## Best Practices

For comprehensive best practices, see the `sanity-best-practices` skill. Key points for blocks:

- **File naming**: Always use kebab-case for files and schema names
- **Type safety**: Use `PagebuilderType<"blockName">` for component props
- **Fragment reuse**: Use existing fragments when available
- **Array handling**: Use `array::compact()` in GROQ to remove null/undefined
- **Optional fields**: Handle optional fields with conditional rendering
- **Component key**: Must match schema `name` exactly

## Complete Example

### Schema (`apps/cms/schemaTypes/blocks/feature-cards.ts`)

```typescript
import { customRichText } from "@/schemaTypes/definitions/rich-text";
import { Grid3x3 } from "lucide-react";
import { defineField, defineType } from "sanity";

export const featureCards = defineType({
  name: "featureCards",
  title: "Feature Cards",
  icon: Grid3x3,
  type: "object",
  fields: [
    defineField({
      name: "title",
      title: "Title",
      description: "The main heading for this section",
      type: "string",
    }),
    customRichText(["block"]),
    defineField({
      name: "cards",
      title: "Cards",
      type: "array",
      of: [
        {
          type: "object",
          fields: [
            defineField({
              name: "title",
              type: "string",
              title: "Title",
            }),
            customRichText(["block"]),
          ],
        },
      ],
    }),
  ],
  preview: {
    select: {
      title: "title",
    },
    prepare: ({ title }) => ({
      title,
      subtitle: "Feature Cards",
    }),
  },
});
```

### GROQ Fragment (`apps/web/src/lib/sanity/query.ts`)

```typescript
const featureCardsBlock = /* groq */ `
  _type == "featureCards" => {
    ...,
    ${richTextFragment},
    "cards": array::compact(cards[]{
      ...,
      ${richTextFragment},
    })
  }
`;

// Add to pageBuilderFragment
const pageBuilderFragment = /* groq */ `
  pageBuilder[]{
    ...,
    _type,
    ${featureCardsBlock},
    // ... other blocks
  }
`;
```

### Component (`apps/web/src/components/sections/feature-cards.tsx`)

```typescript
import type { PagebuilderType } from "@/lib/sanity/types";
import { RichText } from "@/components/elements/rich-text";

type FeatureCardsProps = PagebuilderType<"featureCards">;

export function FeatureCardsBlock({
  title,
  richText,
  cards,
}: FeatureCardsProps) {
  return (
    <section className="mt-4 md:my-16" id="feature-cards">
      <div className="container mx-auto px-4 md:px-6">
        {title && <h2>{title}</h2>}
        {richText && <RichText richText={richText} />}
        {cards && (
          <div className="grid gap-6 md:grid-cols-2 lg:grid-cols-3">
            {cards.map((card) => (
              <div key={card._key}>
                {card.title && <h3>{card.title}</h3>}
                {card.richText && <RichText richText={card.richText} />}
              </div>
            ))}
          </div>
        )}
      </div>
    </section>
  );
}
```

### Registration (`apps/web/src/components/page-builder.tsx`)

```typescript
import { FeatureCardsBlock } from "./sections/feature-cards";

const BLOCK_COMPONENTS = {
  // ...
  featureCards: FeatureCardsBlock,
} as const;
```
