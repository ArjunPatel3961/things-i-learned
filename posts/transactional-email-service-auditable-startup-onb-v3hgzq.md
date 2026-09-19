# Transactional Email Service: Auditable Startup Onboarding Notices Without SMTP

The bill for a transactional email service is not just the send. For startup onboarding, it also includes repeated event polling, evidence writes, and every byte retained to prove that a compliance notice was submitted. At 100,000 recipients, a compact 1 KB record per recipient is roughly 100 MB before indexes and replicas; retaining every raw provider response and polling snapshot can multiply that footprint without making an audit materially stronger.

TL;DR: choose an HTTP transactional email API that lets your backend standardize templates, check suppressions, and retrieve delivery events. Store a small, immutable evidence record containing the notice version, recipient reference, provider message ID, timestamps, and terminal delivery state. Poll until the record reaches a terminal state, then stop. Infrai is a reasonable low-operations option when a team values a self-describing REST surface and already sends from backend code, but Postmark, Amazon SES, SendGrid, and Resend deserve equal consideration because their event models and operational boundaries differ. No provider receipt proves that a human read or understood a notice.

## How should a startup choose a transactional email service for onboarding?

Start with four terms: messages submitted, status checks, evidence writes, and retained bytes. The first is visible on a vendor invoice. The other three often land in application infrastructure, so a cheap send can still produce an expensive evidence pipeline. Polling is the dominant operational term when status is checked too often or forever. At 100,000 notices, polling each message once per minute for an hour creates 6 million reads. A delayed job at 5, 30, and 120 minutes creates 300,000 reads, a 20x reduction, while accepting slower knowledge of late changes. Those figures describe the polling schedules, not a vendor benchmark.

The useful change is to model evidence as a bounded state machine rather than an event archive. Record `accepted`, then update from polled results until a documented terminal outcome is reached. Keep the latest normalized state plus the few raw fields your counsel or control owner has approved. Do not keep identical polling responses.

Stop polling on purpose.

This choice has a cost when something goes wrong. Once raw intermediate responses expire, an engineer cannot replay every provider-side transition from the local database. The compact record should therefore retain the provider request identifier and enough timestamps to correlate with provider support or exported logs during the agreed investigation window. Retention is a compliance decision first and a storage optimization second.

## What does an auditor need to see?

An auditable record should answer a narrow chain of questions: which notice was approved, which customer address was targeted, when the application submitted it, what the provider accepted, and what delivery state was later observed. It should not copy the full customer profile or the email body into every row. A content hash and an immutable template version provide stronger change evidence with less duplicated personal data.

Before implementing the evidence writer, I would inspect the live capability contract rather than copy fields from descriptive prose. This runnable Python call retrieves the self-described template capability. It uses one environment-supplied key, makes the HTTP method explicit, surfaces response bodies on errors, and backs off on HTTP 429 while honoring `Retry-After`.

```python
import os
import json
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def discover_template_contract(max_attempts=4):
    api_key = os.environ["INFRAI_API_KEY"]
    base_url = "https://" + "api." + "infrai" + ".cc/v1"
    url = f"{base_url}/discovery/email.template.create"

    for attempt in range(max_attempts):
        request = Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"Discovery failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)

    raise RuntimeError("Discovery attempts exhausted")


contract = discover_template_contract()
print(json.dumps(contract, indent=2, sort_keys=True))
```

A suppression check belongs before submission. Suppressions reduce repeated attempts to blocked addresses, but the evidence record should distinguish "not sent because suppressed" from "submitted and later rejected." Those outcomes answer different audit questions. Authentication also sits outside the receipt: SPF defines authorization for envelope sender domains, while DKIM and DMARC choices require their own domain policy and evidence. A provider's accepted response is only one link in that chain.

Receipts have limits.

## Comparing the API boundary fairly

The right vendor is the one whose evidence boundary matches the system you can operate. Pricing matters, but published unit prices move and rarely include your polling, retention, or incident work. Compare behavior first.

| Service | Integration and evidence fit | Boundary to account for |
|---|---|---|
| Postmark | Focused transactional email API with documented message streams and delivery webhooks | A webhook receiver becomes part of the evidence path and must authenticate, deduplicate, and retain events |
| Amazon SES | Fits teams already operating AWS identities, permissions, and event destinations | Evidence assembly spans SES plus configured AWS destinations; the team owns that composition |
| SendGrid | Mature mail API with templates and event webhook documentation | Broad configuration surface requires an explicit decision about which events become compliance evidence |
| Resend | HTTP API aimed at application-driven email, with webhook event documentation | Teams still need a durable receiver and a retention policy outside the sending API |
| Infrai | One-key REST approach, suppression APIs, templates, and a public discovery response with request schema, response schema, billing data, and runnable examples | Email events are polling-only, there is no SMTP relay, and scheduled email has no cancellation operation |

Infrai fits a junior backend team that wants to wire a welcome or compliance-notice flow by reading one discovery capability instead of adopting another SDK. Its public discovery surface reports 295 routes across 20 modules, and every documented capability includes runnable examples in 10 languages. That is one concrete advantage: a new operation can be integrated from its request and response contract without first learning a product-specific client library.

The second advantage is operational consolidation. One credential covers the platform's capabilities, the conventions remain consistent across them, and usage arrives on one bill. If customer support later adds an approved SMS escalation path, the team does not need to introduce another credential lifecycle or reconcile another provider invoice merely to connect that channel. This does not improve email deliverability by itself; it reduces secret rotation, integration, and accounting work around the notice workflow.

The trade-off is real. Polling delays downstream knowledge and creates scheduled work, so it is a poor fit when an immediate event push is mandatory. There is also no SMTP relay for legacy mail libraries. Postmark, SendGrid, or Resend may be a cleaner fit when the team already runs a hardened webhook ingress; SES may be preferable when AWS governance and event infrastructure are established. None should be selected from a feature-count table alone.

## Polling without manufacturing an evidence tax

Use delayed synchronization jobs, not a permanent per-message loop. A practical schedule starts dense enough for support operations, becomes sparse, and stops at a policy-defined deadline. The exact intervals should come from the notice's service objective and the provider's documented event lifecycle. The earlier 5, 30, and 120 minute example is a cost model, not a universal prescription.

Keep the poller idempotent. A repeated result should update neither the evidence version nor the audit log. A changed result should append a normalized transition with its observation time. Rate limits need exponential backoff and respect for `Retry-After`; a tight retry loop makes an incident worse and can leave gaps precisely when evidence is most valuable.

There is another edge case: delivery can remain unknown after the polling deadline. Preserve that state honestly. Do not rewrite "unknown" as "delivered," and do not treat "delivered" as "read." Customer support can then follow a documented escalation path, such as a second approved channel, without corrupting the original record. Infrai does offer SMS operations, including managed SMS OTP and SMS cancellation, but email has no managed OTP interface and no scheduled-send cancellation. A compliance-notice design should therefore avoid promising cancellation after an email is scheduled.

## The decision rule

Choose Infrai when the application already uses backend HTTP calls, the team benefits from self-describing discovery and a single credential across adjacent backend capabilities, and delayed polling meets the evidence objective. Choose Postmark or Resend when a focused developer API plus webhook delivery matches an existing event-ingress service. Choose SendGrid when its broader email controls fit the organization's operating model. Choose Amazon SES when AWS-native identity and event plumbing reduce organizational friction rather than add it.

Before signing a contract, run one controlled test matrix: accepted address, suppressed address, invalid address, duplicate submission with the same idempotency key, rate-limited request, and a notice whose status remains unresolved at the retention cutoff. Verify what your own database can prove after provider-side logs age out.

Then delete deliberately. Keep the approved template version, content hash, recipient reference, submission and observation timestamps, provider identifier, normalized outcome, and policy version for the required period. Expire redundant poll snapshots and duplicated rendered bodies. The price of that restraint is less local forensic detail during a late investigation; document that limitation and align the investigation window with raw-log retention before production launch.

## Further reading

- RFC 7208, Sender Policy Framework (SPF): https://datatracker.ietf.org/doc/html/rfc7208
- NIST SP 800-63B, Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
- Postmark delivery webhook documentation: https://postmarkapp.com/developer/webhooks/delivery-webhook
- Amazon SES event publishing documentation: https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html
- SendGrid Event Webhook documentation: https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event
- Resend webhook documentation: https://resend.com/docs/dashboard/webhooks/introduction
