# StarNet — Review Pack (30 September 2026)

Four reports on this repository at `feat/harness-backend` @ `ab86dd0c` (v0.12.5), written after using the
app end to end (see `docs/hands-on-test-2026-09-30/`) and then reading the code. Each report stands on
its own; they cross-reference each other by finding number.

| Report | Question it answers | Findings |
|---|---|---|
| [SECURITY.md](SECURITY.md) | What could let someone, or something, do harm through StarNet? | S1–S11 |
| [CODE_QUALITY.md](CODE_QUALITY.md) | How easy is it to understand, run, change and trust? | Scorecard + 10 areas |
| [CODING_REVIEW.md](CODING_REVIEW.md) | Which specific lines are wrong, and what's the fix? | C1–C23 |
| [IMPROVEMENTS.md](IMPROVEMENTS.md) | What to do, in what order, and how long it takes | Steps 1–15 |

## If you only read one page

* **The foundations are strong.** The local server is well locked down (Host pin, Origin check,
  per-launch token, scoped tickets), web fetching is protected against reaching your local network, the
  consent system defaults to "no" for unattended runs, and the test suite is large and runs on every PR.
* **The most important security gap:** a paired messaging account (e.g. Telegram) gets approval-free
  control of the station, including shell commands, while the bot tells the owner it "silently skips"
  such actions. → Security S1, Improvements step 1.
* **Approvals are broader than they look:** one "Always" on a shell command approves *all* future
  commands; folder approval can cover the whole home folder; only `.env` and `.git` are protected.
  → Security S2–S4.
* **Browser mode is the weak path:** keys in `localStorage` (and invisible to routines, channels and
  step tests), no script policy on the page, and Start fresh stops the server. → Security S5–S6,
  Coding C2, C4.
* **Structure:** `sidecar/index.js` is 22,756 lines and can't be unit-tested; the UI is 167k lines of
  global scripts with no linter or type checker; the repo carries 1.2 GB of assets twice.
  → Code Quality, Improvements steps 6–10.

Phase 0 and 1 of the plan (the trust fixes and first-hour bugs) are about **two weeks** of work.

## How these were produced

* Everything was checked against the code at the revision above; file and line references are to
  that revision.
* Runtime observations come from using the app in a cloud container with a local stand-in model (see
  the hands-on report). Items found only by reading code are labelled as such.
* Security findings describe the risk, where it is and how to fix it. They deliberately don't include
  attack instructions.
