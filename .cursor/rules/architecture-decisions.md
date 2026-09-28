# Kenya County Budget Tracker — Architecture & Decisions Log

_Last updated: 2026-09-28_

This is the standing reference for the project. It is used as:
- **Claude Project Knowledge** (upload this file), and
- **Cursor context** (keep it at `docs/architecture-decisions.md` in the repo; the `.cursor/rules/project-context.mdc` rule points to it).

Keep this file current. When a decision changes, edit the relevant section rather than appending a changelog.

---

## 1. Project Goal

A public-facing website tracking Kenyan national government budget allocation to counties (and, with caveats, wards), year over year, alongside the current political office holder at each level.

---

## 2. Domain Reality (why the data model looks the way it does)

- **County level**: National government allocates funds to counties via the annual **County Allocation of Revenue Act (CARA)**, following formulas from the **Commission on Revenue Allocation (CRA)**. This is clean, standardized, and published year to year (National Treasury, CRA, Controller of Budget reports).
- **Ward level**: There is **no direct national budget line to wards**. Ward-level development funding comes from two separate channels, which must be tracked as distinct fund types and clearly labeled in the UI:
  - **NG-CDF** (National Government Constituency Development Fund): allocated per **constituency**, not per ward. A constituency contains several wards.
  - **Ward-based county development funds**: allocated by the **county government**, not the national government.
- The hierarchy therefore has **four levels**:

  ```
  National → County → Constituency → Ward
  ```

- **Office holders differ by level:**

  | Level | Office(s) |
    |---|---|
  | County | Governor, Deputy Governor, Senator, Woman Representative |
  | Constituency | Member of Parliament (MP) |
  | Ward | Member of County Assembly (MCA) |

---

## 3. Data Sources

- National Treasury (CARA texts, Division of Revenue)
- Commission on Revenue Allocation (CRA): allocation formulas
- Office of the Controller of Budget: implementation and expenditure reports
- NG-CDF Board: constituency-level fund data
- County Fiscal Strategy Papers / County Budget Review: ward development fund data (per county, inconsistent formats)

**Caveat to carry through the whole project**: county-level national allocation data is standardized; ward-level data is not. It is a derived/approximate layer. Every ward figure in the UI must carry a visible fund-type label and a source note, and must never be presented as a clean national allocation.

---

## 4. Tech Stack

- **Backend**: Rails, API-only (`rails new . --api --database=postgresql`). No views, no asset pipeline. JSON only.
- **Frontend**: SvelteKit, run and deployed entirely separately from Rails.
- **Two independently deployable units.** Rails goes to a Render/Fly.io/Heroku-style API host; SvelteKit goes to Vercel/Netlify/a Node host. They are connected via the `PUBLIC_API_URL` env var, never a shared deploy. Do not suggest Rails serving the built Svelte app.
- **CORS**: `rack-cors` configured from day one (separate origins in dev and prod; mandatory given the strict split).
- **JSON shaping**: `blueprinter`. Plain `as_json` is acceptable only for the very first endpoints; move to blueprinter before response shapes multiply. (jbuilder is not used.)
- **Auth**: deferred until an editing/admin feature is needed. When added: token-based (JWT via `devise-jwt`), not Rails' default cookie sessions, since cookies are awkward across separate origins.
- **Database**: PostgreSQL. Note that a plain `rails new` defaults to SQLite, so the app must be generated or switched to PostgreSQL (see §10).

---

## 5. Dependencies (latest stable, checked Sept 2026)

Re-check for newer stable releases before installing; these drift quickly (Rails already moved from 8.1.3.1 to 8.1.4 during setup).

**Backend**

| Dependency | Version | Notes |
|---|---|---|
| Ruby | 4.0.1 | Managed by mise |
| Rails | 8.1.4 | Installed via `gem install rails` (not mise-managed) |
| PostgreSQL | 18.x (latest patch) | |
| pg | latest | Postgres adapter |
| rack-cors | 3.0.0 | |
| blueprinter | 1.2.1 | |
| devise | 5.0.x | Added when auth ships |
| devise-jwt | 0.13.0 | Added when auth ships |

**Frontend**

| Dependency | Version | Notes |
|---|---|---|
| Node.js | 24.x LTS | Do not use Node 26 until it reaches LTS (Oct 2026) |
| svelte | 5.57.1 | |
| @sveltejs/kit | 2.70.3 | |
| vite | 8.3.1 | |
| typescript | 7.0.2 | |
| chart.js + svelte-chartjs | 4.5.1 / 4.0.1 | Default charting choice |
| layerchart | 2.5.0 | Alternative; pick one, not both |

---

## 6. Development Environment

- Runtimes are managed with **mise**. Pin them per project in `mise.toml`:

  ```toml
  [tools]
  ruby = "4.0.1"
  node = "24"
  postgres = "18"
  ```

- mise manages runtimes only. Gems (`bundle`) and npm packages are installed on top.
- Install Rails as a normal user with `gem install rails`. **Never use `sudo`** with a mise-managed Ruby.
- If `rails` is "not found" right after installing it, run `mise reshim` and then `hash -r` (bash) or `rehash` (zsh).
- Local project directory: `govt_budget_track` (Rails API). The SvelteKit app directory name is not yet decided.

---

## 7. Data Model (Rails / ActiveRecord)

Rails 8 requires the positional enum syntax (`enum :level, {...}`); the older `enum level: {...}` keyword form was removed.

```ruby
class Region < ApplicationRecord
  belongs_to :parent, class_name: "Region", optional: true
  has_many :children, class_name: "Region", foreign_key: :parent_id, inverse_of: :parent
  has_many :office_holders
  has_many :budget_allocations

  enum :level, { national: 0, county: 1, constituency: 2, ward: 3 }
end

class OfficeHolder < ApplicationRecord
  belongs_to :region
  # name, title, party, term_start, term_end
end

class BudgetAllocation < ApplicationRecord
  belongs_to :region

  enum :fund_type, { county_allocation: 0, ng_cdf: 1, ward_dev_fund: 2 }
  enum :status, { allocated: 0, disbursed: 1 }
  # year, amount, currency, source_note
end
```

- Self-referential `Region` (parent/children) encodes the whole National → County → Constituency → Ward tree in one table.
- If drill-down queries get complex, consider `ancestry` or `closure_tree` rather than hand-rolled recursive queries.
- The `fund_type` enum enforces the county-allocation vs. NG-CDF vs. ward-dev-fund distinction at the data layer, not just in UI copy.

---

## 8. API Shape (Rails, `/api` namespace)

```
GET /api/regions?level=county                       # list counties
GET /api/regions/:id                                # region + current office holder + children
GET /api/regions/:id/allocations?from=2020&to=2025  # year range for charts
```

Starter controller pattern (replace `as_json` with blueprinter as the API grows):

```ruby
module Api
  class RegionsController < ApplicationController
    def index
      regions = Region.where(level: params[:level])
      render json: regions.as_json(include: :office_holders)
    end

    def show
      region = Region.find(params[:id])
      render json: region.as_json(
        include: { office_holders: {}, budget_allocations: {} }
      )
    end
  end
end
```

---

## 9. Frontend Routes (SvelteKit)

Nested routes mirror the region hierarchy one-to-one:

```
src/routes/
  counties/+page.svelte
  counties/[id]/+page.svelte
  counties/[id]/constituencies/[cid]/+page.svelte
  counties/[id]/constituencies/[cid]/wards/[wid]/+page.svelte
```

- Each `+page.js` / `+page.server.js` fetches "this region + office holder + allocations" in one shot via `load` functions.
- API base URL comes from `PUBLIC_API_URL` (e.g. `http://localhost:3000/api` in dev).
- Charting: chart.js + svelte-chartjs unless changed (see open questions).

---

## 10. Open Questions / To Revisit

- [ ] **Database switch**: the Rails app was scaffolded with the SQLite default. Switch to PostgreSQL (`bundle remove sqlite3 && bundle add pg`, update `config/database.yml`) or regenerate with `--database=postgresql` before writing migrations. Confirm Postgres is running (`pg_isready`).
- [ ] Confirm chart.js + svelte-chartjs vs. layerchart
- [ ] Confirm hosting targets for Rails and SvelteKit (not yet chosen)
- [ ] Decide how ward development fund data (per county, inconsistent format) gets normalized/ETL'd; likely a per-county adapter rather than one universal parser
- [ ] Decide admin/editing workflow and timing for adding JWT auth
- [ ] Decide initial seed scope: hand-seeded prototype vs. the full 47 counties / 290 constituencies / ~1,450 wards
- [ ] Name and scaffold the SvelteKit app

---

## 11. How to Use This Document

- **Claude Project**: re-upload this file to Project Knowledge whenever it changes. Projects don't auto-sync edits made elsewhere, and conversations don't share context with each other, so decisions made in a chat must be written back here.
- **Cursor**: keep the file at `docs/architecture-decisions.md`, referenced from an always-apply rule at `.cursor/rules/project-context.mdc`. Edits in the repo are picked up automatically.
- Treat this file as the source of truth. If a request conflicts with it, flag the conflict and ask before deviating.