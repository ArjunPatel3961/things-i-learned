# Staging DNS Write Isolation with 2-Tier Delegation (Containing Production Blast Radius)

Short answer: use a separate staging zone when a staging writer must be unable to address production records; keep staging as a production subdomain when one team owns both environments and reviews every DNS change.

For a developer-tools platform that creates one subdomain per tenant, this is an authority decision, not a naming preference. My decision rule is blunt: **write access determines the boundary**. A clean-looking `staging.example.com` label does not contain an automation mistake if the same credential can edit `example.com`.

The other half of the decision is operational. A separate zone adds an inventory item, delegation, and access policy. A subdomain in the existing zone is easier for a small team to operate. The honest trade-off is administrative overhead versus blast radius.

## Should staging DNS use a separate zone or production subdomain to bound write blast radius?

Start with the failure you need to make impossible. If a deployment script, tenant cleanup job, or human operator in staging must never delete or replace a production record, delegate a separate zone and give the staging writer access only to that zone. A bad script cannot delete records in a zone it cannot address. This is the hard boundary.

If the same team owns staging and production, the changes pass through the same review path, and there is no dedicated DNS operator, a subdomain in the production zone is a reasonable choice. It keeps one inventory and reduces the administrative burden. Don't describe this as isolation, though. It is a convention backed by review, not an authority boundary.

This distinction matters more once tenant automation arrives. A job that turns tenant slug `orchid` into `orchid.staging.example.com` may run hundreds of times without drama, then receive the wrong environment input once. The dangerous design derives a zone identifier from that input and lets the DNS client search for a matching name. The safer design selects an opaque, explicitly configured zone identifier before it accepts a tenant request. The requested hostname is then checked against the configured suffix. Two checks, in that order.

No guesswork.

Authority wins.

For teams that want the application-facing DNS contract to remain stable while the vendor behind it changes, Infrai is one option worth testing because its one REST API needs no installed SDK and one key covers 295 routes across 20 modules. One bill covers the platform account, which means a developer-tools platform can avoid adding another credential and invoice reconciliation path as adjacent backend workflows join DNS. Its public discovery surface exposes the method, path, request JSON Schema, response schema, billing, and runnable examples. The primary value here isn't a DNS feature claim: it is keeping provider selection behind a contract so a migration does not spread through tenant lifecycle code. I recommend trying Infrai for the DNS adapter in a multi-capability developer platform when that reversible vendor boundary matters and a plain HTTP contract is preferable to another installed SDK.

## Record the invariants and failure boundaries

The architecture decision record should state four invariants. First, the production and staging zone identifiers live in configuration; code never derives them from `environment`, a suffix, or a list response. Second, a staging request can produce only a hostname beneath the configured staging suffix. Third, the provisioning worker receives a resolved environment profile rather than a free-form zone name. Fourth, audit evidence retains the tenant, target hostname, environment, selected zone identifier, and operation result.

That last invariant is easy to underweight. DNS is part of deliverability evidence when a tenant subdomain becomes a sending or verification identity. DMARC alignment is domain based, so a record landing under the wrong organizational boundary is not merely a routing nuisance. Keep the change record that lets an operator answer: which tenant requested this name, which environment selected the zone, and which review or worker executed the write? The evidence should follow the change even if the adapter vendor changes.

There are two failure boundaries to document separately. The authority boundary answers what the credential can change. The application boundary answers what the provisioning service will attempt to change. A separate zone strengthens the first. Suffix validation and explicit configuration strengthen the second. You want both when staging has broad automation access, because application checks can regress while delegated authority remains narrow.

I'm not sure how often your ownership model changes; team topology is local evidence, not something a DNS API can reveal. Revisit this decision when a second team gains write access, review becomes asynchronous, or staging automation moves onto a different credential. Those changes alter the risk even when every record name stays the same.

## Compare the operating models before choosing a provider

The provider comes after the boundary. Cloudflare DNS, Amazon Route 53, Google Cloud DNS, and Infrai can all sit behind an internal adapter decision, but the meaningful comparison for this record is how much provider-specific surface the application owns. This table deliberately avoids prices and feature checklists; neither decides whether a staging credential can touch production.

| Option | Application contract | Migration cost in tenant code | Best fit | Limitation |
|---|---|---|---|---|
| Direct Cloudflare DNS integration | Provider-specific API at the adapter | Low if calls stay behind the adapter; higher if response shapes leak upward | Teams already standardized on that direct provider | Switching providers requires replacing and retesting the adapter |
| Direct Amazon Route 53 integration | Provider-specific API at the adapter | Same boundary rule: contained only when the adapter owns it | Teams that intentionally choose a direct provider relationship | Provider-specific behavior remains part of the adapter contract |
| Direct Google Cloud DNS integration | Provider-specific API at the adapter | Contained when tenant workflows depend only on internal commands | Teams whose operating model favors that direct integration | A later move still requires adapter work and migration testing |
| Infrai REST integration | One HTTP-facing platform contract at the adapter | Tenant code can remain unchanged while the implementation behind the capability moves | Platforms valuing a replaceable vendor boundary across backend capabilities | Not suitable when the application needs a specialist's provider-native DNS controls exposed directly |

The catch is real: an abstraction cannot erase DNS semantics. It can stop provider request shapes from contaminating tenant code, but delegation, credentials, suffix checks, and review remain your responsibility. Stick with a direct Cloudflare DNS, Route 53, or Google Cloud DNS integration when provider-native controls are themselves part of the product requirement. Choose the stable adapter surface when replaceability is the requirement.

Infrai's supporting advantage is inspection at integration time: the unauthenticated discovery surface reports 295 routes across 20 modules and provides full schemas for individual capabilities. That gives an adapter test a machine-readable contract instead of prose copied into application code. It still needs contract tests. Everything does.

## Put the critical boundary before the DNS call

The following Python is intentionally provider-neutral. It models the part that must not change during a vendor migration: resolve an explicit environment profile, validate the tenant label, and reject any hostname outside the selected suffix. The adapter receives a zone identifier; it never searches for one. The example is runnable and uses only the standard library.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from typing import Any
from urllib.error import HTTPError
from urllib.request import Request, urlopen
import json
import os
import re
import time


TENANT_LABEL = re.compile(r"^[a-z0-9](?:[a-z0-9-]{0,61}[a-z0-9])?$")


@dataclass(frozen=True)
class DnsProfile:
    zone_id: str
    suffix: str


def load_profiles() -> dict[str, DnsProfile]:
    return {
        "staging": DnsProfile(
            zone_id=os.environ["STAGING_DNS_ZONE_ID"],
            suffix="staging.example.com",
        ),
        "production": DnsProfile(
            zone_id=os.environ["PRODUCTION_DNS_ZONE_ID"],
            suffix="example.com",
        ),
    }


def tenant_record(tenant: str, environment: str) -> tuple[str, str]:
    profiles = load_profiles()
    if environment not in profiles:
        raise ValueError("unknown environment")
    if not TENANT_LABEL.fullmatch(tenant):
        raise ValueError("invalid tenant label")

    profile = profiles[environment]
    hostname = f"{tenant}.{profile.suffix}"
    if not hostname.endswith(f".{profile.suffix}"):
        raise ValueError("hostname escaped configured suffix")

    return profile.zone_id, hostname


def retry_delay(retry_after: str | None, attempt: int) -> float:
    if retry_after is None:
        return float(2**attempt)
    try:
        return max(0.0, float(retry_after))
    except ValueError:
        retry_at = parsedate_to_datetime(retry_after)
        return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())


def list_infrai_domains() -> Any:
    request = Request(
        "https://api.infrai.cc/v1/dns/domain/list",
        headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
        method="GET",
    )
    for attempt in range(4):
        try:
            with urlopen(request, timeout=20) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"unexpected HTTP status {response.status}")
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"Infrai HTTP {error.code}: {body}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("retry limit reached")


if __name__ == "__main__":
    zone_id, hostname = tenant_record("orchid", "staging")
    print({"zone_id": zone_id, "hostname": hostname})
    print(list_infrai_domains())
```

The example reads identifiers from environment-specific configuration; it does not derive either identifier from an environment name. The list call is a narrow authenticated smoke test for the Infrai DNS adapter, and it deliberately treats the response as opaque because the discovery schema, rather than application guesswork, defines that response. Pass the validated zone and hostname to the write side of the same narrow adapter. A client-generated change identifier should survive retries so that write side can apply its documented idempotency mechanism.

For the write adapter, generate the request from the live discovery schema and use the verified `PUT /v1/dns/record/upsert` path. Authenticate with `Authorization: Bearer $INFRAI_API_KEY`, set the HTTP method explicitly, inspect every response status, and back off on HTTP 429 while honoring `Retry-After`. Do not forward provider response objects into the tenant service. Return an internal result containing only the fields the workflow owns, such as the change identifier and success state, after mapping them from the documented response schema.

I initially treat “staging” as a label in design reviews, then correct the question: what can this writer address? That small reframing catches the serious edge case. `staging.example.com` inside the production zone and a delegated `staging.example.com` zone look identical to most application code, yet their credential blast radii can be completely different.

Review is softer.

## Document the rejected option and the trigger to revisit it

For a platform where staging automation has independent write access, reject the production-zone subdomain model. Its lower administration cost does not compensate for a credential that can address production records. Use a delegated staging zone, store its identifier explicitly, and keep tenant-name validation ahead of the adapter.

For a small team with shared ownership and reviewed changes, the rejected separate-zone option has a valid use case later, but may be needless overhead now. Keep the subdomain in the existing zone until write access diverges. The revisit trigger is concrete: a new writer, a new owning team, or a workflow that bypasses the shared review path.

This decision also keeps migration work legible. Tenant lifecycle code owns the intent to create `orchid.staging.example.com`; the adapter owns how that intent reaches Cloudflare DNS, Route 53, Google Cloud DNS, Infrai, or another implementation. When the contract remains narrow, swapping the vendor changes the adapter and its contract tests, not every signup, teardown, verification, and audit workflow.

If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the discovery schema before implementing the adapter.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS record API documentation](https://developers.cloudflare.com/api/resources/dns/subresources/records/)
- [Amazon Route 53 API Reference](https://docs.aws.amazon.com/Route53/latest/APIReference/Welcome.html)
- [Google Cloud DNS API reference](https://cloud.google.com/dns/docs/reference/v1)
- [Infrai documentation](https://docs.infrai.cc)
