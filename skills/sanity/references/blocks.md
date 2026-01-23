# Blocks Reference

## Complete Workflow

1. Create block schema in CMS
2. Register block in blocks index
3. Add GROQ fragment
4. Create React component
5. Register component in page-builder
6. Generate types

## Step 1: Create Block Schema

**Location**: `apps/cms/schemaTypes/blocks/[block-name].ts`  
**File naming**: kebab-case (e.g. `feature-cards.ts`, `testimonials-carousel.ts`)

### Template (Schema)

```typescript
import { buttonsField, imageWithAltField } from "@/schemaTypes/common";
import { customRichText } from "@/schemaTypes/definitions/rich-text";
import { IconName } from "lucide-react";
import { defineField, defineType } from "sanity";

export const blockName = defineType({
  name: "blockName",
  title: "Block Display Name",
  icon: IconName,
  type: "object",
  fields: [
    defineField({ name: "title", title: "Title", description: "The primary focus of the block", type: "string" }),
    customRichText(["block"]),
    imageWithAltField(),
    buttonsField,
    defineField({ name: "customField", title: "Custom Field", description: "Description", type: "string" }),
  ],
  preview: {
    select: { title: "title" },
    prepare: ({ title }) => ({ title, subtitle: "Block Display Name" }),
  },
});
```

### Common Field Helpers (`apps/cms/schemaTypes/common.ts`)

- `buttonsField` – button array
- `imageWithAltField()` – image with alt
- `richTextField` – rich text
- `iconField` – icon picker
- `customRichText(["block"])` – rich text with block types

## Step 2: Register Block

**Location**: `apps/cms/schemaTypes/blocks/index.ts`

```typescript
import { blockName } from "./block-name";

export const pageBuilderBlocks = [hero, cta, featureCards, blockName];
```

`pageBuilderBlocks` is used in `apps/cms/schemaTypes/definitions/pagebuilder.ts`.

## Step 3: Add GROQ Fragment

**Location**: `apps/web/src/lib/sanity/query.ts`

### Block fragment

```typescript
const blockNameBlock = /* groq */ `
  _type == "blockName" => {
    ...,
    ${richTextFragment},
    ${buttonsFragment},
    ${imageFragment},
    "customField": customField,
    "items": array::compact(items[]->{ _id, _type, title, ${imageFragment} })
  }
`;
```

### Add to `pageBuilderFragment`

```typescript
const pageBuilderFragment = /* groq */ `
  pageBuilder[]{
    ...,
    _type,
    ${ctaBlock},
    ${heroBlock},
    ${blockNameBlock}
  }
`;
```

### Available fragments

`imageFields`, `imageFragment`, `richTextFragment`, `buttonsFragment`, `baseUrlFragment`, `customLinkFragment`

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

export function BlockNameBlock({ title, buttons, badge, image, richText, items }: BlockNameProps) {
  return (
    <section className="mt-4 md:my-16" id="block-name">
      <div className="container mx-auto px-4 md:px-6">
        {badge && <Badge variant="secondary">{badge}</Badge>}
        {title && <h2>{title}</h2>}
        {richText && <RichText richText={richText} />}
        {buttons && <SanityButtons buttons={buttons} className="grid gap-2 sm:grid-flow-col" />}
        {image && <SanityImage image={image} className="rounded-3xl object-cover" width={800} height={800} />}
        {items && (
          <div className="grid gap-6 md:grid-cols-2 lg:grid-cols-3">
            {items.map((item) => (
              <div key={item._id}>{/* Item content */}</div>
            ))}
          </div>
        )}
      </div>
    </section>
  );
}
```

### Simple block

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

### Card grid

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

The key must match the schema `name`:

```typescript
import { BlockNameBlock } from "./sections/block-name";

const BLOCK_COMPONENTS = {
  cta: CTABlock,
  hero: HeroBlock,
  blockName: BlockNameBlock,
} as const satisfies Record<PageBuilderBlockTypes, React.ComponentType<any>>;
```

## Step 6: Generate Types

```bash
pnpm typegen
```

Or:

```bash
cd apps/cms && sanity schema extract && sanity typegen generate --enforce-required-fields
```

## Best Practices

- kebab-case files; `name` in camelCase.
- `PagebuilderType<"blockName">` for props.
- Reuse fragments; `array::compact()` for arrays.
- Optional fields: conditional render. Use `_key` or `_id` in maps.
- `BLOCK_COMPONENTS` key must equal schema `name`.
