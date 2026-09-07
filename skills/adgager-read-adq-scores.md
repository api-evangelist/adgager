---
name: adgager-read-adq-scores
description: Read AdQ advertising-effectiveness scores, competitor comparisons and trends for a brand from the Adgager GraphQL API, and read the published AdQ plan catalogue without any credentials.
api: adgager-graphql
endpoint: https://api.adgager.com/graphql
transport: graphql
auth: Laravel Sanctum bearer token (the plan catalogue is anonymous)
generated: '2026-09-07'
method: generated
source: graphql/adgager.graphql (live introspection, 2026-09-07)
operations:
  - plans
  - login
  - adQBrandDashboard
  - brands
  - brand
  - comparison
  - comparisons
  - upsertComparison
  - getTrends
  - getTrendsByQuestion
  - getTrendsByMonthYear
  - getProjects
  - projectScores
---

# Read AdQ scores and comparisons

AdQ is Adgager's advertising-effectiveness product: Gagers rate an ad film 1–10 across a fixed
question set and the platform reduces that to a single AdQ score per brand, comparable against
category, competitor and market averages. Every operation below is verified against
`graphql/adgager.graphql`.

## No credentials needed: the plan catalogue

```graphql
query { plans { id code type plan_key plan_name description monthly_price yearly_price } }
```

Returns 8 rows anonymously — five `membership` tiers and three `report` tiers. **The `Plan` type has
no currency field**, so do not label these prices. The AdQ product page publishes a different ladder
(10,000–50,000 USD/month) and the provider does not reconcile the two; both are recorded in
`plans/adgager-plans-pricing.yml`. If a user asks what AdQ costs, present both and say they disagree.

## Authenticated: the brand dashboard

```graphql
query { adQBrandDashboard { name logo adq_score avg_adq_score category { ... } subcategory { ... } } }
```

`adq_score` is the brand's own score; `avg_adq_score` is the benchmark it is read against. It takes
no arguments — the brand is resolved from the token, so one token sees one brand.

## Comparisons

`comparisons` lists saved comparisons; `comparison(id:)` reads one; `upsertComparison(input:)`
creates or updates one; `deleteComparison(id:)` removes it. A `Comparison` is
`{ id, name, description, projects[], created_at }` — a named bundle of projects, so build the
comparison out of project ids you already resolved via `getProjects(is_adq: true, ...)`.

## Trends

- `getTrends(input: TrendsFilter, first: Int!, page: Int)`
- `getTrendsByQuestion(input: TrendsByQuestionFilter, first: Int!, page: Int)`
- `getTrendsByMonthYear(...)`

All three return a `ProjectPaginator!`, i.e. trends are expressed as ranked projects, not as a
time-series type. `first` is required.

## Per-project scoring detail

`projectScores(input: ProjectScoreInput!)` returns a bare `JSONScalar`. The underlying `ProjectScore`
type is `{ id, project_id, gender, age, scores: JSONScalar, created_at, updated_at }`, so scores are
broken out by gender and age band — but the inner `scores` shape is untyped. Inspect a live response;
do not assume keys.

## Cautions

- **Errors come back with HTTP 200.** Branch on `errors[]`, and remember that several failure modes
  arrive as *data* on typed response objects instead (`GenericResponse { status, message }`).
- **No deprecation signal.** Zero fields in this schema carry `@deprecated`, so the schema will never
  warn you before something changes. Re-introspect before a release rather than trusting a cached SDL.
- **`brand`, `brands`, `getBrands` and `brandsOfTheBrand` are different queries** with different
  scoping. `brands(first:, page:, filter:, orderBy:)` is the paginated admin-shaped one.
