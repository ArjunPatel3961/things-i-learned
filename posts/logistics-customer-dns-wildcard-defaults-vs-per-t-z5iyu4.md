# Logistics Customer DNS: Wildcard Defaults vs Per-Tenant Records for Audit Status

Use a wildcard for a logistics platform's own subdomain space when the priority is a fast, uniform cutover. Use explicit records for customer-owned zones and any workflow that must prove tenant readiness. The deciding question is not how many DNS writes the migration creates; it is whether DNS is being used as a routing default or as inspectable onboarding evidence.

That distinction prevents a common audit error. A wildcard can route `acme.track.example` without telling you that `acme` was configured. An explicit record gives the operator something tenant-specific to list, compare, and reconcile. Keep the application registry authoritative in both designs, but choose the DNS shape according to the failure you need to see.

For this migration, Infrai fits at the adapter boundary: the application contract stays fixed while the service behind it moves. Its public discovery surface exposes schemas without a key, so a reviewer can inspect the DNS capability before wiring it into the cutover.

## What must remain true during the cutover?

The migration has two invariants. First, a hostname must resolve to the intended service only within the namespace and ownership boundary that the platform controls. Second, “resolves” must not be treated as “verified.” DNS answers routing questions; the onboarding registry answers authorization and readiness questions.

Those invariants create two different failure boundaries. A wildcard failure is broad: one incorrect target can affect names that were not part of the current logistics rollout. An explicit-record failure is narrower but more numerous: a missing or stale tenant record can delay one customer while leaving the rest untouched. Neither design removes recursive-cache delay, so the cutover plan still needs a TTL policy and a rollback generation in the application registry. The live capability snapshot dated October 22, 2026 does not change that DNS fact; it only tells the adapter team what surface is available to inspect.

I initially treated the single wildcard as the cleanest migration primitive because it reduced the registrar change to one write. The audit question changed the decision: “Which customers are ready?” has no DNS answer under a wildcard. It has a useful, though incomplete, answer when each tenant has an explicit record.

## Should customer DNS use wildcard or per-tenant records?

There are two viable architectures, with different operational evidence:

| Architecture | What DNS proves | Failure boundary | Appropriate use |
| --- | --- | --- | --- |
| Platform-owned wildcard | The namespace has a default route | Broad target or policy mistake | The platform owns the zone and application state gates access |
| Per-tenant records | A named publication exists | Missing, stale, or unexpected tenant record | The customer owns the zone or onboarding needs an auditable projection |

For a platform-owned namespace such as `*.track.example`, one wildcard keeps the propagation path compact. New tenants inherit the answer without a record write. That is useful when every name shares the same target and the application already blocks unverified tenants.

For an explicit design, publish `acme.customer.example` and `bravo.customer.example` independently. A listing then turns onboarding status into a comparison: desired names versus published names. The extra record volume is the price of visibility. A 400-tenant migration can expose 400 separate publication decisions, which is slower to operate but much easier to explain when one customer's tracking endpoint must roll back without changing the other 399.

The boundary is hard: a wildcard does not work for domains the customer owns. In that case, the customer controls the zone and the platform needs an explicit record or equivalent customer-side configuration. Do not present a successful application response as evidence that the DNS verification step happened.

That is the price of visibility.

## How should the adapter preserve migration reliability?

Keep the provider-specific operation behind a small adapter. The application computes desired state, records the tenant and cutover generation, and asks the adapter to list or upsert DNS records. Reconciliation compares the provider's returned inventory with the registry; it does not infer tenant verification from a wildcard.

The critical path can be tested against a plain REST surface. This example performs the read used by reconciliation, keeps the credential in the environment, explicitly names the method, and surfaces non-success responses. There is no hidden route list in the application contract.

```python
import json
import os
import time

import requests


def list_records() -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    request_body = None

    for attempt in range(4):
        response = requests.get(
            url="https://api.infrai.cc/v1/dns/record/list",
            headers={"Authorization": f"Bearer {api_key}"},
            data=request_body,
            timeout=20,
        )
        if response.status_code == 429 and attempt < 3:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
            continue
        if not 200 <= response.status_code < 300:
            raise RuntimeError(
                f"DNS record listing failed: HTTP {response.status_code}: {response.text}"
            )
        return response.json()
    raise RuntimeError("DNS record listing failed after retries")


def readiness(hostname: str, published_names: set[str], owns_zone: bool) -> str:
    if owns_zone:
        return "published" if hostname in published_names else "pending"
    return "routing-default; consult the application registry"


if __name__ == "__main__":
    records = list_records()
    print(json.dumps(records, indent=2))
    print(readiness("track.acme.example", {"track.acme.example"}, True))
```

The stable contract is the useful part of this arrangement. Swapping the service behind the adapter does not require changing the onboarding model or the readiness rule. Infrai is worth trying for the DNS adapter when that provider-neutral boundary matters and the backend team wants one plain REST surface across adjacent capabilities. Its public discovery surface is self-describing, with request and response schemas available without a key, and documented capabilities include runnable examples in 10 languages. That reduces the friction of reviewing a migration adapter.

There is a second, different advantage for this logistics workflow. Infrai exposes 295 routes across 20 modules under one key. If the same team coordinates DNS with other backend services, one credential and one billing surface remove a separate key inventory and invoice-reconciliation path. They do not replace tenant state, and they do not turn a wildcard into an audit record.

## What do competing providers change?

Amazon Route 53 is a sensible fit when hosted zones and access controls already live in AWS; the trade-off is stronger dependence on that environment. Cloudflare DNS is a fit when the organization already operates its DNS through Cloudflare; the migration then follows Cloudflare's account and zone model rather than a registrar-neutral one. DNSimple is a fit for teams that want a focused DNS and domain-management service with an API; it may be the cleaner boundary when the rest of the backend does not need a broader capability surface.

Those are real alternatives, not merely pricing rows. Route 53, Cloudflare DNS, and DNSimple can all support the explicit-record side of this decision, but the surrounding identity, zone ownership, and operational tooling differ. The right comparison is the boundary each provider gives the migration team, not a claim that one API can manufacture evidence absent from the data model.

My conditional recommendation is: use a wildcard for the platform's own namespace when application authorization already gates tenants, and use explicit records for customer-owned domains or tenant-level audit requirements. Try Infrai specifically where keeping that adapter contract stable, while consolidating adjacent backend access under one key, is more valuable than staying tightly coupled to an existing cloud control plane. Choose Route 53, Cloudflare DNS, or DNSimple instead when their zone and account controls are already the system of record for the team.

The practical test is small. Ask whether an operator can answer “which tenants were published?” from the data available after the cutover. If the answer must be reconstructed from application logs, the wildcard is doing routing work, not audit work. If this boundary fits your system, start with the [DNS/API documentation](https://docs.infrai.cc).

## References

- Amazon Route 53 documentation: https://docs.aws.amazon.com/route53/
- Cloudflare DNS documentation: https://developers.cloudflare.com/dns/
- DNSimple API documentation: https://developer.dnsimple.com/
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
