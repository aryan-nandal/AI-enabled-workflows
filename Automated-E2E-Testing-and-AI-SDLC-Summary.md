# Automated End‑to‑End Integration Testing — and How Far AI Can Take the Product Lifecycle

**Author:** Aryan Nandal  ·  **Date:** 30 June 2026  ·  *Executive summary (a companion technical report covers the architecture in depth.)*

---

## Context

The brief was to design and build an **automated end‑to‑end integration testing system** that exercises a frontend and a backend together. I delivered that first. I then used the momentum to investigate a question I was genuinely curious about: **how much of the entire product‑development lifecycle can one engineer drive with an AI coding agent — without giving up engineering rigor?** That produced two further projects, each going a layer deeper. This document leads with the deliverable, then walks the progression.

---

## 1 · The deliverable — Pomodoro Planner: the E2E integration testing system

**[Repo](https://github.com/aryan-nandal/pomodoro_planner) · [Design doc](https://github.com/aryan-nandal/pomodoro_planner/blob/main/docs/e2e-integration-testing.md)**

A cross‑platform **Flutter + Firebase** productivity app (tasks, Pomodoro timer, statistics, accounts) serves as a realistic vehicle. The real deliverable is the testing system around it: it operates the **actual app** against an **actual backend** and proves a feature works *holistically* — sign up, write to the database, see the UI reflect it.

Key design decisions:

- **Real backend, not mocks.** Runs the real app against the **Firebase Emulator Suite** (genuine Auth + Firestore, local). The app doesn't know it's an emulator — it's redirected at runtime.
- **Fully containerized.** The entire toolchain (Flutter, JDK 21, Node 20, Firebase CLI, headless Chromium + ChromeDriver) lives in **Docker**. The only host dependency is Docker, so runs are hermetic and reproducible across machines and CI.
- **Deterministic.** The database is **wiped and re‑seeded from JSON fixtures before every test** — control the starting state, control the outcome.
- **Production‑safe.** The app connects to the emulator *only* when compiled with `--dart-define=USE_EMULATOR=true` (defaults to `false`), so a production build can never point at a test backend.
- **Per‑test isolation.** Each test runs in its own disposable (`--rm`) container with its own emulator — already parallel‑ready.
- **Evidence.** Screenshots are captured at each key step and written back to the host, making pass/fail tangible and failures debuggable.

**What it proves:** it covers the genuinely hard parts that unit and widget tests skip — real authentication, Firestore reads/writes, async timing, serialization, and the UI reflecting real backend state. Verified flows include an **auth flow** (register → land on home) and a **tasks flow** (sign in → see seeded tasks → complete one → create a new one → confirm it persists). One command runs everything: `./run_integration_tests.sh`.

**Honest scope:** web‑first, CI‑ready by architecture (Docker‑only); the main remaining flakiness risk is fixed timing waits, which I'd replace with widget‑polling.

---

## 2 · Going further — Super Sudoku: AI across the full SDLC, test‑first

**[Repo](https://github.com/aryan-nandal/Super-Sudoku) · [Pipeline design doc](https://github.com/aryan-nandal/Super-Sudoku/blob/main/docs/PRODUCT_AUTOMATION_DESIGN.md)**

With the testing system delivered, I wanted to see whether the same discipline could wrap the *entire* lifecycle. Super Sudoku is a Flutter Sudoku app built strictly test‑first; the real artifact is the **documented pipeline** behind it — a single human operator acting as architect/PM/reviewer plus one AI coding agent doing execution, spanning **Market Analysis → Requirements → Architecture → Engineering (TDD) → Verification → Delivery**.

- **TDD by default** — red → green → refactor; **137 automated tests** across four layers (domain unit, application/controller, widget, and end‑to‑end + "actually run it").
- **Rigor over vibes** — *"tests green is necessary, not sufficient."* A live smoke test caught a Firebase initialization bug that a fully green suite had missed.
- **Disciplined delivery** — one concern per PR, no direct commits to `main`, human review at every gate; scripted, zero‑cost deploy (`flutter build web && firebase deploy`) on the free plan.

---

## 3 · Going deeper — Super Chess: agent‑driven review + risk‑gated CI

**[Repo](https://github.com/aryan-nandal/Super-Chess) · [Merge‑flow doc](https://github.com/aryan-nandal/Super-Chess/blob/main/docs/AUTOMATED_MERGE_FLOW.md) · [Sample PR #12](https://github.com/aryan-nandal/Super-Chess/pull/12)**

Reusing that product‑design flow, Super Chess (a teaching‑first chess app with a full rules engine and a 528‑puzzle tactics trainer) adds the layer I most wanted to test: **can code review itself be automated so a human only looks when risk is genuinely high?**

The model is a **two‑gate, "draft‑until‑green" merge convention** — both gates must be green before a PR can leave draft:

1. **An AI‑driven local pipeline (`no-mistakes`):** intent → rebase → **review** → test → document → lint → push → PR. The review stage flags issues by severity, **auto‑fixes what it safely can, re‑checks**, and produces a **graded risk assessment**, escalating to a human **only on high risk**.
2. **A GitHub Actions CI job (`validate`):** static analysis + tests + web build. A guard script promotes a draft PR to "ready" only when **both** gates pass.

**Concrete proof — [PR #12](https://github.com/aryan-nandal/Super-Chess/pull/12)** ("add a motif picker to the tactics trainer"): the agent review caught **four real issues** — two missing UI labels, a feedback‑persistence regression, a latent stale‑load race, and a title/selected‑motif mismatch affecting ~33% of puzzles — **auto‑fixed them with new tests**, landed a green **127/127** suite, captured screenshots as evidence, and graded the change **Low risk**, so it merged **without needing human review**.

---

## What this demonstrates

- **I can build the asked‑for system:** a robust, reproducible, production‑safe E2E harness that tests frontend and backend together *for real*, not against mocks.
- **And the process around it:** I can design how AI‑assisted development *ships* so that speed never costs correctness — test‑first, verified against the running app, gated, and reviewed by an agent with a human in the loop only where the risk actually warrants it.

| Project | What it adds | Stack |
|---|---|---|
| **Pomodoro Planner** | The E2E integration testing system (the deliverable) | Flutter · Firebase Auth/Firestore · Firebase Emulator Suite · Docker |
| **Super Sudoku** | AI across the full SDLC, strictly TDD | Flutter · Riverpod · Drift · Firebase/Cloud Functions |
| **Super Chess** | Agent‑driven code review + risk‑gated CI | Flutter · Riverpod · Drift · GitHub Actions |

**Links:** [Pomodoro Planner](https://github.com/aryan-nandal/pomodoro_planner) ([E2E design](https://github.com/aryan-nandal/pomodoro_planner/blob/main/docs/e2e-integration-testing.md)) · [Super Sudoku](https://github.com/aryan-nandal/Super-Sudoku) ([pipeline design](https://github.com/aryan-nandal/Super-Sudoku/blob/main/docs/PRODUCT_AUTOMATION_DESIGN.md)) · [Super Chess](https://github.com/aryan-nandal/Super-Chess) ([merge flow](https://github.com/aryan-nandal/Super-Chess/blob/main/docs/AUTOMATED_MERGE_FLOW.md), [PR #12](https://github.com/aryan-nandal/Super-Chess/pull/12))
