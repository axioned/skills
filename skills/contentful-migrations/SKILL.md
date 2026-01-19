---
name: contentful-migrations
description: Guide for creating and managing Contentful migration scripts. Use when asked to "create a Contentful migration", "modify content types programmatically", "transform Contentful data", or "version control content model changes".
metadata:
  author: axioned
  version: "1.0.0"
---

# Contentful Migrations

Guide for creating and managing Contentful migration scripts to version control content model changes.

## When to Use Migrations

**Required for:**

- Breaking changes (field deletions, content type changes)
- Data transformations (renaming fields, changing types)

**Optional for, but recommended:**

- Adding new optional fields
- Creating new content types
- Simple additions that don't affect existing content

**General rule:** If you can't make the change in the Contentful web interface safely, use migrations.

**Manual changes** are fine for prototyping, but **migration scripts** provide version control and consistency across environments.

## Migration Process

### 1. Write Migration Scripts

Create migration files using the Contentful Migration DSL:

```javascript
// migrations/add-blog-author-field.js
module.exports = function (migration) {
  const blogPost = migration.editContentType("blogPost");

  blogPost.createField("author").name("Author").type("Symbol").required(true);

  blogPost
    .createField("authorBio")
    .name("Author Bio")
    .type("Text")
    .validations([{ size: { max: 500 } }]);
};
```

### 2. Create Test Environment

```bash
contentful space environment create --environment-id test-migration
```

### 3. Test in Sandbox

```bash
contentful space migration --environment-id test-migration migrations/add-blog-author-field.js
```

### 4. Apply to Production

Once tested, apply to master environment:

```bash
contentful space migration --environment-id master migrations/add-blog-author-field.js
```

## Migration Best Practices

### Script Organization

- Keep migrations small and focused on single changes
- Use descriptive filenames with timestamps: `2024-01-15-add-blog-author-field.js`
- Store migrations in version control
- Document complex transformations

### Testing Strategy

- Always test migrations in sandbox environments first
- Verify content model changes don't break existing entries
- Test rollback procedures when possible
- Use `--dry-run` flag to preview changes

### Content Transformation

When modifying existing content types, handle data migration carefully:

```javascript
// Renaming a field
module.exports = function (migration) {
  const blogPost = migration.editContentType("blogPost");

  // Create new field
  blogPost.createField("newTitle").name("New Title").type("Symbol");

  // Copy data from old field to new field
  migration.transformEntries({
    contentType: "blogPost",
    from: ["oldTitle"],
    to: ["newTitle"],
    transformEntryForLocale: function (fromFields, currentLocale) {
      return {
        newTitle: fromFields.oldTitle[currentLocale],
      };
    },
  });

  // Remove old field after transformation
  blogPost.deleteField("oldTitle");
};
```

### Rollback Considerations

While Contentful doesn't provide automatic rollback for migrations, you can:

- Use environment aliases to quickly switch to previous environments if needed
- Keep migration scripts in version control for reference
- Document manual rollback procedures for complex changes

## Common Migration Patterns

### Adding a Field

```javascript
module.exports = function (migration) {
  const contentType = migration.editContentType("contentTypeId");

  contentType.createField("newField").name("New Field").type("Symbol").required(false);
};
```

### Deleting a Field

```javascript
module.exports = function (migration) {
  const contentType = migration.editContentType("contentTypeId");
  contentType.deleteField("oldField");
};
```

### Changing Field Type

```javascript
module.exports = function (migration) {
  const contentType = migration.editContentType("contentTypeId");
  const field = contentType.editField("fieldId");

  field.type("Text"); // Change from Symbol to Text
};
```

### Adding Validation

```javascript
module.exports = function (migration) {
  const contentType = migration.editContentType("contentTypeId");
  const field = contentType.editField("fieldId");

  field.validations([{ size: { min: 10, max: 100 } }, { unique: true }]);
};
```

## Resources

- [Contentful CLI - Migration](https://github.com/contentful/contentful-cli)
- [Contentful Migration DSL](https://github.com/contentful/contentful-migration)
