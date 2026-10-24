# HTTP Error Tracking: Filter, Interceptor, and Queue Boundaries for Property Notifications

TL;DR: Capture an exception once, at the outermost boundary that owns its outcome. Use a global exception filter for HTTP requests, an explicit wrapper for scheduled jobs, and an explicit wrapper around each queue message. Interceptors may add timing and request context, but they should not report the same thrown error again. Normalize all three paths into one failure envelope, attach delivery-stage fields, and preserve the original retry behavior.

For a property-management notification service, the decision rule is blunt: an event is useful only if an operator can tell which resident or staff notification failed, where delivery stopped, and whether another attempt is expected. A stack trace without `channel`, `template`, `delivery_stage`, and a stable operation identifier creates noise. Capturing the same failure in an interceptor, a filter, and a queue consumer creates louder noise.

## Decision and invariants

This architecture records failures at three outcome boundaries: the HTTP adapter, the scheduler callback, and the queue-message handler. Each boundary converts local context into the same envelope before handing it to a generic sink. The sink could write structured logs, emit an OpenTelemetry exception event, or call an error-tracking adapter; the boundary contract does not depend on that choice.

Four invariants keep the signal usable:

1. One failed operation produces one primary error record. A later failed retry is a separate attempt, linked by the same operation ID and distinguished by its attempt number.
2. Reporting never changes the application outcome. An HTTP exception still reaches the framework response path, a scheduled invocation still fails according to its scheduler contract, and a queue message is still acknowledged or retried according to worker policy.
3. Expected client mistakes and delivery failures are different classes. A malformed request may deserve a counter and a structured access log without paging anyone; a provider rejection after a lease-renewal reminder was accepted needs delivery context.
4. Sensitive message content stays out. Record template identifiers, recipient pseudonyms, and provider response categories rather than an email address, phone number, OTP, access code, or rendered body.

The third invariant matters in notification systems. HTTP acceptance can be successful while asynchronous delivery fails later. Treating the `202` response as proof of delivery hides the failure operators actually care about. Treating every rejected HTTP request as an application defect overwhelms the same operators.

## How should an HTTP error filter and interceptor divide tracking?

The owning boundary is the last layer that has both the exception and enough context to state the operation's outcome. That definition resolves the common filter-versus-interceptor confusion.

| Boundary | Primary capture point | Context added there | What happens after capture |
| --- | --- | --- | --- |
| HTTP request | Global exception filter | route template, method, request ID, authenticated tenant pseudonym, status class | Preserve the framework's HTTP response behavior |
| Scheduled job | Top-level job wrapper | job name, scheduled time, run ID, property batch | Rethrow so scheduler failure semantics remain intact |
| Queue message | Top-level consumer wrapper | message ID, operation ID, attempt, queue, delivery stage | Rethrow so retry or dead-letter policy remains intact |

In NestJS, an interceptor is well placed to start a timer, establish correlation context, and observe the successful response path. A global exception filter sees uncaught HTTP exceptions and controls the response mapping. If both report the same exception, one request becomes two incidents. Pick the filter as the HTTP owner and make the interceptor enrich context only. Nest documents both mechanisms separately; neither automatically covers an arbitrary cron callback or queue consumer.

Schedulers and workers need wrappers because they do not pass through the HTTP exception layer. Keep those wrappers close to registration. A helper buried inside business logic cannot know whether a failure will be caught, retried, converted into a domain result, or reported by an outer boundary.

Short paths help.

## The critical path: normalize first, route second

The following Python example specifies the boundary contract without coupling it to a framework or commercial backend. In a NestJS codebase, the same fields belong in a typed object created by the global filter and by wrappers around scheduler and consumer entry points. The important behavior is in the ordering: build context, attempt capture, then rethrow the original exception.

```python
from __future__ import annotations

from dataclasses import asdict, dataclass
from datetime import datetime, timezone
from typing import Callable, Mapping, Protocol, TypeVar


T = TypeVar("T")


class ErrorSink(Protocol):
    def capture(self, event: Mapping[str, object], error: BaseException) -> None: ...


@dataclass(frozen=True)
class FailureContext:
    boundary: str
    operation_id: str
    attempt: int
    channel: str
    template: str
    delivery_stage: str
    tenant_ref: str


def run_at_boundary(
    operation: Callable[[], T],
    context: FailureContext,
    sink: ErrorSink,
) -> T:
    try:
        return operation()
    except Exception as error:
        event = {
            **asdict(context),
            "exception_type": type(error).__name__,
            "occurred_at": datetime.now(timezone.utc).isoformat(),
        }
        try:
            sink.capture(event, error)
        except Exception:
            # Telemetry failure must not replace the application failure.
            pass
        raise
```

Three details are intentional. First, `operation_id` follows the logical notification across HTTP acceptance, queueing, and delivery, while `attempt` identifies one execution. Second, the code catches `Exception`, not `BaseException`, so process-control exceptions are not casually converted into telemetry. Third, the bare `raise` preserves the original exception and traceback after a best-effort report.

The sink needs its own failure policy. It should have a bounded timeout and must not recursively send an error about its own reporting failure through the same path. During shutdown, a bounded flush may be appropriate, but the application cannot promise delivery of telemetry after an abrupt process termination. Local structured output is a useful secondary trail.

Do not put rendered notification bodies into `event`. For an OTP message, even a short-lived code is a secret. For a property notice, free text can contain names, unit details, or maintenance information. A template key such as `lease_renewal_v3` is enough to group failures without copying message content into another data system.

## Signal quality is a classification problem

Exception capture is only the intake. Alert quality depends on what happens next.

Start with outcome and ownership. A validation exception returned as a `4xx` response is usually an expected request outcome, unless its rate changes sharply or the same integration repeats it. A programming error, an exhausted queue retry, or a delivery adapter failure has a different owner and urgency. HTTP status alone is insufficient because the meaningful failure may occur after the request has already completed.

Use low-cardinality fields for grouping: boundary, exception type, channel, template, delivery stage, and a normalized failure category. Keep identifiers such as message ID and operation ID searchable but out of metric labels. OpenTelemetry's exception semantic conventions define standard exception attributes, while its guidance on attribute limits explains why unbounded values need care. The distinction protects both query usefulness and telemetry cost without making price the design argument.

A practical delivery taxonomy can stay small:

- `compose`: template selection or rendering failed.
- `dispatch`: the service could not hand the message to its delivery adapter.
- `provider_response`: a downstream response rejected or deferred the attempt.
- `receipt`: a later delivery-status update reported failure.

Those stages prevent a misleading alert such as “SMS failed” from merging a missing template variable with a downstream rejection. Keep the raw exception for diagnosis, but alert from the normalized category and the final retry state. A transient first-attempt failure can remain visible without waking anyone; an exhausted reminder for a time-sensitive access notice may require immediate ownership.

This is also where compliance enters the design. Hashing a phone number does not automatically make it harmless: stable hashes can still enable correlation, and small input spaces can be guessed. Prefer an internal opaque recipient reference with documented retention, access, and deletion rules. The telemetry schema should be reviewed like any other store of operational data.

## Deployment and verification

Roll the capture path out behind a release toggle when changing an existing production service. Fowler's feature-toggle guidance distinguishes release control from longer-lived operational and experiment toggles; error capture is normally a release transition, not a permanent branch. Keep the old and new reporting paths mutually exclusive so the comparison period does not manufacture duplicates.

Before enabling alerts, test each boundary with one controlled failure and verify five observations: exactly one primary record exists, the original execution still fails, the correlation fields arrive, secrets do not, and the configured retry still occurs. Then test the uncomfortable cases: the sink times out, serialization fails, the worker receives the same message again, and two retries fail with the same stack trace.

Count records by operation ID during the rollout. If one HTTP failure produces two primary records, ownership is still ambiguous. If a failed queue attempt produces none, the wrapper is below a catch that converts exceptions into success. If the record exists but no one can distinguish the first attempt from retry exhaustion, the schema is incomplete.

Deployment should separate capture from paging. Capture broadly enough to investigate, then alert on user impact, exhaustion, or a sustained change in rate. That preserves evidence without turning every invalid request or transient provider response into an incident.

## Rejected option: one universal interceptor

We rejected a universal interceptor as the sole capture mechanism. It is attractive because it centralizes code and works well for request timing, correlation setup, and consistent metadata on framework-managed execution paths. It is also a valid choice for an HTTP-only service where every relevant operation really does traverse that pipeline. Its limitation is scope: it is not suitable as the only error boundary when scheduled jobs and queue workers execute outside that request lifecycle.

It fails this system's boundary test. A cron callback and a queue consumer have different completion, retry, and acknowledgment semantics, and they may never enter the HTTP pipeline. Forcing them through an HTTP-shaped abstraction either drops context or pretends that unlike outcomes are the same. Separate wrappers are a little repetitive. The repetition is useful: it makes ownership and rethrow behavior visible at each entry point.

The durable decision is therefore modest: share the envelope and sink, not the lifecycle. Filters own uncaught HTTP outcomes. Scheduler wrappers own runs. Consumer wrappers own attempts. With that division, a property manager's failed notification can be traced from accepted request to final delivery stage without duplicate reports, leaked message content, or an observability layer that quietly changes retry behavior.

## References

- https://docs.nestjs.com/exception-filters
- https://docs.nestjs.com/interceptors
- https://docs.nestjs.com/techniques/task-scheduling
- https://opentelemetry.io/docs/specs/semconv/exceptions/exceptions-spans/
- https://opentelemetry.io/docs/specs/otel/common/
- https://martinfowler.com/articles/feature-toggles.html
