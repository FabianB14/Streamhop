# codex.md — Streamhop

> Codex reads `AGENTS.md` by convention. Keep this as `codex.md` for your own reference, and also save a copy as `AGENTS.md` at the repo root so the agent picks it up automatically.

---

## 1. What we're building

**Streamhop** — a web app that tells you the cheapest way to watch everything on your list by *rotating* streaming subscriptions instead of stacking them.

You give it the shows and movies you want to watch. It knows which service each one is on and when it drops. It hands you a month-by-month plan: which one or two services to keep active each month, what to binge while you have them, and when to cancel — plus a running total of what you're saving versus paying for everything all year.

**One-liner:** "Stop paying for five streaming services you don't watch. Streamhop plans the rotation for you."

## 2. Why this exists (the wedge)

Rotation is already mainstream behavior — people subscribe, binge one show, cancel, move on. But every guide today tells users to manage it by hand with a spreadsheet or calendar. Two mature tool categories each own half the problem and nobody connects them:

- **Watchlist / "where to watch" apps** (JustWatch, Reelgood, Trakt) know *what's on which service* but never tell you what to pay for or when to cancel.
- **Subscription trackers** (Rocket Money, Trim) know *what you're paying* but are content-blind.

Streamhop is the missing middle: the **optimizer** that turns a watchlist + release dates + prices into an actual subscribe/cancel schedule.

**Known strategic risks (build with these in mind):**
1. **We rent the core data.** Availability data comes from a third-party API. Wrap it behind one adapter so we can swap providers without touching the rest of the app.
2. **The optimizer is a feature, not a moat.** Defensibility comes from being the *savings brand* — a hard dollar number, well-timed cancel reminders, a delightful flow — not the algorithm alone.
3. **Model tension.** Affiliate revenue rewards *more* sign-ups; our value prop is *fewer*. Only ever use affiliate links on the timed subscribe events the plan recommends. Never upsell extra services.

## 3. MVP scope

**In scope (v1):**
- Add titles to a watchlist (search by name, resolve via the data adapter).
- A seed catalog of major US streaming services with monthly prices.
- The optimizer: generate a month-by-month rotation plan over a chosen horizon (default 6 months) with a max number of concurrent services (default 2).
- A plan view: per-month services to hold, titles to watch, monthly cost, and total savings vs. the "keep everything" baseline.
- Plain-language reasoning per month ("Hold Max in March to watch *A*, *B*, *C*").

**Explicitly out of scope for v1 (note as TODO, don't build yet):**
- Real auth / accounts (use a lightweight single-user/demo session — see §8).
- Auto-import from Trakt/JustWatch.
- Email/calendar reminders.
- Household / multi-profile coordination.
- Sports / league-pass logic.
- Live price scraping (prices come from a hand-maintained seed file in v1).

## 4. The optimizer (the heart of the product)

Implement this as a **pure, fully unit-tested function** with no I/O. Everything else is plumbing around it.

**Inputs:**
- `watchlist`: list of titles, each with `id`, `name`, and a set of `availability` windows: `{service_id, available_from, available_until?}`. For already-streaming titles, `available_from` is "now." For upcoming titles, it's the release date.
- `services`: catalog with `{service_id, monthly_price_cents}`.
- `horizon_months`: int (default 6).
- `max_concurrent`: max services active in any single month (default 2).
- `today`: anchor date.

**Output:**
- `months`: ordered list of `{month, active_service_ids[], titles_watched[], cost_cents}`.
- `total_cost_cents`, `baseline_cost_cents` (= all services that appear in any availability window, every month, for the horizon), `savings_cents`.
- Every watchlist title that is watchable within the horizon must be assigned to exactly one (month, service) slot where it's available.

**v1 algorithm — greedy month-by-month set cover (keep it simple and explainable):**
1. Build, for each title, the set of (month, service) pairs where it's watchable inside the horizon (month ≥ its `available_from`, and ≤ `available_until` if present).
2. Walk months in order. For each month, score each candidate service by how many *still-unassigned* titles it would unlock that month, per dollar.
3. Activate up to `max_concurrent` services that unlock the most unassigned titles (skip a service if it unlocks zero). Prefer batching: if waiting one month lets a service cover more titles at once, defer (so the user binges then cancels rather than holding it two thin months).
4. Mark covered titles as assigned to that month/service. Carry the rest forward.
5. After the walk, anything still unassigned but watchable later than the horizon → return as `deferred` with a note. Anything never available on any catalog service → return as `unavailable`.
6. Compute baseline and savings.

Keep the algorithm behind an interface (`Optimizer.plan(...)`) so we can later swap the greedy heuristic for an ILP/CP-SAT solver without changing callers. Add a docstring explaining the heuristic and its known suboptimality in plain terms.

## 5. Data models

Use SQLModel (SQLAlchemy + Pydantic). Money is always integer cents.

```
Service(id, slug, name, monthly_price_cents, ad_tier_price_cents?, logo_url?)
Title(id, external_id, name, kind[movie|series], release_date?, poster_url?)
TitleAvailability(id, title_id, service_id, available_from, available_until?)
WatchlistItem(id, session_id, title_id, status[want|watched], created_at)
Plan(id, session_id, generated_at, horizon_months, max_concurrent,
     total_cost_cents, baseline_cost_cents, savings_cents)
PlanMonth(id, plan_id, month[YYYY-MM], service_ids[json], title_ids[json], cost_cents, reason)
```

`session_id` stands in for a user until real auth lands.

## 6. Data sources (behind one adapter)

Define an `AvailabilityProvider` interface with two methods: `search_titles(query)` and `get_availability(external_id)`. Ship two implementations:

- **`WatchmodeProvider`** — Watchmode API (purpose-built for streaming availability, has a free tier). Reads key from `WATCHMODE_API_KEY`.
- **`FixtureProvider`** — reads from `backend/fixtures/*.json`. **This is the default when no API key is set**, so the app and all tests build and run with zero external dependencies. Seed it with ~15 real-ish titles across services for the demo.

Release dates and posters can come from the same provider or TMDB; keep that detail inside the adapter.

**Attribution:** if you wire up TMDB or JustWatch-derived data, follow their attribution requirements in the UI footer. Don't hardcode availability anywhere outside fixtures or the live adapter.

**Prices:** `backend/seed/services.json`, hand-maintained, with a clear `# verify before launch — prices change` comment. Do not assert prices as live truth.

## 7. API (FastAPI)

```
GET    /api/health
GET    /api/services                 -> catalog
GET    /api/titles/search?q=         -> provider search
POST   /api/watchlist                -> add {external_id} (resolves + stores availability)
GET    /api/watchlist
DELETE /api/watchlist/{id}
POST   /api/plan/generate            -> body {horizon_months, max_concurrent} -> Plan
GET    /api/plan/latest
```

Graceful failure: if the provider errors or times out, return a clear message and fall back to fixtures in dev. Never crash the plan view because one title failed to resolve.

## 8. Tech stack

- **Backend:** Python 3.12, FastAPI, SQLModel, httpx, Uvicorn. SQLite in dev, Postgres in prod.
- **Frontend:** React + Vite + TypeScript, Tailwind, TanStack Query. Keep components small.
- **Auth (v1):** none. Generate a `session_id` cookie on first visit. Leave a clear `# TODO: real auth` boundary so it's a drop-in later. Do not build login flows for the showcase.
- **Deploy:** DigitalOcean App Platform — one repo, `backend/` and `frontend/` as separate components. Include a `do/app.yaml` and a root README with deploy steps.
- **Monorepo layout:**
```
/backend   (app/, tests/, fixtures/, seed/, pyproject.toml)
/frontend  (src/, index.html, package.json)
/do        (app.yaml)
README.md
AGENTS.md  (copy of this file)
```

## 9. Screens (frontend)

1. **Watchlist** — search + add titles, see your list with posters and which service(s) each is on.
2. **Controls** — sliders/inputs for horizon (3–12 mo) and max concurrent services (1–3).
3. **Plan** — the payoff screen. A horizontal month timeline; each month card shows the service logos to keep, the titles you'll watch, and the cost. A big hero number at the top: **"You save $___/year."** Each month has a one-line human reason.
4. **Landing** (showcase polish, last) — the pitch, a "Try the demo" button that loads the seed watchlist so a reviewer sees a full plan in one click.

Design for the savings number to be the emotional centerpiece. This is what makes it a savings brand, not a tracker.

## 10. Monetization (design in, keep v1 free)

- **Free:** plan generation, manual watchlist.
- **Pro (~$2–3/mo, later):** reminders, auto-import, household coordination, price-hike alerts.
- **Affiliate:** only on the subscribe events the plan already recommends. No extra-service upsell, ever. Disclose it.

## 11. Agent working agreement

**Setup:**
```
# backend
cd backend && python -m venv .venv && . .venv/bin/activate
pip install -e ".[dev]"
# frontend
cd frontend && npm install
```

**Run:**
```
cd backend && uvicorn app.main:app --reload
cd frontend && npm run dev
```

**Test / lint (must pass before any milestone is "done"):**
```
cd backend && pytest && ruff check . && black --check .
cd frontend && npm run test && npm run lint
```

**Conventions:**
- Type everything. Backend: full type hints + Pydantic. Frontend: no `any`.
- Comments and docstrings in a **direct, plain, experience-grounded voice** — explain *why*, not *what*. No filler.
- The optimizer stays pure and has the highest test coverage in the repo. Write the tests first.
- Small modules, clear names, no dead code, no commented-out blocks.
- Conventional Commits (`feat:`, `fix:`, `test:`, `chore:`).
- Never invent availability or price data. All of it flows through the adapter or seed files.
- Handle every external call with timeouts and a fixture fallback in dev.

## 12. Build order (milestones)

- **M0 — Scaffold:** monorepo, `/api/health`, services seed catalog endpoint, CI running lint+test. App boots with zero API keys.
- **M1 — Watchlist:** `FixtureProvider`, title search, watchlist CRUD, availability stored. UI to add/see titles.
- **M2 — Optimizer:** pure `Optimizer.plan(...)` + thorough unit tests against fixtures. No UI yet.
- **M3 — Plan end-to-end:** `/api/plan/generate`, the timeline plan screen, the savings hero number. This is the demoable core.
- **M4 — Showcase polish:** landing page, one-click demo seed, footer attribution, README + deploy config.
- **M5 (stretch):** `WatchmodeProvider` live wiring, calendar export for cancel/subscribe reminders.

## 13. Definition of done (v1 / showcase)

A reviewer opens the deployed URL, clicks "Try the demo," and immediately sees a 6-month rotation plan with a believable annual savings number, month-by-month service picks, and plain-language reasoning — built entirely on seed data, with the live Watchmode path available behind an API key. Lint and tests green. README explains the idea, the data approach, and the known risks in §2.
