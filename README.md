# Kenya County Budget Tracker (API)

A JSON API for tracking Kenyan national government budget allocations to counties (and, with caveats, wards), year over year, alongside the current political office holder at each level.

This repository is the **Rails API backend**. A separate SvelteKit frontend will consume it and live in its own repository/deploy.

> **Status:** early development. The data model and read-only API exist; the data currently in `db/seeds.rb` is **placeholder sample data, not real figures**.

## What it tracks

The data follows Kenya's administrative hierarchy:

```
National → County → Constituency → Ward
```

| Level | Office holder(s) |
|---|---|
| County | Governor, Deputy Governor, Senator, Woman Representative |
| Constituency | Member of Parliament (MP) |
| Ward | Member of County Assembly (MCA) |

### Important data caveat

- **County-level** national allocations (via the County Allocation of Revenue Act and Commission on Revenue Allocation formulas) are standardized and published year to year.
- **Ward-level** national allocations do not exist as a clean dataset. There is no direct national budget line to wards. Ward-level figures are derived from separate channels and are always labeled by `fund_type`:
    - `county_allocation`: national allocation to a county
    - `ng_cdf`: National Government Constituency Development Fund (allocated per constituency, not per ward)
    - `ward_dev_fund`: ward development funds allocated by the county government

Ward figures must never be presented as a clean national allocation.

## Tech stack

- Ruby 4.0.1
- Rails 8.1 (API-only)
- PostgreSQL 18
- rack-cors
- Runtimes managed with [mise](https://mise.jdx.dev)

## Getting started

### Prerequisites

- [mise](https://mise.jdx.dev)
- PostgreSQL running locally (check with `pg_isready`)

### Setup

```bash
git clone <repo-url>
cd govt_budget_track

mise install          # installs the runtimes pinned in mise.toml
bundle install

bin/rails db:create
bin/rails db:migrate
bin/rails db:seed     # loads placeholder sample data
```

### Run

```bash
bin/rails server
```

The API is served at `http://localhost:3000`.

### CORS

Requests are allowed from `http://localhost:5173` (the SvelteKit dev server). For other origins, set `FRONTEND_ORIGIN`:

```bash
FRONTEND_ORIGIN=https://your-frontend-domain bin/rails server
```

## API

All endpoints are read-only and namespaced under `/api`.

| Method | Path | Description |
|---|---|---|
| GET | `/api/regions` | List regions. Filter with `?level=county` and/or `?parent_id=ID` |
| GET | `/api/regions/:id` | A region with its current office holders and child regions |
| GET | `/api/regions/:id/allocations` | Budget allocations for a region. Filter with `?from=2022&to=2024` |

Example:

```bash
curl "localhost:3000/api/regions?level=county"
curl "localhost:3000/api/regions/2/allocations?from=2022&to=2024"
```

Notes:

- `amount` is returned as a decimal string (for example `"5000000000.0"`) to avoid floating-point precision loss.
- `regions/:id` returns only **current** office holders (no `term_end`, or one in the future).

## Data model

- `Region`: self-referential tree (`parent_id`) with a `level` enum (`national`, `county`, `constituency`, `ward`)
- `OfficeHolder`: belongs to a region; `name`, `title`, `party`, `term_start`, `term_end`
- `BudgetAllocation`: belongs to a region; `fund_type`, `status` (`allocated` or `disbursed`), `year`, `amount`, `currency` (default `KES`), `source_note`

An allocation is unique per region, fund type, year, and status.

## Data sources

Real data is to be sourced from:

- National Treasury (CARA, Division of Revenue)
- Commission on Revenue Allocation (CRA)
- Office of the Controller of Budget
- NG-CDF Board
- County Fiscal Strategy Papers / County Budget Review reports

## Project documentation

Architecture decisions, the data model rationale, dependency versions, and open questions are tracked in [`docs/architecture-decisions.md`](docs/architecture-decisions.md).

## Roadmap

- [ ] Import real county-level allocation data (CARA)
- [ ] Import constituency-level NG-CDF data
- [ ] Ward development fund data (per-county adapters)
- [ ] Office holder data for all 47 counties, 290 constituencies, and wards
- [ ] Switch responses to Blueprinter serializers
- [ ] SvelteKit frontend
- [ ] Admin/editing with JWT authentication

## License

TBD