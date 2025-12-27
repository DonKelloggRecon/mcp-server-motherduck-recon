# MCP Server MotherDuck - Recon Edition

## Overview

This is an enhanced fork of the [official MotherDuck MCP Server](https://github.com/motherduckdb/mcp-server-motherduck) that adds features to prevent SQL binding errors when working with complex table and column names.

**Package Name:** `mcp-server-motherduck-recon`
**Command:** `mcp-server-motherduck-recon`
**Fork:** https://github.com/DonKelloggRecon/mcp-server-motherduck-recon

---

## Problem Statement

When working with MotherDuck/DuckDB databases that contain tables or columns with special characters, AI assistants frequently generate SQL queries that fail with binding errors.

### Common Failure Scenarios

```sql
-- Table name contains hyphens - FAILS
SELECT * FROM ACSDT5Y2023_B19080-Data;
-- Error: Parser Error: syntax error at or near "-"

-- Column name contains colon - WRONG RESULT
SELECT Unnamed: 12 FROM my_table;
-- Returns literal "12" instead of the column value

-- Database name with underscores - MAY FAIL depending on context
USE Census_ACS_Income_Distribution_Detail;
```

### Root Cause

DuckDB (and SQL in general) requires identifiers containing special characters to be quoted with double quotes (`"`). Without proper quoting:

- **Hyphens (`-`)** are interpreted as the minus operator
- **Colons (`:`)** are interpreted as slice operators or statement terminators
- **Spaces** break the identifier into multiple tokens
- **Reserved words** conflict with SQL syntax

---

## Solution

This fork adds two key enhancements:

### 1. New `list_tables` Tool

A new MCP tool that returns table and column names **pre-quoted** and ready for use in SQL queries.

**Tool Definition:**
```json
{
  "name": "list_tables",
  "description": "List all tables in a database with properly quoted identifiers",
  "parameters": {
    "database": "Database name (optional)",
    "include_columns": "Include column names (boolean, default: false)"
  }
}
```

**Example Output:**
```
# Tables with Properly Quoted Identifiers

Database: "Census_ACS_Income_Distribution_Detail"

## Table: "ACSDT5Y2023_B19080-Data"
   Full reference: "Census_ACS_Income_Distribution_Detail".main."ACSDT5Y2023_B19080-Data"
   Columns:
     - "GEO_ID" (VARCHAR)
     - "NAME" (VARCHAR)
     - "Unnamed: 12" (INTEGER)
     - "_alteryx_load_timestamp" (TIMESTAMP WITH TIME ZONE)
```

### 2. Enhanced Prompt Template

The system prompt now includes comprehensive guidance for AI assistants on:

- When identifiers MUST be quoted
- Examples of correct quoting patterns
- Workflow changes to use `list_tables` before writing queries
- Troubleshooting guidance for binding errors

---

## Changes Made

### File: `src/mcp_server_motherduck/server.py`

**Added `list_tables` tool registration:**
```python
types.Tool(
    name="list_tables",
    description="List all tables in a database with properly quoted identifiers...",
    inputSchema={
        "type": "object",
        "properties": {
            "database": {"type": "string", ...},
            "include_columns": {"type": "boolean", ...},
        },
        "required": [],
    },
),
```

**Added helper functions:**
```python
def _quote_identifier(identifier: str) -> str:
    """Quote an identifier for safe use in SQL."""
    escaped = identifier.replace('"', '""')
    return f'"{escaped}"'

def _needs_quoting(identifier: str) -> bool:
    """Check if an identifier needs quoting."""
    # Checks for special chars, reserved words, etc.
```

**Added tool handler:**
- Switches to specified database (if provided)
- Fetches table list via `SHOW TABLES`
- Parses and quotes all table names
- Optionally fetches and quotes column names via `DESCRIBE`
- Returns formatted, copy-paste ready output

### File: `src/mcp_server_motherduck/prompt.py`

**Added `<identifier-quoting-rules>` section:**
```
CRITICAL: Always quote identifiers to prevent binding errors!

DuckDB uses double quotes (") for identifiers. You MUST quote identifiers when they contain:
- Hyphens: "ACSDT5Y2023_B19080-Data"
- Colons: "Unnamed: 12"
- Spaces: "My Table Name"
- Special characters: "column@name", "field#1"
- Reserved words: "select", "from", "order"
```

**Updated workflow guidance:**
```
2. Database Exploration (CRITICAL - Do this BEFORE writing queries):
   - ALWAYS use list_tables tool first to get properly quoted table/column names
   - Use list_tables with include_columns=true for full schema details
```

**Added binding error troubleshooting:**
```
- Binding errors (e.g., "Binder Error: Table X does not exist"):
  * This usually means an identifier needs quoting
  * Use list_tables tool to get the correct quoted names
  * Example fix: FROM table-name → FROM "table-name"
```

### File: `pyproject.toml`

**Renamed package for parallel installation:**
```toml
[project]
name = "mcp-server-motherduck-recon"
description = "Enhanced MCP server for MotherDuck with list_tables tool and identifier quoting"

[project.scripts]
mcp-server-motherduck-recon = "mcp_server_motherduck:main"
```

**Relaxed duckdb version constraint:**
```toml
dependencies = [
  "duckdb>=1.4.1",  # Was pinned to ==1.4.1
  ...
]
```

**Added build configuration:**
```toml
[tool.hatch.build.targets.wheel]
packages = ["src/mcp_server_motherduck"]
```

---

## Installation

### From GitHub (Recommended)

```bash
pip install git+https://github.com/DonKelloggRecon/mcp-server-motherduck-recon.git
```

### From Source (Development)

```bash
git clone https://github.com/DonKelloggRecon/mcp-server-motherduck-recon.git
cd mcp-server-motherduck
pip install -e .
```

## Configuration

### Claude Desktop App

Add to `%APPDATA%\Claude\claude_desktop_config.json` (Windows) or `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS):

```json
{
  "mcpServers": {
    "motherduck": {
      "command": "mcp-server-motherduck-recon",
      "args": [
        "--saas-mode",
        "--motherduck-token",
        "YOUR_TOKEN"
      ]
    }
  }
}
```

### Claude Code CLI

```bash
claude mcp add motherduck-recon -- mcp-server-motherduck-recon --saas-mode --motherduck-token YOUR_TOKEN
```

Or manually add to `~/.claude.json`:

```json
{
  "mcpServers": {
    "motherduck-recon": {
      "command": "mcp-server-motherduck-recon",
      "args": ["--saas-mode", "--motherduck-token", "YOUR_TOKEN"]
    }
  }
}
```

---

## Usage Examples

### Discovering Tables

```
User: What tables are in the Census database?

AI: [calls list_tables with database="Census_ACS_Income_Distribution_Detail"]

Result shows:
- "ACSDT5Y2023_B19080-Data"
- "ACSDT5Y2023_B19080-Column-Metadata"
- etc.
```

### Writing Queries

```sql
-- AI now generates properly quoted queries:
SELECT "GEO_ID", "NAME", "B19080_001E"
FROM "Census_ACS_Income_Distribution_Detail".main."ACSDT5Y2023_B19080-Data"
WHERE "NAME" LIKE '%California%'
LIMIT 10;
```

---

## Comparison: Original vs Recon Edition

| Feature | Original | Recon Edition |
|---------|----------|---------------|
| `query` tool | ✅ | ✅ |
| `list_tables` tool | ❌ | ✅ |
| Identifier quoting guidance | Basic | Comprehensive |
| Binding error troubleshooting | ❌ | ✅ |
| Pre-quoted output | ❌ | ✅ |

---

## Data Patterns That Benefit

This enhancement is particularly useful for databases with:

- **Census/Government data**: Table names like `ACSDT5Y2023_B19080-Data`
- **Auto-generated columns**: `Unnamed: 0`, `Unnamed: 12`
- **ETL timestamps**: `_alteryx_load_timestamp`, `_load_date`
- **Imported data**: Column names preserved from CSV/Excel with spaces or special chars

---

## Contributing

This is a personal fork. For issues with the core MCP server, please contribute to the [upstream repository](https://github.com/motherduckdb/mcp-server-motherduck).

---

## License

Same license as the original project (see LICENSE file).
