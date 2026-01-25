---
name: L6 Memory PostgreSQL Notebook
overview: Create a new notebook L6_memory_postgres.ipynb that mirrors L6_memory.ipynb but connects to a local PostgreSQL Northwind database instead of SQLite Chinook, with connection URI, schema-agnostic discovery, and example questions adapted to Northwind.
todos:
  - id: add-psycopg2
    content: Add psycopg2-binary to requirements.txt and pyproject.toml
    status: completed
  - id: create-notebook
    content: Create L6_memory_postgres.ipynb from L6_memory.ipynb with Postgres URI and Northwind adaptations
    status: in_progress
  - id: optional-env
    content: "Optional: add NORTHWIND_DATABASE_URI to example.env and use it in the notebook"
    status: pending
isProject: false
---

# L6 Memory - PostgreSQL / Northwind Notebook

## Summary

Create **[L6_memory_postgres.ipynb](lca-langchainV1-essentials-main/python/L6_memory_postgres.ipynb)** by copying the structure of [L6_memory.ipynb](lca-langchainV1-essentials-main/python/L6_memory.ipynb) and changing: (1) the database connection to local PostgreSQL (Northwind), (2) SQLite-specific SQL to PostgreSQL-compatible queries, (3) example questions and system prompt to Northwind’s schema.

---

## 1. PostgreSQL connection

**In the “Setup” / DB cell** (equivalent of current cell 4), replace:

```python
db = SQLDatabase.from_uri("sqlite:///Chinook.db")
```

with:

```python
# Local PostgreSQL Northwind (host port 55432 -> container 5432)
DATABASE_URI = "postgresql://postgres:postgres@localhost:55432/northwind"
db = SQLDatabase.from_uri(DATABASE_URI)
```

Optional: read from env for flexibility, e.g. `os.getenv("NORTHWIND_DATABASE_URI", DATABASE_URI)`.

---

## 2. PostgreSQL driver

`SQLDatabase.from_uri` uses SQLAlchemy, which needs a PostgreSQL driver. The project’s [requirements.txt](lca-langchainV1-essentials-main/python/requirements.txt) and [pyproject.toml](lca-langchainV1-essentials-main/python/pyproject.toml) do not include one.

- Add **`psycopg2-binary`** to `requirements.txt` and to the `dependencies` list in `pyproject.toml`.
- In the notebook, add a short note or a try/import in a setup cell that `psycopg2`/`psycopg2-binary` must be installed for PostgreSQL.

---

## 3. SQLite-specific SQL → PostgreSQL

**`execute_sql` tool (cell 6)**

- Change docstring from “Execute a SQLite command” to “Execute a SQL command” (or “PostgreSQL”) so it stays correct for Postgres.

**System prompt (cell 7)**

- Replace “SQLite” with “PostgreSQL” (or “SQL”):  

`You are a careful PostgreSQL analyst.`

**Table/schema discovery**

- The notebook implicitly relies on SQLite’s `sqlite_master` in the “What were the titles?” style flow. For Postgres, discovery should use the information schema, e.g.:
  ```sql
  SELECT table_name FROM information_schema.tables
  WHERE table_schema = 'public' AND table_type = 'BASE TABLE'
  ORDER BY table_name;
  ```

- The agent will infer this from the system prompt and tool results; you can add an example in a markdown cell or in the system prompt that for “list tables” it should query `information_schema.tables` (schema `public`), not `sqlite_master`.

---

## 4. Example questions for Northwind

Northwind has: `customers`, `orders`, `order_details`, `products`, `employees`, etc. (exact names can vary: `Customers`/`customers`). The Chinook examples are:

- **“This is Frank Harris, What was the total on my last invoice?”**  

Northwind equivalent: a customer plus “last order” and `orders` (and possibly `order_details` for amount). Example:

**“This is Maria Anders. What was the total on my last order?”**

- **“What were the titles?”**  

In Chinook this meant track titles on the last invoice. In Northwind: product names on the last order. Example:

**“What products were on that order?”**

Use these in the “Repeated Queries” and “Add memory” / “Try your own queries” cells so the flow (customer → last order total → follow-up on line items) stays the same. If your Northwind has different customer names, you can use a first run to list customers or adjust the name in the example.

---

## 5. Optional: `example.env` and `.env`

To avoid hardcoding credentials, you can:

- Add to `example.env`:
  - `NORTHWIND_DATABASE_URI=postgresql://postgres:postgres@localhost:55432/northwind`
- In the notebook, after `load_dotenv()`:
  ```python
  DATABASE_URI = os.getenv("NORTHWIND_DATABASE_URI", "postgresql://postgres:postgres@localhost:55432/northwind")
  db = SQLDatabase.from_uri(DATABASE_URI)
  ```


This is optional; a hardcoded default in the notebook is sufficient if you prefer.

---

## 6. Notebook structure (cells to add/change)

| Section        | L6_memory.ipynb reference | Changes in L6_memory_postgres.ipynb |

|----------------|---------------------------|-------------------------------------|

| Title / intro  | Cell 0                    | Same; optional note: “PostgreSQL / Northwind” |

| Setup          | Cells 1–4                 | 1–3: same (dotenv, `doublecheck_env`). Cell 4: `SQLDatabase.from_uri(DATABASE_URI)` with Postgres URI. |

| RuntimeContext | Cell 5                    | No change. |

| `execute_sql`  | Cell 6                    | Docstring: “SQL” or “PostgreSQL” instead of “SQLite”. |

| `SYSTEM_PROMPT`| Cell 7                    | “PostgreSQL” instead of “SQLite”. Optional: hint to use `information_schema.tables` for listing tables. |

| Agent creation | Cells 8, 14               | No structural change; keep `AzureChatOpenAI` / `create_agent` as in L6. |

| Repeated       | Cells 10–11               | Replace questions with Northwind examples (e.g. Maria Anders, “last order”, “products on that order”). |

| Add memory     | Cells 12–16               | Same logic; ensure the same Northwind-style questions and `thread_id` usage. |

| Try your own   | Cells 17–18               | Same; optional markdown: “Use Northwind tables: customers, orders, order_details, products, etc.” |

---

## 7. Dependencies to add

- **[requirements.txt](lca-langchainV1-essentials-main/python/requirements.txt)**: append `psycopg2-binary`.
- **[pyproject.toml](lca-langchainV1-essentials-main/python/pyproject.toml)**: add `psycopg2-binary` to the `dependencies` list.

---

## 8. What stays the same

- `RuntimeContext`, `execute_sql` logic, `create_agent`, `InMemorySaver`, `stream_mode="values"`, `context=RuntimeContext(db=db)`, and `{"configurable": {"thread_id": "1"}}`.
- Use of `env_utils.doublecheck_env` and `load_dotenv()`.
- Overall flow: setup → repeated queries without memory → add `InMemorySaver` and `thread_id` → follow-up that relies on memory.

---

## 9. Pre-run checklist

- PostgreSQL is running with Northwind loaded, host `localhost`, port **55432**, database **northwind**, user **postgres**, password **postgres**.
- `psycopg2-binary` installed (`pip install -r requirements.txt` or `uv sync` after pyproject change).
- `.env` (and optionally `NORTHWIND_DATABASE_URI`) set if you use the env-based URI.

---

## 10. Mermaid: data flow (unchanged from L6)

```mermaid
flowchart LR
    subgraph setup [Setup]
        Env[.env / load_dotenv]
        DB[(PostgreSQL Northwind)]
        DB <-. "SQLDatabase.from_uri" -. Env
    end

    subgraph agent [Agent]
        LLM[LLM]
        Tool[execute_sql]
        Mem[InMemorySaver]
        LLM <--> Tool
        Mem -.->|thread_id| LLM
    end

    User[User] -->|messages| LLM
    Tool -->|run(query)| DB
    RuntimeContext -.->|db| Tool
```

---

## File and dependency checklist

| Action | Target |

|--------|--------|

| Create | [L6_memory_postgres.ipynb](lca-langchainV1-essentials-main/python/L6_memory_postgres.ipynb) |

| Edit   | [requirements.txt](lca-langchainV1-essentials-main/python/requirements.txt): add `psycopg2-binary` |

| Edit   | [pyproject.toml](lca-langchainV1-essentials-main/python/pyproject.toml): add `psycopg2-binary` to `dependencies` |

| Edit   | [example.env](lca-langchainV1-essentials-main/python/example.env): optional `NORTHWIND_DATABASE_URI` |