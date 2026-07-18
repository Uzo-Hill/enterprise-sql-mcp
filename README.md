# enterprise-sql-mcp
MCP server enabling natural-language querying of a MySQL retail database via Claude Desktop — with a safety-validated, read-only SQL execution layer.

---

# Enterprise SQL MCP — Natural Language Retail Analytics Assistant

An MCP (Model Context Protocol) server that lets an AI assistant query a real retail/e-commerce MySQL database using plain English — no SQL knowledge required from the end user.

---

## Project Overview

Business stakeholders often need answers from company data — "which product sold the most?", "who are our top customers?" — but writing SQL, or waiting on an analyst, slows things down. This project closes that gap by building a **secure, read-only MCP server** that connects any MCP-compatible AI application (like Claude Desktop) directly to a retail database, allowing natural-language questions to be answered with real, live query results.

This was built as a hands-on project to learn and demonstrate **MCP (Model Context Protocol)** — the emerging open standard for connecting AI applications to external tools and data sources.

---

## Objective

- Build a working MCP server from scratch using the official Python SDK
- Connect it to a real MySQL database with realistic, relational retail data
- Allow an AI assistant to safely explore the schema and answer business questions in natural language
- Implement production-shaped safety measures — not just a "toy" AI-to-database connection

---

## What is MCP?

MCP is an open standard (built by Anthropic) that defines a consistent way for AI applications to connect to external tools and data, often described as "USB-C for AI apps." Instead of writing custom integration code for every tool an AI needs to use, MCP standardizes the connection so any compatible AI host can plug into any compatible MCP server.

In this project:
- **Host** → Claude Desktop (the AI application)
- **Client** → the MCP client built into Claude Desktop
- **Server** → `server.py`, the program built in this repo, exposing tools over MCP

---

## 🏗️ Architecture

```
                    ENTERPRISE SQL MCP
           Natural Language to Safe SQL Architecture
══════════════════════════════════════════════════════════════════════

                         ┌──────────────────────────────┐
                         │            USER              │
                         │ "Which product generated     │
                         │  the highest revenue?"       │
                         └──────────────┬───────────────┘
                                        │
                                        ▼
                  ┌──────────────────────────────────────┐
                  │         CLAUDE DESKTOP (Host)        │
                  │ • Understands user intent            │
                  │ • Selects the appropriate MCP tool   │
                  └──────────────┬───────────────────────┘
                                 │
                                 ▼
                  ┌──────────────────────────────────────┐
                  │         MCP PROTOCOL (stdio)         │
                  │ • Tool discovery                     │
                  │ • Tool invocation                    │
                  │ • Request / Response transport       │
                  └──────────────┬───────────────────────┘
                                 │
                                 ▼
        ┌────────────────────────────────────────────────────────────┐
        │                 server.py (MCP Server)                     │
        │                                                            │
        │  Registered Tools                                          │
        │  ───────────────────────────────────────────────────────   │
        │  ✓ get_schema()                                            │
        │      • Returns database schema                             │
        │      • Lists tables and columns                            │
        │                                                            │
        │  ✓ run_query(sql)                                          │
        │      • Validates SQL                                       │
        │      • Executes approved queries                           │
        │      • Returns JSON results                                │
        │                                                            │
        │  SQL Safety Layer                                          │
        │      ✓ SELECT statements only                              │
        │      ✓ Blocks DELETE, UPDATE, DROP, INSERT                 │
        │      ✓ Prevents stacked statements (;)                     │
        │      ✓ Automatically applies row LIMIT                     │
        └──────────────────────┬─────────────────────────────────────┘
                               │
                               │  mcp_reader
                               │  (Read-only MySQL User)
                               ▼
                  ┌──────────────────────────────────────┐
                  │         MySQL Database               │
                  │          retail_mcp                  │
                  │                                      │
                  │  • categories                        │
                  │  • products                          │
                  │  • customers                         │
                  │  • orders                            │
                  │  • order_items                       │
                  └──────────────┬───────────────────────┘
                                 │
                                 │ JSON Result Set
                                 ▼
                  ┌──────────────────────────────────────┐
                  │         CLAUDE DESKTOP (Host)        │
                  │ • Interprets SQL results             │
                  │ • Explains findings                  │
                  │ • Responds in natural language       │
                  └──────────────┬───────────────────────┘
                                 │
                                 ▼
                         ┌──────────────────────────────┐
                         │            USER              │
                         │ "Whole Grain Pasta generated │
                         │ the highest revenue at       │
                         │ $45,206.32."                 │
                         └──────────────────────────────┘

══════════════════════════════════════════════════════════════════════
SECURITY LAYERS

1. Claude Desktop
   • Declines unsafe or destructive requests

2. server.py Validation
   • Deterministic SQL validation
   • Read-only (SELECT) enforcement
   • Cannot be bypassed by the model

══════════════════════════════════════════════════════════════════════
                   
``` 


**Flow:**
1. User asks a question in plain English inside Claude Desktop
2. Claude calls `get_schema()` to discover available tables/columns
3. Claude writes its own SQL query based on the question
4. Claude calls `run_query(sql)`, which validates and safely executes the query
5. Results are returned as JSON and explained back to the user in natural language

---

## ⚙️ Tech Stack

| Layer | Tool |
|---|---|
| Database | MySQL 8.0 |
| Server language | Python 3.13 |
| MCP framework | Official `mcp` Python SDK (FastMCP) |
| Data generation | Faker, Pandas |
| DB connectivity | PyMySQL, SQLAlchemy |
| AI Host | Claude Desktop |

---

## 📊 Database Schema

```
                    RETAIL_MCP DATABASE
              Entity Relationship Diagram

═══════════════════════════════════════════════════════════

┌───────────────────────────┐
│        categories         │
├───────────────────────────┤
│ PK  category_id            │
│     category_name          │
└─────────────┬─────────────┘
              │
              │ 1
              │
              │ ∞
┌─────────────▼─────────────┐
│         products           │
├───────────────────────────┤
│ PK  product_id              │
│ FK  category_id             │
│     product_name            │
│     cost_price               │
│     unit_price                │
└─────────────┬─────────────┘
              │
              │ 1
              │
              │ ∞
┌─────────────▼─────────────┐        ┌───────────────────────────┐
│        order_items         │   ∞    │         orders             │
├───────────────────────────┤ ◄──────┤───────────────────────────┤
│ PK  order_item_id           │   1   │ PK  order_id                │
│ FK  order_id                 │        │ FK  customer_id             │
│ FK  product_id                │        │     order_date               │
│     quantity                    │        │     status                    │
│     unit_price                   │        └─────────────┬─────────────┘
└───────────────────────────┘                      │
                                                     │ ∞
                                                     │
                                                     │ 1
                                       ┌─────────────▼─────────────┐
                                       │        customers           │
                                       ├───────────────────────────┤
                                       │ PK  customer_id              │
                                       │     customer_name             │
                                       │     email                      │
                                       │     city                        │
                                       └───────────────────────────┘

═══════════════════════════════════════════════════════════
LEGEND:  PK = Primary Key   FK = Foreign Key
         1 ──── ∞  = one-to-many relationship

RELATIONSHIPS:
- One category has many products
- One product appears in many order_items
- One order has many order_items
- One customer places many orders
```

<!-- 🖼️ INSERT ER DIAGRAM / SCREENSHOT OF SHOW TABLES OUTPUT HERE -->

![Database Schema](https://github.com/Uzo-Hill/enterprise-sql-mcp/blob/main/project_image/tables.PNG)

5 relational tables, generated as realistic synthetic data:

| Table | Description |
|---|---|
| `categories` | Product category groupings |
| `products` | 150 products, tied to categories, with cost & unit price |
| `customers` | 500 customer records |
| `orders` | 1,000 orders across a 2-year period |
| `order_items` | Line items linking orders to products |

---

## Safety & Security Design

A core focus of this project was making the AI-to-database connection **safe by design**, not just functional:

- **Read-only database user** — the server connects as `mcp_reader`, a MySQL account with `SELECT`-only privileges. Even a successfully "tricked" query cannot modify data, because the underlying account physically cannot write.
- **Query validation layer** — every query passes through a `validate_sql()` function before execution:
  - Rejects anything that isn't a single `SELECT` statement
  - Blocks stacked queries (`;` separated statements)
  - Blocks dangerous keywords (`DELETE`, `DROP`, `UPDATE`, `ALTER`, etc.)
  - Automatically appends a row `LIMIT` if the AI doesn't specify one
- **Two layers of protection, confirmed by testing:**
  1. The AI model itself frequently declines destructive requests based on judgment
  2. The code-level validator blocks them regardless — a deterministic backstop that doesn't rely on model behavior

```python
# Core of the safety layer — server.py
FORBIDDEN_KEYWORDS = re.compile(
    r"\b(INSERT|UPDATE|DELETE|DROP|ALTER|TRUNCATE|CREATE|GRANT|REVOKE)\b",
    re.IGNORECASE,
)

def validate_sql(sql: str) -> str:          # Standardizes the string by stripping whitespace and removing trailing semicolons.
    cleaned = sql.strip().rstrip(";").strip()
    if ";" in cleaned:  # Blocks SQL injection stacking (running multiple queries separated by a semicolon).
        raise ValueError("Multiple statements are not allowed.")   # This stops attackers from appending a destructive query after a benign SELECT statement.
    if not re.match(r"^\s*SELECT\b", cleaned, re.IGNORECASE):
        raise ValueError("Only SELECT queries are allowed.")       # Enforces a strict read-only baseline. The query MUST begin with "SELECT"
    if FORBIDDEN_KEYWORDS.search(cleaned):
        raise ValueError("Query contains a disallowed keyword.")    # Scans the entire query against the blocklist. Even if the query starts with SELECT,
                                                                    # this stops dangerous commands hidden in subqueries (e.g., SELECT ... WHERE x = (DROP TABLE ...))
    if not re.search(r"\bLIMIT\s+\d+\s*$", cleaned, re.IGNORECASE):
        cleaned = f"{cleaned} LIMIT {MAX_ROWS}"
    return cleaned
```

---

## 🏢 Production Considerations (Beyond This Project's Scope)

The safety measures in this project — a read-only database user and a SQL validation layer — are genuine, meaningful safeguards, not just theoretical ones. But they represent an MVP security layer suitable for a learning project with synthetic data. A real company deployment connecting an AI assistant to a production database would need additional layers:

| Area | Current State (This Project) | What Production Would Require |
|---|---|---|
| **Credentials** | Hardcoded in `server.py` | Loaded from environment variables or a secrets manager (e.g. AWS Secrets Manager, HashiCorp Vault) — never in source code |
| **Access scope** | Read-only across all tables | Column- and table-level restrictions, so sensitive data (salaries, PII, financials etc) isn't queryable just because a table exists |
| **Data residency** | N/A (synthetic data) | Legal/compliance review of what data categories may pass through a third-party AI provider, considering regulations like GDPR or Nigeria's NDPR |
| **Authentication** | Anyone with local Claude Desktop access | Per-user authentication tied to the MCP server, with role-based access control (a sales rep and a CFO shouldn't see the same data) |
| **Audit trail** | None | Full query logging - who asked what, when, and how many rows were returned, for accountability and security monitoring |
| **Prompt injection** | Not addressed | Awareness that data *returned* by queries (e.g. a product name or note field) could theoretically contain manipulative text — an active, unsolved area of AI security research |

### Why this matters

This project intentionally focuses on demonstrating **correct MCP architecture and query-level safety** — the foundational layer everything else builds on. Understanding what's *missing* for a production deployment is just as important as building what's *present*: it shows the difference between a working demo and a system ready for real company data.

*Several of the technical gaps above (credential management, access control, audit logging) are tracked as concrete next steps in the [Future Improvements](#-future-improvements) section.*

---

## 🚀 Setup & Installation

### 1. Clone the repository
```bash
git clone https://github.com/Uzo-Hill/enterprise-sql-mcp.git
cd enterprise-sql-mcp
```

### 2. Install Python dependencies
```bash
pip install faker pandas sqlalchemy pymysql mcp
```

### 3. Generate synthetic retail data
```bash
python generate_data.py
```

### 4. Create the MySQL database and schema
```bash
mysql -u root -p < schema.sql
```

### 5. Load the data
```bash
python load_data.py
```

### 6. Connect the server to Claude Desktop

Edit your `claude_desktop_config.json` (**Settings → Developer → Edit Config**):

```json
{
  "mcpServers": {
    "retail-sql": {
      "command": "python",
      "args": ["/absolute/path/to/server.py"]
    }
  }
}
```

Restart Claude Desktop, then check **Settings → Developer** — `retail-sql` should show as **running**.

---

## 💬 Example Usage

**Sample questions this project can answer:**

- Which product generated the highest revenue?

![Response](https://github.com/Uzo-Hill/enterprise-sql-mcp/blob/main/project_image/Query1_Top_Product.PNG)
 
- "What's the average order value?

![Response](https://github.com/Uzo-Hill/enterprise-sql-mcp/blob/main/project_image/Query2_Order_Value.PNG)

- Which category is declining over time?

![Response](https://github.com/Uzo-Hill/enterprise-sql-mcp/blob/main/project_image/Query3.PNG)

---

## Challenges Faced

- **Python environment mismatch** — Claude Desktop resolved `python` to a different, unconfigured Python installation than the one used in the terminal, causing `ModuleNotFoundError`. Solved by pointing the MCP config directly at the full path of the correct interpreter.
- **Windows PATH configuration** — MySQL was installed but not accessible from the command line until its `bin` folder was manually added to the system PATH.
- **MSIX sideload restrictions** — Windows blocked the Claude Desktop installer by default; resolved via Developer Mode.
- **Config file not persisting** — an early edit to `claude_desktop_config.json` silently reverted; resolved by fully terminating background processes via Task Manager before restarting, rather than just closing the window.
- **Stale conversation context** — an early test appeared to return cached results from a previous chat rather than a fresh query; resolved by testing in a new chat thread and directly verifying database contents independently.

---

## Key Achievements

- Built and tested a real MCP server from scratch using the official Python SDK — not a wrapper or simulation
- Verified the server over the **actual MCP protocol** (not just direct function calls) using a live MCP client test
- Designed and implemented a genuine SQL-injection-resistant safety layer, tested against real attack patterns (`DELETE`, stacked queries)
- Connected a local AI application (Claude Desktop) to a live MySQL database, enabling natural-language business intelligence queries
- Generated a realistic, relationally consistent synthetic dataset (categories → products → orders → order_items) with an intentionally engineered "declining category" trend for demo purposes

---

## Future Improvements

- Add a chart-generation tool so results render as visuals instead of raw text/JSON
- Extend the schema with `warehouses` and `inventory` tables to support stock-level questions
- Add a query audit log tool for traceability — who asked what, when, and how many rows returned
- Move database credentials out of source code into environment variables or a secrets manager
- Implement per-user authentication with role-based access control, so different roles see different data
- Add column-level access restrictions to protect sensitive fields even within otherwise-accessible tables
- Containerize the full stack (MySQL + server) with Docker Compose for one-command setup
- Deploy as a remote MCP server (HTTP/SSE transport) instead of local stdio only

---

## Author

**Chukwudum Hillary Uzoh**

Data Scientist & AI Engineer

- GitHub: https://github.com/Uzo-Hill
- LinkedIn: https://www.linkedin.com/in/hillaryuzoh/
- Email: uzoh.hillary@gmail.com























