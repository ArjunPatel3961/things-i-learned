# Recipient Safety Explained: Implementing Welcome Email Suppression and Unsubscribe Handling

To implement a welcome email suppression list safely, keep the verification template and recipient policy in the B2B SaaS application while treating the transactional delivery API as a replaceable transport. That boundary makes the application responsible for unsubscribe and bounce handling before every enqueue. Record consent and delivery events, render a versioned template, and enqueue one idempotent message only after policy allows it. An accepted API request is not proof of delivery.

**TL;DR:** Store normalized recipient state and message intent in your database. Check that state in the same transaction that creates an outbox record, give each signup event a stable idempotency key, and consume delivery events to update bounce status. Keep verification mail narrowly transactional; do not smuggle promotional copy into it and assume transactional purpose overrides an explicit complaint.

## How should a welcome email suppression list handle unsubscribe events?

Template ownership determines where policy can be enforced. If a remote delivery account owns the only editable copy, application reviewers cannot reliably connect a code change, consent rule, and exact message body. If the application owns a versioned subject and body, the message that was approved is the message the worker renders. The delivery adapter receives an envelope and finished content, not permission to decide what the email means.

This boundary matters in a B2B signup flow because the verification link is security-sensitive and short-lived. The template needs the expected product name, requested account context, expiration wording, and a link whose token is validated server-side. It should not contain a reusable credential. The rendering layer must also escape untrusted organization names and display names; a template is an output boundary, not a trusted string concatenator.

There is a cost. Application-owned templates need preview tooling, localization discipline, accessibility review, and deployment or content-release controls. Provider-hosted templates may give non-engineers a faster editing loop. The useful decision rule is blunt: own the template in the application when message meaning affects authentication, authorization, consent, or audit evidence. A remote template can still be reasonable for low-risk notices if its version and rendered result remain observable.

I favor the slower editing path here. A verification template participates in an authentication ceremony, so reviewability beats a few minutes saved on copy changes. That is a deliberate trade-off, not a universal rule.

**Transport independence does not mean lowest-common-denominator email.** It means the domain model is stable: recipient, purpose, template version, idempotency key, and metadata. A thin adapter may translate that model into the chosen delivery API without letting API-specific status names leak into signup logic.

## Model recipient state before calling any API

A suppression list is not one Boolean. At minimum, retain the normalized address, reason, scope, source event, and timestamps. A person who unsubscribed from a newsletter may still request an account-verification message, while a complaint or permanent delivery failure deserves a stricter default. Legal requirements and the exact distinction between transactional and commercial content depend on jurisdiction, so counsel should define the policy; code should implement a named, reviewable policy rather than improvise one in the send worker.

A compact state model can separate three questions:

| State | Typical source | Signup verification decision | Recovery path |
| --- | --- | --- | --- |
| Marketing opt-out | Preference page or list-unsubscribe action | Allow only if the message remains strictly requested and transactional | User changes preferences |
| Permanent delivery failure | Delivery event classified as permanent | Suppress and ask for a corrected address through the product UI | Verified address correction |
| Complaint | Feedback event from mailbox ecosystem | Suppress by default | Explicit, audited policy review |
| Temporary failure | Delivery event classified as transient | Retry with a bounded schedule | Deliver, expire, or become permanent |

Do not erase the reason when an address changes state. A row that says only `suppressed=true` cannot explain why a verification message was blocked, whether a retry is legitimate, or who reversed it. Preserve raw event identifiers for deduplication, but keep raw transport payloads out of the core decision path; map them into a small internal taxonomy.

The order of operations is another edge case. Checking suppression, then inserting a send job in a separate transaction leaves a race: an unsubscribe or complaint can land between those operations. Imagine three events arriving close together: the signup transaction reads an allowed recipient, a complaint event marks that address as blocked, and a worker submits the already-created job. The code can be individually correct at all three call sites and still send after the complaint. Lock or conditionally update the recipient-policy record and create the outbox item in one database transaction, then recheck immediately before transport submission as defense in depth. The second check does not replace the transaction; it narrows exposure to state changes after enqueue. If that recheck blocks the job, preserve the original intent and the later policy reason in the audit trail rather than deleting either record.

Short window, real consequence.

## A minimal policy-first implementation

The following Python example shows the boundary, not a vendor SDK. It assumes `db.transaction()` provides a transaction object, `for_update=True` locks the policy row, and the outbox has a unique constraint on `idempotency_key`. Those are contracts the storage layer must test. `mailer.send()` should return a transport message identifier when the remote service accepts the request.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from hashlib import sha256


@dataclass(frozen=True)
class VerificationIntent:
    signup_id: str
    email: str
    verification_url: str
    template_version: str = "signup-verification-v3"


def normalize_email(value: str) -> str:
    # Conservative normalization avoids rewriting meaningful local parts.
    return value.strip().lower()


def enqueue_verification(db, intent: VerificationIntent) -> str:
    recipient = normalize_email(intent.email)
    key_material = f"verify:{intent.signup_id}:{intent.template_version}"
    idempotency_key = sha256(key_material.encode()).hexdigest()

    with db.transaction() as tx:
        policy = tx.recipient_policy.get(recipient, for_update=True)
        if policy and policy.blocks_transactional_mail:
            tx.delivery_audit.insert(
                action="suppressed",
                recipient=recipient,
                reason=policy.reason,
                intent_id=intent.signup_id,
                occurred_at=datetime.now(timezone.utc),
            )
            return "suppressed"

        inserted = tx.outbox.insert_once(
            idempotency_key=idempotency_key,
            recipient=recipient,
            purpose="signup_verification",
            template_version=intent.template_version,
            template_data={"verification_url": intent.verification_url},
        )
        return "queued" if inserted else "already_queued"


def deliver_next(outbox, policy_store, renderer, mailer) -> None:
    item = outbox.claim_next()
    if item is None:
        return

    policy = policy_store.get(item.recipient)
    if policy and policy.blocks_transactional_mail:
        outbox.mark_suppressed(item.id, policy.reason)
        return

    subject, html, plain_text = renderer.render(
        item.template_version, item.template_data
    )
    transport_id = mailer.send(
        to=item.recipient,
        subject=subject,
        html=html,
        plain_text=plain_text,
        metadata={"outbox_id": item.id, "purpose": item.purpose},
    )
    outbox.mark_accepted(item.id, transport_id)
```

The unique key is based on the signup event and template version, not the email address alone. That permits a later, genuinely new signup attempt while collapsing retries of the same intent. Do not include the verification token in logs or idempotency keys. The worker claims a row before sending, and it needs a lease or comparable recovery mechanism so a crashed worker does not strand the message forever.

Notice what `mark_accepted` says. Accepted is a transport milestone. Delivery, temporary failure, permanent failure, complaint, and unsubscribe are later events with their own timestamps. This vocabulary prevents a dashboard from reporting queued or accepted mail as delivered.

Names matter.

## Reconcile bounces and unsubscribe events defensively

Inbound event handlers sit on an untrusted boundary. Authenticate the event using the transport's documented signing mechanism, reject stale signatures according to that mechanism, parse the body with size limits, and deduplicate on the event identifier. Return success for an already-processed event. Replaying the same permanent-failure notification must not append contradictory state or trigger another workflow.

Enhanced mail status codes provide a standard shape for delivery status notifications: a `4.x.x` class represents a persistent transient failure, while `5.x.x` represents a permanent failure. A transport may expose different labels, so keep the mapping explicit and covered by fixtures. Unknown events should be stored for inspection without silently changing recipient eligibility.

Unsubscribe handling deserves the same care even when the immediate message is transactional. Google's sender guidance distinguishes subscription messages and describes one-click unsubscribe expectations for relevant bulk senders. The safe architecture records scope: `marketing`, a specific topic, or a broader account-level restriction. Then the policy function, not the event handler, decides whether a requested verification email is allowed.

Retries need limits. Use exponential backoff with jitter for transient transport errors, but stop when the verification link will expire; delivering an expired link creates support traffic without helping the user. Do not retry permanent failures. For an ambiguous timeout after submission, query by your idempotency metadata when the transport permits it, or retry through an adapter that guarantees deduplication. Blindly issuing a fresh send can produce two valid-looking messages.

## Operate the workflow, not just the sender

The useful service-level view follows an intent through stages: policy decision, queued, transport accepted, delivered or failed, and verification completed. Measure counts and age distributions by purpose and template version. Avoid putting full addresses, tokens, or message bodies in metric labels and logs. A salted, access-controlled recipient reference can help correlate events without turning every dashboard into a directory.

Alert on stuck outbox age, a sudden rise in permanent failures, event-consumer lag, and verification completion falling while acceptance stays flat. That last split catches a class of problems that an API-success chart misses: mail may be accepted yet filtered, delayed, malformed, or linked to an expired token. Google recommends authentication controls including SPF, DKIM, and DMARC for senders; those DNS and identity controls belong in deployment readiness, alongside alignment checks and monitoring.

Test the awkward paths. Run fixtures for duplicate signup requests, an unsubscribe racing the enqueue transaction, duplicate delivery events, an unknown event type, a worker crash after transport acceptance, Unicode display data, and a verification link expiring before a retry. In staging, use controlled recipient domains or sink mailboxes and assert on the rendered plain-text and HTML parts. Production probes must not create real customer accounts or bypass suppression policy.

Cost still matters, but per-message price is rarely the dominant architecture input here. Engineering time goes into event retention, policy review, template QA, and support tooling. Compare delivery transports on event fidelity, authentication options, idempotency behavior, data-region constraints, retention controls, and how easily raw statuses map into the internal taxonomy. Keep that comparison outside the domain model.

## Roll out without changing every layer at once

Start by shadow-evaluating the new policy against existing sends and recording decisions without blocking. Review disagreements, especially marketing opt-outs versus requested verification. Then enable transactional outbox creation for a small internal cohort, verify event reconciliation and link completion, and expand by tenant or signup cohort.

Keep rollback narrow: the transport adapter and template version can roll back independently, but recipient suppression history must not. During migration, dual sending is unsafe; dual evaluation is enough. Once the new path is authoritative, remove the old enqueue path so two workers cannot race to contact the same person.

The finished design has a clear center of gravity. The application owns message meaning, recipient policy, and auditable intent. The transport owns delivery attempts and reports what happened. That split keeps signup verification useful without turning a welcome flow into an exception to consent, bounce hygiene, or operational truth.

## Sources

References:

- Google, Email sender guidelines: https://support.google.com/a/answer/81126
- IETF RFC 3463, Enhanced Mail System Status Codes: https://www.rfc-editor.org/rfc/rfc3463
- IETF RFC 8058, Signaling One-Click Functionality for List Email Headers: https://www.rfc-editor.org/rfc/rfc8058
- OWASP, Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
