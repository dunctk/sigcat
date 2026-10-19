# SigCat — Implementation tracker

> **Source of truth for implementation status.** Product requirements: [PRD.md](PRD.md). Contributor rules: [../AGENTS.md](../AGENTS.md).
>
> **Status as of 2026-10-19:** planning / feasibility not yet validated. The original MIT licence, README, PRD and AGENTS instructions exist; a working SigCat receiver, web app, Docker deployment and speech engine have **not** been verified.

## At a glance

| Field | Current value |
|---|---|
| Current milestone | **M0 — Feasibility** |
| Next task | **M0-01** — document reference hardware + reproducible test environment |
| Critical path | SoapySDR/Seify inside unprivileged Linux Docker → FutureSDR IQ/FFT → verified CPU local ASR → go/no-go ADR |
| Blocking external dependency | Real Linux host + RTL-SDR required to verify physical-device support; replay tests can run without it |
| Overall state | **Not started** (documentation alone is not implementation) |
| Last reviewed | 2026-10-19 |

**How to read this tracker:** all task boxes start unchecked. `[x]` means *Done with evidence*, not “some code exists”. A task with work underway stays `[ ]` and is marked `In progress` beside its line. `Blocked` always includes the blocker and a concrete unblock condition.

## M0 — Feasibility (must pass before M1)

**Exit gate:** document the actual hardware/driver/build versions and reproducible proof that minimally privileged Docker USB acquisition, FutureSDR DSP, an offline speech fixture, and SQLite/HTMX integration work—or record precise blockers/fallbacks. Review PRD §17.

- [ ] **M0-01 — Reference setup.** Record Linux distro/kernel, CPU/RAM, RTL-SDR model, USB permissions, build/toolchain pins, drivers and exact test commands. **Evidence:** pending.
- [ ] **M0-02 — Receiver in Docker.** Prove Seify/SoapySDR RTL-SDR acquisition without `privileged: true` or Docker socket; handle absent device; add deterministic IQ replay. **Evidence:** pending.
- [ ] **M0-03 — FutureSDR flowgraph.** Demonstrate FFT/spectrum from real IQ and replay, retunes, sample-loss telemetry, graceful shutdown and bounded memory. **Evidence:** pending.
- [ ] **M0-04 — Offline speech feasibility.** Benchmark Parakeet + NeMo-Speech.cpp on CPU with redistributable authorised fixture; pin model/runtime/hash/licence and record accuracy, latency, RAM. **Evidence:** pending.
- [ ] **M0-05 — Web and data smoke.** Demonstrate Axum + HTMX round trip, SQLx migration, SQLite WAL and FTS5 read/write/search after restart. **Evidence:** pending.
- [ ] **M0-06 — Go/no-go report.** Commit `docs/adr/0001-m0-feasibility.md` listing measured support, open blockers, rejected alternatives and next steps. **Evidence:** pending.

## M1 — Headless receiver and browser

**Entry gate:** M0 completed with credible evidence.

- [ ] **M1-01 — Rust workspace and CI.** Runnable `sigcatd`, tests/lint, deterministic fixtures, pinned dependency builds. **Evidence:** pending.
- [ ] **M1-02 — Compose deployment and persistent paths.** Non-root where feasible; least-privilege USB, database/media/model volumes; health checks. **Evidence:** pending.
- [ ] **M1-03 — Secure access.** First-run admin, session protection, localhost-only default, documented opt-in LAN and Tailscale Serve (never Funnel). **Evidence:** pending.
- [ ] **M1-04 — Receiver control.** Capabilities, single-owner lease/scheduler, tuning, safe reconnect and diagnostics. **Evidence:** pending.
- [ ] **M1-05 — Live spectrum and demo.** Bounded waterfall updates in browser; genuine receiver and clearly labelled replay/demo modes. **Evidence:** pending.

## M2 — Signal discovery, history and search

- [ ] **M2-01 — Scan plans and true coverage.** Dwell/retune intervals, sample bandwidth, missed-coverage gaps, hardware validation. **Evidence:** pending.
- [ ] **M2-02 — Detection and cautious classification.** Noise baseline, event segmentation, immutable observation, unknown-first labels and algorithm provenance. **Evidence:** pending.
- [ ] **M2-03 — SQLite evidence and FTS5.** Schema, migrations, consistent indexes, 100k fixture, time/frequency/text search and cursor pagination. **Evidence:** pending.
- [ ] **M2-04 — Investigation UI.** Dense event list, filters, detail pages, annotations, metrics and signal history. **Evidence:** pending.
- [ ] **M2-05 — Selective SigMF IQ capture.** Recording disabled by default; short recordings, hashes, quotas, retention and replay. **Evidence:** pending.

## M3 — Protocols and local speech

- [ ] **M3-01 — rtl_433 adapter.** Versioned, isolated protocol worker; validation and searchable structured decode; deterministic fixture. **Evidence:** pending.
- [ ] **M3-02 — Authorised audio/VAD.** Explicit enabled-by-user policy, demodulated clip path, VAD, queue limits, storage and purge. **Evidence:** pending.
- [ ] **M3-03 — ASR backend and searchable transcripts.** Local, pluggable, bounded inference; transcript provenance and FTS5 linking to clip. **Evidence:** pending.
- [ ] **M3-04 — RF-audio model Pareto benchmark.** Same fixtures across models; WER/CER, SNR, false transcription, CPU, RAM, latency and licence review. **Evidence:** pending.
- [ ] **M3-05 — Optional translation (P1).** Original-language text preserved; separately versioned offline translation where supported. **Evidence:** pending.

## M4 — Operations and hardening

- [ ] **M4-01 — Recovery and retention.** USB disconnect, worker crash, DB busy, disk full, prune/pin reliability. **Evidence:** pending.
- [ ] **M4-02 — Backups and exports.** Consistent SQLite/media backup/restore, JSON/CSV/SigMF exports with provenance. **Evidence:** pending.
- [ ] **M4-03 — Security, soak and remote validation.** Auth/CSRF/sandbox tests; measured 24h SDR soak, two-device tailnet test and documented limitations. **Evidence:** pending.
- [ ] **M4-04 — Hardware and model expansion (P1/P2).** ARM64 and additional SDRs/backends only after independent validation. **Evidence:** pending.

## Task lifecycle and agent handoff

1. **At start of every coding session:** read [AGENTS.md](../AGENTS.md) and this file; choose the first unblocked task, check PRD requirements and any relevant ADR.
2. **When starting work:** set the task to `In progress` (keep `[ ]`), and update **At a glance / Next task** if necessary. Avoid two agents claiming the same task; use a GitHub issue/PR for coordination when working concurrently.
3. **During work:** build a small end-to-end slice. Write repeatable tests or document a manual hardware validation procedure and exact environment.
4. **Before marking done:** link concrete proof beside the task (PR/commit and tests, benchmark output or hardware report). Only then change `[ ]` to `[x]`. Never substitute a mock, screenshot or third-party benchmark for hardware proof.
5. **At end of every coding session:** update status, blockers, tested/unverified distinctions and **Next task**. If no change was completed, record what was tried and the next unblock step. Keep the README status factual.
6. **In each pull request:** mention task IDs, tests executed/skipped, tracker changes and any PRD divergence (create an ADR for substantive decisions).

**Progress reporting:** calculate completion from evidenced checked tasks, not lines of code, commit count or subjective percentage. Milestone completion requires all mandatory tasks and their exit gate; P1/P2 options are explicitly exempt if deferred in the ADR.

## Decision and evidence log

Add dated concise entries here when something materially changes; link the source instead of copying long logs. Each entry should include task ID, outcome, evidence and the resulting next action.

| Date | Task | Outcome / evidence | Next action |
|---|---|---|---|
| 2026-10-19 | Planning | PRD and engineering rules documented; no hardware or implementation validated | Begin M0-01 |

## Rules for scope changes

The [PRD](PRD.md) owns product requirements. This file owns implementation status. [AGENTS.md](../AGENTS.md) owns coding-agent operating rules. Do not duplicate long requirements here or check boxes because a design document exists. New tasks must use stable IDs, cite the relevant PRD requirement and state verifiable evidence. If requirements or architecture change, record an ADR and update the PRD before silently re-scoping this list.
