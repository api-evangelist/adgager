---
name: adgager-run-a-research-project
description: Scope, price, create, confirm and close a market-research study on the Adgager platform via its GraphQL API, using the real operation names in the published schema.
api: adgager-graphql
endpoint: https://api.adgager.com/graphql
transport: graphql
auth: Laravel Sanctum bearer token in the Authorization header
generated: '2026-09-07'
method: generated
source: graphql/adgager.graphql (live introspection, 2026-09-07)
operations:
  - login
  - sectors
  - cities
  - countries
  - upsertTargetFilter
  - createTargetGroup
  - addTargetGroupFromList
  - upsertQuestionGroup
  - projectCost
  - upsertProject
  - confirmProject
  - projectAnswers
  - projectScores
  - closeProject
---

# Run a research project on Adgager

Every operation named here exists in `graphql/adgager.graphql`, verified by live introspection of
`https://api.adgager.com/graphql` on 2026-09-07. Adgager publishes no developer documentation, so
the schema is the only contract — check a field signature there before you send it.

## Before you start

- **One endpoint.** Everything is `POST https://api.adgager.com/graphql` with
  `Content-Type: application/json`. There is no REST surface and no versioned path.
- **Get a token.** `mutation { login(input: {email: ..., password: ..., otp_code: ...}) { ... } }`
  returns a `LoginResponse`. Send the token back as `Authorization: Bearer <token>`. `otp_code` is
  optional, so expect a two-step login on accounts that have it enabled.
- **Errors arrive with HTTP 200.** Read `errors[]` in the body, not the status line. An expired or
  missing token comes back as `"Unauthenticated."` with `extensions.guards: ["sanctum"]`.
  See `errors/adgager-problem-types.yml`.
- **There is no idempotency key.** None of the 141 mutations accepts one. If a mutation's response
  is lost, do **not** blindly retry a create — read back with `getProjects` first and only then
  decide. The `upsert*` mutations are the safe ones: passing an existing `id` makes them replayable.
  See `conventions/adgager-conventions.yml`.

## 1. Build the audience

Reference vocabularies answer anonymously, so you can assemble a filter before you even log in:
`sectors`, `cities(filter:)`, `countries`, `districts`, `regions`, `positions`, `universities`.

Then, authenticated:

- `mutation { upsertTargetFilter(input: UpsertTargetFilterInput!) { id } }` — the demographic /
  behavioural filter. Inspect it with `targetFilters` and `targetFilterStat`.
- `mutation { createTargetGroup(input: CreateTargetGroupInput!) { id } }` — an explicit named panel.
  Two bulk loaders exist: `addTargetGroupFromList(list:, name:, brand_id:)` and
  `addTargetGroupFromFile`.

> `mergeTargetGroups` is destructive and the schema has no unmerge. Treat it as one-way.

## 2. Build the questionnaire

`mutation { upsertQuestionGroup(input: QuestionGroupInput!) { id } }`, with
`upsertGagerQuestion` and `upsertGagerQuestionOption` for individual items. Question payloads move
as `SurveyJsScalar` (SurveyJS JSON) — a vendor scalar, not a survey-interchange standard, so there
is no Triple-S or DDI import path (see `conformance/adgager-conformance.yml`).

## 3. Price it before you commit — this is the rehearsal step

`query { projectCost(id: <projectId>, filter: <TargetFilterScalar>) }` returns a `Float!`.
`projectTotal` and `projectGagerCount` size the reachable panel. Adgager has no general dry-run
flag, so **these three queries are the only preview you get**. Run them before `confirmProject`,
because confirmation spends brand credit and there is no refund or reversal mutation anywhere in
the schema (`spendBrandCredit` has no counterpart).

## 4. Create and confirm

1. `mutation { upsertProject(input: UpsertProjectInput!) { id status } }` — creates or updates.
   Idempotent when you pass an existing `id`.
2. `mutation { confirmProject(id:, status:, sendNewGagers:) { id status } }` — **the committing
   step.** Money-equivalent credit moves here.
3. `mutation { CloneProject(input: CloneProjectInput) { ... } }` to start from a previous wave.

Reversal: `closeProject(id:, status:)` and `deleteProject(id:)` exist, but **no window is stated
anywhere** — the provider publishes no policy. Do not tell a user how long they have to undo.

## 5. Read the results

- `query { projectAnswers(project_id:, filter: ProjectGagerFilter) { ... } }` — respondent-level.
- `query { projectScores(input: ProjectScoreInput!) }` — returns `JSONScalar`; the shape is not in
  the schema, so inspect a real response rather than assuming fields.
- `query { projectsForReport(filter:, limit:) { ... } }` and `getInsights(input:, first:, page:)`.

## Paging

Every paginated field requires `first: Int!` and takes `page: Int` (1-based). The envelope is
`{ paginatorInfo { count currentPage hasMorePages lastPage perPage total }, data { ... } }`.
There are no cursors. Loop on `hasMorePages`.

## Rate limits

`X-RateLimit-Limit: 250` is emitted on the API root but **not** on `/graphql` responses, so you get
no runtime budget signal on the calls that matter. Pace yourself conservatively and treat a sudden
non-JSON response as throttling. See `rate-limits/adgager-rate-limits.yml`.
