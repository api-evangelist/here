---
name: here-run-a-batch-geocoding-job
description: Submit, start, monitor, stop and collect a large batch geocoding job with the HERE Batch API v7, including webhook notification on completion.
api: HERE Batch API v7
spec: openapi/here-geocoding-batch-v7-openapi.yml
base_url: https://batch.search.hereapi.com/v7
operations:
  - postJob
  - startJob
  - getJob
  - getJobs
  - getJobResults
  - getJobErrors
  - stopJob
  - deleteJob
  - postNotification
  - testNotification
  - getNotification
  - deleteNotification
---

# Run a batch geocoding job

## Limits to respect before you submit

| Limit | Value | What happens if you exceed it |
|---|---|---|
| Records per job | 1,000,000 | HTTP 413 |
| Uncompressed input size | 500 MB | HTTP 413 |
| `recId` length | 512 UTF-8 chars | entry in the job error log |
| Record length | 4096 UTF-8 chars | entry in the job error log |
| Jobs in a startable state (QUEUED / SUBMITTED / FAILURE) | 500 | HTTP 429 |

Split larger inputs into multiple jobs. HERE does not guarantee throughput or queue time for any job.

## The job lifecycle

`submitted → queued → pending → inProgress → completed | failure | stopped | deleted`

1. **Register a webhook first (optional but recommended).** `postNotification` creates a `WEB_HOOK`
   with a URL template. Available substitutions: `${JOB_ID}`, `${JOB_NAME}`, `${JOB_STATUS}`,
   `${JOB_OUTPUT_TYPE}`, `${JOB_RECORDS_SUCCEEDED}`, `${JOB_RECORDS_FAILED}`, `${JOB_RECORDS_TOTAL}`,
   `${JOB_RECORDS_VALID}`, `${JOB_RECORDS_INVALID}`.
   Then call `testNotification` (`GET /batch/notifications/{notificationId}/test`) to confirm HERE can
   reach your endpoint **before** a real job depends on it. This is the only pre-flight check the platform offers.
   The notification is BETA and HERE documents no signing secret — your receiver cannot verify the caller,
   so treat the webhook as a trigger to go and `getJob`, never as trusted data.
2. **Create the job.** `postJob` (`POST /batch/jobs`) returns a `jobId`. Input for a job cannot be modified
   once defined — a changed input means a new job.
3. **Start it.** `startJob` (`PUT /batch/jobs/{jobId}/start`). A job that is already queued or in progress
   cannot be started again; stop it first if you want to force a re-run.
4. **Watch it.** `getJob` (`GET /batch/jobs/{jobId}`) for status, or wait for the webhook.
5. **Collect.** `getJobResults` on success, `getJobErrors` for the per-record failures. Errors are only
   available once the job has started and completed.

## Reversibility

`stopJob` (`PUT /batch/jobs/{jobId}/stop`) is the reversal for a running job. It fails if the job has already
completed, failed or been deleted.

Completed jobs are retained and re-retrievable for **30 days** after creation
(https://docs.here.com/geocoding-and-search/docs/job-lifecycle). `deleteJob` is terminal within that window.
This 30-day retention is the only reversal window HERE states anywhere in the portfolio.

## Errors you will actually hit

- `Job has already been completed` — create a new job rather than restarting.
- `Too many concurrent Jobs running` — the 500-job startable queue is full; wait or stop jobs.
- `Invalid billing tags` — 4-16 chars, `[A-Za-z0-9]` plus `-_.` (not at the start or end), up to 6 joined
  by `+` **or** `,` but never both.
- `Results for a failed Job is not retrievable` — read `getJobErrors` instead.

Full registry: https://docs.here.com/geocoding-and-search/docs/error-messages
