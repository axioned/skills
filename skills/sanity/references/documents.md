# Documents Reference

## Complete Workflow

1. Create document schema in CMS
2. Register document in documents index
3. Create GROQ queries
4. Create dynamic route (if slug-based)
5. Add to sitemap (if public)
6. Generate types

## Step 1: Create Document Schema

**Location**: `apps/cms/schemaTypes/documents/[document-name].ts`  
**File naming**: kebab-case (e.g. `page.ts`, `work.ts`, `testimonial.ts`)

### Template

```typescript
import { defineField, defineType } from "sanity";
import { seoFields } from "@/utils/seo-fields";
import { ogFields } from "@/utils/og-fields";
import { documentSlugField, pageBuilderField } from "@/schemaTypes/common";
import { IconName } from "lucide-react";

export const documentName = defineType({
  name: "documentName",
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
    documentSlugField("documentName", { description: "The web address where people can find this page" }),
    pageBuilderField,
    ...seoFields,
    ...ogFields,
    defineField({ name: "customField", title: "Custom Field", description: "Description", type: "string" }),
  ],
  preview: {
    select: { title: "title", subtitle: "slug.current" },
    prepare: ({ title, subtitle }) => ({ title, subtitle: subtitle ? `/${subtitle}` : "No slug" }),
  },
});
```

### Common Field Helpers (`apps/cms/schemaTypes/common.ts`)

- `documentSlugField(documentType, options)` – slug with validation
- `pageBuilderField` – page builder array
- `imageWithAltField()` – image with alt
- `buttonsField` – button array
- `richTextField` – rich text

### SEO

For public pages:

```typescript
...seoFields,  // seoTitle, seoDescription, seoHideFromLists
...ogFields,   // ogTitle, ogDescription, ogImage
```

## Step 2: Register Document

**Location**: `apps/cms/schemaTypes/documents/index.ts`

```typescript
import { documentName } from "./document-name";

export const documents = [page, work, documentName];
```

## Step 3: Create GROQ Queries

**Location**: `apps/web/src/lib/sanity/query.ts`

### By slug

```typescript
export const queryDocumentSlugData = defineQuery(`
  *[_type == "documentName" && slug.current == $slug][0]{
    ...,
    "slug": slug.current,
    ${pageBuilderFragment}
  }
`);
```

### All slugs (for `generateStaticParams`)

```typescript
export const queryDocumentSlugPaths = defineQuery(`
  *[_type == "documentName" && defined(slug.current)].slug.current
`);
```

### Sitemap

```typescript
export const queryDocumentSitemapData = defineQuery(`
  *[_type == "documentName" && defined(slug.current)]{
    "slug": slug.current,
    "lastModified": _updatedAt
  }
`);
```

### List with filters

```typescript
export const queryDocumentList = defineQuery(`
  *[_type == "documentName" && (seoHideFromLists != true)]
  | order(orderRank asc) [$start...$end]{
    _id, _type, title, "slug": slug.current, ${imageFragment}
  }
`);
```

## Step 4: Create Dynamic Route

**Location**: `apps/web/src/app/[...slug]/page.tsx` or `apps/web/src/app/[category]/[...slug]/page.tsx`

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
    isEnabled ? { perspective: "drafts", useCdn: false, stega: true } : undefined
  );
}

async function fetchDocumentPaths() {
  try {
    const slugs = await client.fetch(queryDocumentSlugPaths);
    if (!Array.isArray(slugs) || slugs.length === 0) return [];
    return slugs.filter(Boolean).map((slug) => ({ slug: slug.split("/").filter(Boolean) }));
  } catch (error) {
    console.error("Error fetching paths", error);
    return [];
  }
}

export const dynamicParams = true;

export async function generateStaticParams() {
  return await fetchDocumentPaths();
}

export default async function Page({ params }: { params: Promise<{ slug: string[] }> }) {
  const { slug } = await params;
  const slugString = slug.join("/");
  const documentData = await fetchDocumentData(slugString);

  if (!documentData) notFound();

  const { _id, _type, pageBuilder } = documentData ?? {};
  return <PageBuilder id={_id} pageBuilder={pageBuilder ?? []} type={_type} />;
}
```

### Metadata

```typescript
import type { Metadata } from "next";
import { client } from "@/lib/sanity/client";
import { queryDocumentOGData } from "@/lib/sanity/query";

export async function generateMetadata({ params }: { params: Promise<{ slug: string[] }> }): Promise<Metadata> {
  const { slug } = await params;
  const slugString = slug.join("/");
  const data = await client.fetch(queryDocumentOGData, { id: slugString });
  if (!data) return {};
  return {
    title: data.title,
    description: data.description,
    openGraph: { title: data.title, description: data.description, images: data.image ? [data.image] : [] },
  };
}
```

## Step 5: Add to Sitemap

**Location**: `apps/web/src/app/sitemap.ts`

### Query

```typescript
export const querySitemapData = defineQuery(`{
  "slugPages": *[_type == "page" && defined(slug.current)]{ "slug": slug.current, "lastModified": _updatedAt },
  "workPages": *[_type == "work" && defined(slug.current)]{ "slug": slug.current, "lastModified": _updatedAt },
  "documentPages": *[_type == "documentName" && defined(slug.current)]{ "slug": slug.current, "lastModified": _updatedAt }
}`);
```

### Sitemap implementation

```typescript
const baseUrl = env.NEXT_PUBLIC_APP_URL;

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const { slugPages, workPages, documentPages } = await client.fetch(querySitemapData);

  return [
    { url: baseUrl, lastModified: new Date(), changeFrequency: "weekly", priority: 1 },
    ...slugPages.map((p) => ({
      url: `${baseUrl}${p.slug}`,
      lastModified: new Date(p.lastModified || Date.now()),
      changeFrequency: "weekly" as const,
      priority: 0.8,
    })),
    ...workPages.map((p) => ({
      url: `${baseUrl}${p.slug}`,
      lastModified: new Date(p.lastModified || Date.now()),
      changeFrequency: "weekly" as const,
      priority: 0.5,
    })),
    ...documentPages.map((p) => ({
      url: `${baseUrl}${p.slug}`,
      lastModified: new Date(p.lastModified || Date.now()),
      changeFrequency: "weekly" as const,
      priority: 0.7,
    })),
  ];
}
```

## Step 6: Generate Types

```bash
pnpm typegen
```

Or from CMS package:

```bash
cd apps/cms && sanity schema extract && sanity typegen generate --enforce-required-fields
```

## Best Practices

- Use `documentSlugField()` for slugs.
- Add `seoFields` and `ogFields` for public pages.
- Support draft mode in fetch.
- Use `notFound()` when missing.
- Use `generateStaticParams` and add public pages to the sitemap.
- kebab-case for file names.
