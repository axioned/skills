---
name: sanity-best-practices
description: Comprehensive best practices for working with Sanity CMS. Use when reviewing code, setting up new features, or ensuring consistency across Sanity implementations.
---

# Sanity Best Practices

Comprehensive best practices for working with Sanity CMS across queries, documents, blocks, and automation.

## When to Use

- Use this skill when reviewing Sanity code for consistency and quality
- This skill is helpful for ensuring all Sanity implementations follow project standards
- Use when setting up new features or onboarding team members to Sanity workflows

## Instructions

### GROQ Query Best Practices

**Fragment Composition:**

1. **Start with base fragments**: Create the most granular fragments first
2. **Compose upward**: Build larger fragments from smaller ones
3. **Reuse existing fragments**: Check for existing fragments before creating new ones
4. **Use template literals**: Use `${fragmentName}` for interpolation
5. **Keep fragments focused**: Each fragment should have a single responsibility

**Query Structure:**

1. **Explicit filtering**: Use `_type == "x"` rather than implicit type checking
2. **Projection over full docs**: Only fetch fields you need
3. **Explicit sorting**: Use `order()` rather than relying on document order
4. **Null safety**: Use `defined(field)` before accessing
5. **Array safety**: Use `array::compact()` to remove null/undefined
6. **Default values**: Use `coalesce()` for fallbacks

**Image Handling:**

- **Never expand images directly** in GROQ queries
- **Always use fragments** for image fields
- Use `imageFragment` or `imageFields` fragments

**Code Style:**

1. **Template literals**: Use template literals for query strings
2. **Indentation**: Indent nested structures for readability
3. **Comments**: Add comments for complex logic
4. **Consistent spacing**: Maintain consistent whitespace
5. **Group related parts**: Keep related query parts together

**Naming Conventions:**

- **Queries**: camelCase with "Query" suffix (e.g., `queryHomePageData`)
- **Fragments**: camelCase with "Fragment" suffix (e.g., `imageFragment`)
- **Block fragments**: Block name + "Block" (e.g., `heroBlock`)
- **Internal fragments**: Prefix with underscore (e.g., `_richText`)

### Schema Best Practices

**File Naming:**

- Always use kebab-case for files and schema names
- Match filename to schema name (e.g., `feature-cards.ts` → `featureCards`)
- Keep naming consistent across schema, component, and GROQ

**Field Definitions:**

1. **Descriptions**: Include descriptions for all fields (simple terms for non-technical users)
2. **Validation**: Use validation rules for required fields
3. **Icons**: Use lucide-react icons or sanity/icons
4. **Preview**: Always include preview configuration in schema
5. **Groups**: Organize fields into logical groups

**Common Field Helpers:**

- Use `documentSlugField()` for slug fields with validation
- Use `pageBuilderField` for page builder arrays
- Use `imageWithAltField()` for images with alt text
- Use `buttonsField` for button arrays
- Use `customRichText()` for rich text with specific block types

### Document Best Practices

**Slug Validation:**

- Always use `documentSlugField()` helper for proper validation
- Ensure slugs are unique within document type
- Use descriptive slug patterns

**SEO:**

- Include `seoFields` and `ogFields` for public pages
- Use `seoHideFromLists` to control visibility
- Provide fallbacks for missing SEO fields

**Draft Mode:**

- Always support draft mode for preview
- Use `perspective: "drafts"` in queries when draft mode is enabled
- Set `useCdn: false` and `stega: true` for draft queries

**Error Handling:**

- Use `notFound()` for missing documents
- Handle undefined/null data gracefully
- Provide fallback values where appropriate

**Static Generation:**

- Generate static params for better performance
- Use `generateStaticParams()` for dynamic routes
- Handle slug arrays properly for nested routes

**Sitemap:**

- Add all public pages to sitemap
- Include `lastModified` timestamps
- Set appropriate priorities and change frequencies

### Block Best Practices

**Component Registration:**

- Component key in `BLOCK_COMPONENTS` must match schema `name`
- Use type-safe component mapping
- Handle unknown block types gracefully

**GROQ Fragments:**

- Always add block fragment to `pageBuilderFragment`
- Reuse existing fragments when available
- Use `array::compact()` for arrays to remove null values

**Type Safety:**

- Use `PagebuilderType<"blockName">` for component props
- Regenerate types after schema changes
- Use generated types from Sanity

**Fragment Reuse:**

- Use `richTextFragment` for rich text fields
- Use `buttonsFragment` for button arrays
- Use `imageFragment` for image fields
- Use `baseUrlFragment` for URL handling

**Array Handling:**

- Use `array::compact()` in GROQ to remove null/undefined
- Use `_key` or `_id` for mapped arrays
- Handle empty arrays gracefully in components

**Optional Fields:**

- Handle optional fields with conditional rendering
- Use nullish coalescing for defaults
- Don't render empty sections

### Page Builder Best Practices

**Block Naming:**

- Schema name must match component key in `BLOCK_COMPONENTS`
- File naming: Use kebab-case for files
- Consistency: Keep naming consistent across schema, component, and GROQ

**Component Registration:**

```typescript
// ✅ CORRECT - Key matches schema name
const BLOCK_COMPONENTS = {
  featureCards: FeatureCardsBlock, // schema name: "featureCards"
} as const;

// ❌ INCORRECT - Key doesn't match
const BLOCK_COMPONENTS = {
  featureCardsBlock: FeatureCardsBlock, // schema name: "featureCards"
} as const;
```

**Visual Editing:**

- Add `data-sanity` attributes for visual editing
- Configure Sanity Presentation tool properly
- Ensure environment variables are set correctly

**Error Handling:**

- Show error component for unknown block types
- Log helpful error messages
- Guide developers to fix missing components

### Automation Best Practices

**Token Management:**

1. **Separate tokens**: Use different tokens for deploy and backup
2. **Minimal permissions**: Grant only necessary permissions to tokens
3. **Rotate tokens**: Regularly rotate authentication tokens
4. **Review access**: Regularly review who has access to secrets

**Workflow Configuration:**

1. **Path filters**: Use path filters to trigger only on relevant changes
2. **Caching**: Use dependency caching for faster builds
3. **Error handling**: Include proper error handling in workflows
4. **Notifications**: Set up notifications for workflow failures

**Backup Strategy:**

1. **Regular backups**: Schedule backups at appropriate intervals
2. **Retention**: Set appropriate retention periods for backups
3. **Testing**: Test backup restoration procedures
4. **Documentation**: Document backup and restore procedures

**Security:**

1. **Never commit secrets**: Always use GitHub Secrets
2. **Limit permissions**: Grant minimal required permissions
3. **Audit logs**: Review workflow execution logs regularly
4. **Environment separation**: Use different tokens for different environments

### Type Generation Best Practices

**When to Regenerate:**

- After creating new document types
- After creating new block types
- After modifying schema fields
- After adding new GROQ queries

**Commands:**

```bash
# From root
pnpm typegen

# Or from CMS directory
cd apps/cms && sanity schema extract && sanity typegen generate --enforce-required-fields
```

**Type Usage:**

- Use generated types from `@/lib/sanity/types`
- Use `PagebuilderType<"blockName">` for block components
- Use document types for document props
- Avoid manual type definitions when types are generated

### Code Organization Best Practices

**File Structure:**

- Keep schemas in `apps/cms/schemaTypes/`
- Keep queries in `apps/web/src/lib/sanity/query.ts`
- Keep components in `apps/web/src/components/sections/`
- Use consistent directory structure

**Import Organization:**

- Group imports: external, internal, types
- Use absolute imports with `@/` alias
- Import types separately from values

**Component Patterns:**

- Use consistent component structure
- Extract reusable logic into hooks
- Use proper TypeScript types
- Handle loading and error states

### Performance Best Practices

**Query Optimization:**

- Only fetch needed fields
- Use projections instead of full documents
- Limit array sizes when possible
- Use pagination for large datasets

**Image Optimization:**

- Use Sanity image CDN
- Specify appropriate image sizes
- Use responsive images
- Lazy load images when appropriate

**Caching:**

- Use CDN for published content
- Cache queries appropriately
- Use static generation when possible
- Implement proper revalidation strategies

### Testing Best Practices

**Local Testing:**

- Test queries locally before committing
- Test component rendering with sample data
- Test error handling scenarios
- Test draft mode functionality

**Schema Testing:**

- Test schema changes in development environment
- Verify preview configurations
- Test validation rules
- Test field relationships

**Integration Testing:**

- Test complete workflows end-to-end
- Test page builder rendering
- Test dynamic route generation
- Test sitemap generation

## Common Patterns

### Error Handling Pattern

```typescript
const data = await client.fetch(query);
if (!data) {
  notFound();
}
```

### Optional Field Pattern

```typescript
{field && <Component field={field} />}
```

### Array Mapping Pattern

```typescript
{items?.map((item) => (
  <Item key={item._key || item._id} {...item} />
))}
```

### Type-Safe Component Pattern

```typescript
type Props = PagebuilderType<"blockName">;
export function BlockName({ ...props }: Props) {
  // ...
}
```

## Resources

- [Sanity Documentation](https://www.sanity.io/docs)
- [GROQ Query Language](https://www.sanity.io/docs/groq)
- [Sanity Schema Types](https://www.sanity.io/docs/schema-types)
