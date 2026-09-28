# Kenya County Budget Tracker — Architecture & Decisions Log

_Last updated: 2026-09-27_

This document is the standing reference for the project. Upload it to Claude Project Knowledge so every conversation in the Project starts from these decisions instead of re-explaining them.

---

## 1. Project Goal

A public-facing website tracking Kenyan national government budget allocation to counties (and, with caveats, wards), year over year, alongside the current political office holder at each level.

---

## 2. Domain Reality (why the data model looks the way it does)

- **County level**: National government allocates funds to counties via the annual **County Allocation of Revenue Act (CARA)**, following formulas from the **Commission on Revenue Allocation (CRA)**. This is clean, standardized, published year-to-year (National Treasury, CRA, Controller of Budget reports).
- **Ward level**: There is **no direct national budget line to wards**. Ward-level development funding comes from two separate channels, which must be tracked as distinct fund types and clearly labeled as such in the UI:
  - **NG-CDF** (National Government Constituency Development Fund) — allocated per **constituency**, not per ward. A constituency contains several wards.
  - **Ward-based county development funds** — allocated by the **county government**, not national government.
- Because of this, the hierarchy has **four levels**, not two:

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
- Commission on Revenue Allocation (CRA) — allocation formulas
- Office of the Controller of Budget — implementation/expenditure reports
- NG-CDF Board — constituency-level fund data
- County Fiscal Strategy Papers / County Budget Review — ward development fund data (per-county, inconsistent formats)

**Caveat to carry through the whole project**: county-level national allocation data is standardized; ward-level data is not — it's a derived/approximate layer. Every ward figure in the UI should carry a visible fund-type label and a source note, never presented as if it were a clean national allocation.

---

## 4. Tech Stack

- **Backend**: Rails, API-only mode (`rails new kenya-budget-tracker --api`). No views, no asset pipeline — JSON only.
- **Frontend**: SvelteKit, deployed and run entirely separately from Rails.
- **Two independently deployable units.** Rails → Render/Fly.io/Heroku-style API host. SvelteKit → Vercel/Netlify/Node host. Connected via `PUBLIC_API_URL` env var, not a shared deploy.
- **CORS**: `rack-cors` configured from day one (separate origins in dev and prod — this is mandatory, not optional, given the strict split).
- **JSON shaping**: jbuilder (built into Rails) to start; move to `blueprinter` or `jsonapi-serializer` once response shapes multiply past a handful of ad hoc `as_json` calls.
- **Auth**: deferred until an editing/admin feature is needed. When added: token-based (JWT via `devise-jwt`), not Rails' default cookie sessions — cookies get awkward across separate origins.

---

## 5. Data Model (Rails / ActiveRecord)

```ruby
class Region < ApplicationRecord
  belongs_to :parent, class_name: "Region", optional: true
  has_many :children, class_name: "Region", foreign_key: :parent_id
  has_many :office_holders
  has_many :budget_allocations
  enum level: { national: 0, county: 1, constituency: 2, ward: 3 }
end

class OfficeHolder < ApplicationRecord
  belongs_to :region
  # name, title, party, term_start, term_end
end

class BudgetAllocation < ApplicationRecord
  belongs_to :region
  enum fund_type: { county_allocation: 0, ng_cdf: 1, ward_dev_fund: 2 }
  enum status: { allocated: 0, disbursed: 1 }
  # year, amount, currency, source_note
end
```

- Self-referential `Region` (parent/children) encodes the whole National → County → Constituency → Ward tree in one table.
- If drill-down queries get complex, consider `ancestry` or `closure_tree` gems rather than hand-rolling recursive queries.
- `fund_type` enum is what enforces the county-allocation vs. NG-CDF vs. ward-dev-fund distinction at the data layer, not just in the UI copy.

---

## 6. API Shape (Rails, `/api` namespace)

```
GET /api/regions?level=county                    # list counties
GET /api/regions/:id                              # region + current office holder + children
GET /api/regions/:id/allocations?from=2020&to=2025 # year range for charts
```

Example controller pattern:

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

## 7. Frontend Routes (SvelteKit)

Nested routes mirror the region hierarchy, one-to-one:

```
src/routes/
  counties/+page.svelte
  counties/[id]/+page.svelte
  counties/[id]/constituencies/[cid]/+page.svelte
  counties/[id]/constituencies/[cid]/wards/[wid]/+page.svelte
```

- Each `+page.js` / `+page.server.js` fetches "this region + office holder + allocations" in one shot via `load` functions.
- Charting: `layerchart` or `svelte-chartjs` — sufficient for year-over-year line/bar charts, no heavier library needed.

---

## 8. Open Questions / To Revisit

- [ ] Confirm hosting targets for Rails and SvelteKit (not yet chosen)
- [ ] Decide how ward development fund data (per-county, inconsistent format) gets normalized/ETL'd — likely needs a per-county adapter rather than one universal parser
- [ ] Decide admin/editing workflow and timing for adding JWT auth
- [ ] Decide initial seed scope: how many counties/constituencies/wards to hand-seed for prototype vs. full 47/290/1450 dataset

---

## 9. How to Use This Document

Add new confirmed decisions to the relevant section above (don't just append a changelog at the bottom — keep the doc itself current). Re-upload the updated file to Project Knowledge whenever it changes, since Projects read files fresh each conversation but don't auto-sync edits made outside an upload.
