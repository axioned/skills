# GROQ Queries Reference

## File Location

All GROQ queries: `apps/web/src/lib/sanity/query.ts`

## Query Structure

### Required Imports

```typescript
import { defineQuery } from "next-sanity";
```

### Basic Query Template

```typescript
export const queryNameQuery = defineQuery(`
  *[_type == "documentType" && condition][0]{
    ...,
    _id,
    _type,
    "slug": slug.current,
    // Add projections here
  }
`);
```

## Fragment Composition

### Creating Reusable Fragments

Define fragments at the top of the file. Use descriptive names:

```typescript
// Base image fields (most granular)
const imageFields = /* groq */ `
  "id": asset._ref,
  "preview": asset->metadata.lqip,
  "alt": coalesce(
    alt,
    asset->altText,
    caption,
    asset->originalFilename,
    "untitled"
  ),
  hotspot { x, y },
  crop { bottom, left, right, top }
`;

// Full image object (uses imageFields)
const imageFragment = /* groq */ `
  image { ${imageFields} }
`;

// Rich text with mark definitions
const richTextFragment = /* groq */ `
  richText[]{
    ...,
    _type == "block" => {
      ...,
      markDefs[]{
        ...,
        ...customLink{ openInNewTab, ${baseUrlFragment} }
      }
    },
    _type == "image" => { ${imageFields}, "caption": caption }
  }
`;

// Buttons array
const buttonsFragment = /* groq */ `
  buttons[]{ text, variant, _key, _type, ${baseUrlFragment} }
`;
```

### Fragment Naming

- **Base**: `imageFields`, `baseUrlFragment`
- **Composed**: `imageFragment`, `richTextFragment`
- **Block**: `heroBlock`, `ctaBlock`
- **Internal**: `_richText`, `_buttons`

## Common Fragment Patterns

### URL Handling

```typescript
const baseUrlFragment = /* groq */ `
  "openInNewTab": url.openInNewTab,
  "href": select(
    url.type == "internal" => url.internal->slug.current,
    url.type == "external" => url.external,
    url.href
  )
`;
```

### Image Handling

**Do not expand images inline. Use fragments.**

```typescript
// ✅ CORRECT
const blockFragment = /* groq */ `
  _type == "block" => { ..., ${imageFragment} }
`;

// ❌ INCORRECT
const blockFragment = /* groq */ `
  _type == "block" => { ..., image.asset->url }
`;
```

### Array Handling

```typescript
// Inline arrays
"cards": array::compact(cards[]{ _key, title, ${imageFragment}, ${baseUrlFragment} })

// Referenced arrays (joins)
"items": array::compact(items[]->{ _id, _type, title, ${imageFragment} })

// Nested arrays
"columns": columns[]{
  _key, title,
  "links": links[]{ _key, name, ${baseUrlFragment} }
}
```

## Block Fragment Patterns

### Simple Block

```typescript
const simpleBlock = /* groq */ `
  _type == "simpleBlock" => { ..., ${richTextFragment} }
`;
```

### Block with Image and Buttons

```typescript
const heroBlock = /* groq */ `
  _type == "hero" => {
    ...,
    ${imageFragment},
    ${buttonsFragment},
    ${richTextFragment}
  }
`;
```

### Block with Referenced Array

```typescript
const testimonialsCarouselBlock = /* groq */ `
  _type == "testimonialsCarousel" => {
    ...,
    "testimonials": array::compact(testimonials[]->{
      _id, _type, title, text,
      client->{ _id, name, logo { ${imageFields} } }
    })
  }
`;
```

### Block with Inline Array

```typescript
const featureCardsBlock = /* groq */ `
  _type == "featureCards" => {
    ...,
    ${richTextFragment},
    "cards": array::compact(cards[]{ ..., ${richTextFragment} })
  }
`;
```

## Page Builder Fragment

Compose all block fragments:

```typescript
const pageBuilderFragment = /* groq */ `
  pageBuilder[]{
    ...,
    _type,
    ${ctaBlock},
    ${heroBlock},
    ${faqAccordionBlock},
    ${featureCardsBlock},
    ${imageLinkCardsBlock},
    ${workCarouselBlock},
    ${testimonialsCarouselBlock},
    ${technologyGridBlock},
    ${stickyScrollBlock}
  }
`;
```

## Query Patterns

### Single Document by Slug

```typescript
export const querySlugPageData = defineQuery(`
  *[_type == "page" && slug.current == $slug][0]{
    ...,
    "slug": slug.current,
    ${pageBuilderFragment}
  }
`);
```

### All Documents of Type

```typescript
export const queryAllPages = defineQuery(`
  *[_type == "page" && defined(slug.current)]{
    _id, "slug": slug.current, title
  }
`);
```

### Paginated Queries

```typescript
export const queryPaginatedItems = defineQuery(`
  *[_type == "work" && (seoHideFromLists != true)]
  | order(orderRank asc) [$start...$end]{ ${workCardFragment} }
`);
```

### Count Queries

```typescript
export const queryCount = defineQuery(`
  count(*[_type == "work" && (seoHideFromLists != true)])
`);
```

### Conditional Fields

```typescript
export const queryWithCondition = defineQuery(`
  *[_type == "page"][0]{
    ...,
    "title": select(defined(ogTitle) => ogTitle, defined(seoTitle) => seoTitle, title),
    "hasImage": defined(image)
  }
`);
```

### References and Joins

```typescript
const workCardFragment = /* groq */ `
  _type, _id, title, description, "slug": slug.current,
  client->{ _id, name, logo { ${imageFields} } },
  ${imageFragment}
`;
```

## Common Fragment Library

- **Base**: `imageFields`, `baseUrlFragment`
- **Composed**: `imageFragment`, `richTextFragment`, `buttonsFragment`, `customLinkFragment`
- **Block**: `heroBlock`, `ctaBlock`, `faqAccordionBlock`, `featureCardsBlock`, `imageLinkCardsBlock`, `workCarouselBlock`, `testimonialsCarouselBlock`, `technologyGridBlock`, `stickyScrollBlock`
- **Page**: `pageBuilderFragment`, `workCardFragment`, `ogFieldsFragment`

## Example: New Block Query

### 1. Block fragment

```typescript
const newBlockFragment = /* groq */ `
  _type == "newBlock" => {
    ...,
    ${richTextFragment},
    ${buttonsFragment},
    "items": array::compact(items[]->{ _id, _type, title, ${imageFragment} })
  }
`;
```

### 2. Add to `pageBuilderFragment`

```typescript
pageBuilder[]{ ..., _type, ${newBlockFragment}, ... }
```

### 3. Use in document query

```typescript
*[_type == "page" && slug.current == $slug][0]{ ..., ${pageBuilderFragment} }
```

## Advanced Patterns

### Conditional Projections

```typescript
"image": select(defined(image) => { ${imageFields} }, null)
```

### Nested Conditionals

```typescript
columns[]{
  _key,
  _type == "navbarColumn" => { "type": "column", title, links[]{ _key, name, ${baseUrlFragment} } },
  _type == "navbarLink" => { "type": "link", name, ${baseUrlFragment} }
}
```

### Array Operations

```typescript
"activeItems": items[active == true]
"firstThree": items[0...3]
"processedItems": items[]{ ..., "isActive": active == true }
```

## Best Practices

- **Fragments**: Start with base, compose up, reuse. Use `${name}`. One concern per fragment.
- **Queries**: Explicit `_type`, projections, `order()`, `defined()`, `array::compact()`, `coalesce()`.
- **Images**: Always use fragments; never expand inline.
- **Naming**: Queries `queryXxxData`, fragments `xxxFragment` or `xxxBlock`.
