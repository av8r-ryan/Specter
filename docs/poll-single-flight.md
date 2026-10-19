# Single-flight Wazuh polling

One scheduled Wazuh poll can run at a time in each Specter process. The poll
acquires its guard synchronously before its first asynchronous operation and
holds it through fetch, response decoding, mapping, scoring, storage, and the
synchronous subscriber callback. A `finally` block releases the guard on settlement,
including empty responses and failures. A repeated `startAlertIngestion` call
does not create a second polling interval or trigger another initial request.
Its existing callback/file-watcher behavior is otherwise unchanged.

The interval is still `WAZUH_POLL_INTERVAL`, defaulting to 30000 milliseconds.
There is still one immediate poll at startup, and the timer is registered once
before it starts. Ticks that arrive during an active poll are skipped, not queued
or replayed. A later scheduled tick may poll after the active operation settles;
there is no catch-up request after settlement. A never-settling fetch, response
reader, or scorer consequently prevents later polls. This change adds
no timeout, cancellation, backoff, or scheduler lifecycle API.

## Scope and proposed followups

This is overlap prevention, not lossless ingestion. It does not change the
newest-100 query, timestamp-only watermark, mapping, alert IDs, TLS options,
scoring logic, or retained store. In particular, the existing watermark moves
before scoring/storage succeeds: a scoring failure can still leave the cursor
ahead of unstored evidence. A subscriber failure can occur after a batch was
already stored; this change does not roll back that batch or ensure notification.
Repeated starts may still replace the subscriber callback under the existing API.
Returned subscriber promises are not awaited under the existing callback API.
The guard serializes scheduled polls only; HTTP and file ingestion are separate.

Before claiming lossless polling, maintainers would need to review pagination,
equal-timestamp handling, durable cursor/commit rules, response completeness,
source identity/deduplication, and restart behavior. Bounded deadlines and
cancellation would require their own provider/runtime contract and fixtures.
These are followup proposals, not implemented features or adopted commitments.
This slice makes no authentication, TLS trust, remote deployment, or live Wazuh
compatibility claim.

## Offline validation

Use Bun 1.3.14 from `apps/server`:

```bash
bun install --frozen-lockfile --no-env-file
bun --no-env-file test
bun run --no-env-file typecheck
```

On the upstream-based single-flight branch, the full server suite has 93 tests:
81 existing cases and 12 isolated polling subprocess scenarios. Focus only the
new scenarios with `bun --no-env-file test src/poll-single-flight.test.ts`.
The same full suite and TypeScript check run on Ubuntu and macOS in
`.github/workflows/poll-single-flight.yml` with the tracked server lock.

The new fixtures invoke the production `startAlertIngestion` API and capture its
real interval callback. They drive that callback deterministically while holding
fetch, JSON decoding, or scoring promises, verify that skipped ticks do not
queue, and confirm that a later tick uses the accepted batch's newest timestamp.
They also cover storage callback reentry, synchronous first-fetch startup
reentry, repeated starts, an empty result, and
recovery after fetch, HTTP, JSON, mapping, scorer, and subscriber failures. The
callback-failure fixture confirms the existing stored batch remains present.

Each scenario uses a fresh process, synthetic credentials, a disposable missing
JSONL path, mocked fetch/scoring, and cleared inherited Wazuh trust settings. No
real interval, Wazuh, model, SSH, provider, or file watcher is started. The existing
HTTP integration fixture also points its JSONL fallback at a disposable missing
file. These tests establish the local overlap contract, not a deployed data-loss
or polling-completeness guarantee.

## Maintenance notes

Keep guard acquisition before the first await and release in `finally` around the
whole operation. Keep timer registration idempotent and before the initial poll
can reenter application code. Do not release the guard at fetch completion while
response parsing or scoring remains pending. Preserve separate reviews for
watermark, mapping, ID, TLS, and store changes; the fixture path deliberately
avoids those other contributions' production hunks.
