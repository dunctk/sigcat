# SigCat integration specification — ADS-B, PiAware, SkyAware and FlightAware

**Status:** Proposed / not implemented  
**Owner:** SigCat  
**Priority:** P1 — incremental integration; initial read-only bridge can be developed without replacing existing receiver software  
**Task IDs:** ADS-01 through ADS-05 in [the implementation tracker](../IMPLEMENTATION.md)  
**Date:** 2026-10-19

## Outcome and non-negotiable acceptance criteria

An operator already using **PiAware + SkyAware** to contribute ADS-B to FlightAware must be able to use SigCat for local aircraft discovery and history **without requiring another RTL-SDR or a second ADS-B decoder**. An optional later deployment provides **one managed Docker Compose project and one primary SigCat web interface**, while retaining a PiAware uploader as the supported FlightAware integration.

**Protect the feeder:** Do not interrupt the existing PiAware site or change its unique `feeder-id` implicitly. Retain the ability to contribute with the same site identity and verify the FlightAware site's actual online/feed statistics after migration. Complimentary FlightAware Enterprise access is a **FlightAware-administered benefit** for qualifying active data feeders, not a benefit that SigCat can grant or guarantee.

**Do not misstate “one app”:** A managed stack can run multiple cooperating services/processes (e.g., dump1090-fa or readsb, a PiAware feeder and SigCat), but needs only one 1090 MHz RTL-SDR, one decoder and one project lifecycle. SigCat's web interface can become the main UI; SkyAware can remain optional for diagnosis.

## Stage A (first): connect to existing PiAware *read-only*

Data flow:

    Existing RTL-SDR (1090 MHz)
       → existing dump1090-fa / readsb decoder
          ├── PiAware uploader → FlightAware (existing registration)
          ├── existing SkyAware UI (unchanged)
          └── Beast TCP output (typically port 30005)
                   → SigCat ADS-B read-only client
                      → validated aircraft messages/events
                      → SQLite timeline / search / aircraft UI

- The existing decoder alone owns the RTL-SDR. SigCat must **not open the same USB device**, retune 1090 MHz or launch a duplicate dump1090 process.
- Treat Beast TCP output on port 30005 as an **input feed**, not an unprotected internet endpoint. Configure host and port explicitly, support reconnect/backoff, stream status and receive-only access.
- Do not need to modify PiAware's `feeder-id`, account, MLAT settings or uploader process.
- Support hostnames, local bridge networking or explicit private LAN addresses. Where Docker runs in the same host's isolated network, verify reachability; `127.0.0.1` inside a container is not the host loopback interface.
- Never assume port 30005 is available to the container or accessible off-box. If not, document least-exposed options; don't recommend broad `0.0.0.0:30005` binding as a default.
- Implement enough Beast framing/error handling for 1090 MHz Mode S/ADS-B message ingestion, timestamps where present, dedupe and quality metrics. Preserve raw frame and arrival provenance without claiming absolute decoding quality.
- Track receiver name, source host, decoder software/version if known, message count, bytes, last seen, disconnects, invalid frames and event ingest lag.
- Make existing FlightAware feed status **operator-confirmed** initially. SigCat's incoming TCP message count is not proof that FlightAware is receiving uploads or that an Enterprise account is active.

**Acceptance:** with current feeder still online, connect SigCat to its Beast output, populate a local read-only aircraft/event view and searchable log for a reproducible replay fixture. Existing SkyAware, dump1090 and PiAware continue to work. Tests must not claim production feeder continuity without on-device and FlightAware account/site verification.

## Stage B: enrich SigCat's local aircraft view

- Aircraft observation records: ICAO 24-bit address (with correct category/source handling), time last observed, callsign if broadcast, lat/lon if decoded, barometric/geometric altitude where present, ground speed, track, message source, validity and message confidence as appropriate.
- Build a **current aircraft** table/map and historical aircraft/message timeline from **locally received** frames, avoiding reliance on paid FlightAware APIs.
- Search by ICAO/callsign/time/altitude/distance where computable; include receiver coverage quality, source attribution and data gaps.
- Aircraft tracks are created from messages actually seen by the receiver. Do not imply completeness beyond radio range or retune gaps.
- Keep local direct ADS-B/Mode S data separate from FlightAware-derived MLAT or enriched commercial data. Do not silently store, merge, scrape, republish or redistribute restricted MLAT outputs.
- Avoid undue retention of unneeded identifiers; keep local-only defaults, configurable retention and access restrictions consistent with [the PRD](../PRD.md).

**Acceptance:** deterministic Beast fixture produces deduplicated correctly attributed observations and aircraft tracks; live UI reflects disconnect/reconnect and stale aircraft; long-term data survives restart and is searchable using indexed SQLite queries.

## Stage C: optional managed Docker Compose installation

Target shape:

    RTL-SDR (1090 MHz) ──> dump1090-fa OR verified readsb backend
                                  ├── Beast 30005 → SigCat consumer
                                  ├── Beast 30005 → PiAware uploader → FlightAware
                                  └── optional SkyAware / decoder admin UI
    SigCat → Axum + HTMX UI → localhost / opt-in LAN / Tailscale Serve

- Package as an **opt-in Compose profile** such as `adsb-managed`, not the default radio scanning pipeline. It must reuse one decoder and one physical ADS-B receiver.
- Prefer the existing FlightAware-supported decoder path first; validate readsb compatibility/MLAT timing before making it a replacement option.
- PiAware remains the FlightAware **uploader**. SigCat does not reimplement authentication/feed protocol or attempt to spoof a feeder. Configure PiAware to consume the external Beast stream (e.g. `receiver-type other`, `receiver-host`, `receiver-port`), with exact configuration verified for the packaged PiAware version.
- Preserve a durable, backed-up site `feeder-id` during migrations. **No new feeder identity** unless the operator requests a new site. Treat feeder ID as sensitive installation configuration, not a secret requiring secrecy like a password, and never publish it in screenshots/README/logs.
- Move receiver ownership once: stop the old decoder **only during the explicit cutover**, keep a tested rollback to the old system, then start the managed decoder + PiAware + SigCat.
- Verify upstream health and actual FlightAware contribution after cutover; a local green PiAware process indicator alone is insufficient. Protect MLAT data and source timestamps; compare MLAT status before and after.
- Use separate internal networks and bounded port exposure. Only SigCat UI is accessible remotely through Tailscale Serve by default. No public port 30005 or externally exposed SkyAware by default.
- Avoid needless host dependencies; run services as minimally privileged as compatible with RTL-SDR USB. No `privileged: true` just to simplify container access.
- Package/licensing: review FlightAware and decoder licences, redistribution rights, images and exact versions; if needed document supported external installation instead of bundling an incompatible image.
- Never silently enable FlightAware uploads: managed feeding is an **explicit external-data-sharing opt-in** and exception to SigCat's local-only operating default. Local aircraft discovery continues if the uploader is disabled.

**Acceptance:** one Compose project starts/stops all components; one 1090 MHz dongle and one decoder; PiAware feeds a verified, existing FlightAware site with unchanged feeder ID; MLAT health checked; SigCat's local aircraft UI remains functional. Validate restart, decoder failure and rollback.

## Stage D: operational observability

A compact **FlightAware / ADS-B** panel inside SigCat should show:
- decoder input connected, valid messages/s, aircraft count and receiver/dongle status;
- SigCat Beast client connected and local processing errors/backlog;
- PiAware process and upstream connection/feed state (only when known via local status interfaces);
- last verified FlightAware site check and its method, ideally a user-opened link to FlightAware's official My ADS-B site statistics; do not pretend entitlement can be verified from local packets;
- MLAT status when available from authorised local PiAware status without redistributing MLAT results;
- a clear warning on interrupted feeding, receiver conflict or stale telemetry.

For single dongle operation: a **dedicated 1090 MHz receiver is continuously assigned to ADS-B**; SigCat frequency scanning must run on another SDR (or deliberately suspend ADS-B with an explicit warning that uploads and MLAT will be disrupted).

## Data model and interfaces (planned)

- `adsb_sources`: id, transport, configured host:port reference, decoder type/version, local identity, status.
- `adsb_messages`: source ID, receiver clock/arrival time, message/format, ICAO if valid, validated parse payload, hash and dedupe key, TTL/retention.
- `aircraft_observations`: source/message refs, ICAO, callsign, position, altitude/speed/track where broadcast, source quality, first/last-seen.
- `aircraft_tracks` (optional materialized view/segments): local positions only and source attribution.
- `feeder_status`: latest known PiAware state, checked_at, config mode, no external credential or login token.
- Data must include local-origin vs MLAT-origin distinction. Never merge proprietary derived results without rights/permission.

Proposed API namespace: `/api/v1/adsb/sources`, `/api/v1/adsb/aircraft`, `/api/v1/adsb/aircraft/{icao}`, `/api/v1/adsb/status`. Mutating source/settings routes require admin auth and CSRF. Optional outbound upload controls must be explicit and separately permissioned.

## Failure cases / QA checklist

1. PiAware is up but no Beast frames → warn source down, don't claim FlightAware is offline.
2. Beast TCP drops → bounded reconnect and gap in observed coverage.
3. Duplicate Beast frames → one canonical observation with provenance; no duplicate aircraft track points.
4. Incorrect decoder port/address within Docker → helpful actionable error.
5. FlightAware upstream disconnect → operator warning from authoritative PiAware signal, not inference from SigCat receiving Beast.
6. Receiver claimed by dump1090 → SigCat must never open it via FutureSDR scheduler.
7. Retuning another SDR → aircraft receiver remains undisturbed.
8. Existing feeder ID moved → identity preserved only with operator confirmation and post-migration FlightAware site check.
9. App/network restart → no duplicate feed identity, no database corruption, controlled recovery.
10. Model/aircraft metadata enrichment remains offline unless user opts into specifically documented third-party services.
11. Stale aircraft automatically age out of current list while remaining queryable according to retention.
12. Authorized local Beast data and restricted MLAT results cannot be confused in export or public interfaces.

## Implementation dependencies / precedence

- **ADS-01** read-only Beast ingest is independent of running SigCat's own SDR capture pipeline; can be started after basic web/SQLite scaffolding exists.
- **ADS-02** local aircraft UI/search depends on ADS-01 and storage/search plumbing.
- **ADS-03** feeder health panel depends on PiAware status compatibility and tested separation of local vs upstream states.
- **ADS-04** managed Compose/profile and feeder-ID-preserving migration comes **only after** passive integration is validated and must not be required for SigCat MVP.
- **ADS-05** real end-to-end cutover/MLAT/FlightAware benefit verification needs physical hardware, operator credentials and on-site checks. Agents must not mark this done from a mock.

## Sources (recheck when implementing)

- FlightAware official: [ADS-B sharing and account benefits](https://www.flightaware.com/adsb/) — eligibility depends on active data feeding and provider policies.
- FlightAware: [Streaming data from your ADS-B receiver](https://support.flightaware.com/hc/en-us/articles/37944925213975-Streaming-Data-from-your-ADS-B-Receiver) — ports including 30005.
- FlightAware: [Preserving PiAware feeder ID](https://support.flightaware.com/hc/en-us/articles/33379001938583-How-Do-I-Claim-The-FlightFeeder-Site-number-UID-and-Feeder-History-From-An-Old-PiAware) — official migration steps.
- [PiAware official repository](https://github.com/flightaware/piaware) and [receiver handling source](https://github.com/flightaware/piaware/blob/master/package/piaware.tcl).
- [FlightAware's ADS-B FAQ](https://www.flightaware.com/adsb/faq/).
- [Community docker-piaware implementation](https://github.com/sdr-enthusiasts/docker-piaware) — useful reference, not yet an endorsed dependency.
