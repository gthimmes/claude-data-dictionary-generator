---
name: data-dictionary-generator
description: Generate comprehensive data dictionaries by analyzing database schemas, migration files, backend code, frontend code, and documentation sources. Use when creating or updating database documentation.
user-invocable: true
allowed-tools: Read, Write, Edit, Glob, Grep, Task, Bash, WebFetch, mcp__postgres__query
---

# Data Dictionary Generator

A skill that automatically generates comprehensive data dictionaries by analyzing multiple documentation sources across your technology stack. The skill synthesizes information from databases, codebases (backend and frontend), and documentation platforms to create a unified, living reference document for database entities.

## What This Skill Does

When invoked, you (Claude) will:

1. **Set up integrations** - Guide user through MCP connections (PostgreSQL required, Notion optional)
2. **Gather configuration** - Ask about project structure and output preferences
3. **Analyze sources** following a priority hierarchy - Extract information from each available source
4. **Synthesize findings** - Merge information with conflict resolution based on source priority
5. **Generate output** - Create a comprehensive data dictionary document

## Documentation Source Priority Hierarchy

When multiple sources provide information about the same table/column, use this priority hierarchy:

### Tier 1: Database & Schema Evolution (Authoritative)
**Highest Priority - Direct Schema Truth**

1. **PostgreSQL Database Comments** - Direct `COMMENT ON` statements
   - Most authoritative source
   - Lives with the schema
   - Cannot drift from reality

2. **Migration Files** (Flyway/Liquibase) - Schema change history with context
   - SQL comments in migration files
   - Explains "why" behind changes
   - Temporal context for evolution

### Tier 2: Implementation Code (Current Reality)
**High Priority - What's Actually Implemented**

3. **Backend Entity Code** (Kotlin/Spring Boot)
   - JPA entity classes and annotations
   - Repository interfaces and custom queries
   - Service layer business logic
   - Data validation rules
   - KDoc/JavaDoc comments

4. **Frontend Code** (TypeScript/React)
   - TypeScript interfaces and types
   - Zod/Yup validation schemas
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

### Phase 1: PostgreSQL MCP Setup (Required)

Database access via MCP is essential for generating accurate data dictionaries. Start by checking if PostgreSQL MCP is configured.

**Step 1: Check for existing MCP connection**

Try to execute a simple query:
```sql
SELECT current_database(), current_user, version();
```

**If the query succeeds:** PostgreSQL MCP is already configured. Proceed to Phase 2.

**If the query fails or MCP is not available:** Guide the user through setup.

**Step 2: Guide MCP Setup**

Tell the user:

```
To generate an accurate data dictionary, I need direct access to your PostgreSQL database via MCP.

Let me help you set this up. I'll need:
1. Database host (e.g., localhost, db.example.com)
2. Port (default: 5432)
3. Database name
4. Username
5. Password

Which database would you like to connect to?
```

**Step 3: Gather connection details**

Ask for:
- Host and port
- Database name
- Username
- Password handling preference (direct in config vs environment variable)

**Step 4: Configure MCP**

Once you have the details, help the user add the MCP server configuration.

**Option A: Direct configuration (simpler)**

Edit the user's `~/.claude.json` file to add the MCP server to their project:

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

**Option B: Environment variable (more secure)**

1. Tell user to set environment variable:
   ```bash
   export POSTGRES_URL="postgresql://username:password@host:port/database"
   ```

2. Configure MCP to use the variable:
   ```json
   "mcpServers": {
     "postgres": {
       "command": "npx",
       "args": [
         "-y",
         "@modelcontextprotocol/server-postgres"
       ],
       "env": {
         "DATABASE_URL": "${POSTGRES_URL}"
       }
     }
   }
   ```

**Step 5: Restart Claude Code**

Tell the user:
```
I've added the PostgreSQL MCP configuration. Please:
1. Update the password in ~/.claude.json if needed
2. Restart Claude Code to load the MCP server
3. Come back and I'll verify the connection
```

**Step 6: Verify connection**

After restart, test the connection:
```sql
SELECT current_database(), current_user, version();
```

Then list available schemas:
```sql
SELECT schema_name FROM information_schema.schemata
WHERE schema_name NOT IN ('pg_catalog', 'information_schema', 'pg_toast')
ORDER BY schema_name;
```

Ask the user which schema contains their application tables.

---

### Phase 2: Notion MCP Setup (Optional)

After PostgreSQL is configured, offer Notion integration for additional documentation context.

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

**Step 4: Identify relevant pages**

After setup, ask:
```
Which Notion pages contain your database documentation?

Please provide:
1. Page URLs or IDs for data model docs
2. Any specific databases or pages with table descriptions
3. ADR pages related to schema decisions
```

**If no Notion:** Proceed without it, noting that Tier 4 documentation will be unavailable.

---

### Phase 3: Project Configuration

Now gather information about the codebase:

```
Now let's configure the code analysis. Please provide:

1. **Backend code path** - Where is your backend code?
   (e.g., ./backend/src/main/kotlin or ./src/main/java)

2. **Frontend code path** - Where is your frontend code?
   (e.g., ./frontend/src or ./web/src)

3. **Migration files path** - Where are your database migrations?
   (e.g., ./src/main/resources/db/migration for Flyway)

4. **Schema to analyze** - Which database schema(s)?
   (e.g., "public", "dev", or specific schema name)

5. **Scope** - Which tables?
   - All tables in the schema
   - Specific domain (e.g., "client", "billing")
   - List of specific tables

6. **Output format** - Markdown, YAML, JSON, or HTML?

7. **Output location** - Where should I save the data dictionary?
   (e.g., ./docs/data-dictionary.yaml)
```

---

### Phase 4: PostgreSQL Schema Analysis

Execute these queries via MCP to extract schema information:

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

**Extract constraints:**
```sql
SELECT
    tc.table_name,
    tc.constraint_name,
    tc.constraint_type,
    kcu.column_name,
    ccu.table_name AS foreign_table_name,
    ccu.column_name AS foreign_column_name,
    rc.delete_rule,
    rc.update_rule
FROM information_schema.table_constraints tc
LEFT JOIN information_schema.key_column_usage kcu
    ON tc.constraint_name = kcu.constraint_name
    AND tc.table_schema = kcu.table_schema
LEFT JOIN information_schema.constraint_column_usage ccu
    ON ccu.constraint_name = tc.constraint_name
LEFT JOIN information_schema.referential_constraints rc
    ON rc.constraint_name = tc.constraint_name
WHERE tc.table_schema = '{schema_name}';
```

**Extract indexes:**
```sql
SELECT tablename, indexname, indexdef
FROM pg_indexes
WHERE schemaname = '{schema_name}'
ORDER BY tablename, indexname;
```

---

### Phase 5: Migration File Analysis

Scan the migration directory for Flyway (`V{version}__{description}.sql`) or Liquibase files.

**For each migration file, extract:**
- Version number and timestamp
- File-level comments (header block comments explaining "why")
- COMMENT ON statements within the SQL
- ALTER TABLE and CREATE TABLE statements
- Related JIRA tickets from filename or comments (e.g., `ART-1234`)
- Author information if present

**Pattern to look for:**
```sql
/*
 * [Description of change]
 *
 * Related: [JIRA-TICKET]
 * Author: [author-name]
 * Date: [date]
 *
 * Business Context:
 * [Why this change was made]
 */
```

Use Glob to find migration files:
```
**/db/migration/*.sql
**/flyway/*.sql
**/liquibase/**/*.sql
```

---

### Phase 6: Backend Code Analysis

**Find entity/model classes:**

For JPA/Hibernate:
```
@Entity
@Table
```

For JDBC-based DAOs (custom pattern):
```
TABLE_NAME =
CrudDao
RowMapper
```

**For each entity, extract:**
- Class-level documentation (KDoc/JavaDoc)
- Table name mapping
- Column annotations and types
- Validation annotations (`@NotNull`, `@NotBlank`, `@Size`, etc.)
- Relationship annotations
- Custom validation methods

**Find Repository/DAO interfaces:**
```
Repository
JpaRepository
CrudDao
```

**Extract from repositories:**
- Custom query methods
- `@Query` annotations
- Method documentation

---

### Phase 7: Frontend Code Analysis

**Find TypeScript interfaces/types:**
```
export interface
export type
```

**For each interface, extract:**
- TSDoc comments
- Property names and types
- Optional markers
- Property-level comments

**Find validation schemas (Zod/Yup):**
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
  schema: "{schema_name}"

  sources_analyzed:
    - type: "postgresql_schema"
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
          db_comment: "{from COMMENT ON}"
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

## Example Session

```
User: /data-dictionary-generator

Claude: I'll help you generate a comprehensive data dictionary. First, let me check if you have database access configured.

[Attempts MCP query - fails]

To generate an accurate data dictionary, I need direct access to your PostgreSQL database via MCP.

Let me help you set this up. I'll need:
1. Database host (e.g., localhost)
2. Port (default: 5432)
3. Database name
4. Username
5. Password handling preference

What database would you like to connect to?

User: localhost:5432, database is "myapp", user is "dev_user"

Claude: I'll add the PostgreSQL MCP configuration to your Claude settings.

[Edits ~/.claude.json to add MCP config]

Done! Please:
1. Edit ~/.claude.json to add your password
2. Restart Claude Code
3. Come back and say "ready"

User: ready

Claude: [Tests connection - succeeds]

Connected to myapp database as dev_user. Found schemas: public, audit.

Would you like to include Notion documentation? This can provide additional business context for your tables.

User: Yes, I have data model docs in Notion

Claude: Great! To connect to Notion:

1. Go to https://www.notion.so/my-integrations
2. Create a new integration
3. Share your data model pages with it
4. Give me the integration token

User: [provides token]

Claude: [Configures Notion MCP]

Now let's configure the code analysis:

1. Backend code path?
2. Frontend code path?
3. Migration files path?
4. Which schema to analyze?
5. All tables or specific domain?
6. Output format?
7. Output location?

User:
1. ./backend/src/main/kotlin
2. ./frontend/src
3. ./backend/src/main/resources/db/migration
4. public schema
5. Start with the "user" and "account" tables
6. YAML
7. ./docs/data-dictionary.yaml

Claude: [Executes analysis across all sources]

Generated data dictionary for 2 tables:
- user: 15 columns, 8 validation rules
- account: 22 columns, 12 validation rules

Saved to ./docs/data-dictionary.yaml

Key findings:
- user.email has different max lengths in DB (255) vs frontend validation (100)
- account.status lacks a database comment - consider adding one
- Found ADR in Notion explaining the account hierarchy design
```

---

## Key Principles

1. **MCP is essential** - Guide users through setup; don't skip database access
2. **Prioritize authoritative sources** - Database comments and migration files are truth
3. **Preserve business context** - The "why" is as important as the "what"
4. **Flag discrepancies** - When sources conflict, note it for review
5. **Be comprehensive** - Include all relevant details from each source
6. **Provide traceability** - Always cite source file and line numbers
