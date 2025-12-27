# Installing MCP Server MotherDuck (Recon Edition) for Claude Desktop

This guide walks you through installing the enhanced MotherDuck MCP server for use with the Claude Desktop app.

## Prerequisites

- **Python 3.10+** installed on your system
- **Claude Desktop app** installed ([download here](https://claude.ai/download))
- **MotherDuck account** with an access token ([sign up here](https://motherduck.com/))

### Verify Python Installation

Open a terminal and run:

```bash
python --version
```

You should see `Python 3.10.x` or higher. If not, [install Python](https://www.python.org/downloads/).

---

## Step 1: Install the Package

Open a terminal (Command Prompt, PowerShell, or Terminal) and run:

```bash
pip install git+https://github.com/DonKelloggRecon/mcp-server-motherduck-recon.git
```

This installs the `mcp-server-motherduck-recon` command and all dependencies (including DuckDB).

### Verify Installation

```bash
mcp-server-motherduck-recon --help
```

You should see the available options. If you get "command not found", you may need to add Python Scripts to your PATH (see Troubleshooting below).

---

## Step 2: Get Your MotherDuck Token

1. Log in to [MotherDuck](https://app.motherduck.com/)
2. Click your profile icon (top right) → **Settings**
3. Go to **Access Tokens**
4. Click **Create Token**
5. Copy the token (you'll need it for the next step)

---

## Step 3: Configure Claude Desktop

### Find the Config File

| Platform | Location |
|----------|----------|
| **Windows** | `%APPDATA%\Claude\claude_desktop_config.json` |
| **macOS** | `~/Library/Application Support/Claude/claude_desktop_config.json` |

### Windows Quick Access

Press `Win + R`, type `%APPDATA%\Claude`, and press Enter.

### macOS Quick Access

In Finder, press `Cmd + Shift + G` and paste `~/Library/Application Support/Claude/`

---

### Edit the Config File

Open `claude_desktop_config.json` in a text editor. If it doesn't exist, create it.

Add or update the `mcpServers` section:

```json
{
  "mcpServers": {
    "motherduck": {
      "command": "mcp-server-motherduck-recon",
      "args": [
        "--saas-mode",
        "--motherduck-token",
        "YOUR_TOKEN_HERE"
      ]
    }
  }
}
```

**Replace `YOUR_TOKEN_HERE` with your actual MotherDuck token.**

### Windows Note

If `mcp-server-motherduck-recon` is not in your PATH, use the full path to the executable:

```json
{
  "mcpServers": {
    "motherduck": {
      "command": "C:\\Users\\YOUR_USERNAME\\AppData\\Roaming\\Python\\Python3XX\\Scripts\\mcp-server-motherduck-recon.exe",
      "args": [
        "--saas-mode",
        "--motherduck-token",
        "YOUR_TOKEN_HERE"
      ]
    }
  }
}
```

To find the exact path, run:
```bash
where mcp-server-motherduck-recon
```

---

## Step 4: Restart Claude Desktop

1. Fully quit Claude Desktop (not just close the window)
   - **Windows**: Right-click the system tray icon → Quit
   - **macOS**: Claude → Quit Claude (or `Cmd + Q`)
2. Reopen Claude Desktop
3. Check the MCP server status in Settings → Developer

You should see `motherduck` listed as connected.

---

## Step 5: Test the Connection

In a new Claude conversation, try:

> "Use the list_tables tool to show me what databases are available"

or

> "Run this query: SELECT database_name FROM duckdb_databases()"

---

## Available Tools

Once connected, Claude has access to these tools:

| Tool | Description |
|------|-------------|
| `query` | Execute SQL queries against MotherDuck/DuckDB |
| `list_tables` | List tables with properly quoted identifiers |

### Why `list_tables` Matters

When working with tables that have special characters in their names (hyphens, colons, spaces), you need to quote them in SQL:

```sql
-- This FAILS:
SELECT * FROM ACSDT5Y2023_B19080-Data;

-- This WORKS:
SELECT * FROM "ACSDT5Y2023_B19080-Data";
```

The `list_tables` tool returns pre-quoted identifiers so Claude generates correct SQL automatically.

---

## Troubleshooting

### "Command not found" Error

The Python Scripts directory may not be in your PATH.

**Windows:**
```bash
# Find where pip installed the command
pip show mcp-server-motherduck-recon

# Add Scripts folder to PATH or use full path in config
```

**macOS/Linux:**
```bash
# Usually installed here
~/.local/bin/mcp-server-motherduck-recon
```

### MCP Server Shows "Failed" in Claude Desktop

1. Check your token is correct
2. Verify the command path exists
3. Try running the command manually in terminal:
   ```bash
   mcp-server-motherduck-recon --saas-mode --motherduck-token YOUR_TOKEN
   ```
4. Check for error messages

### "Server disconnected" Error

This usually means the server started but crashed. Common causes:
- Invalid or expired token
- Network connectivity issues
- Missing Python dependencies

Run the command manually to see the actual error.

### Updating to Latest Version

```bash
pip install --upgrade git+https://github.com/DonKelloggRecon/mcp-server-motherduck-recon.git
```

---

## Optional: Read-Only Mode

If you want to prevent accidental data modifications:

```json
{
  "mcpServers": {
    "motherduck": {
      "command": "mcp-server-motherduck-recon",
      "args": [
        "--saas-mode",
        "--motherduck-token",
        "YOUR_TOKEN_HERE",
        "--read-only"
      ]
    }
  }
}
```

---

## Support

- **Issues with this fork**: [GitHub Issues](https://github.com/DonKelloggRecon/mcp-server-motherduck-recon/issues)
- **MotherDuck documentation**: [docs.motherduck.com](https://docs.motherduck.com/)
- **Claude Desktop help**: [support.anthropic.com](https://support.anthropic.com/)
