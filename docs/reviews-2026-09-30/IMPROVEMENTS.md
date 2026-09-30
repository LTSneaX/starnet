# StarNet — Improvement Plan

**Date:** 30 September 2026 · **Revision:** `feat/harness-backend` @ `ab86dd0c` (v0.12.5)
**Question this report answers:** *Given everything found in the security, quality and coding reviews and in
hands-on use, what should be done, in what order, why that order, and how do you know each step is done?*

Each step says **what**, **why**, **where**, **how to know it's done**, and a rough **effort** for one
developer who knows the codebase. References like [S1], [C5] or [B2] point to the Security Review, the
Coding Review, and the hands-on bug list respectively.

---

## How the plan is ordered

StarNet's foundations are strong. The work splits into four phases:

1. **Phase 0: trust fixes (about a week).** Close the gaps where the product's safety rules are weaker
   than what it tells the user. These matter most because StarNet's whole value is "agents you can
   trust with your computer".
2. **Phase 1: first-hour experience (about a week).** Fix what a new user hits in their first hour:
   recovery-mode traps, browser-mode keys, silent buttons, invented facts.
3. **Phase 2: make change cheap (ongoing, 4–8 weeks in slices).** Linting and types, breaking up
   `index.js`, UI modules, repository diet. This is what keeps the next year of features from being
   slower than the last.
4. **Phase 3: stronger containment and polish.** A real sandbox, taint tracking, cost realism,
   documentation.

| Step | What | Phase | Effort | Fixes |
|---|---|---|---|---|
| 1 | Make messaging remote control an explicit opt-in | 0 | 1–2 days | S1 |
| 2 | Honest, narrower approvals for shell and folders | 0 | 3–4 days | S2, S3 |
| 3 | One truth for the floor under Full Power | 0 | half a day | S4, C13 |
| 4 | Browser mode: keys on the server, and a page CSP | 0/1 | 3–4 days | S5, S6, C2 |
| 5 | Fix the first-hour bugs | 1 | 3–4 days | C1, C3–C12, C17 |
| 6 | Turn on a linter and a type checker, starting small | 2 | 2–3 days to start | Quality §6 |
| 7 | Make silent failures visible | 2 | 2 days + ongoing | Quality §4 |
| 8 | Break up `sidecar/index.js` | 2 | 1 area per week | Quality §3.1 |
| 9 | Modules for the UI | 2 | 2–4 weeks | Quality §3.2 |
| 10 | Repository diet | 2 | 2–3 days (+ a decision) | Quality §3.4 |
| 11 | Browser-mode journeys in CI | 2 | 3–4 days | Quality §5 |
| 12 | A real sandbox for shell | 3 | 2–4 weeks | S2 |
| 13 | Taint tracking for runs that read outside content | 3 | 1 week | S8, S7 |
| 14 | Cost realism for custom and local models | 3 | 2–3 days | C16 |
| 15 | Documentation and configuration clean-up | 3 | 3–5 days | Quality §7–8 |

---

## Phase 0 — Trust fixes

### Step 1. Make messaging remote control an explicit opt-in [S1]

**What.** Today, pairing a messaging account gives that account approval-free control of the station:
shell, connectors and, on desktop, the mouse and keyboard, while the bot tells the owner it "silently
skips" such actions. Change this so that:

1. Pairing gives the ordinary behaviour: in owner DMs, actions that need permission **ask with buttons**
   (as `/approvals on` does today) or are skipped, never auto-allowed.
2. A new, clearly worded setting under Messaging, **"Remote control from my paired account"**, off by
   default, restores today's behaviour for users who want it.
3. Even with remote control on, shell commands, writes outside the agent's own workspace, sending or
   deleting through connectors, and the desktop lease always ask.
4. The `/approvals` message tells the truth for each case.
5. Every action taken via a channel appears in the station's activity log, and on desktop as a
   notification.

**Where.** `sidecar/index.js:17330` (the `bypass` predicate), `16270-16325` (owner trust → interactive
surface, workbench, desktop lease), `17124-17131`; `sidecar/channels/hub.js:1084-1110` (`/approvals`).

**Done when.** A test proves an owner DM with remote control off can't run `shell.exec` without a
button tap; the `/approvals` text matches behaviour in all four combinations (owner or not, approvals on
or off).

**Why first.** It's the one finding where someone who isn't at the computer can, with realistic
preconditions, run commands on it. The fix is small.

---

### Step 2. Honest, narrower approvals for shell and folders [S2, S3]

**What.**

* **Shell approval card:** the "Always" button says what it does ("Always allow *any* command for this
  agent"), with a line explaining that commands run with the user's full permissions. Add narrower
  choices: "Always allow `npm test`" (this exact command) and "Always allow `git …`" (this program).
  Store them as keys like `workbench:execute:git`. The broker already keys grants by class; add an
  optional more specific key that is checked first.
* **Folder approval card:** refuse to propose or bless the filesystem root, drive roots, the home
  folder itself, and system folders (`/etc`, `/usr`, `C:\Windows`, `/System`…). Offer the file's own
  folder instead. Show how many files are in the folder being approved.
* **Protected floor:** extend `.env`/`.git` to common secret locations (`.ssh`, `.gnupg`, `.aws`,
  `.azure`, `.config/gcloud`, `.kube`, `.docker/config.json`, `.npmrc`, `.pypirc`, `.netrc`,
  `id_rsa*`/`id_ed25519*`, `*.pem`/`*.key`, browser profile folders, OS keychain files), checked
  against the real path after following links, like `.env` is today.

**Where.** `sidecar/permissions.js` (`dangerKey`, `grant`), the approval card in the UI,
`sidecar/pathtrust.js` (`hardlineReason`, `detectRoot`, `guard`), `sidecar/index.js:3428`
(`hardlineFloor`).

**Done when.** Unit tests: blessing `~` is refused; reading `~/.ssh/config` inside a blessed root is
refused; an "Always allow git" grant allows `git status` but still prompts for `rm`.

---

### Step 3. One truth for the floor under Full Power [S4, C13]

**What.** Decide whether "Full Power" crosses the protected floor. Recommendation: **no**. The floor is
small, and a user who really wants an agent to edit `.git` internals can do it by hand. Then:
move the `unrestrictedNow()` check below the hardline in `permissions.js:187-189` and the equivalent
in `pathtrust.js:138`; update the header comments and the in-app description; add a test.

**Done when.** One test asserts the chosen behaviour for both files; the Settings text for Full Power
matches it.

---

### Step 4. Browser mode: keys on the server, and a page CSP [S5, S6, C2]

**What.**

1. When the user saves a key in browser mode, send it to the sidecar (`POST /api/key` with the normal
   API token) instead of keeping it in `localStorage`. Store it in the OS keychain when available
   (`security` on macOS, `secret-tool` on Linux, Credential Manager on Windows), otherwise in an
   encrypted file with a 0600 key file beside it. Migrate existing `localStorage` keys once, then delete
   them from the browser.
2. Encrypt the connector vault in browser mode too, with a locally generated key
   (`sidecar/connector-vault.js` currently stores plain JSON when no key is supplied).
3. Serve `/` with a real Content-Security-Policy (`script-src 'self' 'nonce-…'`, `object-src 'none'`,
   `base-uri 'none'`), putting the nonce on the token script; or deliver the token via a same-origin
   fetch.
4. Show in Settings where keys are stored.

**Why.** It closes two Medium security findings *and* fixes a High functional bug (C2): routines,
channels and step tests start working in browser mode with no environment variables.

**Where.** `frontend/app/harness.js:146-160, 322-330`; `sidecar/index.js:19-20`, `22730-22755`;
`sidecar/connector-vault.js`.

**Done when.** In a fresh browser-mode install, a key saved in the UI makes **Run this step** work,
`localStorage` contains no keys, and the page's response carries a script CSP.

---

## Phase 1 — The first hour

### Step 5. Fix the first-hour bugs [C1, C3–C12, C17]

These are what a new user meets. Each is small; together they decide whether someone keeps using
StarNet. In order of how early a user hits them:

1. **C3:** a failed provider test must not trigger Recovery Mode on the next start. Only count a station
   identity file as prior-install evidence (`sidecar/workspace-lineage.js:16`).
2. **C10:** onboarding shouldn't point at a button that isn't there (`app.js:2843`), and the default
   provider for a fresh install should be one the user can set up from that screen.
3. **C17:** never drop the first chat message during awakening; queue it.
4. **C1:** fix the `heroCaps` shape mismatch (`app.js:3292, 3491`).
5. **C4:** make Start fresh restart the server in browser mode (respawn on exit code 75 in
   `bin/starnet.js`, and have `npm start` use the launcher).
6. **C6:** stop refusing escaped quotes in shell commands on macOS/Linux.
7. **C5:** declare `impact: 'none'` on `deliverable_note` so unattended work gets names.
8. **C9:** remove invented defaults from recipes (`business.js:38` and any others).
9. **C7, C12:** Forget must always do something visible; validate messaging tokens before saving.
10. **C8, C11:** `/api/credits` returns `{configured:false}`; error messages use agent names.

**Done when.** Each has a test (listed in the Coding Review), and a new-user walkthrough (install →
first task → first routine → messaging) completes in browser mode with no dead ends. The hands-on
report's steps are a good script for that walkthrough.

---

## Phase 2 — Make change cheap

### Step 6. Turn on a linter and a type checker, starting small

**What.**

1. Add ESLint with a short rule list: `no-undef`, `no-unused-vars`, `no-empty` (no empty catch blocks
   for new code), `eqeqeq`, `no-implicit-globals` for new files. Run it in the fast gate, but only on
   files changed in the PR at first (`eslint $(git diff --name-only origin/feat/harness-backend -- '*.js')`),
   so the existing 270,000 lines don't block anyone.
2. Add a `jsconfig.json` with `"checkJs": false` and turn on `// @ts-check` **per file**, starting with
   the pure modules that already describe their shapes in comments: `permissions.js`, `pathtrust.js`,
   `apiauth.js`, `apitickets.js`, `inputpolicy.js`, `failopen.js`, `workspace-lineage.js`. Turn those
   comments into JSDoc `@typedef`s. Then `world.js`'s public API (`heroCaps` and friends), which
   would have caught C1.
3. Add a formatter (Prettier with a wide print width, say 140) and apply it one folder at a time, in
   commits that change nothing else, so `git blame` stays useful (use `.git-blame-ignore-revs`).

**Done when.** CI runs lint and `tsc --noEmit -p jsconfig.json` on every PR; the pure security modules
are type-checked.

**Why this step is so valuable.** It's the cheapest way to make every later refactor safe, and it
catches whole classes of bugs found in this review before they ship.

---

### Step 7. Make silent failures visible

**What.** The project already has the right tool, `sidecar/failopen.js` (`swallow(tag)` and `note()`),
written after a bare empty catch hid a feature that failed 100% of the time for weeks. Make it the only
way:

1. Replace the 776 bare empty catches in `sidecar/` with `swallow('area.what')`, a folder at a time.
   Mechanical, but read each: some should actually be errors.
2. Turn on the `no-empty` lint rule for `sidecar/` once done.
3. Do the same in the UI with a small `UI.swallow(tag)` that counts and logs at debug level, and show
   the top tags on the diagnostics page.
4. **Rule for UI actions:** every button either completes, fails with a visible reason, or visibly
   queues. No silent `return`. (C7, C17, C21 are all violations of this.)

**Done when.** `grep -rE "catch \(_\) \{ ?\}" sidecar | wc -l` returns 0, and the diagnostics page lists
fail-open counts.

---

### Step 8. Break up `sidecar/index.js`

**What.** Move each area into its own folder with two files:

* `service.js`: the logic, as functions that take their dependencies as arguments (the pattern
  already used by `permissions.js` and `apiauth.js`), so it can be unit-tested;
* `routes.js`: `module.exports = (deps) => [ { m: 'GET', exact: '/api/…', h: … }, … ]`, returning its
  entries of the route table.

`index.js` keeps start-up, configuration, the request gate, and `ROUTES = [...channels(deps),
...routines(deps), ...]`.

**Suggested order** (each is a self-contained PR, about a week each):

1. **Messaging channels** (Telegram/Slack/Discord/Matrix/Signal wiring, owner trust): security-relevant
   and currently spread over thousands of lines; doing it right after Step 1 locks in that fix.
2. **Recovery, lineage and Start fresh**: touched by C3 and C4.
3. **Routines/cron and the conveyor/step test.**
4. **Updates and release**, **credits**, **voice/Spotify**: leaf features, easy wins.
5. **Consent and tool resolution** last, once the others are out and it's easier to see.

**Rules for each move:** no behaviour change in the same PR; the existing HTTP tests must pass
unchanged; add unit tests for the extracted service.

**Done when.** `index.js` is under 3,000 lines and every feature area has unit tests that run in
milliseconds.

---

### Step 9. Modules for the UI

**What.** Move the UI from 233 classic `<script>` tags with globals to ES modules, gradually:

1. Add a tiny build step (esbuild: one binary, very fast, no config needed) that bundles
   `frontend/app/main.js` into one file for both desktop and browser mode. Keep serving the unbundled
   files in development.
2. Convert leaf files first (those that use globals but define none others need): replace
   `window.X = …` with `export`, and `X.foo()` with `import { foo } from './x.js'`. The bundler lets
   old and new styles coexist during the move.
3. Split the largest files along the seams their own section comments already mark
   (`world.js`, `stationui.js`, `chat.js`, `propsprites.js`).
4. Delete `frontend/agent-demo/` near-copies (`propsprites.js`, `stationbake.js`) by importing the
   main versions with options for the differences.
5. One escaping helper, imported everywhere (Security S6).

**Done when.** The page loads one script bundle; no new globals are added (lint rule); the largest
file is under 3,000 lines.

---

### Step 10. Repository diet

**What.**

1. **Stop committing `website/app/`.** Generate it in the website deploy job from `frontend/` (the
   sync script exists: `npm run sync:website`). This alone removes a 17,022-file duplicate.
2. **Move large binary assets out of normal git**: Git LFS for `frontend/assets/industrial/` (1.2 GB,
   1,125 files), or publish them as a versioned asset pack downloaded by `npm run desktop:prepare`.
3. **Find unused assets**: a script that collects every asset path referenced by code and JSON
   manifests, and lists files that nothing references. Remove those.
4. **Move `output/` studies** (159 MB, 10,087 files) and dated docs into an archive repository or a
   release attachment.
5. Use a shallow checkout in CI where the claims test allows it (it needs `fetch-depth: 0` today;
   consider fetching only the tags or commits it needs).
6. **History rewrite**: making the existing 1.6 GB history smaller needs a history rewrite, which breaks
   every existing clone and open PR. That's a maintainers' decision; the steps above stop the
   growth without it.

**Done when.** A fresh clone of the default branch (excluding LFS content) is under 200 MB.

---

### Step 11. Browser-mode journeys in CI

**What.** Several of the most serious bugs (C2, C4, S5) live only in browser mode, and several others
(C1, C7, C17) are in UI-to-server paths the tests don't exercise. Add a small set of Chrome-driven
journeys that run in the fast gate **in browser mode** and assert on **network requests**, not just on
what's rendered:

1. Fresh install → set a key → first task → approve a write → deliverable appears named.
2. Create a workflow line → **Run this step** → output appears (catches C2).
3. Provider test fails → restart → no Recovery Mode (catches C3).
4. Messaging: save a bad token → refused inline; forget → request sent (catches C7, C12).
5. Routine → Run now → result logged and a notification appears (catches C21).

Use a fake model endpoint (a local OpenAI-compatible stub that returns canned tool calls), as our
hands-on testing did, so the journeys are free and deterministic.

**Done when.** The five journeys run in CI in under three minutes.

---

## Phase 3 — Stronger containment and polish

### Step 12. A real sandbox for shell [S2]

**What.** Add a sandboxed execution backend and make it the default when available:

* **Linux:** `bubblewrap` (`bwrap`): bind the agent's workspace read-write and blessed projects as
  granted, give read-only system folders, no home folder, private `/tmp`, optional no-network.
* **macOS:** `sandbox-exec` with a generated profile allowing the same paths.
* **Windows:** a restricted token / AppContainer, or WSL with the Linux profile.
* **Anywhere with Docker or Podman:** a container with the workspace mounted.

Keep the current unsandboxed path as "Full Power" behaviour, clearly labelled. Show in the approval card
whether a command will run sandboxed.

**Where.** The execution backends already have a home (`sidecar/execution-router.js`,
`execution-profiles.js`, `docs/EXECUTION_BACKENDS.md`); `shell.js` notes "true confinement needs a
container — a deferred backend".

**Done when.** On Linux with `bwrap` installed, an approved command can't read a file in the home
folder outside the workspace, and the test suite proves it.

---

### Step 13. Taint tracking for runs that read outside content [S8, S7]

**What.** Mark a run as *tainted* once it reads content from outside: a web page, a connector
read, a forwarded message, a webhook body, or a file from a downloaded folder. (Forwarded messages are
already flagged; generalise it.) In a tainted run:

* any network call, message or connector write that includes data read from local files needs a fresh
  human approval, even with a standing grant;
* the approval card shows the outgoing content (with secrets redacted, as the write-diff view already
  does).

For webhook-triggered lines, show a "public entry point" badge and warn if agents on the line have Full
Access or terminal/connector grants.

**Done when.** A test run that reads a page and then tries to send a local file's contents to a URL
stops for approval.

---

### Step 14. Cost realism for custom and local models [C16]

**What.** Let users enter prices for custom providers; treat unknown prices as unknown (not $0); enforce
token caps when prices are unknown; show "cost unknown" in the ledger. Also from the hands-on test:
background calls give up after about 5 seconds within a pass, which is right for cloud APIs and fatal
for slow local models. Scale that timeout to the provider (longer for Ollama and custom endpoints) or
make it configurable.

**Done when.** A custom-provider run with a token cap stops at the cap.

---

### Step 15. Documentation and configuration clean-up

**What.**

1. **`docs/README.md`**: a map of current reference docs. Move finished plans and dated memos into
   `docs/archive/`, with a status line at the top of each plan.
2. **Comments state the present**: adopt the rule "comments say what's true now and why; history goes
   in commits and `docs/DECISIONS.md`". Apply it to files as they're touched, not in one sweep.
3. **Environment settings reference**: generate a table of every `ENV('…')` setting with its default
   and meaning (a script can extract them) and publish it in `INSTALL.md`.
4. **npm scripts**: keep the everyday ten in `package.json`; move the 120+ proof and phase scripts
   behind `npm run task -- <name>` with a `--list`.
5. **Retire `SKYNET_` names**: log a deprecation warning when an old name is used; switch CI and docs
   to `STARNET_`; remove support in a stated future version.

---

## What not to do

* **Don't rewrite.** The safety model, request gate, web fetch guard and durable-write layer are
  better than what most rewrites would produce. Extract and tidy; don't replace.
* **Don't loosen the unattended default-deny to fix usability.** Several first-hour problems come from
  unattended runs refusing things. The fix is clearer grants (Steps 2 and 5), not a weaker default.
* **Don't add `--no-sandbox` to Chromium automatically** to make container use easier (C15). Explain
  instead.
* **Don't enforce the linter on the whole codebase at once.** Changed-files-only first; the backlog
  will shrink as files are touched.

---

## Summary

Phase 0 is about a week and brings StarNet's safety behaviour in line with what it tells its users.
Phase 1 is another week and removes the dead ends a new user meets in their first hour. Phase 2 is
the long, steady work that keeps StarNet fast to change as it grows: tooling first, then breaking up
the two giant surfaces (server file and UI), then the repository. Phase 3 turns the careful-but-textual
shell guard into real containment. Every step stands on its own and leaves the project better even if
the plan stops there.
