# Implementation plan

This plan implements `spec.md` (R1 to R8). It reuses the two existing source files and adds one new file. The archived helper was never built and is not used (rejected).

## Ordered steps

1. Write the tests named in `spec.md` first (release gate, privacy, link route, duplicate retry, throughput boundary, bounded retry), so each requirement has a failing check before code exists.
2. Create new file `src/service/notify.mjs` with the release-gate check (R1, R2): build a job only for a result a doctor has reviewed and released, keyed by the released result version.
3. In new file `src/service/notify.mjs`, add the message builder (R3, R5): fixed "result ready" text and a link from `resultRoute` in `src/portal/result-route.mjs`, which stays unchanged; a portal redesign is excluded.
4. Enqueue jobs through the existing `enqueueNotification` in `src/service/queue.mjs` (unchanged), with the released result version as the idempotency key (R6).
5. Add the limiter and batching in new file `src/service/notify.mjs` (R7): at most 50 requests per second and at most 20 messages per batch.
6. Add bounded retries with backoff and the delivered-keys check (R6, R8), and restrict logging to event IDs and delivery states (R4).
7. Run all tests, confirm the privacy test against real log output, and keep channel and recipient configuration patient-only until OPEN-1 and OPEN-2 are approved by their owners.

## Paths and dependencies

- Existing: `src/portal/result-route.mjs` exports `resultRoute` (`/portal/results/:resultId`); reused read-only for the link (step 3).
- Existing: `src/service/queue.mjs` exports `enqueueNotification(queue, job)`; reused for enqueueing (step 4).
- New file: `src/service/notify.mjs` holds the gate, message builder, limiter and retry logic (steps 2, 3, 5, 6).
- Not used: the archived helper was never built and is rejected; no plan step depends on it.
- Order: tests (1) come first; the gate (2) comes before the message (3) and the enqueue (4); the limiter (5) and retries (6) depend on the enqueue path; verification (7) is last.
- External dependencies: the provider contract (50 requests per second, 20 messages per batch) and the approvals that OPEN-1 (product and privacy) and OPEN-2 (clinical and privacy) still need.

## Risks and verification

- **Duplicate delivery on retry:** mitigated by the released result version as the idempotency key and the delivered-keys check; verified by `test_duplicate_retry_single_delivery`.
- **Notifying before release:** mitigated by the release gate as the only trigger; verified by `test_release_gate_blocks_unreleased`.
- **Clinical data leak:** clinical content stays out of notifications and logs, which hold only event IDs and delivery states; verified by `test_privacy_no_clinical_content`.
- **Provider throttling:** mitigated by the 50 per second and 20 per batch limiter; verified by `test_throughput_boundary` at the boundary values.
- **Retry storms:** retries are bounded; verified by `test_bounded_retry_failure`.
- **Unapproved scope:** the channel (OPEN-1) and referring-doctor notifications (OPEN-2) stay with their owners; nothing ships for them until approved.

| Requirement | Step | Existing path | Test |
| --- | --- | --- | --- |
| R1, R2 release gate | 2 | - | `test_release_gate_blocks_unreleased` |
| R3, R4 privacy | 3, 6 | - | `test_privacy_no_clinical_content` |
| R5 portal reuse | 3 | src/portal/result-route.mjs | `test_link_uses_existing_route` |
| R6 idempotency | 4, 6 | src/service/queue.mjs | `test_duplicate_retry_single_delivery` |
| R7 throughput | 5 | - | `test_throughput_boundary` |
| R8 provider failure | 6 | - | `test_bounded_retry_failure` |
