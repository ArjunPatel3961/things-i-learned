# Shared-Device Authentication: Session Boundaries for Account Handoffs

On a family tablet, the dangerous moment is not the Google or GitHub login itself. It is the five seconds after one person taps “switch account” and before the next session is fully established. **Short answer: isolate session state first, then make account switching a server-side transaction with explicit revocation and reauthentication checks.**

I design these flows around a hard constraint: one browser profile can represent several people over a day, while cookies, local storage, cached API responses, and push tokens naturally want to outlive a person. Gaming makes the edge cases visible. A child may borrow a parent’s tablet, sign into a game with Google, then hand it back for a GitHub-linked developer account. The UI looks simple; the identity state is not.

## What should session isolation and account switching guarantee?

The first invariant is ownership. Every access token, refresh token, device token, and cached response must map to one internal user identifier, never merely to a provider subject or an email string. Google and GitHub identifiers are provider-scoped values; they are useful for linking, but they are not your application’s authorization boundary.

The second invariant is atomic replacement. A switch request should revoke the active refresh-token family, clear server-side device bindings that belong to the old session, and issue the new session only after the callback state and PKCE verifier have been validated. If any step fails, the old session stays revoked and the client returns to a signed-out screen. Half-switched sessions are where account data leaks.

The third invariant is observable intent. Record a session identifier, internal user ID, provider, device binding, and reason code such as `user_switch` or `reauthentication`. Do not log raw access tokens, authorization codes, or full profile payloads. OWASP’s Authentication Cheat Sheet also calls for reauthentication after risk events; a switch on an unfamiliar shared device is one of those events.

## How can a shared-device flow contain provider callbacks?

Keep the provider callback boring. The callback endpoint validates `state`, exchanges the code over a back channel, verifies the provider token according to that provider’s documented rules, and then hands a normalized identity to your account-linking service. The browser receives an opaque application session, not a provider token.

Here is the shape I use in Python. It is deliberately an interface, because the provider SDK is not the security boundary.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class ExternalIdentity:
    provider: str
    subject: str
    email_verified: bool

def complete_login(callback, expected_state, pkce_verifier):
    if callback.state != expected_state:
        raise ValueError("invalid oauth state")
    identity = exchange_and_verify(callback.code, pkce_verifier)
    user_id = find_or_require_link(identity)
    revoke_refresh_family_for_device(callback.device_id)
    return issue_application_session(user_id, callback.device_id)
```

The ordering matters. A client-side `localStorage.clear()` is useful housekeeping, but it cannot revoke a refresh token or invalidate a replayed callback. Cookies should be scoped deliberately, and service workers must not serve a previous user’s cached API response. On logout, send a cache-busting response, clear the application cache namespace, and make the API reject the old session immediately.

No shortcuts.

Tiny detail, large blast radius: an avatar URL or display name cached for the previous player can still reveal the wrong identity even when authorization is correct. I include the internal user ID in cache keys and test a switch with two accounts that have deliberately similar names.

## Which failure tests belong in the migration plan?

Start with a matrix, not a happy-path demo. Test Google-to-GitHub and GitHub-to-Google transitions, cancelled consent, an expired callback, a duplicated callback, offline resume, two tabs, and a device shared by an adult and a child account. Assert both positive and negative outcomes: the new user can read their own game profile, and cannot read the prior user’s inventory, linked identities, or recovery channels.

I also run a race test where logout and callback arrive within the same second. The server should use a monotonic session version or a revoked-at check so an older refresh token cannot win a timing contest. A 401 is expected for the stale request; silently retrying it with a different identity is not.

Metrics should distinguish `oauth_callback_rejected`, `session_revoked`, `link_confirmation_required`, and `reauthentication_required`. A spike in rejected state values can indicate a broken redirect deployment or an attack. Rate-limit the expensive exchange and linking operations, while keeping ordinary profile reads fast enough for a family room full of impatient players.

## What trade-offs matter when leaving a managed identity service?

Migration is a data problem before it is an SDK problem. I start with a row-level inventory: provider subject, internal user ID, refresh-token family, device binding, recovery method, last-seen timestamp, and audit-event retention. For each row, I mark whether the provider identity is verified, linked to more than one local account, or waiting for user confirmation. The export is encrypted, access-controlled, and disposable; it is not a second identity database. After importing an immutable-ID mapping, I dual-read for a short window while new sessions are minted by the destination service. During that window, a switch event carries a migration version so support can reconstruct which path issued it. A failed lookup sends the user through explicit account linking, never an automatic guess. Never infer account ownership from a mutable email address alone.

Managed identity products reduce the amount of protocol code your team owns, but they can constrain token shape, regional data placement, callback customization, or the audit fields you need. A self-hosted stack gives control over those boundaries and shifts patching, key rotation, abuse detection, and on-call work to your team. A lower invoice is not a security argument.

The catch is operational capacity. If your team cannot rehearse key rotation, restore the identity database, and investigate suspicious linking events, staying with the managed provider is the safer choice. Your mileage may vary; the deciding evidence is a tested runbook and measured recovery time, not a feature checklist.

For a gaming migration, I stage by cohort: internal testers, one family-facing release channel, then the long tail. Keep the old callback alive until every refresh-token family has either expired or been explicitly revoked. Compare session-switch telemetry across both paths, and make rollback mean “stop issuing new sessions,” not “trust old tokens again.”

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://datatracker.ietf.org/doc/html/rfc7636
- https://datatracker.ietf.org/doc/html/rfc6749
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies
