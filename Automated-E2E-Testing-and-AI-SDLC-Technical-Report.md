# Automated End‑to‑End Integration Testing — and How Far AI Can Take the Product Lifecycle
### Technical Report

**Author:** Aryan Nandal  ·  **Date:** 30 June 2026

---

## Executive summary

The brief was to design and build an **automated end‑to‑end (E2E) integration testing system** that exercises a frontend and a backend together. I delivered that as **Pomodoro Planner** (Part 1): a system that drives the *real* Flutter app against a *real* (emulated) Firebase backend, inside disposable Docker containers, with a wiped‑and‑seeded database per test and screenshots as evidence.

I then pursued a broader question I was curious about — **how much of the whole product‑development lifecycle can one engineer drive with an AI coding agent without losing engineering rigor?** That produced two further projects, each adding a layer:

- **Super Sudoku** (Part 2) — a documented, single‑operator‑plus‑AI‑agent SDLC pipeline, built strictly test‑first (137 tests across four verification layers).
- **Super Chess** (Part 3) — the same product‑design flow plus **agent‑driven code review** and a **risk‑gated CI pipeline** that escalates to a human only when risk is high.

This report covers each in technical depth, then the cross‑cutting principles and honest limitations.

---

## Part 1 — Pomodoro Planner: the E2E integration testing system

**Repo:** <https://github.com/aryan-nandal/pomodoro_planner>
**Design doc:** [`docs/e2e-integration-testing.md`](https://github.com/aryan-nandal/pomodoro_planner/blob/main/docs/e2e-integration-testing.md)

### 1.1 The app as a vehicle

Pomodoro Planner is a cross‑platform **Flutter** productivity app backed by **Firebase**, deliberately chosen to make an E2E flow *meaningful* — it has real auth, real persistence, async timing, and serialization. It combines:

- **Task Planner** — categories, priorities, subtasks, scheduling, search, archive; completing all subtasks auto‑completes the parent task.
- **Pomodoro Timer** — focus / short‑break / long‑break cycles (long break every 4th focus session), custom durations, alarm + haptic + local‑notification cues, and the ability to start a focus session directly from a task.
- **Statistics** — tasks completed today, focus minutes today and over 7 days (charted with `fl_chart`), total Pomodoros, daily streak.
- **Accounts & Sync** — email/password auth with per‑user data isolated in Cloud Firestore under `users/{uid}/…`.

The README calls the E2E system *"the highlight of this project."* The motivation, quoted from the design doc: confidence that the app works *"when the actual app talks to an actual backend: sign up, read and write Firestore data, complete a task, and see the result reflected in the UI."* Unit and widget tests are deemed insufficient because they *"eliminate the genuinely difficult aspects."*

### 1.2 Goals

Operate the **real app** end‑to‑end (real `main()`, BLoCs, Firebase SDKs); run against a **real backend, not mocks**; be **deterministic** and **isolated**; **never touch the production Firebase project**; and be **reproducible** across machines and CI.

### 1.3 Approach (one sentence, from the doc)

> Run the real Flutter app against Google's **Firebase Emulator Suite** (a genuine Auth + Firestore backend, just local), inside a **disposable Docker container**, with the database **wiped and seeded** before each test, and capture **screenshots** of every key screen as evidence.

### 1.4 The seven design decisions

1. **Real backend (emulator), not mocks** — *"The app doesn't know it's talking to an emulator — I redirect it at runtime."*
2. **Everything runs inside Docker** — the full stack is containerized so runs are *"hermetic and reproducible,"* depending only on Docker.
3. **DB cleared and re‑seeded before every test** — *"Determinism comes from controlling the starting state."* A Node seeder wipes Auth + Firestore and writes a known fixture.
4. **App opts into the emulator at compile time** — connects to the emulator only when built with `--dart-define=USE_EMULATOR=true`; defaults to `false`, so a production build can never accidentally point at a test backend.
5. **Each test gets its own fresh container** — *"every test runs in its own `--rm` container with its own emulator,"* enabling future parallelism.
6. **Shared caches for speed** — only Docker volumes for pub caches and build artifacts are reused; no test *state* persists.
7. **Screenshots as evidence** — captured at each meaningful step and written back to the host.

### 1.5 Components

| File | Role |
|---|---|
| `run_integration_tests.sh` | Host orchestrator — builds the image, creates cache volumes, clears old screenshots, iterates the test cases (`auth_flow`, `tasks_flow`) each paired with a seed file, runs each in an isolated container, collects exit codes, prints a pass/fail summary, exits 0 on full success / 1 on any failure. |
| `integration_test_system/Dockerfile` | `debian:bookworm-slim` + **Java 21** (Temurin), **Chromium + chromium‑driver**, **Node.js 20**, **Firebase CLI**; pre‑downloads the Firestore emulator; clones **Flutter stable**, enables web. |
| `integration_test_system/entrypoint.sh` | In‑container lifecycle (see §1.6). |
| `integration_test_system/firebase.json` | Emulator config — Auth on **9099**, Firestore on **8080**, `singleProjectMode: true`. |
| `integration_test_system/seed.js` | Node + Firebase Admin seeder — `clearEmulators()` DELETEs both services; `seed()` creates Auth users and writes Firestore docs from a JSON fixture, substituting a `{TODAY}` date placeholder. |
| `integration_test_system/seeds/*.json` | Fixtures (`auth_flow.json`, `tasks_flow.json`) — e.g. one user + two seeded tasks (one complete, one incomplete). |
| `integration_test/*_test.dart` | The user‑flow tests. |
| `test_driver/integration_test.dart` | Driver using `integrationDriver(onScreenshot: …)`; writes each PNG to `/workspace/screenshots/<name>.png`. |

### 1.6 Data flow — one test run

1. Orchestrator starts a fresh container with the chosen test target + seed file.
2. `entrypoint.sh` (`set -e`) rsyncs the project into an isolated `/app`, installs seeder deps, then starts `firebase emulators:start` with a `trap cleanup EXIT` that tears down the emulator and ChromeDriver.
3. Health‑polls **Firestore (8080)** and **Auth (9099)** via `curl`, each with a 60 s timeout.
4. Seeds the DB: `node seed.js "$SEED_FILE"` (clear → write fixture).
5. Starts `chromedriver`, then `flutter drive --target=<test>` on `-d web-server` with **headless Chromium** flags and `--dart-define=USE_EMULATOR=true --dart-define=EMULATOR_HOST=localhost`.
6. The test simulates user actions and captures screenshots; the container exits with the test's exit code, which the orchestrator records.

### 1.7 What the tests actually verify

- **Auth flow** — open app → Sign Up → register a new account → confirm navigation to the home screen. Screenshots: `01_auth_screen`, `02_signup_screen`, `03_home_screen_after_signup`.
- **Tasks flow** — sign in with the seeded user → assert the seeded "Incomplete" and "Completed" tasks render → tap the incomplete task's checkbox to complete it → open the FAB "Create Task" screen → enter a new task → save → assert it appears. Screenshots: `01_tasks_loaded` … `04_new_task_in_list`.

Both flows' screenshots are committed to the repo as evidence.

### 1.8 Running it

Locally: a single command — `./run_integration_tests.sh`. *"Requires only Docker. The first run builds the image (slow); later runs reuse it and the caches (fast)."*

### 1.9 Honest tradeoffs (from the doc)

- **Timing waits are the primary flakiness risk** — the doc would prefer polling for expected widgets over fixed sleeps.
- **Web‑only** today; extending to Android/iOS is unimplemented.
- **No Firestore security‑rules coverage** — the emulator runs unrestricted, so the suite validates *app behavior*, not security rules.
- **Sequential execution** — tests run one at a time, although the isolation model already supports parallelism.
- **CI‑readiness is architectural, not a committed pipeline** — the Docker‑only design is built to run in CI, but no GitHub Actions workflow is checked in yet.

---

## Part 2 — Super Sudoku: AI automation across the full product lifecycle

**Repo:** <https://github.com/aryan-nandal/Super-Sudoku>
**Design doc:** [`docs/PRODUCT_AUTOMATION_DESIGN.md`](https://github.com/aryan-nandal/Super-Sudoku/blob/main/docs/PRODUCT_AUTOMATION_DESIGN.md)

### 2.1 The thesis

Having delivered the testing system, I wanted to test the *process*: how far AI automation can be pushed across the whole SDLC while keeping rigor. The design doc states it plainly: *"The interesting part isn't 'an AI wrote code.' It's the system around it: how requirements become locked decisions, how every change is test‑driven and verified."* The model compresses a multi‑role product org into **one human operator (architect / PM / reviewer at every gate) + one AI coding agent (execution).**

### 2.2 The app

A cross‑platform **Flutter** Sudoku app — *"the ultimate cognitive gym … built around an honest puzzle engine, a global daily, and a real learning ramp."* Its differentiator is an **"honest" engine that grades difficulty by the logical solving techniques a puzzle requires**, not by clue count. Done: unique‑solution generator, brute‑force solver, a human‑technique logical solver, a full 9×9 board (notes, highlighting, undo/redo, win flow), and a deterministic **Global Daily** with a shareable result card.

### 2.3 The pipeline

> **Market Analysis → Requirements & Strategy → Architecture & Tech → Engineering (TDD) → Verification → Delivery & Deploy**

Each stage produces an artifact: market analysis → a positioning gap ("Cognitive Gym"); requirements → locked product decisions (free Daily funnel + a single one‑time unlock, no ads); architecture → the stack and seams; engineering → code + tests; verification → a green suite + screenshots + a live smoke test; delivery → merged PRs + a live URL.

### 2.4 Architecture (built to change)

- **Pure‑Dart domain core** with zero Flutter/backend imports — *"the durable, testable asset."*
- **Riverpod v2** for state, **Drift (SQLite)** for offline‑first persistence (with `build_runner` codegen; web via `sqlite3.wasm`), **go_router** for routing, and isolate‑based puzzle generation via `compute`.
- **Firebase behind interfaces** — Auth, Firestore, **Cloud Functions (TypeScript)**, Hosting — deferred to a later phase so the MVP stays local‑first. Guiding principle: *"make the reversible decision now, keep the expensive one optional. Flags and seams over big rewrites."*

### 2.5 Quality strategy — automation without losing rigor

TDD is the default: *"tests precede UI/feature code,"* following **red → green → refactor**. Verification has **four layers**:

1. **Domain unit tests** — engine, rating math, ranking logic (pure, fast).
2. **Application tests** — controllers via dependency‑injected fakes (`ProviderContainer` + provider overrides).
3. **Widget tests** — screens render and react (`flutter_test`).
4. **End‑to‑end + "actually run it"** — drive the real app, capture screenshots, run a live smoke test.

Test frameworks: **`flutter_test`** on the app side and **Vitest** for the TypeScript Cloud Functions. The suite currently stands at **137 automated tests**, *"analysis clean, on every merge."* (The README's earlier "~49 tests" reflects an earlier phase.)

The doc is candid about *why* automation alone isn't enough: *"An AI agent is fast but will confidently ship subtly‑broken work if you let it,"* and *"I don't trust 'tests pass.' I run the real app, screenshot it, and smoke‑test the deployed site."* The proof point: a **live smoke test caught a Firebase initialization bug that a fully green test suite had missed** — *"'tests are green' is necessary, not sufficient."*

### 2.6 How work ships

One concern per PR (**33+ PRs**), descriptive history, **no direct commits to `main`**, human review at every gate. Delivery is *"scripted and free": `flutter build web && firebase deploy`* → a live, shareable URL, on Firebase's free (Spark) plan at **₹0 infra cost**. (CI runs as a scripted/per‑PR process; there are no GitHub Actions workflows in this repo — that comes next, in Super Chess.)

### 2.7 Generalization

The doc notes the same pipeline was *"reapplied identically to a sibling chess‑teaching product"* — i.e. it was designed to be reusable, which sets up Part 3.

---

## Part 3 — Super Chess: agent‑driven code review + risk‑gated CI

**Repo:** <https://github.com/aryan-nandal/Super-Chess>
**Merge‑flow doc:** [`docs/AUTOMATED_MERGE_FLOW.md`](https://github.com/aryan-nandal/Super-Chess/blob/main/docs/AUTOMATED_MERGE_FLOW.md) · **Sample:** [PR #12](https://github.com/aryan-nandal/Super-Chess/pull/12)

### 3.1 What it adds

Super Chess **reuses the Super Sudoku product‑design flow** and goes one step deeper into the part of the lifecycle that's hardest to automate responsibly: **code review and merge gating.** The goal — *automate review so that a human is pulled in only when risk is genuinely high.*

### 3.2 The app

A **Flutter** chess **learning** app — *"the teaching‑first chess gym — play locally against a full rules engine and drill a 528‑puzzle, motif‑by‑motif tactics trainer."* It has a **pure‑Dart engine** (with **perft** verification and a **UCI** interface), complete rules (checkmate, stalemate, fifty‑move, threefold repetition, insufficient material), and a tactics trainer over **528 curated CC0 Lichess puzzles** organized by motif. State via **Riverpod**; persistence via **Drift**; layered architecture (*Presentation → Application → Domain → Data*). It carries **127 tests**.

### 3.3 The merge model — "draft‑until‑green," two independent gates

Every PR opens **as a draft** and can only be promoted to "ready" (and thus merged) when **both** gates are green:

1. **`no-mistakes` — an AI‑driven local pipeline.** A staged pipeline where each stage can auto‑fix and re‑run:

   `intent → rebase → review → test → document → lint → push → pr → ci`

   The **review** stage is the core gate: it performs an AI code review, *"parks at a gate where findings are resolved (fix / approve / escalate to a human),"* then the fixed commits flow through the remaining stages and land on the PR. Findings are recorded in the PR with **severity markers** — `⚠️` (warning), `ℹ️` (info/latent) — alongside `🔧 Fix:` lines naming the auto‑fix commit and `✅ Re-checked` confirmations, plus a graded **Risk Assessment**.

2. **GitHub Actions CI — the `validate` job** (`.github/workflows/ci.yml`). Triggers on `pull_request → main` and `push → main`, with concurrency cancellation of superseded runs. Steps: checkout → set up Flutter `3.41.6` → `flutter pub get` → **`flutter analyze`** → **`flutter test`** → **`flutter build web`**. As a required status check, *"a PR cannot be merged until it is green."*

A guard script — **`scripts/mark-ready.sh <pr>`** — flips a draft to *ready* **only if** CI `validate == SUCCESS` **and** the `no-mistakes` run actually passed (review + test completed, not skipped); otherwise it refuses. The PR template enforces a matching pre‑merge checklist (*"CI green; no‑mistakes reported checks‑passed; I reviewed the diff myself — Wait for green."*).

*Design note:* the convention cleverly exploits a platform invariant — **a draft PR can never be merged** — to enforce gating without paid branch protection. It was introduced after two earlier PRs merged before validation finished, shipping un‑validated code.

### 3.4 Risk‑based escalation to human review

The review gate resolves each finding via **fix / approve / escalate‑to‑human**, and every PR carries a pipeline‑generated **Risk Assessment** with a graded level. **Low‑risk, well‑bounded, fully‑fixed changes proceed hands‑off; only high‑risk findings escalate to a human.** (The model documents escalation‑to‑human as the explicit safety valve rather than enumerating a fixed numeric threshold; the live example below was graded *Low* and did not escalate.)

### 3.5 Walkthrough — [PR #12](https://github.com/aryan-nandal/Super-Chess/pull/12)

**Title:** *"feat(tactics): add a motif picker to the tactics trainer."* **+274 / −53 across 8 files, 6 commits**, merged to `main`.

**The change:** a horizontal chip row to filter tactics puzzles to a single motif (mate‑in‑1/2, fork, pin, skewer, discovered attack, deflection, back‑rank, win‑material, sacrifice), new controller state (`motifs` / `selectedMotif` / `setMotif`), a `_set` helper to route state emissions, a generation counter to guard against stale loads, and a refactor decoupling startup from Drift (a Drift‑free `PuzzleAssets`) while switching the shipping bundle to the curated 528‑puzzle CC0 library.

**How the agent review behaved** (from the PR's pipeline section):

- **Review — "3 issues found → auto‑fixed (2)":**
  1. `⚠️` *Missing motif labels* — the picker rendered a chip per theme, but the label map lacked entries for `deflection` (76 puzzles) and `sacrifice` (97 puzzles), so those chips and **47 puzzle titles showed raw camelCase keys.** → fixed.
  2. `⚠️` *Feedback‑persistence regression* — the new `_set` defaulted `feedback` to null, so selecting/deselecting a piece cleared the *"Correct — keep going"* message in multi‑move puzzles — a behavior change. → fixed.
  3. `ℹ️` *Latent stale‑load race* — `setMotif`/`nextPuzzle` fire `_loadNext()` without awaiting/cancelling; safe with the in‑memory repo but a latent race with the Drift‑backed repo. → flagged *"latent; no action needed for the current default."*
  - After re‑check, **one more warning surfaced:** `⚠️` with a motif chip selected, the title could name a *different* motif than the chip, because a puzzle can contain multiple themes — affecting *"177/528 (~33%) of puzzles."* → fixed; *"✅ Re‑checked — no issues remain."*
- **Test — passed:** added tests (`setMotif filters puzzles`, `setMotif(null) returns to all`, `exposes available motifs sorted`, *"intermediate‑move feedback survives selecting/deselecting,"* title‑matches‑selected‑motif); **full suite green 127/127**; a temporary widget test rendered the live screen to PNGs, then was removed.
- **Document / Lint / Push — passed.**
- **Risk Assessment — `✅ Low`:** *"Well‑bounded feature plus a behavior‑preserving asset refactor; all four prior‑round findings are correctly fixed with tests, and a full pass surfaced no new bugs, races, or regressions."* → **no escalation to human review.**

This is the system working as intended: the agent caught four genuine, non‑trivial defects (including a UX regression and a real concurrency hazard), fixed them with accompanying tests, verified a green build, and self‑assessed the residual risk as low enough to merge without human review.

### 3.6 Honest scope

- The agent review runs **locally** (driven by an AI coding agent) and its findings are **recorded in the PR description**, rather than as GitHub‑native review objects; the GitHub‑side gate is the `validate` CI job. The repo does not name a specific model/provider for the reviewer.
- Risk grading is expressed as a reasoned verdict with an escalation path, not a fixed numeric rubric.

---

## Cross‑cutting principles

What stayed constant across all three projects — and is, to me, the real point:

- **Test against reality.** Real backend over mocks (Pomodoro Planner); run and screenshot the deployed app, not just the suite (Super Sudoku); a green build is necessary but never sufficient.
- **Architecture built to change.** A pure, dependency‑free domain core with external systems (Firebase, persistence) hidden behind seams — in all three.
- **Determinism and isolation.** Controlled starting state, disposable environments, one concern per PR.
- **Human‑in‑the‑loop by design.** Automation handles execution and the easy decisions; a human owns the gates — and, in Super Chess, is summoned *automatically and only* when risk is high.
- **Reproducibility and low cost.** Docker‑only test runs; zero‑cost scripted deploys on free tiers.

## Progression at a glance

| | Pomodoro Planner | Super Sudoku | Super Chess |
|---|---|---|---|
| **Adds** | The E2E integration testing system *(the deliverable)* | AI automation across the full SDLC, TDD‑first | Agent‑driven code review + risk‑gated CI |
| **Backend testing** | Real app vs. Firebase Emulator in Docker | 4 verification layers + live smoke test | CI `validate`: analyze + test + build |
| **Process** | One‑command local run | Single operator + AI agent; gated PRs | Two‑gate "draft‑until‑green"; risk‑based human escalation |
| **CI** | Architecturally CI‑ready | Scripted per‑PR | **GitHub Actions** (`ci.yml`) |
| **Tests** | Auth + tasks E2E flows | 137 across 4 layers | 127 |
| **Stack** | Flutter · Firebase Auth/Firestore | Flutter · Riverpod · Drift · Firebase/Functions | Flutter · Riverpod · Drift · pure‑Dart engine |

---

## Links

- **Pomodoro Planner** — <https://github.com/aryan-nandal/pomodoro_planner> · [E2E design doc](https://github.com/aryan-nandal/pomodoro_planner/blob/main/docs/e2e-integration-testing.md)
- **Super Sudoku** — <https://github.com/aryan-nandal/Super-Sudoku> · [Pipeline design doc](https://github.com/aryan-nandal/Super-Sudoku/blob/main/docs/PRODUCT_AUTOMATION_DESIGN.md)
- **Super Chess** — <https://github.com/aryan-nandal/Super-Chess> · [Merge‑flow doc](https://github.com/aryan-nandal/Super-Chess/blob/main/docs/AUTOMATED_MERGE_FLOW.md) · [Sample PR #12](https://github.com/aryan-nandal/Super-Chess/pull/12)
