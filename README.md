# SQL_App
Development of an online App to facilitate the SQL training of students

# SQL Practice Interface — Educational SQL Editor

A lightweight, browser-based SQL editor designed for university-level relational database courses. Students can write and execute SQL queries directly from their browser against real PostgreSQL databases — no installation required.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Databases](#databases)
- [Database Setup (Supabase)](#database-setup-supabase)
- [Security Model](#security-model)
- [Frontend Setup](#frontend-setup)
- [Project Structure](#project-structure)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

This project provides a ready-to-use SQL practice environment for students in a relational databases course. It was built to solve two recurring problems in classroom settings:

- **Client installation issues** — students no longer need MySQL Workbench, DBeaver, or any local SQL client. Everything runs in a browser.
- **Server instability** — instead of rotating GCP free-tier instances every 3 months, the app relies on a managed PostgreSQL host with a stable free tier, ensuring continuous access throughout an academic year.

The interface allows students to select a practice database, explore its schema, write SQL queries, and see results — all in one page.

---

## Features

- **Zero installation** for students — works in any modern browser
- **Multi-schema support** — students can switch between multiple practice databases via a dropdown
- **Live schema explorer** — tables and columns are dynamically listed to help students navigate the database structure
- **Read-only enforcement** — only `SELECT` and `WITH` (CTEs) queries are allowed; all DDL and DML is blocked at the database level
- **Automatic LIMIT** — queries without a `LIMIT` clause automatically get `LIMIT 100` applied, preventing runaway results
- **Dynamic `search_path`** — table names can be written without schema prefix (e.g., `SELECT * FROM trips` rather than `SELECT * FROM bikeshare.trips`)

---

## Architecture

```
┌────────────────────────────────┐
│         Student Browser        │
│                                │
│  ┌──────────┐  ┌────────────┐  │
│  │  Schema  │  │ SQL Editor │  │
│  │ Explorer │  │  + Results │  │
│  └────┬─────┘  └─────┬──────┘  │
└───────┼──────────────┼─────────┘
        │              │
        ▼              ▼
┌──────────────────────────────────┐
│         Supabase (PostgreSQL)    │
│                                  │
│  get_schema(schema_name)         │  ← Schema introspection
│  run_student_query(query, schema) │  ← Query execution (read-only)
│                                  │
│  Schemas(April 2026): bikeshare │ boutique │ movies
└──────────────────────────────────┘
```

The frontend communicates with Supabase exclusively through two PostgreSQL RPC functions exposed via the REST API. No direct table access is granted to the anonymous role.

---

## Databases

Three practice schemas are included: `bikeshare`, `boutique`, and `movies`. Each targets different SQL skill levels and business contexts.

The database descriptions, schemas, and pedagogical notes are maintained in a separate file to allow them to evolve independently of the application:

➡️ **[DATABASES.md](DATABASES.md)**

---

## Database Setup (Supabase)

### 1. Create a Supabase project

Create a free project at [supabase.com](https://supabase.com). The free tier provides persistent PostgreSQL access suitable for a full academic year.

### 2. Import the SQL schemas

Run the provided `.sql` files in the Supabase **SQL Editor** in this order for each database:

1. Open the SQL Editor in your Supabase dashboard
2. Paste and run the schema creation file (e.g., `bikeshare_pg.sql`)
3. Import CSV data using `\COPY` (via `psql`) or the Supabase Table Editor

```sql
-- Example import order for movies
\COPY directors     FROM 'directors.csv'     CSV HEADER ENCODING 'UTF8';
\COPY movies        FROM 'movies.csv'        CSV HEADER ENCODING 'UTF8';
\COPY movie_details FROM 'movie_details.csv' CSV HEADER ENCODING 'UTF8';
\COPY movie_cast    FROM 'movie_cast.csv'    CSV HEADER ENCODING 'UTF8';
\COPY ratings       FROM 'ratings.csv'       CSV HEADER ENCODING 'UTF8';
\COPY tags          FROM 'tags.csv'          CSV HEADER ENCODING 'UTF8';
```

### 3. Create the RPC functions

Run the following SQL in the Supabase SQL Editor to create the two functions the frontend depends on.

#### `run_student_query` — secure query executor

```sql
CREATE OR REPLACE FUNCTION run_student_query(query text, schema_name text DEFAULT 'public')
RETURNS json
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
  result json;
  clean_query text;
BEGIN
  clean_query := trim(query);

  -- Only allow SELECT and CTEs
  IF NOT (lower(clean_query) LIKE 'select%' OR lower(clean_query) LIKE 'with%') THEN
    RAISE EXCEPTION 'Only SELECT queries are allowed';
  END IF;

  -- Block DDL and DML keywords
  IF lower(clean_query) ~ '\b(insert|update|delete|drop|truncate|alter|grant|revoke)\b' THEN
    RAISE EXCEPTION 'Unauthorized query';
  END IF;

  -- Block system catalog access
  IF lower(clean_query) ~ 'information_schema|pg_catalog' THEN
    RAISE EXCEPTION 'Access denied';
  END IF;

  -- Auto-apply LIMIT if missing
  IF lower(clean_query) !~ '\blimit\b' THEN
    clean_query := clean_query || ' LIMIT 100';
  END IF;

  -- Set search_path to selected schema
  EXECUTE 'SET LOCAL search_path TO ' || quote_ident(schema_name) || ', public';

  EXECUTE 'SELECT json_agg(t) FROM (' || clean_query || ') t' INTO result;
  RETURN COALESCE(result, '[]'::json);
END;
$$;

GRANT EXECUTE ON FUNCTION run_student_query(text, text) TO anon;
```

#### `get_schema` — schema introspection

```sql
CREATE OR REPLACE FUNCTION get_schema(schema text DEFAULT 'public')
RETURNS json
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
  result json;
BEGIN
  EXECUTE 'SELECT json_agg(t) FROM (
    SELECT table_name, column_name, data_type, ordinal_position
    FROM information_schema.columns
    WHERE table_schema = ''' || schema || '''
    ORDER BY table_name, ordinal_position
  ) t' INTO result;
  RETURN COALESCE(result, '[]'::json);
END;
$$;

GRANT EXECUTE ON FUNCTION get_schema(text) TO anon;
```

### 4. Configure anonymous access

Ensure the `anon` role has no direct table grants. All access goes through the two RPC functions above. You can verify this in **Supabase → Authentication → Policies**: no RLS policies are needed since the `anon` role has no direct table permissions.

---

## Security Model

| Layer | Mechanism | Effect |
|---|---|---|
| Query type | `LIKE 'select%'` / `LIKE 'with%'` check | Only SELECT and CTEs accepted |
| DDL/DML blocking | Regex on reserved keywords | Prevents INSERT, UPDATE, DELETE, DROP, etc. |
| Catalog access | Regex block on `information_schema` / `pg_catalog` | Students cannot enumerate all schemas or roles |
| Row safety | Auto-`LIMIT 100` | Prevents accidental full-table scans in results |
| Schema isolation | `SET LOCAL search_path` per call | Students query the selected schema without prefix |
| Role permissions | `GRANT EXECUTE … TO anon` only | No direct table read/write via REST API |

> The functions use `SECURITY DEFINER`, meaning they execute with the privileges of the function owner, not the calling role. The `anon` role only has `EXECUTE` rights on these two functions.

---

## Frontend Setup

The frontend is a single HTML file. To deploy it:

1. Copy the HTML file to any static hosting service (GitHub Pages, Netlify, Vercel, etc.) or simply open it locally in a browser.
2. In the HTML file, set your Supabase project URL and anonymous public key in the configuration section at the top of the script:

```js
// Configuration — update these values for your Supabase project
const SUPABASE_URL = "https://<your-project-ref>.supabase.co";
const SUPABASE_ANON_KEY = "<your-anon-public-key>";
```

> **The anon key is safe to expose in the frontend.** It is a public key that only grants access to what is explicitly permitted for the `anon` role — in this case, the two RPC functions above. No data modification is possible via this key.

3. Update the schema list if you have added or renamed schemas:

```js
const SCHEMAS = ["bikeshare", "boutique", "movies"];
```

No build step, no dependencies, no package manager — the file is self-contained.

---

## Project Structure

```
sql-practice-interface/
│
├── frontend/
│   └── index.html              # Single-file SQL editor app
│
├── databases/
│   ├── bikeshare/
│   │   ├── bikeshare_pg.sql    # PostgreSQL schema + DDL
│   │   └── data/               # CSV files for import
│   ├── boutique/
│   │   ├── boutique_pg.sql
│   │   └── data/
│   └── movies/
│       ├── movies_pg.sql
│       └── data/
│           ├── directors.csv
│           ├── movies.csv
│           ├── movie_details.csv
│           ├── movie_cast.csv
│           ├── ratings.csv
│           └── tags.csv
│
├── supabase/
│   └── functions.sql           # run_student_query + get_schema definitions
│
└── README.md
```

---

## Contributing

Contributions are welcome, particularly:

- New practice databases (additional schemas with varied business contexts)
- Additional SQL exercises and solution files
- UI improvements to the frontend editor
- Translations of the interface

Please open an issue before submitting a pull request for significant changes.

---

## License

This project is released under the [MIT License](LICENSE).

---

*Developed for a Master-level relational databases course. Designed to be easy to maintain, cost-free to operate, and frictionless for students to use.*
