# Specification

Scope: the results-ready notification for the fictional clinic, from the approved intent in `intent.md` (I-1 to I-4, OPEN-1, OPEN-2) and the constraints in `inputs/current-policy.md`. The archived proposal in `inputs/archived-design.md` was never approved and is rejected; nothing in this spec adopts it.

## Requirements

- **R1 (release gate, I-1):** A patient notification is created only after a doctor has reviewed and released the result. An unreleased, unreviewed or withdrawn result never produces a notification.
- **R2 (trigger, I-1):** The release event is the only trigger for a notification; notifying on upload is rejected.
- **R3 (privacy of content, I-2):** Clinical content stays out of notifications and logs; the notification says only that a result is ready and links to the portal.
- **R4 (privacy of logs, I-2):** Operational logs record only event IDs and delivery states, never patient identifiers or clinical payloads.
- **R5 (portal reuse, I-3):** The notification link points to the existing route `/portal/results/:resultId` exported by `src/portal/result-route.mjs`; a portal redesign is excluded.
- **R6 (idempotency, I-4):** The idempotency key is the released result version, so one released version is delivered at most once, including across retries.
- **R7 (provider contract, I-4):** Sends respect the provider limits of at most 50 requests per second and at most 20 messages per batch.
- **R8 (provider failure, I-4):** Provider failures use bounded retries with backoff that reuse the same idempotency key; after the retry limit the job is marked failed and logged by event ID and delivery state only.

## Design

- **Trigger:** a handler on the result release event checks the release gate (reviewed and released by a doctor) and builds one notification job per released result version.
- **Queue:** jobs go through the existing adapter `enqueueNotification` in `src/service/queue.mjs`; no new queue is introduced.
- **Sender:** new file `src/service/notify.mjs` holds the release check, the message builder, the rate and batch limiter and the retry logic. The archived helper was never built and is not used (rejected).
- **Message:** a fixed "your result is ready" text plus a link built from `resultRoute` in `src/portal/result-route.mjs`; no result values, diagnosis, test names or patient identifiers.
- **Idempotency:** the job key is `resultId` plus released version; a store of delivered keys is checked before each send and updated after confirmed delivery, so retries and duplicate events are no-ops.
- **Throughput:** a limiter caps the provider at 50 requests per second and splits jobs into batches of at most 20 messages.
- **Retries:** a bounded retry count (proposed 3, to be confirmed against the provider contract) with exponential backoff; final failure is recorded as a delivery state.
- **Channel and recipient:** the sender takes the channel as configuration and sends only to the patient, pending OPEN-1 and OPEN-2 below.

## Open decisions and owners

- **OPEN-1 (channel choice):** which channel (for example SMS, email or app push) is used. Owner: product and privacy, who must both approve. Until then the channel stays configurable and no channel is assumed in the plan.
- **OPEN-2 (referring-doctor notifications):** whether referring doctors are notified. Owner: clinical and privacy, who must both approve. It is out of scope; only the patient is notified.

## Traceability

| Intent | Requirement | Test |
| --- | --- | --- |
| I-1 notify after review and release | R1, R2 | `test_release_gate_blocks_unreleased`: an unreviewed or unreleased result creates no job; a released one creates exactly one |
| I-2 clinical content only in portal | R3, R4 | `test_privacy_no_clinical_content`: message body and every log line contain no clinical payload or patient identifier, only event ID and delivery state |
| I-3 reuse existing portal route | R5 | `test_link_uses_existing_route`: the link matches `resultRoute` from `src/portal/result-route.mjs`, with no new route |
| I-4 no duplicates, provider contract | R6, R7, R8 | `test_duplicate_retry_single_delivery`: a retried or duplicate release of the same version delivers once; `test_throughput_boundary`: 50 requests per second and 20 per batch pass, 51 and 21 are deferred; `test_bounded_retry_failure`: the retry stops at the limit and is logged as failed |
