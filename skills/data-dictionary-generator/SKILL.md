---
name: data-dictionary-generator
description: Generate comprehensive data dictionaries by analyzing database schemas, migration files, backend code, frontend code, and documentation sources. Use when creating or updating database documentation.
user-invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Task, Bash, WebFetch, mcp__postgres__query, mcp__mysql__query, mcp__sqlite__query
---

# Data Dictionary Generator

A skill that automatically generates comprehensive data dictionaries by analyzing multiple documentation sources across your technology stack. The skill synthesizes information from databases, codebases (backend and frontend), and documentation platforms to create a unified, living reference document for database entities.

## Supported Databases

- **PostgreSQL** - Full support including comments, constraints, indexes
- **MySQL/MariaDB** - Full support including comments, constraints, indexes
- **SQL Server** - Full support including extended properties, constraints, indexes
- **SQLite** - Basic support for schema extraction
- **Other SQL databases** - Via standard information_schema queries

## What This Skill Does

When invoked, you (Claude) will:

1. **Set up integrations** - Guide user through MCP connections (database required, Notion optional)
2. **Gather configuration** - Ask about project structure and output preferences
3. **Analyze sources** following a priority hierarchy - Extract information from each available source
4. **Synthesize findings** - Merge information with conflict resolution based on source priority
5. **Generate output** - Create a comprehensive data dictionary document

## Documentation Source Priority Hierarchy

When multiple sources provide information about the same table/column, use this priority hierarchy:

### Tier 1: Database & Schema Evolution (Authoritative)
**Highest Priority - Direct Schema Truth**

1. **Database Comments/Descriptions** - Direct column and table comments
   - Most authoritative source
   - Lives with the schema
   - Cannot drift from reality

2. **Migration Files** (Flyway/Liquibase/Alembic/etc.) - Schema change history with context
   - SQL comments in migration files
   - Explains "why" behind changes
   - Temporal context for evolution

### Tier 2: Implementation Code (Current Reality)
**High Priority - What's Actually Implemented**

3. **Backend Entity Code** (Java/Kotlin/Python/C#/Go/etc.)
   - ORM entity classes and annotations
   - Repository interfaces and custom queries
   - Service layer business logic
   - Data validation rules
   - Code documentation comments

4. **Frontend Code** (TypeScript/JavaScript/etc.)
   - TypeScript interfaces and types
   - Validation schemas (Zod/Yup/etc.)
   - Form field validation rules
   - UI help text and tooltips
   - User-facing perspective

### Tier 3: API & Testing (Behavior Contract)
**Medium Priority - Specified Behavior**

5. **API Documentation** (Swagger/OpenAPI if available)
   - Controller endpoint annotations
   - Request/response schemas

6. **Test Code**
   - Test method names and descriptions
   - Edge cases and business rule validations

### Tier 4: Documentation Platforms (Curated Context)
**Medium-Low Priority - May drift from implementation**

7. **Notion Documentation** - Technical design documents, ADRs, data model docs
8. **JIRA Tickets** - Feature stories, schema change tickets

---

## Execution Workflow

### Phase 1: Database MCP Setup (Required)

Database access via MCP is essential for generating accurate data dictionaries. Start by asking which database the user wants to document.

**Step 1: Ask about database type**

```
I'll help you generate a comprehensive data dictionary. First, I need to connect to your database.

Which database system are you using?
1. PostgreSQL
2. MySQL / MariaDB
3. SQL Server
4. SQLite
5. Other
```

**Step 2: Check for existing MCP connection**

Based on the database type, try to execute a simple test query:

For PostgreSQL:
```sql
SELECT current_database(), current_user, version();
```

For MySQL:
```sql
SELECT DATABASE(), USER(), VERSION();
```

For SQL Server:
```sql
SELECT DB_NAME(), SUSER_NAME(), @@VERSION;
```

For SQLite:
```sql
SELECT sqlite_version();
```

**If the query succeeds:** Database MCP is already configured. Proceed to Phase 2.

**If the query fails or MCP is not available:** Guide the user through setup.

**Step 3: Guide MCP Setup**

Tell the user:

```
To generate an accurate data dictionary, I need direct access to your database via MCP.

Let me help you set this up. I'll need:
1. Database host (e.g., localhost, db.example.com)
2. Port (default varies by database)
3. Database name
4. Username
5. Password

Which database would you like to connect to?
```

**Step 4: Configure MCP**

Once you have the details, help the user add the MCP server configuration to `~/.claude.json`.

**PostgreSQL Configuration:**
```json
"mcpServers": {
  "postgres": {
    "command": "npx",
    "args": [
      "-y",
      "@modelcontextprotocol/server-postgres",
      "postgresql://USERNAME:PASSWORD@HOST:PORT/DATABASE"
    ]
  }
}
```

**MySQL Configuration:**
```json
"mcpServers": {
  "mysql": {
    "command": "npx",
    "args": [
      "-y",
      "@modelcontextprotocol/server-mysql",
      "mysql://USERNAME:PASSWORD@HOST:PORT/DATABASE"
    ]
  }
}
```

**SQLite Configuration:**
```json
"mcpServers": {
  "sqlite": {
    "command": "npx",
    "args": [
      "-y",
      "@modelcontextprotocol/server-sqlite",
      "/path/to/database.db"
    ]
  }
}
```

**SQL Server Configuration:**
```json
"mcpServers": {
  "mssql": {
    "command": "npx",
    "args": [
      "-y",
      "@modelcontextprotocol/server-mssql",
      "Server=HOST;Database=DATABASE;User Id=USERNAME;Password=PASSWORD;"
    ]
  }
}
```

**Environment Variable Option (more secure):**

For any database, recommend using environment variables:
```json
"mcpServers": {
  "postgres": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-postgres"],
    "env": {
      "DATABASE_URL": "${DATABASE_URL}"
    }
  }
}
```

**Step 5: Restart Claude Code**

Tell the user:
```
I've added the database MCP configuration. Please:
1. Update the password in ~/.claude.json if needed
2. Restart Claude Code to load the MCP server
3. Come back and I'll verify the connection
```

**Step 6: Verify connection and list schemas**

After restart, test the connection and list available schemas/databases.

---

### Phase 2: Notion MCP Setup (Optional)

After database is configured, offer Notion integration for additional documentation context.

**Step 1: Ask about Notion**

```
Would you like to include Notion documentation in the data dictionary?

Notion can provide:
- Technical design documents
- Architecture Decision Records (ADRs)
- Data model documentation
- Business context for tables/fields

Do you have Notion documentation for your database schema?
```

**If yes, guide setup:**

**Step 2: Get Notion API Token**

Tell the user:
```
To connect to Notion, you'll need an Internal Integration Token:

1. Go to https://www.notion.so/my-integrations
2. Click "New integration"
3. Name it (e.g., "Claude Data Dictionary")
4. Select your workspace
5. Copy the "Internal Integration Token"

Also, you'll need to share the relevant Notion pages with this integration:
1. Open each Notion page/database you want to include
2. Click "..." menu → "Add connections"
3. Select your integration

What is your Notion integration token?
```

**Step 3: Configure Notion MCP**

Add to the user's MCP configuration:

```json
"notion": {
  "command": "npx",
  "args": [
    "-y",
    "@modelcontextprotocol/server-notion"
  ],
  "env": {
    "NOTION_API_TOKEN": "secret_xxxxx"
  }
}
```

**If no Notion:** Proceed without it, noting that Tier 4 documentation will be unavailable.

---

### Phase 3: Project Configuration

Now gather information about the codebase:

```
Now let's configure the code analysis. Please provide:

1. **Backend code path** - Where is your backend code?
   (e.g., ./backend/src/main/kotlin, ./src, ./api)

2. **Frontend code path** - Where is your frontend code?
   (e.g., ./frontend/src, ./web/src, ./client)

3. **Migration files path** - Where are your database migrations?
   (e.g., ./migrations, ./db/migrate, ./src/main/resources/db/migration)

4. **Schema to analyze** - Which database schema(s)?
   (e.g., "public", "dbo", or specific schema name)

5. **Scope** - Which tables?
   - All tables in the schema
   - Specific domain (e.g., "user", "billing")
   - List of specific tables

6. **Output format** - Markdown, YAML, JSON, or HTML?

7. **Output location** - Where should I save the data dictionary?
   (e.g., ./docs/data-dictionary.yaml)
```

---

### Phase 4: Database Schema Analysis

Execute queries appropriate for the database type to extract schema information.

#### PostgreSQL Queries

**Extract table comments:**
```sql
SELECT
    n.nspname AS schema_name,
    c.relname AS table_name,
    obj_description(c.oid) AS table_comment
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind IN ('r', 'v')
  AND n.nspname = '{schema_name}';
```

**Extract column details with comments:**
```sql
SELECT
    c.relname AS table_name,
    a.attname AS column_name,
    pg_catalog.format_type(a.atttypid, a.atttypmod) AS data_type,
    a.attnotnull AS not_null,
    pg_get_expr(d.adbin, d.adrelid) AS default_value,
    col_description(c.oid, a.attnum) AS column_comment
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
JOIN pg_attribute a ON a.attrelid = c.oid
LEFT JOIN pg_attrdef d ON d.adrelid = a.attrelid AND d.adnum = a.attnum
WHERE n.nspname = '{schema_name}'
  AND c.relkind IN ('r', 'v')
  AND a.attnum > 0
  AND NOT a.attisdropped
ORDER BY c.relname, a.attnum;
```

#### MySQL Queries

**Extract table comments:**
```sql
SELECT
    TABLE_SCHEMA AS schema_name,
    TABLE_NAME AS table_name,
    TABLE_COMMENT AS table_comment
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = '{schema_name}';
```

**Extract column details with comments:**
```sql
SELECT
    TABLE_NAME AS table_name,
    COLUMN_NAME AS column_name,
    COLUMN_TYPE AS data_type,
    IS_NULLABLE = 'NO' AS not_null,
    COLUMN_DEFAULT AS default_value,
    COLUMN_COMMENT AS column_comment
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = '{schema_name}'
ORDER BY TABLE_NAME, ORDINAL_POSITION;
```

#### SQL Server Queries

**Extract table comments (extended properties):**
```sql
SELECT
    s.name AS schema_name,
    t.name AS table_name,
    ep.value AS table_comment
FROM sys.tables t
JOIN sys.schemas s ON t.schema_id = s.schema_id
LEFT JOIN sys.extended_properties ep ON ep.major_id = t.object_id
    AND ep.minor_id = 0
    AND ep.name = 'MS_Description'
WHERE s.name = '{schema_name}';
```

**Extract column details with comments:**
```sql
SELECT
    t.name AS table_name,
    c.name AS column_name,
    ty.name + CASE
        WHEN ty.name IN ('varchar', 'nvarchar', 'char', 'nchar')
        THEN '(' + CAST(c.max_length AS VARCHAR) + ')'
        ELSE ''
    END AS data_type,
    ~c.is_nullable AS not_null,
    dc.definition AS default_value,
    ep.value AS column_comment
FROM sys.columns c
JOIN sys.tables t ON c.object_id = t.object_id
JOIN sys.schemas s ON t.schema_id = s.schema_id
JOIN sys.types ty ON c.user_type_id = ty.user_type_id
LEFT JOIN sys.default_constraints dc ON c.default_object_id = dc.object_id
LEFT JOIN sys.extended_properties ep ON ep.major_id = c.object_id
    AND ep.minor_id = c.column_id
    AND ep.name = 'MS_Description'
WHERE s.name = '{schema_name}'
ORDER BY t.name, c.column_id;
```

#### Standard Information Schema (Fallback)

For any SQL database supporting information_schema:

```sql
SELECT
    TABLE_NAME AS table_name,
    COLUMN_NAME AS column_name,
    DATA_TYPE AS data_type,
    IS_NULLABLE = 'NO' AS not_null,
    COLUMN_DEFAULT AS default_value
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = '{schema_name}'
ORDER BY TABLE_NAME, ORDINAL_POSITION;
```

**Extract constraints (works across most databases):**
```sql
SELECT
    tc.table_name,
    tc.constraint_name,
    tc.constraint_type,
    kcu.column_name,
    ccu.table_name AS foreign_table_name,
    ccu.column_name AS foreign_column_name
FROM information_schema.table_constraints tc
LEFT JOIN information_schema.key_column_usage kcu
    ON tc.constraint_name = kcu.constraint_name
    AND tc.table_schema = kcu.table_schema
LEFT JOIN information_schema.constraint_column_usage ccu
    ON ccu.constraint_name = tc.constraint_name
WHERE tc.table_schema = '{schema_name}';
```

---

### Phase 5: Migration File Analysis

Scan the migration directory for migration files based on the framework used.

**Common patterns to search for:**
```
**/db/migration/*.sql          # Flyway
**/flyway/*.sql                # Flyway
**/liquibase/**/*.sql          # Liquibase
**/liquibase/**/*.xml          # Liquibase XML
**/migrations/*.sql            # Generic
**/migrations/*.py             # Alembic (Python)
**/db/migrate/*.rb             # Rails
**/prisma/migrations/**/*.sql  # Prisma
```

**For each migration file, extract:**
- Version number and timestamp
- File-level comments (header block comments explaining "why")
- COMMENT ON statements within the SQL
- ALTER TABLE and CREATE TABLE statements
- Related JIRA tickets from filename or comments (e.g., `ART-1234`)
- Author information if present

---

### Phase 6: Backend Code Analysis

**Find entity/model classes based on the framework:**

For JPA/Hibernate (Java/Kotlin):
```
@Entity
@Table
```

For Django (Python):
```
class.*Model
```

For SQLAlchemy (Python):
```
Base
declarative_base
```

For Entity Framework (C#):
```
DbContext
DbSet
```

For GORM (Go):
```
gorm.Model
```

**For each entity, extract:**
- Class-level documentation
- Table name mapping
- Column annotations and types
- Validation annotations/decorators
- Relationship mappings
- Custom validation methods

---

### Phase 7: Frontend Code Analysis

**Find TypeScript interfaces/types:**
```
export interface
export type
```

**For each interface, extract:**
- Documentation comments
- Property names and types
- Optional markers
- Property-level comments

**Find validation schemas (Zod/Yup/etc.):**
```
z.object
yup.object
```

**Extract validation rules and error messages.**

**Find form components:**
- Labels and placeholders
- Help text
- Max length constraints

---

### Phase 8: Notion Documentation Analysis (if configured)

If Notion MCP is available:

1. Search for pages containing table names
2. Extract descriptions and business context
3. Find ADRs related to schema decisions
4. Pull data model diagrams/descriptions

---

### Phase 9: Synthesize and Generate Output

For each table/entity:

1. **Start with database schema** as the foundation
2. **Layer in migration history** for evolution context
3. **Add backend code details** for validation rules and business logic
4. **Incorporate frontend insights** for user-facing documentation
5. **Add Notion context** for business rationale
6. **Note any conflicts** between sources

**Conflict resolution:**
- Use highest-priority source's description as primary
- Note discrepancies in a "Notes" section
- Flag potential documentation drift

---

## Output Format: YAML (Recommended)

```yaml
data_dictionary:
  generated_at: "{timestamp}"
  database: "{database_name}"
  database_type: "{postgresql|mysql|sqlserver|sqlite}"
  schema: "{schema_name}"

  sources_analyzed:
    - type: "database_schema"
      database_type: "{type}"
      status: "connected"
    - type: "migration_files"
      path: "{path}"
      count: {n}
    - type: "backend_code"
      path: "{path}"
    - type: "frontend_code"
      path: "{path}"
    - type: "notion"
      status: "connected|not_configured"

  tables:
    - name: "{table_name}"
      schema: "{schema_name}"
      description: "{description}"
      business_context: "{context from migrations/notion}"

      columns:
        - name: "{column_name}"
          type: "{data_type}"
          nullable: {true/false}
          default: "{default_value}"
          db_comment: "{from database comment}"
          description: "{merged description}"

          validations:
            backend:
              - annotation: "{@NotNull}"
                message: "{message}"
            frontend:
              - rule: "{min(1)}"
                message: "{message}"

          ui:
            label: "{User-Facing Label}"
            help_text: "{Help text}"
            placeholder: "{Placeholder}"

      constraints:
        - name: "{constraint_name}"
          type: "{PRIMARY KEY|FOREIGN KEY|UNIQUE|CHECK}"
          columns: ["{col1}"]
          references:
            table: "{ref_table}"
            columns: ["{ref_col}"]
          on_delete: "{CASCADE|RESTRICT}"

      relationships:
        - type: "{many_to_one|one_to_many}"
          target: "{target_table}"
          via: "{column_name}"
          description: "{description}"

      evolution:
        - version: "{V1.001}"
          date: "{2024-01-15}"
          change: "{description}"
          reason: "{business context}"
          jira: "{TICKET-123}"

      source_references:
        database: "{schema.table}"
        backend_model: "{com.example.Model}"
        backend_dao: "{com.example.Dao}"
        frontend_type: "{src/types/model.ts}"
        notion_page: "{page_id}"
```

---

## Key Principles

1. **MCP is essential** - Guide users through setup; don't skip database access
2. **Database-agnostic** - Adapt queries and approach to the specific database type
3. **Prioritize authoritative sources** - Database comments and migration files are truth
4. **Preserve business context** - The "why" is as important as the "what"
5. **Flag discrepancies** - When sources conflict, note it for review
6. **Be comprehensive** - Include all relevant details from each source
7. **Provide traceability** - Always cite source file and line numbers
