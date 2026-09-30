# StarNet — Code Quality Review

**Date:** 30 September 2026 · **Revision:** `feat/harness-backend` @ `ab86dd0c` (v0.12.5)
**Question this report answers:** *How easy is this codebase to understand, run, change and trust, and
what will make it harder over the next year if nothing changes?*

Line-level bugs are in the **Coding Review**; risks to people and data are in the **Security Review**;
the ordered plan is in **Improvements**.

---

## 1. The short version

StarNet is a **large, ambitious and unusually careful** codebase. Its developers clearly think hard
about failure: the code is full of fail-closed defaults, durable writes, recovery paths, and comments
explaining *why* a line exists and which incident it came from. The test suite is big (over a thousand
test files) and runs on every pull request. For a project of this scope, that's rare and valuable.

Its problems are the problems of **growth without pruning**:

* One server file, `sidecar/index.js`, has reached **22,756 lines**, with 307 HTTP routes and about 780
  top-level functions. It can't be loaded by a test, so everything in it can only be tested by starting a
  whole server.
* The browser UI is **167,000 lines of JavaScript loaded by 233 `<script>` tags**, with no module
  system, no bundler, no linter and no type checking. Several UI files are 9,000–12,000 lines each.
* The repository is **heavy and duplicated**: a 1.6 GB git history, 1.2 GB of art assets committed
  **twice** (once under `frontend/`, again under `website/app/`), large near-copies of source files, and
  159 MB of study output.
* **Errors are often swallowed**: 776 empty `catch` blocks in the server code, 347 of them in
  `index.js`. Most are deliberate "this must never break the main path" choices, but together they make
  real failures quiet.
* **Comments carry the project's history** (dates, names, incident notes, "rulings") rather than just
  explaining the current code, which makes files longer and slower to read.

None of this is urgent in the way a security hole is. But each item makes the next change slower and
riskier, and they compound. The Improvements report orders the fixes so that each makes the next easier.

---

## 2. Scorecard

| Area | Grade | One-line reason |
|---|---|---|
| Correctness mindset | **A** | Fail-closed defaults, durable writes, explicit threat models, recovery modes |
| Testing | **B+** | 1,109 test files, CI on every PR; but the core server can't be unit-tested |
| Security engineering | **A−** | See Security Review; strong HTTP and network layers |
| Structure / modularity (server) | **C** | Good small modules around one 22,756-line file |
| Structure / modularity (UI) | **C−** | Global scripts, several 10k-line files, near-duplicate copies |
| Error handling | **C+** | Careful in critical paths; 776 silent catches elsewhere |
| Tooling (lint, types, format) | **D** | None configured |
| Repository hygiene | **D** | 1.6 GB history; assets and code committed twice |
| Documentation | **B−** | Plenty of it (218 docs files), hard to find the current truth |
| Dependencies | **A** | 6 direct runtime deps, 129 packages total, one moderate advisory |

---

## 3. Structure

### 3.1 The server: good modules around one giant file

`sidecar/` has 321 JavaScript files and 104,669 lines. Most files are well-sized (a few hundred lines),
named for what they do (`permissions.js`, `pathtrust.js`, `apitickets.js`, `durable-write.js`), and
several are written as **pure functions with injected dependencies**, so they can be tested without
a real clock, network or disk. `permissions.js` and `apiauth.js` are good examples; `apiauth.js` even
explains in its header that it was pulled out of `index.js` *because* `index.js` can't be loaded in a test.

Then there's `sidecar/index.js`, at **22,756 lines**, a fifth of all server code. It contains:

* process start-up and 85 environment settings, read through an `ENV()` helper that checks both
  `STARNET_` and legacy `SKYNET_` names (`index.js:368`);
* the HTTP server, the request gate, and a single **307-entry route table** (`ROUTES`, lines 9615–9999)
  whose handlers are mostly defined in the same file;
* messaging channel wiring (Telegram, Slack, Discord, Matrix, Signal), cron/routine execution, the
  workflow "conveyor", step tests, updates, recovery mode, credits, voice, Spotify, the consent
  broker set-up, tool resolution, and much more.

**Why this matters, concretely:**

* **It can't be tested directly.** `index.js` starts a server as soon as it's loaded. Any logic
  inside it can only be tested by launching the whole process and making HTTP calls, which is slow
  (the fast gate takes about 4 minutes on CI; one claims test we ran took ~190 seconds) and makes
  failures hard to pinpoint. The project's own answer, pulling logic out into pure modules, is the right
  one; it just needs to keep going.
* **It's a merge-conflict magnet.** Almost every feature touches it, so parallel work collides.
* **Understanding one feature means reading many distant parts of one file.** For example, the
  Telegram owner-trust behaviour described in Security S1 is spread across lines 8565, 15505–15521,
  16270–16325, 17124–17131 and 17330.
* **Load time and memory**: every start parses the whole file and builds every feature, even ones the
  user never enables.

**What good looks like:** `index.js` shrinks to start-up plus wiring, a few hundred lines. Each area
(channels, routines, conveyor, updates, recovery, credits, voice) becomes a folder with a `routes.js`
that registers its own routes and a `service.js` with its logic, taking its dependencies as arguments.
The route table stays one list, but it's assembled from those modules. See Improvements.

### 3.2 The UI: global scripts and very large files

`frontend/` has 268 JavaScript files and **167,024 lines**. The page `frontend/index.html` loads them
with **233 `<script>` tags**, none of them modules. Files talk to each other through globals
(`World`, `Harness`, `U`, `H`, `SK.*`). The largest files:

| File | Lines |
|---|---|
| `frontend/app/propsprites.js` | 11,903 |
| `frontend/agent-demo/propsprites.js` | 11,866 |
| `frontend/app/world.js` | 10,967 |
| `frontend/agent-demo/review-world.js` | 10,059 |
| `frontend/app/stationui.js` | 9,875 |
| `frontend/app/chat.js` | 9,250 |
| `frontend/app/prop-catalog-data.js` | 6,090 |
| `frontend/app/build.js` | 5,999 |
| `frontend/app/stationbake.js` | 5,486 |
| `frontend/agent-demo/stationbake.js` | 5,367 |
| `frontend/app/app.js` | 5,326 |

**Why this matters:**

* **Load order is the dependency system.** With 233 scripts, a file that uses a global before its script
  has run fails, often silently thanks to `try/catch`. Reordering is risky and nothing checks it.
* **No dead-code detection.** Without imports, no tool can tell you a function is unused.
* **Data shape mismatches go unnoticed.** Coding Review B1 is a real example: `World.heroCaps()` returns
  `[{objectType}]`, but two callers in `app.js` treat the result as strings or `{id,label}`. The result is
  "undefined, undefined, …" shown to the user and a permission check that's always false. A type checker
  (even plain JSDoc with `// @ts-check`) would have flagged both.
* **Performance.** The browser makes hundreds of requests and parses 167k lines before the app is
  usable. It works on a fast machine; it's heavy for older laptops.

### 3.3 Duplicated code

* `frontend/agent-demo/` contains near-copies of main UI files: `propsprites.js` (11,866 vs 11,903
  lines, 119 differing lines) and `stationbake.js` (5,367 vs 5,486, 271 differing lines). Bug fixes to
  one don't reach the other.
* `website/app/` is a **committed copy of `frontend/`** (17,022 tracked files). `diff -rq` finds only three
  differences (`index.html`, plus `demo-boot.js` and `demo.css` only in the website). A script
  (`npm run sync:website`) keeps them aligned, but the copy doubles repository size (see 3.4) and
  means every change shows up twice in history.
* The HTML-escaping helper is written out separately in at least four UI files (`cratecard.js:18`,
  `deliverables.js:8`, `workflowpanel.js:34` fallback, `updates.js:17` fallback), next to the shared
  `U.esc`. Security-relevant code should exist once (see Security S6).

### 3.4 Repository size

| Folder | Tracked files | On disk |
|---|---|---|
| `website/` | 17,081 | 1.3 GB |
| `frontend/` | 17,020 | 1.3 GB (1.2 GB of it `assets/industrial/`) |
| `output/` | 10,087 | 159 MB |
| `docs/` | 1,788 | 533 MB |
| `qa/` | 901 | 57 MB |
| **`.git` history** | — | **1.6 GB** |

`frontend/assets/` holds 16,565 PNG files. Every contributor, every CI run (the fast gate uses
`fetch-depth: 0`, the full history) and every release build downloads all of it.

**Why it matters:** slow clones, slow CI, and a practical barrier to outside contributors. It's also
hard to know which assets the app actually uses.

**What to do:** keep the site copy out of git (generate it in the deploy step), move large binary
assets to Git LFS or a release download, move `output/` studies out of the main repo, and find
unreferenced assets with a script that scans code for asset paths. History rewriting is a separate
decision (see Improvements).

---

## 4. Error handling

The server code has **776 empty `catch` blocks** (`catch (_) {}` and similar); `index.js` alone has 347.

It's important to be fair here: many are **deliberate**. The comments explain that a telemetry write,
a notification or a quest update must never break a user's run ("fire-and-forget + fail-open"). And the
critical paths do the opposite: the consent broker fails closed, credential writes fail closed and
read back to verify, and a central route guard turns thrown errors into proper 500 responses instead of
hanging sockets (`index.js:9383-9394`).

The trouble is the *default*. When swallowing is the easy, common pattern:

* **Real bugs hide.** Coding Review B7 (the Telegram "CONFIRM FORGET" button that sends nothing) and B5
  (the first chat message lost during start-up) are the kind of problem that a swallowed error produces:
  no crash, no log, just nothing happening.
* **You can't tell how often fallbacks happen.** The project already has the right tool:
  `sidecar/failopen.js` provides `swallow('tag')` and `note()` (imported into `index.js` as
  `failNote`), which keep the fail-open behaviour but log a throttled warning and count each tag for
  the diagnostics page. Its own header explains why it exists: *"reflection was silently dead on
  trunk for weeks because its caller's bare empty catch handler hid a 100% failure rate."* That's
  exactly the risk, already paid for once. The helper is used in many places, but the 776 bare
  catches show the old habit is still the default.

**What to do:** make "swallow silently" impossible to write by accident. Replace bare empty catches with
`swallow('area.what')`, which counts and logs at debug level, then add a lint rule that forbids empty
catch blocks. Surface the counts in the existing diagnostics page. This keeps the fail-open behaviour
and adds visibility.

---

## 5. Testing and CI

**Strengths.** 1,109 test files under `test/`, using Node's built-in test runner (no framework to
maintain). Two curated lists: `test/fast.list` (954 entries) and `test/http.list` (157). CI
(`.github/workflows/fast-gate.yml`) runs the fast gate on every pull request and every push to the
default branch, with a real Chrome for the end-to-end tests. There are separate workflows for
evaluation gates, secret scanning, desktop builds, soak tests and release. Tests are clearly written to
prove specific incidents stay fixed.

**Gaps:**

1. **The biggest file is only testable end-to-end** (see 3.1). This makes tests slow and coarse, and
   pushes people toward "claims" tests that check documents and outputs rather than behaviour.
2. **Several bugs we found by using the app are not caught by the suite.** B1 (capabilities shown as
   "undefined"), B2 (browser-mode keys invisible to step tests), B3/B4 (a new user locked in recovery
   mode; "Start fresh" leaving a dead server in browser mode) and B7 (a button that sends no request) are
   all in paths between UI and server. That suggests the UI tests check that things render, not that
   they call the right API with the right data.
3. **Browser/source mode is under-tested compared to desktop mode.** Several bugs (B2, B4, S5) only exist
   in browser mode.
4. **Speed.** About 4 minutes on CI for the fast gate is acceptable, but locally the suite is long
   enough that people will run subsets. Faster unit tests (from 3.1) help here.

**What to add:** unit tests for logic as it's extracted from `index.js`; a small set of "journey" tests
that click through the UI in browser mode and assert on the network requests made (the project
already has Chrome-driven tests to build on); and a test for each bug in the Coding Review.

---

## 6. Tooling

There is **no linter, no formatter and no type checking** configured (no ESLint, Prettier,
`tsconfig`/`jsconfig`, or `@ts-check` comments). The only lint-like script is
`scripts/lint-evidence-secrets.mjs`.

For a 270,000-line JavaScript codebase this is the cheapest big improvement available:

* **ESLint** with a small rule set (no-undef, no-unused-vars, no-empty with `allowEmptyCatch: false`,
  eqeqeq, no-implicit-globals for new files) would catch whole classes of the bugs above.
* **TypeScript's checker on plain JavaScript** (`checkJs` in a `jsconfig.json`, or `// @ts-check` per
  file) needs no build step and no rewrite. Start with the pure modules, which already have
  well-documented shapes in comments; turn those comments into JSDoc types.
* **Prettier** (or any formatter) ends formatting debates and makes diffs smaller. Many lines in
  `index.js` are 200+ characters, which makes review harder.

---

## 7. Comments and documentation

### 7.1 Comments

`index.js` has about 4,300 comment lines (roughly one in five), 176 of them containing a date. Many
comments are excellent: they explain threat models, invariants and why a check exists. Others read
like a changelog or meeting notes ("Andrew's ruling 2026-09-22", "hardened after code-review
2026-07-12", "used to … now …").

History is valuable, but **in the code it has costs**: files get longer, the current rule gets buried
under how it evolved, and comments drift out of date (Security S4 is one example: the header says the
floor is "unreachable past any flag" while the code lets Full Power past it).

**Suggested rule:** comments state what's true *now* and why; history goes in the commit message, and
decisions go in `docs/DECISIONS.md` (which already exists). A comment can link to a decision entry.

### 7.2 Documents

`docs/` has 218 files, many of them plans, status memos and dated audits (`AUDIT_0.11.1_FOR_0.11.2.md`,
`FABLE_MARCHING_ORDERS_2026-07-03.md`, many `*_PLAN.md`). `CODE_MAP.md` (94 lines) is a good start as an
entry point.

For a newcomer it's hard to tell which documents describe the current system and which are
historical. **Suggested:** a `docs/README.md` that lists current reference docs; move finished plans
and dated memos into `docs/archive/`; mark each plan's status at the top.

### 7.3 Naming

The project was renamed from Skynet to StarNet, and both names remain: `SKYNET_*` environment variables
are still accepted (78 distinct `SKYNET_` names appear in `sidecar/`, next to 90 `STARNET_` names), the
`x-skynet-token` header is accepted, and variables like `SKYNET_CHROME` appear in CI. Accepting old
names for compatibility is kind to users; the cost is that every setting has two names. Set a removal
date, log a deprecation warning when an old name is used, and use only new names in docs and CI.

---

## 8. Configuration surface

* **136 npm scripts** in `package.json`. Many are one-off proofs (`phase4:ui-proof`,
  `t1:signing:public`, `phase5:evidence:init:decision`). A newcomer running `npm run` sees a wall. Group
  them: keep the ten or so everyday ones in `package.json` and move the rest behind a single
  `npm run task -- <name>` runner with its own help text.
* **Around 85 environment settings** read in `index.js` alone. There's no single list of them with
  defaults and meanings. Generate one (a script can extract every `ENV('…')` call) and publish it
  in `INSTALL.md`.

---

## 9. Dependencies

This is a clear strength. Six direct runtime dependencies (`@huggingface/transformers`, `ajv`,
`kokoro-js`, `node-pty`, `ogg-opus-decoder`, `undici`), two dev dependencies, and 129 packages in the
lock file. Security-sensitive things (HTTP, crypto, the consent broker) are written in-house on Node's
standard library. `npm audit` shows one moderate advisory (Security S11). The Rust side has a normal
`Cargo.lock`.

The only suggestion is to run `npm audit` in CI and let Dependabot open PRs for both npm and Cargo.

---

## 10. Summary

The *thinking* in StarNet is high quality: the safety model, the failure handling, the tests. What
needs attention is the *shape*: one file that everything passes through, a UI without modules or
checks, a repository carrying its assets twice, and a habit of silent catches and history-laden
comments. Each is fixable in steps, without a rewrite, and the most valuable first step is the
cheapest: turn on a linter and a type checker for the pure modules, then keep extracting logic out of
`index.js`.
