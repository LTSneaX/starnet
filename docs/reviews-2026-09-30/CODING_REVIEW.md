# StarNet — Coding Review

**Date:** 30 September 2026 · **Revision:** `feat/harness-backend` @ `ab86dd0c` (v0.12.5)
**Question this report answers:** *Which specific pieces of code are wrong, what goes wrong for the user
because of them, and what exactly should change?*

Every item gives the **symptom** (what a user sees), the **cause** (what the code does, with file and
line), **when** it happens, and a **fix**, usually with a code sketch. The "B" numbers match the bug list
in `docs/hands-on-test-2026-09-30/REPORT.md`, where we first saw most of these while using the app. Items
without a B number were found by reading the code.

A note on confidence: where we traced the cause all the way through the code, the item says
**Cause (confirmed)**. Where we saw the symptom but the cause is our best reading, it says **Likely
cause**, with a way to confirm.

Security issues are in the **Security Review**; structural issues in **Code Quality**.

---

## Summary

| # | Item | Impact | Where |
|---|---|---|---|
| C1 | Capabilities shown as "undefined, undefined…"; cabinet check always false (B1) | High | `frontend/app/app.js:3292, 3491` |
| C2 | Workflow step tests, routines and channels can't see browser-mode keys (B2) | High | `frontend/app/workflowpanel.js:615`, `sidecar/index.js:2342-2364` |
| C3 | A failed provider test makes a brand-new install look like an old one (B3) | High | `sidecar/workspace-lineage.js:16`, wire-test run recording |
| C4 | "Start fresh" stops the server in browser mode (B4) | High | `sidecar/index.js:10073, 10094`, `frontend/app/app.js:5134-5146` |
| C5 | `deliverable_note` is refused in unattended runs (B11) | Medium | `sidecar/inputpolicy.js:40-57`, `sidecar/tools/builtin/deliverable.js:62` |
| C6 | Shell guard refuses any backslash, including escaped quotes (B6) | Medium | `sidecar/tools/builtin/shell.js:52` |
| C7 | Telegram "CONFIRM FORGET" does nothing (B7) | Medium | `frontend/app/windows/messaging.js:599-604` |
| C8 | `/api/credits` returns 404 on every page load (B21) | Low | `sidecar/index.js:9760`, `frontend/app/app.js:2277, 2298`, `harness.js:283` |
| C9 | Recipe invents "the terms you have told me" (B9) | Medium | `frontend/app/recipe-catalog/business.js:38` |
| C10 | Onboarding points to a button that isn't there (B8) | Low | `frontend/app/app.js:2843` |
| C11 | Error text says "target agent agent" | Low | `sidecar/index.js:2364` |
| C12 | Obviously invalid messaging token is saved and looks connected (B22) | Low | channel connect route |
| C13 | Consent comment says the floor can't be bypassed; code lets Full Power bypass it | Medium | `sidecar/permissions.js:13-16, 187-189` |
| C14 | Hardline floor only checks an argument named `path` | Low | `sidecar/index.js:3428-3433` |
| C15 | Browser tool fails with an opaque error when run as root (Linux) | Low | `sidecar/tools/builtin/browser.js` launch |
| C16 | Spending caps can't protect custom/local providers (recorded as $0) | Medium | cost/ledger recording |
| C17 | First chat message during start-up is lost (B5) | Medium | chat input/awakening hand-off |
| C18 | Deliverable "What you asked for" shows the last message, not the brief (B12) | Low | deliverable record |
| C19 | "Shipped" counter counts runs, not deliverables (B10) | Low | station record counters |
| C20 | FORK line shown twice; turns glued together without a space (B15, B16) | Low | `frontend/app/chat.js`, `fork.js` |
| C21 | Stale approval notifications; no notice when an unattended write is refused (B17) | Low | notifications |
| C22 | Clicking a crew card doesn't switch the chat thread (B19) | Medium | crew card / COMMS |
| C23 | Quest/goal progress ignores finished work (B13) | Low | quest completion |

---

## C1. Capabilities shown as "undefined, undefined…"; the cabinet check is always false (B1)

**Symptom.** On the first-task screen, the agent's abilities are listed as
"undefined, undefined, undefined". Separately, the autopilot believes the agent has no file cabinet
even when one is placed, so a hand-off that depends on it never happens.

**Cause (confirmed).** `World.heroCaps()` (`frontend/app/world.js:10929-10949`) returns a list of
**objects** shaped `{ objectType: 'cabinet' }`. Two callers in `app.js` expect something else:

* `app.js:3292`:
  ```js
  getCaps: () => (World.heroCaps('agent') || []).map(c => (typeof c === 'string' ? { id: c, label: c } : c)),
  ```
  Objects pass through unchanged, so `{objectType}` reaches code that reads `c.label || c.id`. Both
  are `undefined`.
* `app.js:3491`:
  ```js
  hasCabinet: () => … (World.heroCaps(agentId) || []).indexOf('cabinet') >= 0
  ```
  `indexOf('cabinet')` on a list of objects is always `-1`.

**When.** Every time; the shape changed in `heroCaps` and these callers weren't updated.

**Fix.**
```js
// app.js:3292
getCaps: () => (World.heroCaps('agent') || []).map(c => {
  const id = typeof c === 'string' ? c : (c && (c.objectType || c.id));
  return { id, label: capLabel(id) };           // capLabel: the existing human name for a capability
}),

// app.js:3491
hasCabinet: () => (World.heroCaps(agentId) || [])
  .some(c => (typeof c === 'string' ? c : c && c.objectType) === 'cabinet'),
```
Better still, add one helper `capId(c)` next to `heroCaps` and use it everywhere. Add a JSDoc return type
to `heroCaps` (`@returns {{objectType: string}[]}`) so a type checker catches the next mismatch (see
Code Quality §6).

**Test.** A UI unit test that seeds a station with a cabinet and asserts `getCaps()` labels are
non-empty strings and `hasCabinet()` is `true`.

---

## C2. Workflow step tests, routines and channels can't see browser-mode keys (B2)

**Symptom.** In browser mode, chat works, but **Run this step** in the workflow editor fails with
"configure the CUSTOM base URL for target agent agent". Routines and messaging channels fail the same
way. Setting `CUSTOM_OPENAI_BASE_URL` in the environment makes it work.

**Cause (confirmed).** In browser mode, API keys and base URLs live only in the page's `localStorage`
(`frontend/app/harness.js:322-330`). Chat requests carry them. Server-side runs look up credentials
through `channelRunConfigFor()` (`sidecar/index.js:2342-2366`), which *can* accept a key offered by
the caller (`candidate`), but the step-test request doesn't offer one:

```js
// workflowpanel.js:615
return api('/api/routing/steptest', 'POST', { line: c.key, text, startAt: pid, single: true });
```
With no runtime key on the server and none in the request, `providerHasCredential()` fails.

Routines and channels run with nobody's browser involved, so they *can't* get the key from the page
at all.

**When.** Browser/source mode only. The desktop build pushes keys into the sidecar (`/api/key`).

**Fix.** Two levels:

1. *Quick:* pass the same candidate the chat path uses in the step-test body:
   `{ line, text, startAt, single: true, candidate: Harness.runCandidate() }`, and have
   `/api/routing/steptest` forward it to `channelRunConfigFor`.
2. *Proper (recommended):* keep keys **server-side** in browser mode too. When the user saves a key,
   `POST` it to `/api/key` with the page's normal API token. This is the same fix as Security S5 and
   makes routines and channels work.

**Test.** A browser-mode HTTP test: set a key via the UI path, then call `/api/routing/steptest` and
assert it doesn't return the credential error.

---

## C3. A failed provider test makes a brand-new install look like an old one (B3)

**Symptom.** On a fresh install, trying a provider that isn't running (for example, Ollama when it isn't
started) fails, which is fine. But after restarting StarNet, the user is put in **RECOVERY MODE — prior
station data found**, although they never created a station.

**Cause (confirmed).** The provider "wire test" is an internal run. It writes `runs.jsonl` and
`ledger.jsonl` into the workspace. The start-up gate (`inspectWorkspaceLineage`,
`sidecar/workspace-lineage.js:49-75`) treats any file matching `STATE_EVIDENCE`
(`workspace-lineage.js:16`) as proof of a previous station, and that pattern includes
`runs.jsonl` and `ledger.jsonl`.

**When.** The first time a user tests a provider before completing onboarding, then restarts.

**Fix.** Pick one (the first is the most robust):

1. Only treat the workspace as a prior station if a **station identity file** exists
   (`agent.save.json` or `agent.roster.json`). Ledgers and run logs alone are not a station.
2. Don't record internal, zero-token wire tests into `runs.jsonl`/`ledger.jsonl` before onboarding
   completes (or record them in a separate `diagnostics.jsonl` that isn't evidence).

**Test.** Unit test on `inspectWorkspaceLineage`: a workspace with only `runs.jsonl` and
`ledger.jsonl` reports `priorInstallEvidence: false` (with fix 1).

---

## C4. "Start fresh" stops the server in browser mode (B4)

**Symptom.** In Recovery Mode, pressing **Start fresh** twice sets the old files aside (good), then the
server stops. In browser/source mode nothing restarts it. The page keeps polling and stays on the
recovery screen.

**Cause (confirmed).** Both recovery actions end with `setTimeout(() => process.exit(75), 150)`
(`sidecar/index.js:10073` and `10094`). The desktop shell's guardian restarts the sidecar on exit code
75. When StarNet was started with `npm start` or `node sidecar/index.js`, nothing does. The UI does set a
status line in the non-desktop case (`app.js:5139`: "restart the manual sidecar; this screen will
continue automatically"), but in our run we didn't notice it, and for a newcomer "restart the manual
sidecar" isn't actionable.

**When.** Browser/source mode, any recovery or start-fresh action.

**Fix.**

1. Have the launcher restart on 75. `bin/starnet.js` already spawns the sidecar; make it (and
   `npm start`, by pointing it at the launcher) respawn when the child exits with 75.
2. Or, when `DESKTOP_SHELL` is false, re-run the start-up sequence in the same process instead of
   exiting (harder, because of module-level state in `index.js`).
3. At minimum, make the message impossible to miss: a full-screen panel saying "StarNet has stopped
   so it can start fresh. In the terminal where you started it, press ↑ and Enter to run it again."

---

## C5. `deliverable_note` is refused in unattended runs (B11)

**Symptom.** Work produced by workflows and routines shows up in the Library as
"index.html — the agent did not name this one". The run log says the naming tool was withheld as
"a tool with unknown external effects".

**Cause (confirmed).** `deliverable_note` is declared with `capability: 'deliverable'` and no
`impact` (`sidecar/tools/builtin/deliverable.js:62`). Its own comment says it "writes no file, reaches
no network, and has no outward effect". But `impactOfTool()` (`sidecar/inputpolicy.js:42-57`) only
treats capabilities in `SAFE_BUILTIN_CAPS` as harmless:

```js
const SAFE_BUILTIN_CAPS = new Set(['compute', 'cabinet', 'memory', 'quest', 'studio', 'orchestrator',
  'taskbrief', 'toolsearch', 'code', 'stationinfo']);    // no 'deliverable'
```
Anything else falls through to `EXTERNAL_UNKNOWN`, which unattended runs refuse (`inputpolicy.js:181`).

**When.** Every unattended run (routines, workflow lines, step tests, channels without approvals).

**Fix.** Declare the impact on the tool itself (clearer than widening the set):
```js
name: 'deliverable_note', capability: 'deliverable', scope: 'write', impact: 'none', requiresConsent: false, …
```
**Test.** Unit test: `impactOfTool(deliverableTool) === IMPACTS.NONE`, plus an unattended run that
calls it and gets a named deliverable.

---

## C6. The shell guard refuses any backslash, including escaped quotes (B6)

**Symptom.** A normal Linux command such as `grep -c "class=\"hero\"" index.html` is refused with
"drive-root paths (\\…) are not allowed". Both `shell.exec` and `verify.run` are affected.

**Cause (confirmed).** `shell.js:52`:
```js
if (/(^|[\s"'`=(])\\(?![\\])/.test(cmd)) return 'drive-root paths (\\…) are not allowed — …';
```
The pattern means "a backslash after a space, quote, backtick, `=` or `(`". `\"` inside a
double-quoted string matches (the character before the backslash is a quote). The rule was written for
Windows, where `\Users\x` means "root of the current drive", but it runs on every platform.

**When.** Any command with an escaped quote, a regular expression, or a `printf` format, on macOS and Linux.

**Fix.**

1. Apply the drive-root rule **only on Windows** (`process.platform === 'win32'`), where it means
   something.
2. On Windows, require the backslash to be followed by a path-like character and to be in the
   first position of a word: `/(^|\s)\\(?![\\"'])[A-Za-z0-9_.-]/`.
3. As Security S2 says, this guard isn't a sandbox; being wrong in the strict direction only trains users
   to reach for Full Access.

**Test.** Table test with commands that must pass (`grep "a=\"b\""`, `printf "%s\n"`, `sed 's/\./,/'`)
and commands that must be refused on Windows (`type \Windows\win.ini`, `cd \`).

---

## C7. Telegram "CONFIRM FORGET" does nothing (B7)

**Symptom.** In Messaging, **FORGET** turns into **CONFIRM FORGET**, but clicking it sends no request
(we watched the network trace over three attempts). Calling
`POST /api/channels/telegram/disconnect {purge:true}` directly works.

**Likely cause.** The click handler (`frontend/app/windows/messaging.js:599-604`) checks
`configuredById[c.id]` on **every** click, including the confirming one, and returns silently if it's
false:
```js
fBtn.addEventListener('click', () => {
  if (!(configuredById[c.id])) { return; }        // silent
  armed(c.pre + '-forget', fBtn, removeLabel, confirmRemoveLabel, async () => { … });
});
```
`configuredById` is rewritten by every status refresh (`paintCard`, line 329; the 30-second poll at
line 489; and `refreshAll()` after other actions), and set to `false` for every channel if one refresh
fails (line 473). In our test the saved token was invalid (see C12), so the channel's reported state was
changing. If `configured` flips to false between the arming click and the confirming click, the
confirm click is swallowed with no message. There's also a two-second arm window (`armed()`, line 450),
after which the label resets.

**How to confirm.** Log `configuredById[c.id]` and `btn.dataset.armed` on each click, and reproduce with
an invalid token.

**Fix.**
```js
fBtn.addEventListener('click', () => {
  const armedNow = fBtn.dataset.armed === '1';
  if (!armedNow && !configuredById[c.id]) {                 // decide once, when arming
    setMsg(msgEl, 'nothing saved for this channel', 'info');  // never silent
    return;
  }
  armed(…);
});
```
Also: extend the arm window to about 5 seconds, and let "forget" always be allowed (purging a
secret that may or may not exist is harmless and is what a worried user wants).

---

## C8. `/api/credits` returns 404 on every page load (B21)

**Symptom.** Every load logs a failed request in the browser console.

**Cause (confirmed).** The route is registered as "404s (no surface) unless managed credits are
configured" (`sidecar/index.js:9760`), and the page calls it unconditionally from three places
(`frontend/app/app.js:2277, 2298`, `frontend/app/harness.js:283`).

**Fix.** Return `200 {"configured": false}` when credits aren't set up. Callers already check
`j.configured`. A 404 should mean "no such route", not "feature off"; otherwise real 404s get ignored.

---

## C9. The proposal recipe invents what the user told it (B9)

**Symptom.** The "Draft a Proposal" recipe writes as if it knows the user's usual terms, even when
none were ever given.

**Cause (confirmed).** The optional "Your terms" field has a **default value** that's a claim, not a
placeholder (`frontend/app/recipe-catalog/business.js:38`):
```js
{ key: 'terms', …, required: false, default: 'the terms you have told me you normally work on' }
```
The field is collapsed by default, so the user never sees it, and the text is inserted into the prompt.

**Fix.** Default to empty and let the template say what to do when it's missing:
```js
{ key: 'terms', …, required: false, default: '' }
// template: "{{#if terms}}Terms: {{terms}}{{else}}No terms were given — propose sensible ones and mark them as suggestions.{{/if}}"
```
Search the other recipe files for similar `default:` values that assert knowledge (`grep -n "default: '" frontend/app/recipe-catalog/`).
(`website/app/app/recipe-catalog/business.js:38` has the same line; see Code Quality §3.3.)

---

## C10. Onboarding points to a button that isn't there (B8)

**Symptom.** Waking the first agent with the default provider says "press 🔗 LINK YOUR STARNET ACCOUNT
above". There's no such button on that screen.

**Cause (confirmed).** `frontend/app/app.js:2843` hard-codes the message; the button lives elsewhere (and
only when managed credits exist, see C8).

**Fix.** Either render the link button next to the message, or change the text to a real path ("choose
a provider in Settings → Model, or add your own API key"). Better, don't default a fresh install to a
provider that needs an account the user can't create from that screen.

---

## C11. Error text says "target agent agent"

**Symptom.** "configure the CUSTOM base URL for target agent agent".

**Cause (confirmed).** `sidecar/index.js:2364` appends the raw agent id, and the lead agent's id is
literally `agent`.

**Fix.** Use the display name: `' for ' + (ident.name || id)`, which gives "…for Nova".

---

## C12. An obviously invalid messaging token is saved and looks connected (B22)

**Symptom.** Saving a made-up Telegram token shows a normal-looking `/pair` code; the only sign of
failure is a toast that disappears.

**Fix.** Validate before saving: call Telegram's `getMe` with the token (StarNet already talks to the
Bot API) and refuse to store it unless it succeeds, showing the error inline next to the field. Only
show the pairing code after a successful check. Apply the same idea to the other channels (Slack
`auth.test`, Discord `users/@me`).

---

## C13. The consent code says the floor can't be bypassed; the code lets Full Power bypass it

**Where.** `sidecar/permissions.js`: header lines 13-16 ("HARDLINE … checked FIRST so no flag can
reach past it") and line 188 ("unreachable past any flag"); but line 187 returns `allow` for Full Power
before the hardline check on line 189.

**Why it matters.** Covered in Security S4. From a coding point of view: a comment that contradicts the
code is a bug. The next person to add a protection will put it in the hardline believing it's absolute.

**Fix.** Decide the rule, then make comment, code and a test agree. If the floor should hold under Full
Power, move the `unrestrictedNow()` line below the hardline check. If not, rewrite the header to
describe two tiers ("Full Power removes every StarNet policy including the floor").

---

## C14. The hardline floor only checks an argument named `path`

**Where.** `sidecar/index.js:3428-3433`:
```js
function hardlineFloor(call) {
  const p = call && call.args && call.args.path;
  if (typeof p === 'string' && (/…\.env…/i.test(p) || /…\.git…/i.test(p))) return 'writing ' + p + ' is blocked…';
  return null;
}
```
Tools that name files differently aren't covered at this layer: `fs.patch` takes a `patch` text
containing the paths; other tools use `file`, `dest` or `files`. Today `pathtrust.js` re-checks project
paths, so this is defence in depth rather than a live hole. But a single, obvious rule is worth more
than two partial ones.

**Fix.** Let each tool declare the paths it will touch (`tool.pathsOf(args) → string[]`, with `fs.patch`
parsing its patch headers), and have `hardlineFloor` check all of them. Unknown tools with no
`pathsOf` get checked on every string argument that looks like a path.

---

## C15. The browser tool fails with an opaque error when run as root on Linux

**Symptom.** In a container (running as root), every browser tool call fails with "exited before CDP
ownership was established". The real reason, Chromium refusing to run as root without `--no-sandbox`,
isn't shown.

**Fix.** Don't add `--no-sandbox` automatically (that would weaken security). Instead, when the launch
fails and `process.getuid && process.getuid() === 0`, say so: "Chromium won't run as the root user.
Run StarNet as a normal user, or set `STARNET_CHROME` to a wrapper that adds `--no-sandbox` if you
accept the risk." Also include the last lines of Chromium's stderr in the error. The CI workflow
already uses exactly that wrapper approach.

---

## C16. Spending caps can't protect custom or local providers

**Symptom.** With a custom OpenAI-compatible endpoint or Ollama, every run is recorded as $0, so
dollar caps never trigger, even if the "custom" endpoint is a paid service.

**Fix.** Let users set a price per million input/output tokens for custom providers (default: unknown,
not zero), and when the price is unknown, enforce the **token** caps instead of dollar caps. Show
"cost unknown" rather than "$0.00" in the ledger.

---

## C17. The first chat message during start-up is lost (B5)

**Symptom.** A message typed and sent just as the agent's "awakening" finished was cleared from the
input box but never shown or sent.

**Likely cause.** The send handler clears the input before checking that a run can start; if the chat
isn't ready (awakening hand-off in progress), the send is dropped without restoring the text.

**Fix.** Never clear the input until the message is accepted (shown in the thread or queued). If the
chat isn't ready, queue the message and send it when ready, showing it immediately as "waiting".

**How to confirm.** Add a test that sends a message while the awakening flag is set, then asserts the
message appears in the thread after awakening.

---

## C18–C23. Smaller correctness issues seen during use

These were all observed in the hands-on test; each is small.

* **C18. Deliverable "What you asked for" shows the last message of the run (B12)**, not the task
  brief. Record the brief (the Task Brief text, or the first user message of the task) when the
  deliverable is created, not at the end.
* **C19. "Shipped" counter counts runs that changed anything (B10)** ("2 deliverables shipped" when one
  file existed; "8 shipped today" with 3 deliverables). Count distinct deliverable records instead.
* **C20. FORK choices appear twice (B15)**, as buttons and as raw `FORK: q || a | b` text, and **two
  assistant turns are glued together without a space (B16)** ("…and starting.Writing the page."). Strip
  the FORK line from the message once it's parsed into buttons (`frontend/app/fork.js` /
  `chat.js`), and join consecutive turns with a paragraph break.
* **C21. Approval notifications stay unread after the approvals are resolved (B17)** (10 of 14 in
  our session), while a refused unattended write produced **no** notification. Resolve a notification
  when its approval is answered; notify when an unattended run is refused something, since that's
  exactly when the user needs to grant a standing approval.
* **C22. Clicking an agent's crew card doesn't switch the chat thread (B19)**; a message meant for
  Nova went to Pixel. Make the crew card set the active thread, and show the recipient's name in the
  input placeholder ("Message Nova…") so a mismatch is visible before sending.
* **C23. Goal progress ignores finished work (B13).** Milestone "Ship one sample client page" stayed
  at 0/5 after the page shipped and was approved. Tie milestone completion to deliverable records
  (C18/C19 make that data reliable).

---

## Patterns behind these bugs

Looking across the list, most bugs come from four patterns. Fixing the pattern prevents the next one:

1. **Data shapes change without the callers noticing** (C1, C5). A type checker on the UI and on the tool
   declarations (Code Quality §6) catches this class at edit time.
2. **Browser mode is a second-class path** (C2, C4, C15; Security S5). Put browser mode into CI with
   the same journeys as desktop.
3. **Silent early returns and swallowed errors** (C7, C17, C21). A rule: every user action ends in a
   visible outcome (done, refused with a reason, or queued), and there are no empty `catch` blocks.
4. **Text that claims more than is true** (C9, C10, C13; Security S1's `/approvals` message). Treat
   user-facing text and code comments as part of the behaviour, and test the important ones.
