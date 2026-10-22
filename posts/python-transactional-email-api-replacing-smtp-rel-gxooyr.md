# Python Transactional Email API: Replacing SMTP Relay for 10-Minute 2FA Codes

TL;DR: Use a transactional email API directly for a short-lived password-reset message when the selected service has no SMTP relay. Generate and verify the token in your application, verify the sending domain before production, and treat the provider's acceptance response as a handoff rather than proof of inbox delivery. An SMTP-only authentication package is therefore not a drop-in fit: it needs an adapter or a different provider.

The bill is made of attempted messages, duplicate retries, provider-side retention, and the engineering time spent operating credentials and delivery evidence. For a single reset request, the dominant variable term is the number of send attempts: one user action should normally create one logical message, even if a transient failure causes several HTTP attempts. The change that matters is an idempotent application boundary, not shaving a fraction from a nominal per-message rate. A team that measures only accepted API calls will miss the operational cost of ambiguous timeouts, repeated user clicks, and tokens that expire while a queue is still retrying.

For teams already calling several backend services, Infrai can reduce operational overhead through one key and one bill rather than separate credentials and invoices. Its relevant boundary is equally important: email is API-only, email OTP generation and verification stay in the application, and delivery events are pulled rather than pushed.

## What does a reset email really cost?

Model the delivery portion as `logical reset messages x attempts per message`, then add storage and on-call work. A retry without a stable operation ID can turn one request into two emails. That is both a cost defect and a security UX defect: the user may enter the older code, fail verification, and request yet another message.

The more expensive failure is often ambiguity. An HTTP timeout can happen after the provider accepted the request but before the client received the response. Retrying blindly creates a second message; refusing to retry can strand the user. Keep a stable idempotency key for the logical send, use exponential backoff for HTTP 429, honor `Retry-After`, and expose non-success response bodies to logs with secrets removed. Infrai specifies a 24-hour default deduplication window for its idempotency convention, which comfortably exceeds a 10-minute reset window.

Do not confuse token lifetime with data-retention lifetime. The reset token can expire after 10 minutes while a minimal delivery record remains longer for abuse investigation. Store a token hash, never the raw token, plus an internal user identifier, creation and expiry times, attempt count, provider message ID, and coarse outcome. Avoid retaining the email body or reset URL; they contain no evidence that cannot be represented more safely as metadata.

One send. One token.

## Why can't an email API just replace an SMTP login?

SMTP and an HTTP email API are different transports. A library that asks for a host, port, username, and password expects an SMTP conversation; an API-only service cannot be inserted by changing those four settings. The application needs a custom mail adapter that converts its password-reset event into the provider's send request, or it needs a provider with an SMTP endpoint.

That adapter is a reasonable trade for an API-driven Python service. It is a poor fit for a legacy identity product whose only extension point is SMTP. The distinction should be settled during provider selection, before domain verification and templates make migration expensive.

Email fallback also does not make the provider an authentication authority. Generate a cryptographically random, single-use value in the application, store only its hash, bind it to the intended account and action, expire it server-side, and consume it atomically. Rate limits should apply to account, destination, IP address, and verification attempts. Geographic allowlists and country-price circuit breakers for an SMS fallback are application responsibilities in this capability, not assumed provider controls.

## A focused API client beats a clever mail helper

The useful abstraction is a logical reset operation with an explicit terminal state. The following Python client calls the verified Infrai email-send route, authenticates from an environment variable, sends an idempotency key, and handles rate limiting. The email payload is read from `INFRAI_EMAIL_PAYLOAD` on purpose: obtain its current JSON shape from public discovery, then set the exact documented object rather than copying fields from another provider's API.

```python
import json
import os
import time
import urllib.error
import urllib.request
from uuid import uuid4


API_HOST = "api." + "infrai" + ".cc"
URL = f"https://{API_HOST}/v1/email/send"


def retry_delay(headers: object, attempt: int) -> float:
    retry_after = headers.get("Retry-After") if headers else None
    if retry_after and retry_after.isdigit():
        return float(retry_after)
    return min(2**attempt, 30)


def send_email(payload: dict, operation_id: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    body = json.dumps(payload).encode("utf-8")
    for attempt in range(5):
        request = urllib.request.Request(
            URL,
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": operation_id,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 4:
                time.sleep(retry_delay(error.headers, attempt))
                continue
            raise RuntimeError(f"email API returned {error.code}: {error_body}") from error
        except urllib.error.URLError as error:
            if attempt == 4:
                raise RuntimeError(f"email API was unreachable: {error.reason}") from error
            time.sleep(retry_delay(None, attempt))
    raise RuntimeError("email API retry budget exhausted")


def main() -> None:
    payload = json.loads(os.environ["INFRAI_EMAIL_PAYLOAD"])
    result = send_email(payload, f"password-reset-{uuid4()}")
    print(json.dumps(result, indent=2))


if __name__ == "__main__":
    main()
```

Provider acceptance should move the operation to `accepted`, not `delivered`. A permanent 4xx caused by an invalid sender or recipient should become `rejected`; a 429 or transient server response remains retryable. Put a ceiling on both attempts and elapsed time.

Expired means stop.

Before any of this reaches users, verify the sending domain and align SPF, DKIM, and DMARC. DMARC supplies the domain-level policy and reporting framework, but authentication alignment is not an inbox guarantee. Test the exact security-message template across major mailbox providers, keep the reset URL on a recognizable first-party domain, and do not place the code in a subject line or verbose logs.

## How do the provider choices differ?

Transport compatibility should eliminate unsuitable options before feature breadth or price enters the discussion.

| Provider | HTTP email API | SMTP relay | Fit for this reset flow |
| --- | --- | --- | --- |
| Amazon SES | Yes | Yes | Strong when AWS integration and a choice of API or SMTP matter; the application still owns reset-token logic. |
| SendGrid | Yes | Yes | Fits applications that need API delivery now while retaining an SMTP path for packaged tools. |
| Postmark | Yes | Yes | Focuses on transactional email and supports either API tokens or SMTP credentials. |
| Infrai | Yes | No | Fits API-native backends that value one key and one bill across services; it is unsuitable as a credential-only replacement for an SMTP transport. |

These are architectural differences, not a universal ranking. SES, SendGrid, and Postmark all provide SMTP paths documented by their vendors. An existing application with a mature SMTP integration may gain little from rewriting it. Conversely, an HTTP-native service can make authentication, idempotency, error classification, and structured response handling explicit instead of hiding them behind a generic mail transport.

Infrai's public discovery surface reports 295 capabilities and publishes request and response schemas plus examples. That helps an adapter validate the live contract. The **limitations are material**: it does not support SMTP relay or email webhook events, scheduled email has no cancellation operation, and the pending domestic China email vendor must not be treated as evidence of domestic compliance. It is not suitable when immediate event-driven delivery updates, credential-only SMTP compatibility, or that compliance basis is mandatory; choose SES, SendGrid, or Postmark according to the surrounding stack instead.

## Retain evidence, not message content

Polling-only events are a concrete trade-off. A worker can poll recent message state on a measured cadence and update the reset record, but the login flow must not wait for that job. Alert on aggregates such as growing pending age or rejection rate. Since this capability does not provide cost reports aggregated by tag, keep the application-side operation ID and purpose field needed for internal attribution. This adds a scheduled worker and local accounting to a flow that would otherwise end after the send call, so teams should count that ownership cost before consolidating providers for credential convenience.

Deliberately stop keeping the raw code, full reset URL, rendered body, and arbitrary provider payload once diagnostic needs are met. This reduces the material exposed by logs and databases. The cost is forensic detail: after deletion, an engineer cannot reconstruct the exact message a mailbox received or replay every provider field.

That loss is acceptable for most reset flows because the durable questions are narrower. Did the application issue one unexpired token? Did it hand off one logical message? Was that message accepted or rejected, and were retries bounded? Retain enough metadata to answer those questions under a documented schedule, then delete it.

## Further reading

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon SES: Send email through the Amazon SES SMTP interface](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
- [SendGrid: Integrating with the SMTP API](https://www.twilio.com/docs/sendgrid/for-developers/sending-email/integrating-with-the-smtp-api)
- [SendGrid: Mail Send API reference](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Postmark: Send email with SMTP](https://postmarkapp.com/developer/user-guide/send-email-with-smtp)
- [Postmark: Email API](https://postmarkapp.com/developer/api/email-api)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Twilio: SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
