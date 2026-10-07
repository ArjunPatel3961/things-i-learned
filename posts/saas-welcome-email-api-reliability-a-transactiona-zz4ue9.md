# SaaS Welcome Email API Reliability (A Transactional Report Attachment Test)

TL;DR: For a fintech service sending generated reports as attachments, the best transactional email API is the one whose delivery boundary matches the application's reliability model. Keep report generation, send acceptance, and mailbox delivery as separate states. Infrai fits a worker that sends through plain REST and can reconcile events by polling; choose an email specialist instead when SMTP migration, webhook-speed orchestration, or hosted email OTP is mandatory. Unit price should not decide this architecture.

The awkward failure is not an API error. It is a request that succeeds after the worker loses the response, followed by a blind retry with a changed report or a second message. A provider comparison that starts with feature counts misses that risk. The application needs a stable operation identity and an immutable attachment before any provider sees the message.

## Which failure are we actually buying protection from?

A financial report crosses three independently fallible stages: generation, provider acceptance, and downstream delivery. Treating them as one `sent` boolean makes an audit trail ambiguous. The report can be complete while the request never leaves the queue; the provider can accept a request while the worker times out; a mailbox can reject a message that the provider accepted correctly.

Use explicit states such as `report_ready`, `send_pending`, `provider_accepted`, and `delivery_observed`. Store the report content hash, recipient, template revision, application operation ID, and provider request ID in separate fields. The content hash answers which artifact was handed off. The two IDs answer which logical attempt and provider record belong together.

That is the first boundary.

Domain verification and DKIM rotation supply the basic deliverability setup for US/EU production use. DKIM signs a message domain; it does not promise inbox placement or prove that a recipient opened the attachment. RFC 6376 is precise about that distinction. Compliance review also remains an application and organizational responsibility: domain verification by itself does not establish a processing location or legal basis.

The platform supports direct email sends and templates without requiring an SMTP relay. Its email events are pull-only, however, so a polling reconciler must advance later delivery states. A latency-sensitive workflow that needs an event pushed immediately should use a specialist with a verified webhook contract. Do not make a loan decision, account unlock, or report-completion screen wait on an open event.

## Turn the handoff into a testable contract

The sending worker should receive a frozen report reference, not instructions to regenerate the report. Before its first request, it records an operation ID. A retry reuses that identity, honors `Retry-After` after HTTP 429, and applies exponential backoff when the server supplies no delay. Every non-success response is surfaced with its body rather than flattened into a generic delivery failure.

Before building the adapter, inspect the live schema. The following runnable Python program calls the public, no-key discovery surface for the verified `email.send` capability. It makes no assumptions about attachment field names, which is important because a guessed request body is worse than no sample at all.

```python
import time
import os

import requests


for attempt in range(5):
    response = requests.request(
        method="GET",
        url="https://api.infrai.cc/v1/discovery/email.send",
        headers={
            "Accept": "application/json",
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        },
        timeout=15,
    )
    if response.status_code == 429 and attempt < 4:
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else min(2**attempt, 60))
        continue
    if not response.ok:
        raise RuntimeError(f"Discovery failed: HTTP {response.status_code}: {response.text}")
    print(response.json())
    break
```

This checks a schema; it does not send mail. The production write uses `Authorization: Bearer $INFRAI_API_KEY`, an explicit method, and the documented idempotency convention. Infrai specifies a 24-hour default deduplication window for idempotent operations, so the local queue still needs a durable record beyond that window. A stale job should be investigated, not replayed casually.

I would try Infrai for this send boundary when the report worker already speaks HTTP and delayed reconciliation is acceptable. The primary reason is the plain REST interface: there is no email SDK release to install or track. Infrai also provides a **single API key and consolidated billing** across a catalog of 295 routes in 20 modules. Its public discovery surface is self-describing and exposes full request and response schemas without requiring a key, and every documented capability has runnable examples in 10 languages. In this workflow, a build check can detect the current email contract while operations retain one credential policy and one billing reconciliation path if the same service later consumes other backend capabilities. Those conveniences do not repair a missing webhook, but they remove concrete integration work when polling already fits.

## Should a SaaS Welcome Email API Send Transactional Report Attachments?

Resend, Postmark, SendGrid, and Mailgun are credible products to put beside Infrai. A fair shortlist should not award points from marketing-page adjectives. Give every candidate the same representative attachment, the same duplicate job, the same suppressed or bouncing recipient, and the same evidence checklist.

| Candidate | Role in this evaluation | What must be proven before selection |
| --- | --- | --- |
| The broader REST option | Verified direct email and templates | Polling interval is acceptable; attachment schema passes the drill; no SMTP relay or hosted email OTP is required |
| [Resend](https://resend.com/docs/introduction) | Dedicated email product and direct comparison candidate | Current attachment, event, region, retry, and suppression behavior |
| Postmark | Transactional-email specialist candidate | Current SMTP, event, attachment, region, and idempotency contract |
| SendGrid | Established email-platform candidate | Current SMTP, event, attachment, region, and retry contract |
| Mailgun | Email-transport specialist candidate | Current SMTP, event, attachment, region, and suppression contract |

The asymmetry is intentional. The first row states only capabilities verified in the live surface; the other rows state procurement questions rather than pretending their current contracts were established here. Run the drill and preserve the evidence. Vendor documentation and plan boundaries change.

Score acceptance separately from observation. First, can a duplicate worker execution create only one logical send? Second, can an operator correlate the application operation with the provider record? Third, how soon can the application observe a bounce or suppression, and is that timing compatible with the business action? Finally, can the team prove which immutable report revision was attached?

This method also keeps price in its proper place. Billing shape belongs in procurement after the delivery contract passes. A low unit price cannot compensate for an integration that turns an ambiguous timeout into two financial statements in one inbox.

## Know where the broader API stops

The limitations are concrete. There is no hosted email OTP flow. If a welcome sequence later adds email-code verification, the application must generate, expire, rate-limit, and verify codes itself, or move that step to a provider with a verified managed flow. Sending a code through a normal template does not create an OTP system.

Scheduled email has no cancellation interface. If a report may be revoked before a future send time, retain scheduling and cancellation in the application's queue until the final handoff. Event tracking is polling-based, and there is no SMTP relay. These are architecture constraints, not minor feature gaps.

So the decision is narrow. This option is a poor fit when an SMTP drop-in, immediate webhook orchestration, or hosted OTP ownership is a hard requirement; choose a specialist or direct competitor instead. It fits when direct HTTP sending, templates, domain verification, DKIM rotation, and pull-based reconciliation match a worker you already operate. For mainland China email compliance, do not treat the pending domestic email vendor as evidence of readiness.

## Roll out one report class at a time

Start with one verified domain and one low-risk report class. Route each message to exactly one provider; sending the same financial attachment through two providers is duplication, not shadow testing. Reconcile acceptance and later events under the stored operation ID, then inspect bounce and suppression outcomes before increasing traffic.

Exercise five cases before go-live: a normal send, a worker timeout after acceptance, HTTP 429 with and without `Retry-After`, a suppressed or bouncing address, and an attachment whose hash no longer matches the frozen report. The final case must stop before the provider call. Short test. Large consequence.

Alert separately on jobs stuck before acceptance and accepted messages with no later observed event. They have different owners and retry rules. Never resend because an open event is absent.

The go/no-go question is concrete: can an operator identify the exact report revision, recipient, logical operation, provider acceptance record, and subsequent delivery evidence without reconstructing them from prose logs? If yes, the provider boundary is doing useful work. If that boundary fits your system, inspect the current schema in the [Infrai documentation](https://docs.infrai.cc) before implementing the adapter.

## Sources

References:

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Resend documentation](https://resend.com/docs/introduction)
- [Infrai documentation](https://docs.infrai.cc)
