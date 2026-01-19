---
name: sanity-queries
description: Create and compose GROQ queries with fragments following project patterns and best practices
---

# Create Sanity GROQ Queries

## When to use this skill

Use this skill when you need to create or modify GROQ queries for fetching data from Sanity CMS. Focus on fragment composition, query patterns, and best practices.

## File Location

All GROQ queries are located in: `apps/web/src/lib/sanity/query.ts`

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

Define fragments at the top of the file for common patterns. Use descriptive names prefixed with the data type:

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
  hotspot {
    x,
    y
  },
  crop {
    bottom,
    left,
    right,
    top
  }
`;

// Full image object (uses imageFields)
const imageFragment = /* groq */ `
  image {
    ${imageFields}
  }
`;

// Rich text with all mark definitions
const richTextFragment = /* groq */ `
  richText[]{
    ...,
    _type == "block" => {
      ...,
      markDefs[]{
        ...,
        ...customLink{
          openInNewTab,
          ${baseUrlFragment},
        }
      }
    },
    _type == "image" => {
      ${imageFields},
      "caption": caption
    }
  }
`;

// Buttons array
const buttonsFragment = /* groq */ `
  buttons[]{
    text,
    variant,
    _key,
    _type,
    ${baseUrlFragment},
  }
`;
```

### Fragment Naming Conventions

- **Base fragments**: Use descriptive names (e.g., `imageFields`, `baseUrlFragment`)
- **Composed fragments**: Use type suffix (e.g., `imageFragment`, `richTextFragment`)
- **Block fragments**: Use block name (e.g., `heroBlock`, `ctaBlock`)
- **Prefix reusable fragments**: Use underscore for internal fragments (e.g., `_richText`, `_buttons`)

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

const customLinkFragment = /* groq */ `
  ...customLink{
    openInNewTab,
    ${baseUrlFragment},
  }
`;
```

### Image Handling

**IMPORTANT**: When there is an image within a GROQ query, do NOT expand it unless explicitly instructed. Use fragments:

```typescript
// ✅ CORRECT - Use fragment
const blockFragment = /* groq */ `
  _type == "block" => {
    ...,
    ${imageFragment}
  }
`;

// ❌ INCORRECT - Don't expand images directly
const blockFragment = /* groq */ `
  _type == "block" => {
    ...,
    image.asset->url  // Don't do this
  }
`;
```

### Array Handling

```typescript
// Inline arrays
const cardsFragment = /* groq */ `
  "cards": array::compact(cards[]{
    _key,
    title,
    ${imageFragment},
    ${baseUrlFragment}
  })
`;

// Referenced arrays (with joins)
const itemsFragment = /* groq */ `
  "items": array::compact(items[]->{
    _id,
    _type,
    title,
    ${imageFragment}
  })
`;

// Nested arrays
const nestedFragment = /* groq */ `
  "columns": columns[]{
    _key,
    title,
    "links": links[]{
      _key,
      name,
      ${baseUrlFragment}
    }
  }
`;
```

## Block Fragment Patterns

### Simple Block

```typescript
const simpleBlock = /* groq */ `
  _type == "simpleBlock" => {
    ...,
    ${richTextFragment}
  }
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
      _id,
      _type,
      title,
      text,
      client->{
        _id,
        name,
        logo {
          ${imageFields}
        }
      }
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
    "cards": array::compact(cards[]{
      ...,
      ${richTextFragment},
    })
  }
`;
```

## Page Builder Fragment

Compose all block fragments into the page builder:

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
    ${stickyScrollBlock},
    // Add new blocks here
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
    _id,
    "slug": slug.current,
    title
  }
`);
```

### Paginated Queries

```typescript
export const queryPaginatedItems = defineQuery(`
  *[_type == "work" && (seoHideFromLists != true)] 
  | order(orderRank asc) 
  [$start...$end]{
    ${workCardFragment}
  }
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
    "title": select(
      defined(ogTitle) => ogTitle,
      defined(seoTitle) => seoTitle,
      title
    ),
    "hasImage": defined(image)
  }
`);
```

### References and Joins

```typescript
const workCardFragment = /* groq */ `
  _type,
  _id,
  title,
  description,
  "slug": slug.current,
  client->{
    _id,
    name,
    logo {
      ${imageFields}
    }
  },
  ${imageFragment}
`;
```

## Best Practices

For comprehensive best practices, see the `sanity-best-practices` skill. Key points for queries:

- **Fragment composition**: Start with base fragments, compose upward, reuse existing fragments
- **Query structure**: Use explicit filtering, projection over full docs, null safety with `defined()`, array safety with `array::compact()`
- **Naming**: Queries use camelCase with "Query" suffix, fragments use "Fragment" suffix
- **Image handling**: Never expand images directly, always use fragments

## Common Fragment Library

The project maintains these reusable fragments:

### Base Fragments

- `imageFields` - Basic image field projection
- `baseUrlFragment` - URL handling for internal/external links

### Composed Fragments

- `imageFragment` - Full image object with fields
- `richTextFragment` - Rich text with markDefs and images
- `buttonsFragment` - Button array with URLs
- `customLinkFragment` - Custom link with URL handling

### Block Fragments

- `heroBlock` - Hero section
- `ctaBlock` - Call to action
- `faqAccordionBlock` - FAQ accordion
- `featureCardsBlock` - Feature cards grid
- `imageLinkCardsBlock` - Image link cards
- `workCarouselBlock` - Work carousel
- `testimonialsCarouselBlock` - Testimonials carousel
- `technologyGridBlock` - Technology grid
- `stickyScrollBlock` - Sticky scroll section

### Page Fragments

- `pageBuilderFragment` - Complete page builder array
- `workCardFragment` - Work card projection
- `ogFieldsFragment` - OpenGraph fields

## Example: Creating a New Block Query

### 1. Create Block Fragment

```typescript
const newBlockFragment = /* groq */ `
  _type == "newBlock" => {
    ...,
    ${richTextFragment},
    ${buttonsFragment},
    "items": array::compact(items[]->{
      _id,
      _type,
      title,
      ${imageFragment}
    })
  }
`;
```

### 2. Add to Page Builder

```typescript
const pageBuilderFragment = /* groq */ `
  pageBuilder[]{
    ...,
    _type,
    ${newBlockFragment},  // Add here
    // ... other blocks
  }
`;
```

### 3. Use in Document Query

```typescript
export const queryPageData = defineQuery(`
  *[_type == "page" && slug.current == $slug][0]{
    ...,
    ${pageBuilderFragment}
  }
`);
```

## Advanced Patterns

### Conditional Projections

```typescript
const conditionalFragment = /* groq */ `
  "image": select(
    defined(image) => {
      ${imageFields}
    },
    null
  )
`;
```

### Nested Conditionals

```typescript
const complexFragment = /* groq */ `
  columns[]{
    _key,
    _type == "navbarColumn" => {
      "type": "column",
      title,
      links[]{
        _key,
        name,
        ${baseUrlFragment}
      }
    },
    _type == "navbarLink" => {
      "type": "link",
      name,
      ${baseUrlFragment}
    }
  }
`;
```

### Array Operations

```typescript
// Filter array
"activeItems": items[active == true]

// Slice array
"firstThree": items[0...3]

// Map with condition
"processedItems": items[]{
  ...,
  "isActive": active == true
}
```
