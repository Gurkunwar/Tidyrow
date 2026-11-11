# Tidyrow — project context

This file gives Claude Code the full context of the project. Keep it at the repo root.

## What we're building

**Tidyrow** is a free, privacy-first hub of small data tools for people who live in spreadsheets
(small shop owners, CAs/bookkeepers, HR and ops staff, students, small NGOs — India first).
Each tool fixes one everyday problem in under a minute: merge two sheets, remove duplicates,
fix phone numbers and dates, turn a bank statement PDF into Excel/Tally.

- Every tool runs **inside the user's browser** on one shared engine. Files are never uploaded.
- A new tool ships every week or two, chosen from what users search for.
- Think "pdf24, but for everyday data problems" — NOT PDF/image tools.

**Mission:** solve real, boring data problems for ordinary people, and learn production data
engineering by doing it.

## About the owner

- Student, working solo, 7–10 hours/week. Budget: under ₹100/month (domain only). No paid services.
- Data Engineer who was previously an SDE. Knows HTML/CSS/JS, React, Node.js, Python, Django,
  Java, Spring Boot, Go.
- Goals: learn data engineering concepts, build a resume-worthy project, and earn some revenue
  to support education. Revenue targets are stretch goals.

## Goals

| Goal     | Target                                                                                   | By       |
| -------- | ---------------------------------------------------------------------------------------- | -------- |
| Learning | Browser data engine, event pipeline, dbt warehouse, orchestrated analytics in production | Month 6  |
| Product  | 15 tools live; recipes (saved tool chains) working                                       | Month 6  |
| Users    | 10,000 monthly visitors; 1,000 returning; 200 saved recipes                              | Month 9  |
| Revenue  | First paid bank-statement conversion                                                     | Month 5  |
| Revenue  | About ₹10,000/month                                                                      | Month 12 |
| Cost     | Under ₹100/month                                                                         | Always   |

**Non-goals:** commodity PDF/image tools, a general BI product, enterprise sales, storing user
files on a server, anything resembling financial advice.

## Architecture

Three zones:

1. **Browser** — each tool page loads the engine in a Web Worker. DuckDB-WASM (SQL work),
   SheetJS (Excel), Papa Parse (CSV), pdf.js (PDFs). Large files are streamed in chunks.
2. **Cloudflare free tier** — Pages (static Astro site, one SEO page per tool), Workers API
   (auth, credits, export tokens, recipe sync, event intake, AI proxy, payment webhook),
   D1 (accounts, credits, recipes), R2 (event lake), Workers AI (optional assistant).
3. **Oracle Always Free ARM VM** — analytics only: Dagster + dbt-duckdb + Metabase in Docker
   Compose. The product must keep working if this VM is down.

### Monorepo layout (pnpm + uv)

```
apps/web          Astro site (React islands)
packages/engine   TypeScript engine (@tidyrow/engine), runs in a Web Worker
tools/<slug>/     one folder per tool: tool.json, faq.md, optional panel, golden files
workers/api       Cloudflare Workers API
data/             Dagster project + dbt models
infra/            Terraform (VM, Cloudflare DNS, R2, Pages)
```

## Design rules (do not break these)

1. **Files stay in the browser.** The tool worker may call only the event endpoint. A CSP and a
   CI test enforce this.
2. **Every tool is a manifest over engine ops.** New logic goes in the engine, not in a tool.
3. **Rules first, AI last.** AI is optional, costs ₹0, and sees column names and types only —
   never cell values.
4. **Ports and adapters** for anything on a free tier (storage, email, AI, payments).
5. **Scope is fixed, dates move.** A milestone starts only when the previous exit check passes.
6. Support Indian formats everywhere: DD/MM dates, lakh/crore, Indian commas, +91 phones, GSTIN, PIN.

## Tool manifest (`tools/<slug>/tool.json`)

```json
{
	"id": "merge-sheets",
	"title": "Merge two sheets on a common column",
	"category": "files",
	"users": ["small-business", "accountant"],
	"inputs": [
		{ "name": "left", "accept": ["csv", "xlsx"] },
		{ "name": "right", "accept": ["csv", "xlsx"] }
	],
	"params": {
		"key": { "type": "column", "from": "left" },
		"how": { "enum": ["left", "inner", "full"], "default": "left" }
	},
	"steps": [
		{ "op": "read" },
		{ "op": "join", "on": "$key", "how": "$how" },
		{ "op": "write", "format": "$input_format" }
	],
	"tier": "free",
	"seo": {
		"keywords": ["merge two excel sheets", "vlookup two files online"],
		"faq": "faq.md"
	}
}
```

A **recipe** is the same `steps` list saved with a name and expected columns. On rerun, check the
new file's columns first and ask the user to remap any that changed.

## Engine ops

| Op               | Does                                                              | Runs on                |
| ---------------- | ----------------------------------------------------------------- | ---------------------- |
| `read`           | Detect delimiter, encoding, header row, sheet; stream large files | Papa Parse, SheetJS    |
| `infer`          | Column types, DD/MM vs MM/DD, Indian number styles                | JS rules               |
| `map`            | Rename/reorder columns to a template                              | JS rules + fuzzy match |
| `clean`          | Trim, case, phone, date, PIN, GSTIN normalisers                   | JS rules               |
| `dedupe`         | Exact, then fuzzy with blocking + similarity threshold            | DuckDB-WASM            |
| `join` / `union` | Merge on keys; stack files with header alignment                  | DuckDB-WASM            |
| `split`          | By rows, size or column value                                     | DuckDB-WASM            |
| `profile`        | Blanks, distinct counts, outliers, duplicates, type mismatches    | DuckDB-WASM            |
| `diff`           | Rows added/removed/changed between two versions                   | DuckDB-WASM            |
| `extract_pdf`    | Text and table positions from PDFs                                | pdf.js                 |
| `write`          | CSV, XLSX, JSON, Tally XML                                        | SheetJS, JS            |

## Tool catalog (shipping order)

| #   | Tool                                   | Ops                            | Tier               | Milestone |
| --- | -------------------------------------- | ------------------------------ | ------------------ | --------- |
| 1   | Merge sheets                           | read, join, write              | Free               | M1        |
| 2   | Remove duplicates (exact + fuzzy)      | read, dedupe, write            | Free               | M1        |
| 3   | Convert files (Excel/CSV/JSON)         | read, write                    | Free               | M1        |
| 4   | Combine many files                     | read, map, union               | Free               | M2        |
| 5   | Split big file                         | read, split                    | Free               | M2        |
| 6   | Phone number fixer (+91)               | read, clean                    | Free               | M2        |
| 7   | Date fixer                             | read, infer, clean             | Free               | M2        |
| 8   | Text cleaner                           | read, clean                    | Free               | M2        |
| 9   | Map to template                        | read, map                      | Free               | M2        |
| 10  | Bank statement → Excel/Tally           | extract_pdf, map, clean, write | Paid per statement | M4        |
| 11  | GSTIN validator                        | read, clean                    | Free               | M4        |
| 12  | PIN code enricher                      | read, join                     | Free               | M4        |
| 13  | Lakh/crore converter + amount in words | read, clean                    | Free               | M4        |
| 14  | Data health check                      | read, profile                  | Free               | M5        |
| 15  | Compare two files                      | read, diff                     | Free               | M5        |

Backlog ideas: CGPA→percentage, attendance tracker, invoice extractor, email list validator,
address splitter, pivot builder, college counselling predictor.

Premium (any tool): files over 50 MB, batch runs, more than 3 saved recipes, no ads.

## Data model

**D1 tables:** `users`, `sessions`, `credit_ledger` (append-only; balance = sum(delta)),
`orders` (deduped on provider_order_id), `export_tokens` (one-time signed), `recipes`,
`tool_requests`, `flags`.

**Event schema** (cookie-less, daily-rotating anonymous session id, no file contents):

```json
{
	"event": "run_completed",
	"ts": "2026-12-14T10:02:11Z",
	"sid": "a91f…",
	"tool": "merge-sheets",
	"tool_version": 3,
	"rows_bucket": "10k-100k",
	"cols": 14,
	"file_mb_bucket": "1-10",
	"ms": 2140,
	"browser": "chrome",
	"device": "desktop",
	"country": "IN",
	"error_code": null
}
```

Other events: `tool_viewed`, `run_started`, `run_failed`, `search`, `recipe_saved`,
`recipe_rerun`, `checkout_started`, `credit_spent`.

**Analytics pipeline (the data engineering core):**

- Bronze: R2 `events/dt=YYYY-MM-DD/hour=HH/*.ndjson.gz` (Worker batches every 60 s / 500 events)
- Silver: R2 Parquet, validated, typed, deduped; bad rows to `quarantine/` with a reason
- Gold (dbt-duckdb): `fct_tool_runs`, `dim_tools`, `daily_tool_metrics`, `fct_funnel`,
  `search_gaps` (the build-next list), `tool_errors`, `revenue_daily`
- Dagster: hourly bronze→silver, nightly dbt build, freshness + row-count checks
- Metabase dashboards

## AI assistant (M5, optional)

Request path, cheapest first: rules + embedding match → cache → Workers AI (free allocation)
→ user's own key (stays in their browser) → fallback to closest tool. Model output must be
JSON matching the recipe schema; validate column indexes and op/type fit; show as editable
steps; the model never runs anything. Release gate: correct recipe for ≥ 80% of a 50-request
golden set.

## Payments (M4)

Credits over UPI. Checkout → signed webhook → `orders` + `credit_ledger`. Paid tool finishes
locally → Worker spends one credit → returns signed one-time export token → download unlocks.
Client-side bypass is an accepted risk. **Provider is undecided** (Razorpay assumed; unverified
whether a student can onboard as an individual — alternatives: Instamojo, Lemon Squeezy, Dodo).

## Roadmap

| Milestone                   | Dates                | Hours | Exit check                                                                                                      |
| --------------------------- | -------------------- | ----- | --------------------------------------------------------------------------------------------------------------- |
| M0 Foundations              | 16 Nov – 29 Nov 2026 | 16    | Site deploys from `main` to Cloudflare Pages; engine loads in a Web Worker; CI green; VM rebuilt from Terraform |
| M1 Engine + first 3 tools   | 30 Nov – 27 Dec 2026 | 36    | 100 MB CSV merge in under 15 s with the page responsive; 3 tools live; soft launch to 20 people                 |
| M2 Tool factory (tools 4–9) | 28 Dec – 31 Jan 2027 | 40    | A simple tool ships in ≤ 6 h from a manifest; 9 tools live and indexed                                          |
| M3 Analytics pipeline       | 1 Feb – 21 Feb 2027  | 24    | Hourly events reach silver and nightly dbt passes unattended for 7 days; first search-gap report                |
| M4 India tools + payments   | 22 Feb – 28 Mar 2027 | 40    | Statement parser passes golden files for 5 banks; first paid statement; 13 tools live                           |
| M5 Recipes, AI, tools 14–15 | 29 Mar – 25 Apr 2027 | 32    | 15 tools live; a recipe reruns on next month's file; assistant ≥ 80% on golden set                              |
| M6 Growth and autopilot     | 26 Apr – 23 May 2027 | 28    | Upkeep under 2 h/week for 4 weeks; request board live; next 3 tools chosen from data                            |

### M0 tasks (start here)

1. Set up monorepo (pnpm + uv), lint, format, pre-commit — 2 h
2. Astro site skeleton: home, category and tool page layouts — 3 h
3. Deploy to Cloudflare Pages from `main`, with preview deploys — 2 h
4. Engine package skeleton running in a Web Worker — 3 h
5. CI: unit tests, golden-file runner, Playwright smoke test — 3 h
6. Terraform: Oracle VM, Cloudflare DNS, R2 buckets — 2 h
7. Buy domain; draft the privacy page — 1 h

## Definition of done (every task)

- Merged through CI with all checks green
- Tests at the right layer (unit for ops, golden files for tools, Playwright for flows)
- Events and error codes emitted
- Docs updated
- Deployed and smoke-checked

## New tool checklist

Manifest validates · ≥ 3 golden files · handles empty file, one row, wrong type, 100 MB ·
Indian formats checked · plain-language errors · FAQ with 5 real questions · events fire ·
works on mobile Chrome.

## Key decisions (ADRs)

An ADR (Architecture Decision Record) is a short note recording one design decision, why it
was made, and what would reopen it.

- ADR-017 Process files in the browser; never upload them
- ADR-018 Static, SEO-first Astro site; one page per tool
- ADR-019 Manifest-driven Tool SDK; recipes are op lists
- ADR-020 Cloudflare Pages + Workers + D1 for the product backend
- ADR-021 Anonymous events → R2 → Dagster + dbt on the Oracle VM
- ADR-022 Credits over UPI; client-side unlock accepted (Proposed — provider undecided)
- ADR-023 AI sees schema only; rules → Workers AI → user key
- ADR-024 No commodity PDF or image tools
- Still valid from the earlier plan: ports and adapters; one shared engine; Dagster
  orchestrates plain functions; Parquet + DuckDB + dbt; rules first, LLM last

## Open questions

- Payment provider for a student seller (blocks M4)
- Which 5 banks first for the statement converter (blocks M4)
- Fuzzy dedupe: DuckDB string-similarity functions or a JS library (blocks M1)
- Domain name (short `.in` or `.app`)
- Ads before 10,000 monthly visitors? Premium at ₹99 or ₹199/month?

## History

The project started as two paid APIs (a messy-data normalisation API and an SEC filings event
API). That plan was dropped on 11 Nov 2026 in favour of this hub: the APIs had weak demand for
a solo student, SEC data is US-only, and server-side LLM use risked costs. Ignore any leftover
references to "Tidyrow Import", "Tidyrow Signals", Postgres, Procrastinate or EDGAR.

## Where the full docs live

- Full spec: TDD v2 — Tidyrow Hub — https://claude.ai/artifact/6KqAVoFLiwR9SkfxTH7Q4j
- Project tracker (Notion, "Tidyrow HQ"): https://app.notion.com/p/3fec348b97cd8124a192e2b0ca87b6f6
  — milestones, 58 tasks, tool catalog, decision log, risks, open questions

## How to work with me

- Explain data engineering concepts as we build them; learning is a main goal.
- Keep everything on free tiers; flag anything that could cost money before using it.
- Work milestone by milestone; don't start the next one until the exit check passes.
- Prefer small, tested steps I can finish in a 2–3 hour sitting.
