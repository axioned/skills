---
name: sanity-documents
description: Complete workflow for creating Sanity document types from schema to dynamic routes and sitemap integration
---

# Create Sanity Documents

## When to use this skill

Use this skill when you need to create a new document type in Sanity CMS that will be rendered as pages on the website. This covers the complete workflow from schema definition to dynamic routes and sitemap integration.

## Complete Workflow

1. Create document schema in CMS
2. Register document in documents index
3. Create GROQ queries for fetching data
4. Create dynamic route (if slug-based)
5. Add to sitemap (if public page)

## Step 1: Create Document Schema

**Location**: `apps/cms/schemaTypes/documents/[document-name].ts`

**File naming**: Use kebab-case (e.g., `page.ts`, `work.ts`, `testimonial.ts`)

### Template

```typescript
import { defineField, defineType } from "sanity";
import { seoFields } from "@/utils/seo-fields";
import { ogFields } from "@/utils/og-fields";
import { documentSlugField, pageBuilderField } from "@/schemaTypes/common";
import { IconName } from "lucide-react";

export const documentName = defineType({
  name: "documentName", // kebab-case
  title: "Document Display Name",
  type: "document",
  icon: IconName,
  fields: [
    defineField({
      name: "title",
      title: "Title",
      description: "The main title for this document",
      type: "string",
      validation: (Rule) => Rule.required(),
    }),
    // Slug field (required for routes)
    documentSlugField("documentName", {
      description: "The web address where people can find this page",
    }),
    // Page builder (if document uses page builder)
    pageBuilderField,
    // SEO fields (for public pages)
    ...seoFields,
    ...ogFields,
    // Custom fields
    defineField({
      name: "customField",
      title: "Custom Field",
      description: "Description",
      type: "string",
    }),
  ],
  preview: {
    select: {
      title: "title",
      subtitle: "slug.current",
    },
    prepare: ({ title, subtitle }) => ({
      title,
      subtitle: subtitle ? `/${subtitle}` : "No slug",
    }),
  },
});
```

### Common Field Helpers

From `apps/cms/schemaTypes/common.ts`:

- `documentSlugField(documentType, options)` - Slug field with validation
- `pageBuilderField` - Page builder array
- `imageWithAltField()` - Image with alt text
- `buttonsField` - Array of buttons
- `richTextField` - Rich text editor

### SEO Fields

For public pages, include:

```typescript
...seoFields,  // seoTitle, seoDescription, seoHideFromLists
...ogFields,   // ogTitle, ogDescription, ogImage
```

## Step 2: Register Document

**Location**: `apps/cms/schemaTypes/documents/index.ts`

Add your document to the exports:

```typescript
import { documentName } from "./document-name";

export const documents = [
  page,
  work,
  documentName, // Add here
  // ... other documents
];
```

## Step 3: Create GROQ Queries

**Location**: `apps/web/src/lib/sanity/query.ts`

### Query by Slug

```typescript
export const queryDocumentSlugData = defineQuery(`
  *[_type == "documentName" && slug.current == $slug][0]{
    ...,
    "slug": slug.current,
    ${pageBuilderFragment}  // If using page builder
  }
`);
```

### Query All Slugs (for static params)

```typescript
export const queryDocumentSlugPaths = defineQuery(`
  *[_type == "documentName" && defined(slug.current)].slug.current
`);
```

### Query for Sitemap

```typescript
export const queryDocumentSitemapData = defineQuery(`
  *[_type == "documentName" && defined(slug.current)]{
    "slug": slug.current,
    "lastModified": _updatedAt
  }
`);
```

### Query with Filters

```typescript
export const queryDocumentList = defineQuery(`
  *[_type == "documentName" && (seoHideFromLists != true)] 
  | order(orderRank asc) 
  [$start...$end]{
    _id,
    _type,
    title,
    "slug": slug.current,
    ${imageFragment}
  }
`);
```

## Step 4: Create Dynamic Route

**Location**: `apps/web/src/app/[...slug]/page.tsx` (for catch-all) or specific route

### For Catch-All Route

If using the existing catch-all route at `apps/web/src/app/[...slug]/page.tsx`, ensure your query is added to handle your document type.

### For Specific Route

Create: `apps/web/src/app/[category]/[...slug]/page.tsx`

```typescript
import { draftMode } from "next/headers";
import { notFound } from "next/navigation";
import { PageBuilder } from "@/components/page-builder";
import { client } from "@/lib/sanity/client";
import { queryDocumentSlugData, queryDocumentSlugPaths } from "@/lib/sanity/query";

async function fetchDocumentData(slug: string) {
  const { isEnabled } = await draftMode();
  return await client.fetch(
    queryDocumentSlugData,
    { slug },
    isEnabled
      ? {
          perspective: "drafts",
          useCdn: false,
          stega: true,
        }
      : undefined,
  );
}

async function fetchDocumentPaths() {
  try {
    const slugs = await client.fetch(queryDocumentSlugPaths);
    if (!Array.isArray(slugs) || slugs.length === 0) {
      return [];
    }
    const paths: { slug: string[] }[] = [];
    for (const slug of slugs) {
      if (!slug) continue;
      const parts = slug.split("/").filter(Boolean);
      paths.push({ slug: parts });
    }
    return paths;
  } catch (error) {
    console.error("Error fetching paths", error);
    return [];
  }
}

export const dynamicParams = true;

export async function generateStaticParams() {
  return await fetchDocumentPaths();
}

export default async function Page({
  params,
}: {
  params: Promise<{ slug: string[] }>;
}) {
  const { slug } = await params;
  const slugString = slug.join("/");
  const documentData = await fetchDocumentData(slugString);

  if (!documentData) {
    notFound();
  }

  const { _id, _type, pageBuilder } = documentData ?? {};

  return (
    <PageBuilder id={_id} pageBuilder={pageBuilder ?? []} type={_type} />
  );
}
```

### Metadata Export

For SEO, add metadata:

```typescript
import type { Metadata } from "next";
import { client } from "@/lib/sanity/client";
import { queryDocumentOGData } from "@/lib/sanity/query";

export async function generateMetadata({ params }: { params: Promise<{ slug: string[] }> }): Promise<Metadata> {
  const { slug } = await params;
  const slugString = slug.join("/");
  const data = await client.fetch(queryDocumentOGData, {
    id: slugString,
  });

  if (!data) {
    return {};
  }

  return {
    title: data.title,
    description: data.description,
    openGraph: {
      title: data.title,
      description: data.description,
      images: data.image ? [data.image] : [],
    },
  };
}
```

## Step 5: Add to Sitemap

**Location**: `apps/web/src/app/sitemap.ts`

### Update Sitemap Query

First, update the GROQ query to include your document type:

```typescript
export const querySitemapData = defineQuery(`{
  "slugPages": *[_type == "page" && defined(slug.current)]{
    "slug": slug.current,
    "lastModified": _updatedAt
  },
  "workPages": *[_type == "work" && defined(slug.current)]{
    "slug": slug.current,
    "lastModified": _updatedAt
  },
  "documentPages": *[_type == "documentName" && defined(slug.current)]{
    "slug": slug.current,
    "lastModified": _updatedAt
  }
}`);
```

### Update Sitemap Component

```typescript
import type { QuerySitemapDataResult } from "@/lib/sanity/sanity.types";
import type { MetadataRoute } from "next";
import { env } from "@/env";
import { client } from "@/lib/sanity/client";
import { querySitemapData } from "@/lib/sanity/query";

type Page = QuerySitemapDataResult["slugPages"][number];

const baseUrl = env.NEXT_PUBLIC_APP_URL;

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const { slugPages, workPages, documentPages } = await client.fetch(querySitemapData);

  return [
    {
      url: baseUrl,
      lastModified: new Date(),
      changeFrequency: "weekly",
      priority: 1,
    },
    ...slugPages.map((page: Page) => ({
      url: `${baseUrl}${page.slug}`,
      lastModified: new Date(page.lastModified || new Date()),
      changeFrequency: "weekly" as const,
      priority: 0.8,
    })),
    ...workPages.map((page: Page) => ({
      url: `${baseUrl}${page.slug}`,
      lastModified: new Date(page.lastModified || new Date()),
      changeFrequency: "weekly" as const,
      priority: 0.5,
    })),
    ...documentPages.map((page: Page) => ({
      url: `${baseUrl}${page.slug}`,
      lastModified: new Date(page.lastModified || new Date()),
      changeFrequency: "weekly" as const,
      priority: 0.7, // Adjust priority as needed
    })),
  ];
}
```

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

For comprehensive best practices, see the `sanity-best-practices` skill. Key points for documents:

- **Slug validation**: Always use `documentSlugField()` helper
- **SEO**: Include `seoFields` and `ogFields` for public pages
- **Draft mode**: Always support draft mode for preview
- **Error handling**: Use `notFound()` for missing documents
- **Static generation**: Generate static params for better performance
- **Sitemap**: Add all public pages to sitemap
- **File naming**: Use kebab-case for all files

## Complete Example

### Schema (`apps/cms/schemaTypes/documents/work.ts`)

```typescript
import { defineField, defineType } from "sanity";
import { seoFields } from "@/utils/seo-fields";
import { ogFields } from "@/utils/og-fields";
import { documentSlugField, pageBuilderField, imageWithAltField } from "@/schemaTypes/common";
import { Briefcase } from "lucide-react";

export const work = defineType({
  name: "work",
  title: "Work",
  type: "document",
  icon: Briefcase,
  fields: [
    defineField({
      name: "title",
      title: "Title",
      type: "string",
      validation: (Rule) => Rule.required(),
    }),
    documentSlugField("work"),
    imageWithAltField(),
    pageBuilderField,
    ...seoFields,
    ...ogFields,
  ],
  preview: {
    select: {
      title: "title",
      subtitle: "slug.current",
    },
    prepare: ({ title, subtitle }) => ({
      title,
      subtitle: subtitle ? `/${subtitle}` : "No slug",
    }),
  },
});
```

### GROQ Queries (`apps/web/src/lib/sanity/query.ts`)

```typescript
export const queryWorkSlugPageData = defineQuery(`
  *[_type == "work" && slug.current == $slug][0]{
    ...,
    "slug": slug.current,
    ${imageFragment},
    ${pageBuilderFragment}
  }
`);

export const queryWorkPaths = defineQuery(`
  *[_type == "work" && defined(slug.current)].slug.current
`);
```

### Route (`apps/web/src/app/work/[...slug]/page.tsx`)

```typescript
import { draftMode } from "next/headers";
import { notFound } from "next/navigation";
import { PageBuilder } from "@/components/page-builder";
import { client } from "@/lib/sanity/client";
import { queryWorkSlugPageData, queryWorkPaths } from "@/lib/sanity/query";

// ... (implementation as shown above)
```

### Sitemap Update

```typescript
// Add to querySitemapData
"workPages": *[_type == "work" && defined(slug.current)]{
  "slug": slug.current,
  "lastModified": _updatedAt
}

// Add to sitemap return
...workPages.map((page: Page) => ({
  url: `${baseUrl}/work${page.slug}`,
  lastModified: new Date(page.lastModified || new Date()),
  changeFrequency: "weekly" as const,
  priority: 0.5,
})),
```
