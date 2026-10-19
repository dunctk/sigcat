# SigCat — Product Requirements Document

**Version:** 1.0 (planning baseline)  
**Date:** 2026-10-19  
**Status:** Proposed; no implementation or validated device support yet  
**Product:** SigCat — local-first, open-source radio signal observatory  
**Repository:** https://github.com/dunctk/sigcat  
**Licence:** MIT for original SigCat source (upstream libraries, model weights, and bundled binaries retain their own licences)

> SigCat turns lawfully monitored radio-frequency activity into a useful, searchable local history. It is a receiver and analysis system, not a transmitter, interception bypass, or universal decoder.

## 1. Executive summary

SigCat is a self-hosted receive-only SDR application. Connect a supported receiver to a Linux computer, launch a Docker Compose deployment, and use a dense, responsive web interface either on that computer, on another LAN device, or privately across Tailscale. SigCat shows spectrum/waterfall activity, scans selected bands, detects candidate emissions, characterises them conservatively, invokes suitable protocol decoders, and records observations in SQLite. When the operator is authorised to process a voice transmission, SigCat can run entirely local speech activity detection, recognition and optional translation. It stores compact structured records and selectively retains audio or IQ evidence for inspection and replay.

The core is Rust, Tokio, FutureSDR, Seify with a SoapySDR-first hardware backend, Axum, HTMX, Tailwind, SQLx and SQLite FTS5. Speech inference uses a pluggable local backend; investigate NeMo-Speech.cpp with Parakeet TDT 0.6B v3 as the first balanced candidate. Whisper is not a mandatory dependency. Model choice is an empirical, hardware- and RF-audio-specific Pareto decision, not a fixed marketing claim.

**Product promise:** all essential capture, decoding, indexing, search and authorised speech inference can work without cloud access after images/models have been provisioned. Acquisition survives browser disconnects and inference overload.

**Not a promise:** decoding unknown or encrypted protocols; simultaneous coverage of arbitrary frequencies using one RTL-SDR; speech recognition on every transmission; compatible USB passthrough on Docker Desktop; or predictable accuracy without real RF fixtures.

## 2. Users, jobs and guiding principles

### Personas

1. **Radio hobbyist / researcher:** understands frequencies; wants rapid discovery and replay of interesting local activity.
2. **Environmental / sensor observer:** wants searchable rtl_433 telemetry with time series and event evidence.
3. **Open radio-data enthusiast:** explores appropriately received and used ADS-B, APRS, AIS or satellite broadcasts via optional adapters.
4. **Home-lab operator:** leaves a Linux mini-PC running, checks it from a Mac/Windows laptop or phone through a browser, including Tailscale.
5. **Developer / data analyst:** wants a stable API, reproducible captures, configurable rules and bulk structured exports.

### Jobs to be done

- See what frequencies and signal classes were observed during a specific period.
- Distinguish a repeatable emission from background noise and receiver artifacts.
- Follow an observation through measured RF evidence, classifier outputs and protocol results.
- Search structured decodes, operator notes and eligible speech transcripts in one interface.
- Replay a retained audio/IQ segment and inspect the detection/classification chain.
- Tune a remote SDR safely without an SSH session.
- Operate autonomously without paying for a cloud service.

### Product principles

- **Evidence before labels:** an unknown signal remains unknown; never conflate modulation guesses with successfully decoded protocols.
- **Local by default:** no third-party speech, analytics, telemetry, recording or remote APIs.
- **Efficient by design:** record observation metadata continuously, media selectively; enforce storage quotas.
- **Clear coverage:** show scan dwell times, retune gaps, bandwidth and missed-observation limitations.
- **One hardware owner:** a scheduler arbitrates receiver access; decoders never independently retune the same device.
- **Graceful degradation:** receiver capture outranks visualisation, decoders, ASR, translation and enrichment in that order.
- **Replaceable backends:** do not couple domain entities to FutureSDR, SoapySDR or any single inference runtime.
- **Safe by default:** receive-only, private network binding, first-run authentication, no public web exposure or hidden recording.

## 3. Goals and success criteria

### MVP outcomes

- Deploy on a supported Linux x86-64 host through Docker Compose with mapped SDR USB device and persistent volumes.
- Open authenticated web UI on localhost, then optionally through a deliberate LAN binding or tailnet-only Tailscale Serve.
- Select receiver, inspect supported capabilities, tune an allowed frequency and watch a live spectrum/waterfall.
- Create a scan plan; observe a representative activity signal; produce bounded, deduplicated observations.
- Persist events and decode at least one well-documented open telemetry protocol via rtl_433.
- Search 100,000 test observations by time, frequency, tags and FTS query with benchmarked performance.
- Process one operator-provided, authorised multilingual audio fixture fully offline into timestamped transcript and, if supported, English translation.
- Export a selected event and its SigMF metadata or JSON without exporting unrelated records.
- Restart the container and browser without losing history, settings or indexing integrity.

### Definition of complete

A checked acceptance test, reproducible hardware/software manifest, documented fixture or automated test, and visible user-facing error state for each MVP requirement. README must not mark a feature as delivered merely because it appears in this PRD.

### Non-goals (initial releases)

- Transmitting RF or controlling transmit capability.
- Circumventing encryption, access controls, digital privacy or radio-system authentication.
- Capturing/recording private conversations without authorisation.
- Global broadband continuous surveillance or mathematically complete RF coverage.
- Automatic universal modulation/protocol classification.
- Replacing laboratory spectrum analysers for calibrated field strength measurements.
- Public multi-tenant SaaS, billing, mobile-native apps or a cloud-only deployment.
- Always-on raw IQ archives.
- Real-time many-language translation on low-end CPU hardware.

## 4. Scope and delivery priorities

| Priority | Capability | Exit evidence |
|---|---|---|
| P0 | Linux Docker Compose, local browser, volumes, authenticated admin | clean install and restart tests |
| P0 | RTL-SDR tuning, sample rate/gain, capability discovery and stream health | physical receiver test |
| P0 | FFT and waterfall with bounded live updates | synthetic and real-stream test |
| P0 | Receiver ownership and scan-plan scheduler | concurrency/retune conflict tests |
| P0 | Activity segmentation and observation persistence | golden IQ fixture |
| P0 | Event list, detail and query/filter UI | 100k row fixture |
| P0 | SQLite + FTS5 + configurable quotas/retention | migration and prune tests |
| P0 | SigMF-compatible selected IQ capture/replay | round-trip fixture |
| P0 | One decoded telemetry protocol through adapter | rtl_433 fixture |
| P0 | Offline VAD and STT on user-provided authorised audio | local model fixture |
| P0 | Restricted LAN and Tailscale Serve documentation | independent device test |
| P1 | Expanded protocol adapters, speech translation and model benchmarking | RF audio benchmark |
| P1 | Multi-SDR workers and scan orchestration | multiple-device test |
| P1 | Per-rule notifications, user bookmarks and custom classification | deterministic tests |
| P1 | JSON/CSV/SigMF exports and stable read-only API | contract tests |
| P2 | Vector similarity, signal clustering, maps and richer analytics | measured value vs overhead |

P0 speech means an end-to-end *fixture and manual on-demand path*, not unrestricted bulk capture or universal live transcription; a live authorised audio path and translation are eligible for P1 when feasibility is proved.

## 5. Deployment requirements and host topology

### Supported deployment

**Primary:** Linux amd64. **Secondary:** Linux arm64, subject to packaged SDR drivers and speech-model runtime compatibility. Supported browsers: current desktop and mobile Chrome/Chromium, Firefox and Safari. Browser client OS can be Windows, macOS or Linux. The SDR host itself is Linux for MVP.

Topology:

    RTL-SDR USB → SoapySDR/Seify → FutureSDR graph → event bus
                                                  ├→ spectrum stream
                                                  ├→ activity detector
                                                  ├→ protocol adapters
                                                  ├→ authorised audio/VAD/ASR
                                                  └→ bounded IQ/audio recorder
                                              → SQLite + media directory
                                              → Axum/HTMX browser UI
                                              → localhost | opted-in LAN | Tailscale Serve

### Docker and storage

- Produce reproducible multi-stage container builds; pin Rust toolchain, system packages and native third-party dependencies.
- Provide a Compose configuration exposing internal TCP 8080 as host **127.0.0.1:8080** by default. Opt-in host bind address for LAN access, documented as an increased attack surface.
- Persistent directories: /data (SQLite and application state), /recordings (IQ/audio), /models (downloaded local models), /config (configuration); all may be mapped to user-owned paths.
- Explicitly pass only necessary USB bus/device nodes for RTL-SDR; enumerate Linux udev permissions/group handling. Do **not** require privileged mode or mounting Docker socket.
- Container must run with dropped Linux capabilities and a non-root application user where technically feasible; if SDR permissions conflict, document the narrowest working policy.
- Health checks: liveness, readiness (database migrated and server listening), receiver status separately. Booting the web app without a dongle must still succeed in demo/replay mode.
- Gracefully handle unplug/replug, USB driver conflicts, Soapy version mismatches and recoverable decoder process exits.
- No internet requirement for normal operation. Downloads of images/models are explicit install-time steps with hashes and licence acceptance when necessary.
- No server-side GPU assumption. Optional GPU-specific worker profiles are independently switchable and never block CPU-only installations.
- Run unattended under Docker restart policy after reboot; allow scheduled, consistent SQLite backup.

### Network modes

1. **Local (default):** browser on host via http://127.0.0.1:8080.
2. **LAN (opt-in):** administrator changes binding/firewall and uses http://host-lan-ip:8080; require authentication and strongly recommend HTTPS reverse proxy for nontrusted LANs.
3. **Tailnet (recommended remote):** host installs Tailscale and uses **Tailscale Serve**, which proxies to 127.0.0.1:8080 with tailnet-only HTTPS. Require existing SigCat login; do not trust unverified forwarded identity headers. Document MagicDNS/HTTPS and ACL requirements. **Never enable Tailscale Funnel by default.**

The preferred Tailscale setup is a host-side Serve configuration, not an extra privileged sidecar or an automatic public port.

## 6. RF hardware, capabilities and scheduling

### Device abstraction

Define ReceiverRegistry, Receiver, ReceiverSession and SampleStream interfaces independent of backend-specific types. Every driver advertises ranges of supported frequency, sample rate, gain modes, antenna, timestamp capability and RX channels. Never present unsupported capabilities as editable controls.

Use Seify with SoapySDR as initial RTL-SDR path. Retain a documented adapter boundary to evaluate native Seify drivers. Open and close sessions safely with idempotent shutdown; detect device identity without assuming a permanently stable USB path.

### One-owner receiver scheduler

A receiver is exclusively leased to one orchestrated capture flowgraph. Scanning, fixed tuning, recording, adapters and web control submit commands to the same scheduler. Apply commands at graph-safe transition points; expose current assignment, queue and estimated dwell duration. Make manual override explicit and auditable.

States: disconnected, idle, acquiring, retuning, recording, degraded, faulted. Recover faults with bounded backoff. Commands carry idempotency keys, cancellation and timeout handling.

### Scan plans

Each plan defines ordered frequency ranges or channels; dwell time; step size; sample rate; gain; demodulation/detection profile; schedule; optional per-band rules; and recording policy. Validate against advertised device capabilities and legal-use setting. Permit fixed monitoring and sequential stepping. Show current centre frequency, instantaneous sample bandwidth, planned coverage and gaps. Log observed tuning intervals rather than implying coverage between them.

A common RTL-SDR sample rate near 2.4 MS/s provides only a few MHz of instantaneous visibility; retuning sacrifices continuity. With unsigned 8-bit IQ, estimated raw size at 2.4 MS/s is 4.8 MB/s, 17.28 GB/hour, 414.72 GB/day (decimal units); these are estimates excluding headers and compression.

### DSP and quality

- FutureSDR flowgraph for complex sample ingestion, DC-offset correction, optional frequency translation, FFT, decimation and demodulation as supported.
- Windowed spectral estimates; configurable FFT size, averaging and update interval; avoid pathological allocations on every frame.
- Adaptive noise floor with guardrails, receiver saturation/overload indication, candidate bandwidth/power estimation.
- Timestamp samples with monotonic and wall-clock references; distinguish SDR hardware timestamps from approximate host timestamps.
- Never describe uncalibrated received power as absolute dBm.
- Expose dropped samples, buffer occupancy, retunes, clipping and sample discontinuities.

## 7. Signal detection, classification and discovery

### Hierarchy of evidence

1. **Spectral detection:** energy above estimated noise; candidate centre, width, duration and SNR proxy.
2. **Characterisation:** measurable traits such as occupied bandwidth, duty cycle, periodicity, amplitude variation and spectral shape.
3. **Modulation hypothesis:** e.g. AM/FM/FSK/ASK/PSK, with model/rule version and uncertainty.
4. **Protocol candidate:** tentative matching decoder, if any.
5. **Verified decode:** protocol-specific payload successfully parsed and validated, with adapter provenance.
6. **Human annotation:** distinct field, never overwrite machine evidence.

Unknown/unsupported signals must be first-class items. Detectors are not permitted to invent protocol labels or infer people/intent from RF measurements.

### Event lifecycle

A raw candidate begins when a configurable threshold is met, accumulates evidence during its active window, and ends by hysteresis/timeout; bursts can be merged within a configurable gap. Create an immutable observation with RF measurements, receiver session, scan-step provenance, software versions and optional media references. Link recurrent observations to an inferred signal identity through versioned clustering; do not treat cluster identity as ground truth.

Compute an explainable interest score from novelty, strength relative to local noise, persistence, known/unknown decode state, match to explicit watch rules and data completeness. Each feature and weight must be inspectable/editable. Distinguish prioritisation from security risk or threat attribution.

## 8. Protocol decoding architecture

Adapters implement: input sample/audio format, frequency constraints, demodulation prerequisites, lifecycle, configuration schema, output schema and errors. Worker isolation must protect the main process from decoder crashes; use bounded pipes/queues and resource limits. Record decoder executable/container version and raw decoder output alongside canonicalized fields.

**First adapter:** rtl_433 for compatible unencrypted ISM-band sensor messages. Its many supported devices are heterogeneous; configure allowed protocols rather than promiscuously declaring every JSON line valid.

**Candidates (not MVP commitments):** read-only ADS-B decoder, APRS packet decode, AIS, selected legally receivable satellite telemetry. Validate antenna, bandwidth, station permissions and source-specific downstream data restrictions separately. Do not ingest every location-related protocol by default.

Canonical decode envelope:

- event_id, observation_id, adapter_id, decoder_version
- protocol_id, event_time_utc, frequency_hz
- raw_payload_reference, decoded_json, validation_status
- source_hash, confidence_if_meaningful, error_if_any

Do not automatically send aircraft, vessel, telemetry or location data to external aggregation networks.

## 9. Authorised audio, speech recognition and translation

### Gatekeeping and data minimisation

Audio/transcript capture is **disabled by default**. Activation requires operator confirmation that monitoring, retention and transcription are lawful for the selected signal. Provide per-profile permission/recording policies, visible recording status, short default clip retention and immediate deletion. Avoid presenting the feature as automatic private-conversation interception. Do not transcribe protected or encrypted services.

### Pipeline

    approved demodulated audio (with time/frequency provenance)
      → resample/normalise; optional noise filtering
      → local VAD / squelch-aware segmentation
      → bounded audio queue and persisted job state
      → selected local ASR backend
      → original text + word/segment timing + language/provenance
      → optional local text translation job
      → index in FTS5 and expose with original clip

Important constraints:

- Speech tasks cannot block IQ acquisition or the web event writer.
- Clip length, queue depth, concurrency, max runtime and memory are configured for the host hardware.
- Persist jobs as pending/running/completed/failed/skipped, with cancellation and bounded retries. Restore interrupted work on restart without duplicate transcript rows.
- Unknown language, non-speech and insufficient quality may return no transcript. Show uncertain outputs as hypotheses, never silently trusted statements.
- Preserve original-language transcript, translated text and per-model metadata separately. Translation must never overwrite original.
- Default to **on-demand batch transcription** for known authorised clips; streaming is an optional independently benchmarked profile.

### Model backend strategy

Expose AsrBackend and TranslationBackend traits/capabilities: model ID/revision/hash/licence, accepted sample rate, language list, streaming/batch support, timestamp support, measured RAM/RTF, accelerators and runtime availability.

**Balanced candidate:** NeMo-Speech.cpp with Parakeet TDT 0.6B v3, called through a local loopback process or in-process FFI only after technical feasibility testing. NeMo-Speech.cpp documents Parakeet support but performance depends on exact build, quantisation and hardware. Do not hardcode claimed 6x throughput as a service-level guarantee.

**Lite candidate:** a smaller, permissively licensed local streaming ASR model (e.g. Moonshine English, pending runtime integration validation).

**Extended multilingual:** optional Qwen3-ASR; **translation:** local text translation (e.g. verified permissively licensed language-pair models) or evaluated speech translation backend. **Fallback:** whisper.cpp/whisper-rs where beneficial. Download/activate only selected models. Never assume a particular quantised conversion is available without verified checksums and reproducible provenance.

Do not redistribute third-party model weights in the default MIT software image without checking individual licences and attribution obligations. License of code does not automatically license model weights.

### Pareto evaluation and automatic selection

Benchmark **model + runtime + quantisation + hardware + audio preprocessing** tuples, not bare model names. Publish fixture manifest and scripts. Include clean voice, weak analog FM/AM/SSB, fading, white/impulsive noise, clipping, squelch edges, mixed languages, short callsigns/numbers, non-speech and false-positive prompts.

Measure WER/CER per language, domain-specific exact-entity accuracy (numbers/callsigns), false speech detections, real-time factor, time-to-first-partial (streaming), wall-clock latency, peak RSS, CPU%, package/model bytes, energy proxy where possible, restart/recovery and licences. Use bootstrapped confidence intervals and stratification by SNR. Plot Pareto frontier; avoid cross-paper numeric comparisons as decisive evidence.

Profiles: Lite, Balanced, Full. Auto-select only among installed and benchmark-validated candidates; show why chosen and permit manual override.

## 10. Storage architecture

### Database

SQLite using SQLx migrations, foreign keys, WAL, an explicit busy timeout, transactional event writes and one bounded write queue. Index by integer Hz and UTC microseconds or milliseconds with a documented convention. Use stable IDs and versioned schemas. Read APIs use connection pooling appropriate for SQLite/WAL. FTS5 virtual tables index transcripts, translations, annotation text and selected decode fields; use triggers or transactional synchronisation tests to prevent stale index entries.

Suggested tables:

| Table | Role / key fields |
|---|---|
| receivers | hardware key, capabilities JSON, last_seen |
| receiver_sessions | receiver_id, driver version, start/end, clock quality |
| scan_plans / scan_steps | declarative frequencies, dwell, applied settings |
| coverage_intervals | measured start/end, centre/bandwidth, gaps |
| signal_candidates | extracted features, lifecycle |
| signal_clusters | versioned recurrent identity, operator merge/split |
| observations | immutable UTC start/end, centre_hz, bandwidth_hz, SNR proxy, session |
| classifications | stage, label, algorithm/model revision, score, explanation |
| decoder_runs / decodes | adapter version, status, protocol, validated JSON |
| media_objects | content hash, type, path, bytes, retention state |
| audio_clips / iq_captures | observation linkage, SigMF/media metadata |
| speech_jobs | backend, model revision, state, attempts, timestamps |
| transcripts / transcript_segments | language, original text, timing, provenance |
| translations | source transcript ID, target language, model/version, text |
| tags / notes / event_tags | user annotations |
| rules / alerts | explicit event matching and bounded notifications |
| users / sessions / audit_log | authentication, actions, configuration history |
| schema_migrations / app_settings | application schema/version/state |

Indexes: observations(start_time DESC), observations(receiver_id,start_time DESC), observations(centre_hz,start_time DESC), decodes(protocol_id,event_time DESC), jobs(state,created_at), media(content_hash), and required composite indexes based on query plans. Do not build every possible index prematurely. Use cursor-based pagination, not deep OFFSET scans.

### Media

Do not place long IQ blobs in SQLite. Save IQ as SigMF-compatible captures (metadata and data files), reference immutable manifest/path + SHA-256 in media_objects. Store selected demodulated audio as FLAC/Opus with codec, sample rate, channel count and lawful-use provenance. Atomic file temp-write/fsync/rename followed by committed DB reference; periodically reconcile missing/orphaned files.

At 2.4 million complex samples/s x 2 bytes per complex sample, raw IQ costs 17.28 decimal GB/hour. Default capture should be OFF except explicit events or short pre/post-trigger ring buffer. Implement configurable daily/total byte budget, max clip length, age-based retention and pinned-record exceptions; show projected storage. Pruning is crash-safe, auditable and never silently removes pinned data.

Backup: consistent online SQLite backup/checkpoint policy plus separate media manifest; restore validates schema/version and content hashes. Document that copying only the SQLite file while active may be inconsistent if WAL files are ignored.

### Exports

Event JSON with provenance; filtered CSV for scalar observations; SigMF for IQ; FLAC/Opus for audio; optional JSONL bulk archive. Redact sensitive transcript/audio content for sharing by explicit selection.

## 11. UI/UX requirements

Visual character: an information-dense field notebook / technical instrument, not a card-heavy consumer dashboard. Use compact, accessible sans-serif typography, legible numeric columns, SI-unit formatting and progressive enhancement. HTMX for server-rendered navigation/filtering; small isolated JavaScript components for waterfall rendering and audio playback; SSE for low-frequency state/event feeds, throttled canvas/WebSocket for high-rate spectra if necessary.

Pages:

1. **Overview** — online/offline receiver status, latest active frequencies, throughput/dropped-sample indicators, recent interesting events, storage, decoder/ASR queue state.
2. **Spectrum** — pan/zoom waterfall and PSD, centre/span/sampling controls, gain, cursor readout, event markers, explicit recording indicator. Responsiveness must not affect capture stability.
3. **Scan plans** — editable named plans, validated against device, scheduler state, coverage timeline, suspend/resume.
4. **Observations** — dense sortable table with time, frequency, estimated width, duration, SNR proxy, candidate class, verified decoder, confidence and tags.
5. **Observation detail** — measurements, evidence, diagnostic plots, decoder JSON, signal history, audio player, transcript versions, IQ playback link, provenance and notes.
6. **Search** — FTS terms + structured filters for time window, frequency/range, receiver, verified protocol, tag, presence of audio, language and confidence. Saved searches optional P1.
7. **Models** — installed model inventory, local-only download instructions, Pareto results, model requirements and CPU/inference budget.
8. **Settings/diagnostics** — SDR driver health, permitted bands/permissions, recording policy, retention, audit, software versions, backup.

Empty states should be useful: demo/replay offer without hardware, troubleshooting if USB absent, and honest “not monitored” status rather than “no signals” for coverage gaps.

Accessibility: keyboard controls; visible focus; reduced-motion mode; high-contrast waterfall palette and non-colour semantics; data available outside canvas; mobile read-only inspection usable at narrow widths.

## 12. API and streaming contracts

Use versioned endpoints under /api/v1. SSR pages and HTMX partials share service-layer queries; avoid a second inconsistent frontend domain model.

Proposed routes (subject to implementation refinement):

    GET    /health/live
    GET    /health/ready
    GET    /api/v1/receivers
    POST   /api/v1/receivers/{id}/sessions
    PATCH  /api/v1/receivers/{id}/tuning
    GET    /api/v1/scan-plans
    POST   /api/v1/scan-plans
    POST   /api/v1/scan-plans/{id}/start
    POST   /api/v1/scan-plans/{id}/stop
    GET    /api/v1/coverage
    GET    /api/v1/observations?from=&to=&hz_min=&hz_max=&q=&cursor=
    GET    /api/v1/observations/{id}
    POST   /api/v1/observations/{id}/notes
    POST   /api/v1/observations/{id}/record
    POST   /api/v1/observations/{id}/transcribe
    GET    /api/v1/observations/{id}/media
    GET    /api/v1/events/stream
    GET    /api/v1/diagnostics
    GET    /api/v1/models
    POST   /api/v1/backups

Mutating endpoints: authenticated admin role, CSRF token for cookie-auth browser calls, input bounds, per-device permission checks and idempotency where appropriate. HTTP status should distinguish invalid frequency, device conflict, insufficient resources, missing model, unavailable decoder and receiver disconnected. Log safe request correlation IDs; never log session cookies or transcript body by default.

Spectrum transport: cap bins, frame rate, client count and backpressure, support disconnect/reconnect; real-time streams are **not** authoritative stored observation history. Events stream with increasing sequence IDs and reconnection/resume semantics.

## 13. Security, lawful-use and privacy

Threat model: exposed dashboard, compromised LAN machine, malicious/invalid decoder payload, prompt-like instructions in decoded text, untrusted IQ fixture, arbitrary uploaded recording, leaked exports, malicious model binary and remote administrator abuse.

- The service is receive-only; prohibit transmit APIs, secret extraction, encryption bypasses and stealth capture features.
- Admin onboarding uses explicit one-time setup flow; Argon2id password hashing, secure session cookies, session rotation on login, rate limiting and CSRF protection.
- Do not infer that a Tailscale-connected client is automatically authorised to view all recordings. SigCat auth always applies.
- Default HTTP listener reachable on host loopback only; no UPnP, public reverse proxy or Funnel.
- Bind internal inference service to a private socket or loopback; never expose model server externally.
- Run decoder subprocesses with limited rights/time/memory, validate messages and sanitize HTML/JSON outputs.
- Strict size/format validation for uploads and decodes; avoid traversal in file paths, command injection in adapter args and untrusted model code.
- Disabled-by-default persistent speech/audio collection; operator-set per-plan lawful use. Provide quick purge and retention audit.
- Explicitly separate observation metadata from optional sensitive text/audio; include export warnings.
- No automatic analytics/tracking, telemetry, cloud ASR, cloud translation or external receiver uploads.
- Document local laws, radio-service restrictions and personal-data duties; SigCat does not grant permission to intercept or redistribute traffic.
- Never treat decoded content, ASR transcripts or downloaded model metadata as instructions to an AI coding agent or execution engine.

## 14. Performance and reliability targets

Targets below are **engineering acceptance objectives to test**, not existing benchmark results.

| Metric | MVP objective and method |
|---|---|
| Core acquisition | no avoidable blocking on UI/ASR/database reads; log dropped sample count |
| Event persistence | eventual durable write; recover cleanly after forced restart |
| Search | target p95 < 300 ms for common indexed filters/FTS over 100k synthetic records on reference host; publish machine specs |
| Overview | target p95 < 500 ms server response with seeded DB on reference host |
| Waterfall | target 5–15 fps for local browser at configured resolution without starving RF worker |
| Startup | app/UI ready without SDR device or model installed; report degraded state |
| Memory | configure finite FFT/queue/clip limits, measure peak RSS under 24h workload |
| Offline | normal operation succeeds with outbound network blocked after initial provisioning |
| Availability | 24-hour capture soak without process crash on selected reference RTL-SDR |
| Storage | no unbounded growth; under-quota enforcement and forced-disk-full error recovery |
| Transparency | every classification/decode/transcript carries provenance and explicit uncertainty |

Test reference hosts: CPU-only amd64 mini-PC (document CPU, RAM, kernel, dongle, drivers) and one arm64 Linux host if supported. Keep real-time factor figures relative to precisely specified fixture lengths and hardware.

## 15. Observability and operation

Structured tracing with event/session IDs, counters for IQ throughput, dropped samples, receiver retunes, detector candidates, queued decodes, ASR latency, FTS query time, storage bytes, pruning and recovery. Local diagnostic page with exportable redacted support bundle. Audit admin actions and recording policy changes. Health status must distinguish app ready, hardware absent, driver failure, AI model missing and storage near-full.

Crash behaviour: transactional DB writes; idempotent job recovery; bounded workers; automatic receiver rescan with exponential backoff; corrupted media row quarantined rather than panicking entire service. Keep migration rollback/backup guidance and compatibility notes for schema upgrades.

## 16. Quality plan and required test fixtures

- Rust unit tests: DSP transforms, segmentation state machine, interval arithmetic, RF unit conversions, schedule validator, storage retention and input validation.
- Synthetic IQ golden files: empty/noise, single narrow FM, bursty FSK/ASK-like, repeated bursts, strong interfering carrier, overlap, clipping, frequency hop; document their simulated nature and labelled truth.
- Integration tests: Seify dummy/recording backend, Soapy hardware smoke test, rtl_433 adapter fixture, migration/FTS sync, restarted unfinished jobs, export/import.
- Browser tests: initial admin setup, receiver-less demo, scan creation, filters, observation detail, keyboard navigation, network reconnect, CSRF and unauthorised requests.
- Security tests: SSRF rejection, traversal/path sanitation, malicious decoder output, oversized IQ/media upload, unauthorized file access, direct access attempts when Tailscale is enabled, accidental external port exposure.
- Speech benchmark: licensed human test fixtures with manually verified reference transcripts, controlled SNR impairments, stratified per-language WER/CER, numeric/callsign accuracy, false positives and peak RAM/RTF.
- Offline test: disconnect WAN or firewall outbound traffic after provisioning; verify scan, decoding, FTS and authorised inference.
- Soak tests: 24h single receiver, disk-full, unplug/replug, container SIGTERM, database checkpoint/backup/restore.

Ship fixtures that do not contain live private communications or unlicensed recordings.

## 17. Milestones and implementation sequence

### M0 — Feasibility spike (blocking)

- Prove RTL-SDR access via SoapySDR + Seify inside a least-privilege Linux container.
- Prove FutureSDR graph ingest, FFT and clean shutdown on physical receiver and synthetic/replay input.
- Measure CPU, sample loss, device conflict and USB permissions on target host.
- Prove one local NeMo-Speech.cpp Parakeet transcription fixture on CPU; check model weights/licences, backend compatibility and packaging.
- Demonstrate SQLite migrations, WAL/FTS consistency and basic HTMX update.
- Record go/no-go findings and substitute plan (e.g. alternate acquisition boundary) before further architecture hardening.

### M1 — Receiver + browser foundation

Workspace layout, CI/lint/tests, Compose image, auth/setup, configuration, receiver discovery, scheduler, tuning, health and live spectrum. Demo mode without dongle. Tailscale instructions.

### M2 — Discover, persist, search

Scan plans/coverage, signal segmentation, immutable observations, events SSE, tables/filters, persistence, FTS and SigMF short captures, retention quotas.

### M3 — Decode and local speech

rtl_433 adapter with provenance, on-demand legal audio demodulation/VAD, background transcription, transcript UI/FTS, model backend registry and tested offline fixture. Add translation only after separate feasibility and licence review.

### M4 — Hardening and extensions

Security review, backup/restore, disk-full recovery, measurement-based optimisations, multi-hardware compatibility, extra decoder plugins, Lite/Balanced/Full profiles and RF-domain Pareto dashboard.

### Explicit dependency gates

M1 cannot declare hardware supported before M0 container evidence. M3 does not require a multilingual translation engine to ship P0 ASR. Multi-SDR, advanced signal machine learning and vector search cannot block M0–M3.

## 18. Acceptance scenarios

1. **New install:** clone repo, follow documented build/install, map RTL-SDR USB, provision a local admin and open localhost dashboard. Restart; settings persist.
2. **Remote view:** on a separate browser computer access a Tailscale Serve HTTPS URL; tailnet ACL and SigCat login both gate access; public internet cannot reach service.
3. **No receiver:** launch with no USB device, see a clearly marked degraded status and load synthetic IQ for replay.
4. **Scan reality:** step two channels and inspect measured coverage intervals and receiver retuning gaps; no false claim of continuous coverage.
5. **Detected unknown:** inject unseen IQ burst, save approximate frequency/time/features with label unknown; no invented protocol.
6. **Verified decode:** process rtl_433 fixture, show protocol/payload in observation detail; search indexed decoded fields.
7. **Search:** seeded 100k dataset supports mixed FTS, frequency and time query with stable cursor pagination and repeatable performance measurement.
8. **Permitted speech:** import authorised multilingual RF-style clip, explicitly request local transcription, see original text, timestamps, model/version and uncertainty. Optionally translate to English with separate local backend.
9. **Resource pressure:** throttle inference and clients; IQ acquisition remains prioritised, while queued work shows lag.
10. **Storage pressure:** quota reached; avoid new media capture, preserve observation metadata if possible, notify operator, do not overwrite pinned clips.
11. **Backup and upgrade:** restore clean database/media snapshot, confirm FTS consistency and readable observation histories.
12. **Unplug/restart:** disconnect SDR, app remains browsable, logs hardware failure; reconnect and re-acquire or require explicit action according to documented policy.

## 19. Risks and mitigations

| Risk | Mitigation |
|---|---|
| FutureSDR/Seify API and driver instability | isolation via traits, pin versions, M0 hardware spike, replay fallback |
| USB permissions / Docker variability | Linux-first support matrix, per-device pass-through, documented udev groups |
| Missed short emissions | explicit coverage intervals, tuning gap visibility, extra receivers later |
| DSP false positives | robust noise estimates, confidence tiers, synthetic/real golden fixtures |
| Overconfident classification | staged evidence hierarchy; verified decoder separate from hypothesis |
| Decoder crashes or hostile input | subprocess isolation, bounded I/O, strict parser validation |
| Slow/offline models on small hardware | on-demand inference, queue limits, Pareto suite, CPU-only profiles |
| Speech hallucination on static | VAD and confidence metadata, no transcript when inappropriate, radio fixtures |
| Huge IQ storage | selective captures, quotas, retention, pinned evidence |
| Ambiguous legal uses | policy gates, explicit user authorisation, receive-only and default-off recording |
| Licences for model/runtime/decoder | third-party inventory and per-artifact review before packaging |
| Device/browser exposed remotely | loopback default, app auth, Tailscale Serve only and no Funnel |

## 20. Open questions and documented defaults

Defaults for implementation unless field test contradicts:

- Single RTL-SDR and one receiver lease at a time.
- Linux amd64 host and browser clients across platforms.
- Detection/persistence on, selected-IQ capture off, audio capture/transcription off.
- Authentication enabled even on localhost and tailnet.
- Simple 8–16 kHz mono demodulated audio fixture to validate STT (resample to backend's accepted format).
- All source-of-truth times UTC; display local timezone as preference.
- Rule-based/noise-baseline classification in MVP; statistical ML classifier follows evidence.
- No prebundled external model weights; explicit provision command/model manifest.
- CPU-only inference and constrained queues; translation optional in first usable release.
- Single SQLite database and media filesystem, no Redis/Postgres/Kafka.
- Default to no externally initiated network connections during service runtime.

Deferred decisions must be recorded in an ADR with alternatives, validation criteria and rollback strategy.

## 21. Suggested Rust workspace and code ownership

    crates/
      sigcat-core/        # domain entities, common errors, traits
      sigcat-radio/       # receiver registry, Seify/Soapy adapters
      sigcat-dsp/         # FutureSDR graph, FFT, demodulation
      sigcat-scheduler/   # leases, scan plans, measured coverage
      sigcat-detect/      # segmentation, features, classification
      sigcat-decoders/    # sandboxed adapters and rtl_433
      sigcat-audio/       # authorized audio policy, VAD, clips
      sigcat-inference/   # local STT/translation backends, jobs
      sigcat-store/       # SQLx, migrations, FTS, retention, media
      sigcat-web/         # Axum, HTMX SSR, API and SSE
      sigcat-testkit/     # simulated receivers, fixtures and benchmark
    apps/
      sigcatd/            # daemon entrypoint and composition root
    docs/
      PRD.md
      adr/
    deploy/
      docker/
    tests/
      fixtures/

Contracts to enforce: core has no dependency on web/Soapy/FutureSDR/model runtimes; adapters depend inward. SQL write access is centralized, acquisition never waits on synchronous transcript generation, and decoder code never owns receiver tuning.

## 22. Sources and verification checklist

Authoritative integration docs (check again at implementation time):

- FutureSDR — https://github.com/FutureSDR/FutureSDR
- Seify — https://github.com/FutureSDR/seify
- SoapySDR — https://github.com/pothosware/SoapySDR
- NeMo-Speech.cpp — https://github.com/NVIDIA/NeMo-Speech.cpp
- Parakeet TDT 0.6B v3 — https://huggingface.co/nvidia/parakeet-tdt-0.6b-v3
- Moonshine — https://github.com/moonshine-ai/moonshine
- Qwen3-ASR — https://huggingface.co/Qwen/Qwen3-ASR-0.6B
- rtl_433 — https://github.com/merbanan/rtl_433
- SigMF — https://sigmf.org/
- SQLite FTS5 — https://www.sqlite.org/fts5.html
- Tailscale Serve — https://tailscale.com/docs/features/tailscale-serve

**Implementation verification gates:** pin exact upstream commits; re-check licences separately for every package/model; verify platform/ISA support, model runtime and throughput on actual host; document provenance and measurements. Any numerical benchmark in vendor documentation is not a SigCat benchmark.

---

This PRD is an implementation specification and requirements baseline. It does **not** assert that any listed feature or Docker image already exists.
