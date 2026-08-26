# Choosing an SMS Service Route for US/EU Passwordless Recovery and Account Alerts

Short answer: choose an SMS route only after it passes a destination-weighted delivery trial, exposes usable delivery evidence, and fits a provider-independent recovery design; a low submitted-message rate cannot compensate for missing or late passwordless backup alerts.

This decision record does not crown a permanent winner. Twilio, Vonage, and Telnyx can be candidates in the same trial, but their names do not settle the decision. The application should own expiry, abuse controls, consent evidence, and notification state. A replaceable adapter should own the syntax of submission and receipts.

That boundary matters because an accepted request is not a message on a handset. I've chased enough delivery gaps to distrust a clean 2xx response — it proves that one interface accepted work, not that the user can finish account recovery. The useful comparison unit is therefore an eligible, terminally observed delivery to a real destination class, not an API call.

## How should US and EU teams compare SMS for passwordless backup account alerts?

Start with invariants. OWASP's forgot-password guidance calls for a consistent response for existing and nonexistent accounts, a side channel for reset material, random tokens or codes, secure storage, single use, expiry, and protection against excessive submissions. Those controls belong to the account system. Switching an SMS route must never reset an account's attempt counter, extend a code's lifetime, or create a second independently valid recovery challenge.

Keep the message modest too. It should not disclose whether a phone number belongs to an account, and operational telemetry should not copy a code or full destination into logs and metric labels. The server remains the authority on validity. A receipt arriving after expiry may be useful delivery evidence, but it cannot revive the challenge.

Consent deserves a separate record from message content. GDPR Article 7 describes conditions that apply when processing is based on consent: the controller must be able to demonstrate it, the request must be distinguishable and intelligible, and withdrawal must be as easy as giving consent. It does not establish that consent is the lawful basis for every security notification. I'm not sure a generic provider checklist can determine the right basis or retention period for a particular product; counsel and the product's actual processing purpose resolve that question. Engineering can still preserve the policy version, capture time, channel, region, and withdrawal state without retaining an OTP.

Then define a destination-weighted trial. Use the countries, sender configurations, languages, and message lengths that resemble the intended traffic. For each candidate, record queued-to-accepted time, accepted-to-terminal time, terminal-status coverage, duplicates, and expiry-before-delivery. Do not mix US and EU observations into one average: a route that looks fine in aggregate can still fail the smaller regional cohort that needs it most.

Test the awkward cases on purpose. Exercise a duplicated job, a repeated receipt, receipts arriving out of order, worker termination after submission, an expired challenge, and a submission whose outcome is temporarily unknown. Also verify that suppression and account-level rate limits live above route selection. Spam filtering, sender rules, and destination behavior can change independently, so the trial result should have an owner and a review date rather than becoming permanent folklore.

No shortcuts.

## Invariants and failure boundaries

The durable object is a notification intent. It contains an opaque intent identifier, an idempotency key, a destination reference, a template and locale, an expiry time, and a security purpose. It is written before a worker contacts an adapter. Message text and the resolved phone number should have the shortest practical lifetime and a narrower access path than ordinary operational metadata.

Use a small state machine: `queued`, `accepted`, `delivered`, `undeliverable`, `expired`, and `unknown`. `accepted` means the route took custody. It is explicitly nonterminal from the user's perspective. An authenticated receipt can advance the intent, while an expiry check can close it without waiting forever. Receipt handling must be idempotent and must reject backward movement such as `delivered` to `accepted`.

The dangerous interval sits between remote submission and the local commit. Picture a worker that claims intent `a17`, submits it, receives an acceptance identifier, and loses its lease before persisting that identifier. The next worker sees no acceptance. If it treats the missing row as proof that nothing happened, the user may receive two codes; if it refuses all further action, the user may receive none. A green request counter can't distinguish those outcomes. The recovery path therefore needs one of two verified properties: a route-supported idempotency contract, or reconciliation by the remote message identifier before another submission is permitted. If neither property is available, mark the outcome `unknown` and apply an explicit operator or product policy. Alert on the share and age of those intents by destination class, template, and route, measure the two latency legs separately, and let support see custody and delivery as different facts. Synthetic traffic can test the full path, but its frequency, numbers, and consent handling need review against local rules. Your mileage may vary because the evidence required for a low-risk informational alert is not the same as the evidence required for the only recovery path into an account. One more boundary follows from the same scenario: fallback does not mean retry. A fallback may be a different verified recovery channel that the user already controls, and it must obey the same challenge lifetime and abuse budget. Sending another SMS through a second route while the first outcome is unknown is a duplicate-risk decision, not harmless redundancy. Don't silently guess.

Receipts decide.

## Option comparison and total-cost evidence

The shortlist should first satisfy security policy, destination eligibility, receipt semantics, data-processing requirements, and operational support needs. Cost comparison comes after those gates. Ask every candidate to price the same country mix, sender setup, segment distribution, and support assumptions, then calculate cost per terminally observed eligible delivery. Registration work, number rental where applicable, engineering ownership, monitoring, duplicate handling, and support contacts belong in the model. Public headline rates alone are not comparable evidence.

| Architecture option | Evidence to collect | Useful when | Limitation |
|---|---|---|---|
| One aggregator behind an adapter | Terminal-state coverage, regional delivery time, duplicate observations, support response | Traffic is modest and the team wants a small operating surface | One route policy and control plane remain a shared boundary |
| Two aggregators behind one policy layer | All single-route measures plus failover duplication and reconciliation drills | Regional evidence justifies route diversity and the team can own it | More compliance review, testing, on-call decisions, and state reconciliation |
| Carrier-direct connections | Destination volume, sender setup, receipt mapping, and staffing requirements | Sustained concentrated traffic supports specialist operations | Poor fit for a small mixed-country workload or a thin operations team |
| SMS plus a verified non-SMS recovery path | Completion, abuse, accessibility, and support outcomes by channel | Phone reachability cannot be the only route back into an account | Requires product work and a carefully reviewed recovery policy |

The catch is organizational. Two routes can reduce dependence on one delivery path while increasing ambiguity during an incident. Someone must be able to explain which policy selected the route, whether a receipt is final, and whether another message may be sent. If no team owns those questions, stick with one adapter and invest in a separately verified recovery channel. Carrier-direct work is not suitable when destination volume is fragmented or compliance and on-call staffing are limited.

Candidate selection should use the same replay set and scoring weights. Record the raw observations as well as the score, since weights change when the product enters a new country or account alerts become authentication-critical. A candidate that wins on one US-heavy dataset may not win on an EU-heavy one. That is not inconsistency; it is the point of weighting the decision with actual traffic.

## Critical path in Python

The code below defines the application-side contract. It intentionally contains no commercial endpoint, credential format, or receipt signature scheme. Each adapter must translate its selected route's documented submission and authenticated receipt behavior into this small model, while shared policy stays above it.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from enum import Enum
from typing import Protocol


class DeliveryState(str, Enum):
    ACCEPTED = "accepted"
    DELIVERED = "delivered"
    UNDELIVERABLE = "undeliverable"
    UNKNOWN = "unknown"


@dataclass(frozen=True)
class SmsIntent:
    intent_id: str
    idempotency_key: str
    destination_ref: str
    template_id: str
    expires_at: datetime


@dataclass(frozen=True)
class Submission:
    message_id: str
    state: DeliveryState


class SmsRoute(Protocol):
    def submit(self, intent: SmsIntent) -> Submission: ...


class IntentStore(Protocol):
    def claim(self, intent: SmsIntent) -> bool: ...

    def record_acceptance(self, intent_id: str, message_id: str) -> None: ...

    def record_unknown(self, intent_id: str) -> None: ...


def dispatch(intent: SmsIntent, route: SmsRoute, store: IntentStore) -> Submission:
    if intent.expires_at <= datetime.now(timezone.utc):
        raise ValueError("expired notification intent")

    # The claim is atomic across workers and all available SMS routes.
    if not store.claim(intent):
        raise ValueError("idempotency key already claimed")

    try:
        submission = route.submit(intent)
    except TimeoutError:
        store.record_unknown(intent.intent_id)
        raise

    if submission.state is not DeliveryState.ACCEPTED:
        raise ValueError("route violated the submission contract")

    store.record_acceptance(intent.intent_id, submission.message_id)
    return submission
```

The example stops at custody because delivery belongs to the receipt path. That handler authenticates the callback using the route's documented method, locates the intent by message identifier, maps external statuses to internal states, and applies a monotonic transition. Tests should feed it duplicate and reordered receipts. They should also prove that `delivered` after expiry records evidence without changing challenge validity.

I reject direct submission from an account controller for passwordless recovery. It ties user-facing request latency to a remote system, scatters rate-limit and consent decisions, and makes acceptance look temptingly close to completion. The valid use case is narrower: a low-risk internal alert with tiny volume, modest consequences for delay, and a team prepared to reconcile it manually may not justify a queue and receipt state machine. Keep the direct integration there. Do not carry it into the only backup path for a user account.

The decision is complete when the record names the invariant owner, trial dataset, evidence threshold, review date, and exit trigger. It should select the smallest architecture that preserves expiry, idempotency, abuse controls, consent evidence, and delivery reconciliation. The provider can change. Those properties cannot.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://gdpr-info.eu/art-7-gdpr/
