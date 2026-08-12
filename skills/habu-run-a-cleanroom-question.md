---
name: habu-run-a-cleanroom-question
description: Run an existing clean room question on the Habu Clean Room API (LiveRamp Clean Room) and retrieve its results, respecting the 20-runs-per-hour ceiling and the absence of idempotency.
api: Habu Clean Room API
base_url: https://api.habu.com/v1/
docs: https://developers.liveramp.com/clean-room-api
generated: '2026-08-12'
method: generated
source: openapi/habu-clean-room-api-openapi.yml, conventions/habu-conventions.yml, rate-limits/habu-rate-limits.yml
operations:
  - getHealth
  - getAllCleanrooms
  - getAllCleanroomQuestions
  - getAllCleanroomQuestionDatasets
  - createCleanroomQuestionRun
  - getCleanroomQuestionRunById
  - getCleanroomQuestionRunData
  - getCleanroomQuestionRunOutputFile
---

# Run a clean room question and fetch its results

This is the highest-value flow on the Clean Room API: take a question that is already
attached to a clean room, run it, wait for it, and read the output.

## Before you start

- **This flow is rate limited.** `createCleanroomQuestionRun` is capped at **20 calls per
  hour**, counted per API user, per organization *and* per IP address. Status polling is
  capped at 20 checks per hour too. Budget accordingly.
- **There is no idempotency key.** A retried run creation creates a *second run* and
  consumes a second slot from the hourly budget. Never blind-retry step 4 — re-list runs
  with `getAllCleanroomQuestionRuns` and check whether yours already exists.

## Steps

1. **Check connectivity.** `GET /health` (`getHealth`). This is the only unauthenticated
   operation on the API — a 200 with `{"status":"UP"}` proves the host is reachable before
   you spend a token.

2. **Authenticate.** `POST /oauth/token` with
   `Content-Type: application/x-www-form-urlencoded`, body `grant_type=client_credentials`,
   and `Authorization: Basic base64(client_id:client_secret)`. The response carries
   `accessToken`, `tokenType`, `expiresIn` (43200 = 12 hours) and `expiresAt`.
   **Cache this token.** You get only **2 tokens per 24 hours**; re-authenticating on every
   process start will lock you out. Send it as `Authorization: Bearer <accessToken>` on
   every subsequent call.

3. **Find the clean room and the question.** `getAllCleanrooms` returns the clean rooms
   your organization can see; `getAllCleanroomQuestions` (path parameter `cleanroomId`)
   returns the questions attached to one of them. Take the `cleanroomQuestionId`.

4. **Confirm the datasets are bound.** `getAllCleanroomQuestionDatasets` (path parameter
   `cleanroomQuestionId`) shows which clean room dataset is bound to each dataset macro in
   the question. A run against unbound macros fails; check before you spend a run slot.

5. **Create the run.** `createCleanroomQuestionRun` —
   `POST /cleanroom-questions/{cleanroomQuestionId}/create-run`. Capture the returned run
   id. This is the rate-limited call.

6. **Poll for completion.** `getCleanroomQuestionRunById` —
   `GET /cleanroom-question-runs/{cleanroomQuestionRunId}`. LiveRamp recommends **hourly**
   polling and caps you at 20 checks per hour. Do not tight-loop.

7. **Read the results.** `getCleanroomQuestionRunData` —
   `GET /cleanroom-question-runs/{cleanroomQuestionRunId}/data` for the result rows, or
   `getCleanroomQuestionRunOutputFile` —
   `GET /cleanroom-question-runs/{cleanroomQuestionRunId}/download/{fileName}` to pull the
   output file.

## Error handling

Errors come back as JSON, **not** RFC 9457 problem+json. The live envelope is
`{"status","code","timestamp","message","details"}` — richer than the `{code, message}`
the spec declares.

| Status | What it means here | What to do |
|---|---|---|
| 401 | Token missing or expired (12h lifetime) | Reuse your cached token; only re-issue if `expiresAt` has passed — you have 2 per day |
| 403 | The service account lacks the clean room role for this resource | Fix the role in the console; retrying will not help |
| 404 | Wrong `cleanroomId` / `cleanroomQuestionId` / run id | Re-list rather than guess |
| 409 | Conflict with the current state of the resource | Re-read the resource before acting |
| 429 | You exceeded 20 runs (or 20 status checks) in the hour | Back off a full hour — **no `Retry-After` header is sent**, so you must time it yourself |
| 500 | Server error | Retry once with backoff; a duplicate run is possible since there is no idempotency key |

Every response carries an `x-request-id` header. Log it — it is the only correlation
handle this API gives you.

## Operations not to call

Four operations in this spec are tagged **Internal**. LiveRamp states they are "not part of
the supported external customer contract and may change without notice." Do not call them
from an agent.
