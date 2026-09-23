# Node.js Webhook Signature Verification: Raw Body Before Express JSON Parsing

A production API-key rotation changes the webhook verifier's trust boundary. The service must authenticate the callback using the exact bytes received, record which key made the decision, and keep accepting the old key only during a measured overlap. Parsing JSON first is the Express mistake that breaks this chain.

Short answer: capture the request as a Buffer before any JSON parser runs, verify the signature over that Buffer, then parse it. Store a key identifier and verification result in an audit event; never log the secret or the complete payload.

## How should webhook signature verification use the raw body before JSON parsing?

Signatures cover bytes, not an abstract JavaScript object. JSON permits insignificant whitespace, escaped characters, and member ordering choices. A parser followed by `JSON.stringify` can produce equivalent data with different bytes. Even middleware that changes character encoding can alter the signed message.

Bytes matter.

I first assumed a parsed object was enough because the fields looked identical in logs. The failure appeared only for payloads containing escaped Unicode and a trailing newline.

Express middleware is ordered. If `express.json()` consumes the stream first, a later handler usually sees an already-read request and has no canonical byte sequence to verify. The fix is a narrow ingress route with a raw parser, kept separate from ordinary JSON routes so the byte-preservation rule is visible in review. This costs a little memory for the Buffer and gives up the convenience of one global parser; that is a deliberate trade-off at a security boundary.

```python
# Conceptual flow; keep the actual implementation in Node.js.
raw_body = request.read_bytes()
key_id = request.headers.get("x-hook-key-id")
signature = request.headers.get("x-hook-signature")
expected = hmac_sha256(keys[key_id], raw_body)
if not constant_time_equal(signature, expected):
    return response(401)
event = json.loads(raw_body)
return response(204)
```

The generic contract is byte capture, constant-time comparison, and parsing only after authentication. In Node.js, use a raw-body parser or a verified `verify` callback, and ensure no upstream middleware has consumed the stream.

That is the boundary.

## What should the rotation protocol record?

Treat rotation as an auditable state transition, not a file replacement. Generate a new key, assign it an identifier, distribute it to the verifier, and switch the sender to that identifier. During overlap, accept both identifiers but emit an event for every verification with timestamp, key ID, route, outcome, and correlation ID. Redact payload content.

Reject unknown IDs, malformed signatures, and timestamps outside the replay window when the signing scheme includes one. Do not silently fall back to whichever key works; that makes reconstruction ambiguous. Retire the old key after delivery retries and queue lag are covered by observed behavior.

OWASP guidance favors scoped access, rotation, and audit trails rather than credentials in source or logs. A key ID can be searchable metadata; the secret must remain in a protected store readable only by the verifier process.

## Where do common implementations go wrong?

A global JSON parser is the usual trap. Mounting a raw parser after it has run cannot recover bytes. A second trap is verifying a reconstructed string, which creates encoding and whitespace differences. A third is comparing hexadecimal text with a base64 signature without an explicit decoding step.

Replay handling is another edge. A valid MAC does not prove freshness. If the protocol supplies a timestamp and nonce, bind both to the signed bytes and retain a short-lived nonce record. Return the same external error class for unknown key IDs and bad signatures; detailed distinctions belong in restricted logs.

Express leaves middleware ordering to the application; Fastify exposes content-type parsers that must be configured for the webhook route; Koa relies on whichever body-parser middleware the stack includes. None removes the protocol obligation. The portable test is a fixture containing exact bytes, headers, and expected verdicts, run before and after rotation. Framework convenience is the limitation: a managed parser can reduce boilerplate, while a raw route gives tighter audit evidence and more application code to maintain.

Start with a dry run that computes verification with the candidate key but does not change acceptance. Compare key IDs and failure reasons in metrics. Then deploy the verifier with both keys, switch the sender, and watch acceptance by key ID. Only after the old key shows no legitimate traffic should you revoke it. Keep rollback as a key-state change, not a code redeploy.

Start with a dry run that computes verification with the candidate key but does not change acceptance. Compare key IDs and failure reasons in metrics. Then deploy the verifier with both keys, switch the sender, and watch acceptance by key ID. Only after the old key shows no legitimate traffic should you revoke it. Keep rollback as a key-state change, not a code redeploy.

Test UTF-8, duplicate-looking fields, newline endings, oversized bodies, missing headers, and delayed retries. Assert that parsing never occurs on a failed signature, and that the audit event contains no secret material. A small integration test using the real middleware stack catches ordering regressions that unit tests miss.

Version the fixture format (for example, `fixture-v2`) so a future signing change does not silently invalidate historical audit checks.

The durable design is an ingress boundary with raw bytes, explicit key IDs, constant-time verification, and observable rotation states. JSON parsing remains useful, but it is a post-authentication operation.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://expressjs.com/en/resources/middleware/body-parser.html
- https://nodejs.org/api/crypto.html
- https://www.rfc-editor.org/rfc/rfc2104
