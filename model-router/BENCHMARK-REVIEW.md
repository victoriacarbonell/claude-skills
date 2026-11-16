# model-router benchmark: quality review evidence

This is the report written by the model session that reviewed four documentation sets without knowing how each was produced. It is reproduced as written, with local paths shortened. The sets were shuffled under neutral labels:

| Label | Run in [BENCHMARK.md](BENCHMARK.md) |
| --- | --- |
| P | A |
| K | B2 |
| T | F1 |
| M | F2 |

---

# Review of documentation sets K, M, P, T

All checks were applied identically to the four sets, by reading the code only (nothing was installed or run). Line numbers refer to files under `code`.

## 0. Ground truth

### Endpoints exposed by the server (40)

`app.ts:32` mounts the router at `/api`; `routes/index.ts:14-22` mounts the nine routers.

| Router (mount) | Endpoints |
|---|---|
| health (none) | `GET /api/healthz` |
| profile (`/profile`) | `GET /api/profile`, `PUT /api/profile`, `GET /api/profile/weight`, `POST /api/profile/weight`, `DELETE /api/profile/weight/:id` |
| foods (`/foods`) | `GET /api/foods/categories`, `GET /api/foods`, `POST /api/foods`, `GET /api/foods/:id`, `PUT /api/foods/:id`, `DELETE /api/foods/:id`, `GET /api/foods/:id/portions`, `POST /api/foods/:id/portions`, `DELETE /api/foods/portions/:id` |
| recipes (`/recipes`) | `GET /api/recipes`, `POST /api/recipes`, `GET /api/recipes/:id`, `PUT /api/recipes/:id`, `DELETE /api/recipes/:id` |
| logs (`/logs`) | `GET /api/logs/:date`, `POST /api/logs/:date/entries`, `PUT /api/logs/:date/entries/:entryId`, `DELETE /api/logs/:date/entries/:entryId`, `GET /api/logs/exercises/all`, `POST /api/logs/exercises`, `DELETE /api/logs/exercises/:id` |
| exercises (`/exercises`) | `GET /api/exercises`, `POST /api/exercises`, `DELETE /api/exercises/:id` |
| weight (`/weight`) | `GET /api/weight`, `POST /api/weight`, `DELETE /api/weight/:id` |
| dashboard (`/dashboard`) | `GET /api/dashboard/today`, `GET /api/dashboard/calendar/:year/:month`, `GET /api/dashboard/stats/weekly`, `GET /api/dashboard/stats/monthly` |
| ai (`/ai`) | `POST /api/ai/analyze-image`, `GET /api/ai/weekly-summary`, `GET /api/ai/monthly-summary` |

### Spec operations (34)

`lib/api-spec/openapi.yaml` has 34 `operationId`s. 33 have a code counterpart at the same path; `deletePortion` is at `/portions/{id}` (openapi.yaml:243) while the code serves `/foods/portions/:id` (foods.ts:111). Six code endpoints are absent from the spec (`/profile/weight` x3, `/logs/exercises` x3).

Other facts relied on below: Express `^5` with no error middleware (api-server/package.json, app.ts), so a thrown `ZodError` or DB error becomes a default-handler 500; `log_entries.food_id`/`recipe_id` are real FKs (lib/db/src/schema/logs.ts:11-12); `log_entries.date` and `exercises.date` are Postgres `date` columns (logs.ts:9, 23).

## 1. File coverage (35 in-scope files)

K 35/35, M 35/35, P 35/35, T 35/35. No missing docs in any set.

## 2. Line limit (10 lines)

No file doc exceeds 10 lines in any set. Line-count distribution: K 26x5 + 9x6; M 35x6; P 7-10 (one at 10); T 6-10 (four at 10).

## 3. Endpoint coverage

All four api-reference.md files document all 40 ground-truth endpoints and none that does not exist (checked by script against headings and tables, then by reading). K and P document the two duplicate groups (`/profile/weight`, `/logs/exercises`) as grouped entries that refer to the primary endpoints; M and T list them individually.

## 4. File-doc accuracy, 5 sampled files

Errors counted = statements the code contradicts. Omissions not counted.

| File | K | M | P | T |
|---|---|---|---|---|
| pages/Profile.tsx | 0 | 1 | 0 | 1 |
| schema/portions.ts | 0 | 0 | 0 | 1 |
| pages/Home.tsx | 0 | 0 | 0 | 0 |
| routes/ai.ts | 1 | 0 | 0 | 0 |
| components/BarcodeScanner.tsx | 0 | 0 | 0 | 0 |
| **Total** | **1** | **1** | **0** | **2** |

- **K, ai.ts doc:** "There is no caching, so every summary view is a paid LLM call." The server has no cache, but the page that views summaries sets `staleTime: 5 * 60 * 1000` (pages/AISummaries.tsx:29, 32), so repeat views within five minutes do not call the endpoint. Minor; K's own architecture.md states the staleTime correctly.
- **M, Profile.tsx doc:** "`calculatedCalorieGoal` is read via `(profile as any)`, suggesting it is missing from the generated type." The field is in the generated type (lib/api-client-react/src/generated/api.schemas.ts:51). Counted, but note it is phrased as an inference.
- **T, Profile.tsx doc:** "`(profile as any).calculatedCalorieGoal` (L112): field missing from the generated type." Same evidence (api.schemas.ts:51); stated as fact.
- **T, portions.ts doc:** "It is the only domain table without `created_at`." `recipe_ingredients` has none either (lib/db/src/schema/recipes.ts:20-25).

Checked and found correct (examples): K/P Home docs on the `{ date, id }` vs `entryId` bug (Home.tsx:28; api.ts:1778) and UTC vs local date; all four on the forbidden `User-Agent` header (BarcodeScanner.tsx:30); M/P/T on the 8-day window in `getRecentData(7)` (ai.ts:75); T's line references in the five docs (all verified).

## 5. Endpoint accuracy, 5 sampled endpoints

`POST /api/ai/analyze-image`, `POST /api/recipes`, `GET /api/recipes`, `DELETE /api/recipes/:id`, `DELETE /api/logs/:date/entries/:entryId` compared with ai.ts:10-70, recipes.ts:16-63 and 121-125, logs.ts:158-162.

Factual errors found: **K 0, M 0, P 0, T 0.** All four give the right method, path, body, status codes, the `500 {error, raw}` case, the empty-`ingredients` 500 with a leftover recipe row, and the ignored `date` on entry delete.

Errors seen in api-reference.md outside the fixed sample (reported, not counted in the sampled number):
- **K** (line 139): a bare `GET /api/logs/exercises` "returns an empty `DayLog` for the date `"exercises"`". It reaches `eq(logEntriesTable.date, 'exercises')` on a `date` column (logs.ts:24) and fails with 500; K's own line 115 says invalid dates give 500.
- **P** (lines 99 and 247): `PUT /api/foods/:id` with an unknown id returns "`200 {"category":"otros"}`". `food` is `undefined` and `food.category` throws (foods.ts:88-89), so the result is 500.

## 6. Spec mismatches

Classification used for every claim, in every set:
- **Real:** the code fact and the spec fact are both accurate, and either the spec states something the code contradicts, or the code exposes an HTTP-contract element (endpoint, parameter, content type, status code, response body) that the spec lacks.
- **Not a divergence (unsupported):** the facts are accurate but code and spec agree or the spec is simply silent on internal semantics (several such items say so themselves).
- **False:** a stated code or spec fact is wrong.

`mismatches_false` in scores.json = unsupported + false; the split is given in two extra fields.

| Set | Claimed | Real | Not a divergence | False | Missed from union (22) |
|---|---|---|---|---|---|
| K | 18 | 13 | 5 | 0 | 7 |
| M | 25 | 19 | 6 | 0 | 2 |
| P | 21 | 14 | 6 | 1 | 7 |
| T | 28 | 27 | 1 | 0 | 0 |

Raw counts are affected by granularity: T lists the six undocumented endpoints as six claims (K, M and P use two), and splits the 404-body and integer items in two.

### Per-set detail

**K** (items 1-18). Real: 1-10, 12, 14, 15. Not a divergence: 11 (category labels vs normalised values; spec only says `string`), 13 (unreachable optional-ingredients branch; spec and validator both require `ingredients`), 16 (fallback omits `score`; `score` is not in `required`, openapi.yaml:1294), 17 (recipe ingredient `food` not normalised; also its aside "unlike every other `Food` response" is wrong, since log entries embed raw foods too, logs.ts:31), 18 (enum vs free-text DB column; the item itself says it is unreachable through the API).

**M** (items 1-25). Real: 1-16, 19, 20, 25 (25 repeats the invalid-date case of 16). Not a divergence: 17 (category labels), 18 (nested food not normalised), 21 (AI goal vs dashboard goal; "the spec describes no goal logic"), 22 (omitted optional profile fields keep their value), 23 (deletes return 204 for unknown ids; the item says this matches the spec), 24 (`date` path parameter ignored).

**P** (items 1-21). Real: 1-9, 11, 13, 15, 16, 17. Not a divergence: 12 ("valid per the spec"), 14 (unreachable branch), 18 (0-kcal entry when neither id is sent; spec allows it), 19 (category labels), 20 (`Food.unit` free string), 21 ("that's not a mismatch with the code"). **False: 10**, "for unknown ids the code returns `200 {"category":"otros"}` (foods)": the handler throws at foods.ts:89 and returns 500. The `PUT /recipes` part of the same item (500) is correct and is credited in the union table.

**T** (items 1-28). Real: 1-26, 28. Not a divergence: 27 (`date` segment ignored; same treatment as M24). Spec and code line references in this section were spot-checked (openapi.yaml:243-256, 933, 1294; api.ts:300, 377, 410-412, 711-714; recipes.ts:58-60, 102-105) and are correct.

### Verified union of real mismatches (22) and coverage

Coverage is judged on each set's final mismatch section only.

| # | Mismatch | K | M | P | T |
|---|---|---|---|---|---|
| U1 | `deletePortion`: spec `/portions/{id}`, code `/foods/portions/:id` | y | y | y | y |
| U2 | `/profile/weight` GET/POST/DELETE not in spec | y | y | y | y |
| U3 | `/logs/exercises/all`, `/logs/exercises`, `/logs/exercises/:id` not in spec | y | y | y | y |
| U4 | `GET /profile` with no profile: spec 404, code 200 `null` (profile.ts:31) | y | y | y | y |
| U5 | `hasBarcode` query param not in spec (foods.ts:53) | y | y | y | y |
| U6 | `POST /ai/analyze-image` undocumented `500 {error, raw}` (ai.ts:68) | y | y | y | y |
| U7 | `POST /ai/analyze-image` 200 body not validated against `FoodAnalysisResult` | y | y | y | y |
| U8 | Validation failures return 500; spec documents no 4xx/5xx | y | y | y | y |
| U9 | `PUT /logs/{date}/entries/{entryId}` undocumented 404 (logs.ts:122) | y | y | y | y |
| U10 | 404 JSON body on `GET /foods/{id}`, `GET /recipes/{id}` not in spec | y | y | y | y |
| U11 | `PUT /foods/{id}` unknown id returns 500, no 404 (foods.ts:88-89) | y | y | **n** (described as 200) | y |
| U12 | `PUT /recipes/{id}` unknown id returns 500, no 404 | y | y | y | y |
| U13 | `GET /exercises` summary says "date range", code filters one date | y | y | y | y |
| U14 | `ingredients: []` allowed by spec, returns 500 after a partial write (POST and PUT) | y | y | **n** | y |
| U15 | Calendar `year`/`month` typed integer, never validated, bad values give 500 (dashboard.ts:66-67) | y | y | y | y |
| U16 | Spec `integer` fields validated as plain numbers (api-zod), fractions fail at the DB with 500 | **n** | y | y | y |
| U17 | `servings` required in spec (openapi.yaml:933), optional with default 1 in the validator (api.ts:300, 377) | **n** | y | **n** | y |
| U18 | Undocumented 500s from FK constraints (`DELETE /foods/{id}` in use by a recipe; unknown food/recipe ids on create) | **n** | y | **n** | y |
| U19 | Date strings not validated; invalid date gives 500 on logs/exercises | **n** | y | y | y |
| U20 | AI summary success body not guaranteed to match `AiSummary` (model JSON spread in, can drop required fields or override `period`, ai.ts:141-146) | **n** | y | **n** | y |
| U21 | Form-encoded bodies accepted (app.ts:30); spec declares only `application/json` | **n** | **n** | **n** | y |
| U22 | `nullable: true` used in an OpenAPI 3.1.0 document, where it is not a keyword, while the code returns `null` | **n** | **n** | **n** | y |
| | **Missed** | **7** | **2** | **7** | **0** |

K and P describe several of their "missed" items in the endpoint sections (for example the `DELETE /foods/{id}` 500, the empty-ingredients 500 in P, integer handling in K's conventions) but do not list them as mismatches.

## 7. Cross-file findings

All four issues are real:
1. Home.tsx:28 calls `deleteEntry.mutate({ date: today, id })`; the generated hook's variables are `{ date: string; entryId: number }` (api.ts:1778). The request goes to `/api/logs/<date>/entries/undefined` and fails validation (500).
2. CreateRecipeModal.tsx:43 always sends `ingredients: []`. recipes.ts:49-60 inserts the recipe row and then calls `.values([])` on the ingredient insert with no transaction. (That Drizzle throws on an empty `values()` was not executed here; all four sets assume the same.)
3. Spec and generated client use `/api/portions/${id}` (openapi.yaml:243; api.ts:977); the server route is `/api/foods/portions/:id` (foods.ts:111).
4. Server "today" is `new Date().toISOString().slice(0, 10)` (dashboard.ts:29, 108-109; ai.ts:76-77); the client uses `format(new Date(), "yyyy-MM-dd")` (Home.tsx:17, Scan.tsx:81, and four other places).

| Issue | K | M | P | T |
|---|---|---|---|---|
| (i) Home delete field name | yes (Home doc) | no | yes (Home doc, architecture) | no |
| (ii) Recipe modal sends `[]`, server fails | yes (modal doc, recipes doc, architecture) | partial | yes (modal doc, recipes doc, architecture) | partial |
| (iii) Delete-portion path | yes | yes | yes | yes |
| (iv) UTC vs local date | yes | yes (architecture, one line, AI window only) | yes | yes (architecture, dashboard doc) |
| **Found (explicit)** | **4** | **2** | **4** | **2** |
| Partial | 0 | 1 | 0 | 1 |

"Partial" for (ii): M and T state both halves in separate places (modal doc: always sends `[]`; api-reference: `[]` gives 500 with a leftover row) but never connect them, and their modal docs give the consequence as "recipes get no nutrition" (M) / "every recipe has 0 kcal/macros" (T), which hides that the request fails. scores.json counts only explicit findings in `cross_file_found` and reports partials separately.

## 8. Architecture documents

| Set | Lines | Four parts present | Factual errors | Score |
|---|---|---|---|---|
| K | 76 | yes | 0 | 5 |
| M | 63 | yes | 1 | 4 |
| P | 139 | yes | 0 | 5 |
| T | 161 | yes | 2 | 4 |

- **K (5):** correct and repository-specific; the photo flow matches Scan.tsx:47-129, ai.ts and logs.ts step by step; risks are concrete (UTC dates, base64 photos, error handling). Checked details hold (`limit: 100` in Foods.tsx:269, 5-minute staleTime, import-time throw in the OpenAI client). One overstatement, not counted: a recipe created without ingredients "stays at 0 kcal for good" (`PUT /recipes/:id` can recompute it, though the UI has no such screen).
- **M (4):** correct flow and sound decisions/risks, shortest of the four. Error: decision 2 says "an unknown `foodId`/`recipeId` quietly logs 0 kcal instead of failing"; the insert violates the FK (schema/logs.ts:11-12) and returns 500, as M's own api-reference states.
- **P (5):** correct and the most useful for onboarding: runtime topology verified against the artifact.toml files (port 8080, `/api/healthz`, static SPA with rewrite), numbered sequence diagram consistent with its own references, and the risks section names three concrete bugs with files (Home delete, Calendar.tsx:48-50 `proteinG`, recipe modal), all verified. Loose wording, not counted: catalogue rows described as "Comida analizada" rows (that string is only the last fallback name, Scan.tsx:90).
- **T (4):** the most detailed, with line references that were spot-checked and are accurate. Errors: (a) 3.2 says "if `foodId` doesn't resolve, the entry is silently stored with 0 kcal (`logs.ts:76-84`)"; same FK evidence as M, and T's own api-reference says 500; (b) 4.3 says `GET /api/foods` returns "up to 50 at a time"; 50 is only the default with no cap (foods.ts:59) and the app requests `limit: 100` (Foods.tsx:269).

## 9. Not verified, or outside the fixed sample

- Nothing was executed. Runtime behaviour taken on reading and general knowledge of the libraries: Drizzle throwing on `values([])`; Express 5 turning rejected handlers into 500; Postgres rejecting invalid `date` literals.
- M (api-reference, `POST /recipes`): "`servings: 0` ... is likely to fail at the DB (500)". Postgres `real` columns normally accept `NaN`/`Infinity`, so this is probably wrong, but it is hedged and could not be confirmed without a database. Not counted.
- T (BarcodeScanner doc): "`h-13` is not a default Tailwind class". The project pins Tailwind `^4.1.14` (pnpm-workspace.yaml:63), where numeric spacing utilities are generated on demand, so this is probably wrong. Framework behaviour, not confirmed by running a build. Not counted.
- K (api-reference AI header): "no ... retry". The handler has none; whether the OpenAI SDK retries by default was not checked.
- T architecture cites commit `fecb61c`; the review copy has no git metadata.
- Errors noticed in non-sampled file docs, not counted anywhere: M `api-routes-logs.md` ("if the food/recipe id is not found the entry is silently saved with 0 calories"; FK violation in fact) and T `api-server-routes-recipes.md` ("PUT on a missing id returns 200 with `undefined`"; 500 in fact, as T's api-reference says). Non-sampled file docs of K and P were not read in full, so this list is not a comparison.
- Check 6 depends on the real / not-a-divergence rule stated above. A looser rule that also counts "spec is silent" items as real would raise the real counts of K, M and P (by up to 5, 6 and 6) and T (by 1), and enlarge the union; only P item 10 is factually wrong under any rule.
