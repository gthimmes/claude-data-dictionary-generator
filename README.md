# Data Dictionary Generator

Generate comprehensive data dictionaries by analyzing database schemas, code, and documentation sources.

## Supported Databases

- **PostgreSQL** - Full support including comments, constraints, indexes
- **MySQL/MariaDB** - Full support including comments, constraints, indexes
- **SQL Server** - Full support including extended properties, constraints, indexes
- **SQLite** - Basic support for schema extraction
- **Other SQL databases** - Via standard information_schema queries

## What This Is

This is a **Claude Code skill** - a plugin that extends Claude Code with specialized capabilities for generating database data dictionaries. When installed, you can invoke it with `/data-dictionary-generator` or Claude will automatically use it when you're discussing database documentation.

## Installation

### Step 1: Add the Marketplace

```bash
/plugin marketplace add gthimmes/claude-marketplace
```

### Step 2: Install the Plugin

```bash
/plugin install data-dictionary-generator
```

> **Note**: If the slash command doesn't work immediately after installation, restart Claude Code to refresh the skill registry.

## Quick Start

After installation, invoke the skill:

```
/data-dictionary-generator
```

Or simply ask Claude to help with database documentation:

```
"Generate a data dictionary for our user management tables"
"Document the schema for the orders database"
"Create documentation for all tables in the public schema"
```

## What This Does

The Data Dictionary Generator analyzes multiple sources to create comprehensive database documentation:

### Sources Analyzed (by priority)

1. **PostgreSQL Database Comments** - `COMMENT ON` statements (highest priority)
2. **Migration Files** - Flyway/Liquibase with schema history and business context
3. **Backend Code** - Kotlin/Spring Boot entities, repositories, services
4. **Frontend Code** - TypeScript types, Zod/Yup validation schemas, form components
5. **API Documentation** - Swagger/OpenAPI specifications
6. **Test Code** - Edge cases and business rule documentation

### Output Includes

For each table/entity:

- **Column details** - types, nullability, defaults, descriptions
- **Constraints** - primary keys, foreign keys, unique constraints, check constraints
- **Indexes** - names, columns, types, purposes
- **Validation rules** - from both backend and frontend code
- **User-facing labels** - form labels, help text, placeholders
- **Relationships** - many-to-one, one-to-many mappings
- **Schema evolution** - migration history with business context
- **Source references** - links to exact file:line locations

### Output Formats

- Markdown (default)
- YAML
- JSON
- HTML

## Requirements

- **Claude Code** with plugin support
- **Database MCP server** (required) - The skill will guide you through setup for your specific database
- **Notion MCP server** (optional) - For additional documentation context
- Access to your codebase (backend and/or frontend)

### MCP Setup

When you first run the skill, it will:

1. Ask which database type you're using (PostgreSQL, MySQL, SQL Server, SQLite)
2. Check for existing database connectivity
3. Walk you through setting up the appropriate MCP server if needed
4. Configure `~/.claude.json` with the MCP server
5. Verify the connection works

Optionally, you can also configure Notion integration for richer business context in your data dictionaries.

## Example Output

```markdown
## Table: job_templates

**Schema:** public
**Description:** Core template management. Stores reusable job configurations.

### Columns

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| id | bigint | no | nextval(...) | Primary key, auto-generated |
| firm_id | bigint | no | - | FK to firms table for multi-tenant isolation |
| name | varchar(255) | no | - | User-facing template name, unique per firm |

### Validation Rules

**Backend (Kotlin):**
- name: @NotBlank - "Template name is required"
- name: @Size(max=255) - "Must not exceed 255 characters"

**Frontend (Zod):**
- name: min(1) - "Template name is required"
- name: refine(!includes('//')) - "Cannot contain '//' characters"
```

## Documentation

Full skill documentation is available in [skills/data-dictionary-generator/SKILL.md](skills/data-dictionary-generator/SKILL.md).

## License

MIT

---

Built with [Claude Code](https://claude.ai/code)
