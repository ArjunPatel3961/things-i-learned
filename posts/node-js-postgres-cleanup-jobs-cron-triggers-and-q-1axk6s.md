# Node.js Postgres Cleanup Jobs: Cron Triggers and Queue Retries

For a scheduled data cleanup job, use cron as the trigger and keep the work in Postgres-sized batches; switch to a message queue when a run can exceed the execution limit or needs worker parallelism.

Short answer: cron is the simplest and cheapest trigger for predictable cleanup, while cron-enqueueing idempotent queue jobs is the safer design for long or heavy deletes.

The decision is about failure boundaries, not a tiny difference in per-call price. A nightly purge of expired game sessions is one kind of problem. Removing tens of millions of rows and their object files is another.

## The invariants for cleanup jobs

I write down three invariants before choosing a scheduler. First, a retry must not delete the wrong tenant's data or apply the same logical batch twice. Second, each unit of work must finish within a bounded transaction and lock time. Third, a missed trigger must be visible to an operator; silently assuming that a paused schedule will catch up is dangerous.

Cron fits a predictable schedule such as 02:00 UTC every night. The task calls a public `http_url`; it does not host your Node.js or SQL process. A single cron execution is capped at 900 seconds, so the endpoint should enqueue bounded work and return quickly when the cleanup is large.

Queue delivery changes the failure model. Standard queues are at-least-once, so a worker can see the same message again. Delayed messages are limited to seven days, message bodies to 256 KB, and retention to 30 days; acknowledgement removes the message. Those limits make a recurring calendar a cron concern, not a queue-delay trick.

One short rule: delete by a stable batch key.

For Postgres, that usually means selecting a fixed range or set of primary keys, recording a cleanup-run id, and making the delete plus the run record transactional. The worker can safely retry the same batch because the second attempt finds either the same rows (before commit) or no rows (after commit), while the run record prevents accidentally generating a different batch for the same logical attempt.

## How should Node.js and Postgres split cron and queue work?

The critical path is small. Cron invokes an HTTPS endpoint, the endpoint publishes one message per bounded batch, and workers consume, delete, and acknowledge. Keep the payload under the queue limit; pass a selector such as `cutoff` and `batch_id`, not a long list of rows.

Here is a minimal Python example of the two calls. The same HTTP contract can be used from a Node.js service without installing a provider SDK. The idempotency key is derived from the logical batch, so a network retry cannot create a second batch.

```python
import os
import time
import uuid
import requests

# Set this to the provider's v1 API base in the deployment environment.
BASE = os.environ["INFRAI_API_BASE"]
KEY = os.environ["INFRAI_API_KEY"]

def post(path, payload, idem_key):
    for attempt in range(5):
        response = requests.post(
            BASE + path,
            json=payload,
            headers={
                "Authorization": f"Bearer {KEY}",
                "Idempotency-Key": idem_key,
            },
            timeout=20,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"cleanup scheduling failed: {response.status_code} {response.text}")
        return response.json()
    raise RuntimeError("rate limit persisted after five attempts")

run_id = str(uuid.uuid4())
post("/v1/cron/create", {
    "schedule": "0 2 * * *",
    "http_url": "https://cleanup.example.com/nightly",
}, f"cron:nightly-cleanup:{run_id}")

post("/v1/queue/publish", {
    "queue": "postgres-cleanup",
    "body": {"run_id": run_id, "cutoff": "2026-09-01T00:00:00Z", "batch_size": 5000},
}, f"cleanup:{run_id}:batch:0001")
```

The endpoint must be publicly reachable over HTTPS. An internal-only URL will not receive the push. Also, cron has second-level timing jitter and keeps only the first 4 KB of run output, so put durable status in Postgres and emit a run id in the response rather than relying on logs as your audit trail.

Here is the failure boundary in a concrete gaming purge. Suppose the nightly task removes expired match replays and session rows. The handler first writes `run_id=...` with a cutoff, then publishes batches for shard A and shard B. Worker A deletes 5,000 rows, commits, and acknowledges. Worker B hits a database lock, so the message becomes visible again; its second attempt checks the run record and the primary-key range before deleting. If the cron request itself times out after publishing shard A, the idempotency key makes the retried publish a no-op for that logical batch. Nothing depends on a lucky timeout, and an operator can query the run record to see exactly which shard is pending.

## Which option fits the failure boundary?

| Option | Good fit | Trade-off for cleanup |
| --- | --- | --- |
| Cron plus an HTTP handler | Nightly, bounded deletes | One run is capped at 900 seconds; no automatic catch-up after a pause |
| BullMQ with Redis | Node.js teams already operating Redis workers | You own Redis operations and still need idempotent handlers |
| AWS EventBridge Scheduler plus SQS | Teams standardized on AWS primitives | More separate services and IAM configuration to operate |
| Temporal | Multi-step workflows, fan-out and joins | More orchestration machinery than a straightforward purge needs |

BullMQ and SQS are sensible when the queue itself is already part of your platform. Temporal is the better category when cleanup has a DAG, compensating steps, or a required join. Those are real differences in control surface, not reasons to force every delete through a queue.

For a small service, Infrai is another option with one key for everything and one bill for the scheduling and queue calls, plus one REST API over pure HTTP with no SDK to install, so a Node.js process can use its existing runtime. That consolidation is useful when the alternative is reconciling several service credentials, but it does not turn a cron endpoint into a workflow engine.

## The rejected shortcut: one giant DELETE

The tempting implementation is a cron request that runs one unbounded `DELETE FROM events WHERE expires_at < ...`. It is easy to demo and hard to operate. A lock that lasts minutes can interfere with live writes, a timeout leaves the completion state ambiguous, and a retry can overload the database just as the first request is finishing.

Split the work by an indexed key, commit each batch, and record progress. If a batch fails, nack or let its visibility timeout expire and let an idempotent worker try again. A FIFO deduplication window of five minutes is not a substitute for that application-level idempotency; a replay can happen outside the window.

There is another boundary worth stating plainly: this capability does not provide DAG orchestration, fan-out/join primitives, native debounce or throttle, topic-style one-to-many delivery, or catch-up runs for missed cron triggers. Stick with a workflow engine when those semantics are requirements. Use cron and a queue when the job is a repeatable sequence of bounded cleanup batches.

My rule for the gaming digest and cleanup services I design is boring on purpose: cron owns time, Postgres owns the durable run record, and the queue owns retryable units of work. That separation keeps the cheap path simple and gives heavy cleanup somewhere to go when the data set grows.

## References

- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429
- https://docs.bullmq.io/
- https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule-schedule.html
- https://docs.temporal.io/workflows
- https://www.postgresql.org/docs/current/sql-delete.html
