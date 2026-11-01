# Handoff: route tables and Bun.serve routes

- Repo: `maxschuetz06/floorplan` (fork of `haydenschmitty/floorplan`), worktree `/Users/maximilianschuetz/Projects/floorplan`
- Branch: `codex/bun-route-tables`, base `main` @ `cec1075` (= upstream main), date 2026-11-01
- PR target: the fork's own `main`. Not upstream without Max's explicit OK. Drop this handoff file before any upstream PR.
- Skills: architect-loop (entry); `$deslop` (builder, own diff); `$test-audit` (new test file). No `$qa`: no UI change (the map viewer is unchanged; only scan data grows). No migrations.

## Command profile
- Tests (clean env required; Codex env vars change agent detection and make `tests/blueprint.test.mjs` fail/hang):
  `env -i PATH="$PATH" HOME="$HOME" TMPDIR="$TMPDIR" node --test <files>`
- Full suite: same with no file list (slow locally, several minutes).
- No lint, typecheck or build step exists (plain Node ESM, zero dependencies, Node ≥ 18).
- Real-project check (read-only, output outside the project):
  `node skills/floorplan/scripts/scan.mjs /Users/maximilianschuetz/.codex/worktrees/8fbe/praxis-ki --out <tmpdir>/floorplan/.cache/scan.json`
  Baseline @cec1075: 23.5 s, routes 0, http 0.

## Goal
Floorplan recognises HTTP endpoints declared (a) as route-table object literals and (b) as `Bun.serve({ routes: { "/path": … } })` keys, so frontend requests are matched to them.

## Frozen design
1. New module `skills/floorplan/scripts/lib/route-tables.mjs` holds all new detection logic. It exports one function used by `extractJs` in `lib/extract.mjs`: given the comment-stripped source, the `at` line-index function and the file text, it returns route records in the existing shape `{ method, path, var: null, line, framework, handler? }`. `extract.mjs` only gets an import and a call that appends the records to `routes`. `extract.mjs` (598 lines) must not otherwise grow.
2. **Route-table entries** (`framework: 'route-table'`): an object literal is a route when it has all of
   - `method: "<VERB>"` with a string literal verb in GET|POST|PUT|PATCH|DELETE|HEAD|OPTIONS (any case of quotes: `"`, `'`, `` ` `` without `${`);
   - `path:` whose value is a string literal starting with `/`, or a regex literal;
   - a handler key `handle|handler|fetch|action|resolve` (value: any expression).
   Keys may appear in any order within the same object. Objects lacking the handler key (e.g. client request descriptors `{ method: "POST", path: "/x" }`) are not routes. `path:` values that are identifiers, calls or `new RegExp(...)` are skipped silently.
   `line` = the line of the `path:` key. `handler` = for `handle: name` the name; for an arrow/function value, the first called identifier in its body (`=> handleX(...)` → `handleX`); otherwise omitted.
3. **Regex path → route path** (pure helper, unit-tested): strip leading `^` and trailing `$`/`\/?$`; drop lookarounds `(?!…)`, `(?=…)`, `(?<!…)`, `(?<=…)`; unescape `\/`, `\.`, `\-`; a non-capturing group whose alternatives are all plain literals without `/` (`(?:mails|posteingang)`) expands into one route per alternative (cap 8 combined expansions per regex; beyond the cap, that group becomes a wildcard segment); a capturing group or `[^/]+`/`[^/]*`/`\w+`/`\d+`/`[...]+` style class becomes `:p1`, `:p2`… (1-based capture order); `.*`/`.+` or any group that can span `/` becomes `*`. If any remaining segment still contains regex syntax, that segment becomes `*`. Discard the result when it does not start with `/` or has no literal segment of ≥ 2 letters. Never throw on odd input.
4. **Bun.serve literal keys** (`framework: 'bun'`), only in files containing `Bun.serve(`: inside a `routes: {` object, a string-literal key starting with `/` whose value is an object with method keys (GET|POST|PUT|PATCH|DELETE|HEAD|OPTIONS) yields one route per method key; any other value (function, `new Response`, `Response.json`, `Bun.file`, identifier) yields one `ANY` route. Bun's `:param` and `*` syntax stays as is (route matching already understands both). Handler as in 2 where cheap; otherwise omitted.
5. `addDocs` in `extract.mjs` must not override the handler of these routes with its "first declaration after the route" heuristic: skip both frameworks there (one condition), keeping an explicit `handler` from step 2/4.
6. **Health:** in `lib/health.mjs` route findings, return early for `framework` `route-table` and `bun` (no finding → "framework we don't analyse: say nothing"). Rationale: such projects authenticate in wrappers the scanner cannot see; claiming "no login check" would be a false alarm on every endpoint. One added condition; no other health change.
7. Performance: a cheap pre-test (e.g. `/\bmethod\s*:\s*["'`]/` and `/\bpath\s*:/`, or `Bun.serve(`) before any per-object parsing; the praxis-ki scan must stay within +25 % of the 23.5 s baseline.

## Ownership (single builder; work is tightly coupled)
- New: `skills/floorplan/scripts/lib/route-tables.mjs`, `tests/route-tables.test.mjs`, `tests/fixtures/route-tables/**`.
- Minimal hook edits: `skills/floorplan/scripts/lib/extract.mjs`, `skills/floorplan/scripts/lib/health.mjs`.
- Docs: one README line (the **Endpoints:** list gains "Bun.serve routes, route tables (`{ method, path, handle }`)"), one `CHANGELOG.md` line under a new `## Unreleased` heading at the top (match the file's existing style).
- Architect: this handoff file.

## Acceptance criteria
- AC1 Route-table objects (string and regex paths, both key orders, multi-line) become routes with correct method, path, line, framework, handler.
- AC2 Regex conversion covers: `^\/api\/items\/([^/]+)\/content$` → `/api/items/:p1/content`; `^\/api\/rules\/(?!check$)([^/]+)$` → `/api/rules/:p1`; `^\/api\/(?:mails|inbox)\/([^/]+)\/files$` → two routes; `^\/api\/templates\/([^/]+)\/(?:reset-to-defaults|reset)$` → two routes; `^\/files\/(.+)$` → `/files/*`; garbage input → no route, no throw.
- AC3 Objects with method + path but no handler key yield no route; identifier / `new RegExp` paths yield no route and no error.
- AC4 `Bun.serve` literal keys: method objects → one route per method; function/Response values → `ANY`; ignored in files without `Bun.serve(`.
- AC5 End-to-end fixture (generic names, a small "shop"-like app: a server file with a route table, a Bun.serve file, a frontend file calling the endpoints with `fetch` and a `request()` helper including template-literal paths): after scan + build (`tests/helpers.mjs` style), `graph.routes` contains the routes and the frontend file is linked to the route files by HTTP edges.
- AC6 Health produces no `unprotected-endpoint` issue for the fixture's endpoints.
- AC7 Existing tests unchanged and passing in the clean env; new tests fail on base `cec1075`.
- AC8 Zero dependencies, Node 18 compatible, no version bump, `CHECKSUMS.txt` untouched.
- AC9 (architect-verified) praxis-ki scan: ≥ 800 routes, ≥ 300 matched call→route pairs, time ≤ 30 s.

## Anti-goals
- No auth/guard inference for the new frameworks; no screen/router detection; no `url.pathname === "/x"` dispatch detection; no resolution of identifier or `new RegExp` paths; no viewer/UI changes; no refactor of existing detectors; no version/CHECKSUMS changes; nothing pushed upstream.

## Standing orders (restate in every spawn and send_input)
- You are not alone in this repo; preserve others' edits; edit only owned files.
- Fixtures and tests use generic example names only; never copy code or names from praxis-ki or any private project; never print secret values.
- praxis-ki may only be read and scanned with `--out` pointing outside it; never write into it.
- No push, no PR, no GitHub writes; commit locally on `codex/bun-route-tables` (Conventional Commits) only.
- Run tests only in the clean env shown above; keep every test invocation under 20 minutes.
- Follow `/Users/maximilianschuetz/.codex/skills/architect-loop/references/engineering-standards.md` (adapted: plain JS ESM, JSDoc where helpful; upstream file-size habits allowed for existing files, new module ≤ 500 lines, functions ≤ 80 lines) and `CONTRIBUTING.md` / `docs/development.md`.

## Results log
- Baseline: `node --test` in Codex env: `tests/blueprint.test.mjs` fails test 1 and stalls; clean env: passes (runs slowly). Full clean-env baseline run: see below.
- Builder (2026-11-01): new tests red on base `cec1075` (6/8 fail with a stubbed helper import; the 2 passing are false-positive/never-throw guards, mutation-checked), green after (8/8). Full clean-env suite: 153 tests, 152 pass, 0 fail, 1 skip (~3.5 s here). praxis-ki scan: 24.7 s (base on same machine 25.4 s), routes 858, http pairs 2261 (947 from non-test files), praxis-ki worktree unchanged. `dataRoutes(code, at)` takes two args (the file text was not needed).
- Builder round 2 (2026-11-01): B1 (health filter where `jsRoutes` takes `f.routes`), design 3a, Bun generic pre-test, CHANGELOG period. New/changed tests red on `5e96487` (4/9 fail incl. the mixed-file Health test reporting `GET /api/admin/stats` open), green after (9/9); mutation checks on with/without expansion, cross-branch cap, syntax discard, anchors, trailing `\/?` and the health filter each fail a test. Also discards a capture inside a word (`item-(\d+)`) instead of `*`. Full suite 154 / 153 pass / 0 fail / 1 skip. `wild.mjs`: only `GET /api/items`, no http link. praxis-ki scan 24.5 s, routes 858, http 2261 (947 non-test), 0 wildcard routes; praxis-ki unchanged.

## Finding ledger
Round 1 = architect inspection + independent reviewer (`/tmp/floorplan-review-r1.md`) on `5e96487`.

| ID | Class | Evidence | Decision |
|---|---|---|---|
| B1 | BLOCKER | `/tmp/fprev-exp/mixed.mjs`: base `open: []`, head `open: ["GET /api/admin/stats"]`; data routes in the shared `routes` array shrink the next-route window at `health.mjs:334` | Fix: filter the two frameworks out where `jsRoutes` takes `f.routes` (`health.mjs:291`), drop the in-loop return; regression test with the mixed file |
| R1-N1 | BLOCKER (architect, spec gap) | `/tmp/fprev-exp/wild.mjs`: `^\/api\/items(?:\/)?$` → `GET /api/*`, links unrelated `/api/orders` calls | Spec amended (design 3a). Fix |
| R1-N2 | NON_BLOCKING → fix with N1 | top-level `a|b` keeps only first branch, widened | Covered by design 3a |
| R1-N4a | NON_BLOCKING → cheap fix | `Bun.serve<T>(` not detected | Fix pre-test regex |
| R1-N3, N4b, N5, N6, N7, N8-CHANGELOG | NON_BLOCKING | report | N8 CHANGELOG nit: fix (period). Rest: ledgered, not fixed |
| Server→server links | NON_BLOCKING | architect scan: 21 of 2261 http pairs from route files | Ledgered |
| OOS-1..3 | OUT_OF_SCOPE | report | — |

### Design 3a (amendment, frozen for round 2)
- Split a regex source on top-level `|` first; strip `^`/`$` per branch; each branch converts independently (expansion cap 8 per whole regex).
- A `?`-quantified group of plain literals (`(?:\/)?`, `(?:\/edit)?`) expands into "with" and "without" variants (counts toward the cap). A trailing optional `\/?` is dropped as before.
- A segment that would become `*` only because of leftover regex syntax (not a capture or class that can span `/`) discards that variant instead of widening it. `*` survives only from `.*`/`.+` or a spanning capture group.
- Bun pre-test: `/\bBun\.serve\s*(?:<[^>]*>)?\s*\(/`.
