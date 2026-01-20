---
name: base-skill
description: Base skill demonstrating Agent Skills format. Load when learning to create skills or testing the skill creation process.
---

# Base Skill

This is a base skill demonstrating the Agent Skills format.

## Purpose

This skill shows how to structure procedural guidance for AI coding agents using Agent Skills format.

## When to Use

Load this skill when:

- Learning how skills work
- Creating new skills
- Testing the how skills get loaded

## Instructions

To create a skill:

1. Create a directory: `mkdir my-skill/`
2. Add SKILL.md with YAML frontmatter:

   ```yaml
   ---
   name: my-skill
   description: When to use this skill
   ---
   ```

3. Write instructions in imperative form (not second person)

## Best Practices

- Write in imperative/infinitive form: "To do X, execute Y"
- NOT second person: avoid "You should..."
- Keep SKILL.md under 5,000 words
