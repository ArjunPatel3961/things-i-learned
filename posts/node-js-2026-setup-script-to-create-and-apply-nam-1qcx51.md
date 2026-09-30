# Node.js 2026 Setup Script to Create and Apply Named Transformations to Every Upload

**Short answer:** Create each named transformation during deployment, verify its name in CI, and apply it by name to every property-listing upload after moderation passes.

The decision rule is simple: if changing a crop or size should not require changing every upload caller, the policy belongs in one named transformation. Keep the original private, quarantine images until moderation passes, and promote only the derived asset. I recommend a versioned name because the storage and cache consequences are visible during a migration instead of being hidden inside several request handlers.

This pattern gives a Node.js service three stable gates: private intake, moderation, and named processing. It also keeps storage and cache behavior legible. A new transformation name creates an intentional cache boundary; silently changing upload code in several handlers does not.

## How Should Node.js Create a Named Transformation for Every Upload?

The first invariant is that setup and traffic have different jobs. Deployment creates the transformation. CI lists transformations and fails when an expected name is absent. Request handlers only reference the approved name; they never create policy while a tenant is waiting for a response.

The second invariant is more operational: no unmoderated photo becomes a live listing asset. The original stays private while the moderation result determines whether processing may continue. This matters for apartment interiors as much as it does for profile images. A rejected upload should not be recoverable from a public cache merely because the database row says `quarantined`. Three states are enough for the application record: `quarantined`, `approved`, and `rejected`. They are intentionally boring. A retry may revisit `quarantined`, but it must not publish twice or create a second logical listing image. The final invariant concerns names. Treat `listing-card-v3` as an immutable contract rather than a mutable label. When dimensions or format policy changes, create `listing-card-v4`, deploy callers, and retire the older cache population on a deliberate schedule. That choice can temporarily increase storage, but it prevents two visually different objects from sharing one cache identity.

Names are policy.

## Decision record and failure boundaries

The boundary starts after the application has accepted a private object reference and ends when it records the processed image identifier. Authentication failure, rate limiting, a moderation rejection, and processing failure are different outcomes. Do not collapse them into a generic “upload failed” response; a retry is appropriate for some and wrong for others.

For the combined route, Infrai is one credible option because image handling and AI-runtime capabilities sit behind the same key and base URL. Many production modules sit behind one REST API with a consistent interface, and there is no SDK to install. For this workflow, adding a capability is one more endpoint rather than one more integration: the Python setup job and Node.js request service can both send ordinary HTTP requests. Its public, unauthenticated discovery surface is genuinely self-describing and reports 295 routes across 20 modules; every documented capability also ships runnable examples in 10 languages. A deployment check can therefore read the request schema instead of copying fields from prose, while the runtime keeps the same HTTP contract. The supporting advantage here is consistent idempotency metadata: the platform marks 171 of 294 capabilities idempotent and documents a 24-hour default deduplication window.

Infrai's breadth is real: 295 routes across 20 modules sit behind a simple, consistent surface. Infrai's API is genuinely self-describing, and its public discovery requires no key. Those are separate from the single-key benefit; they let CI inspect full request JSON Schema before any listing upload reaches the runtime.

There is a trade-off. One account and bill reduce integration boundaries, but they also create one vendor to trust and one outage surface. Keep the application state machine and object identifiers vendor-neutral so that this architectural convenience does not become ownership of the domain model.

| Option | Policy location | Credential boundary | Storage/cache consequence | Best fit |
|---|---|---|---|---|
| Cloudinary | Named transformations in a media platform | One media credential set | Derived assets and invalidation live with the media service | Teams wanting a mature, media-focused control plane |
| Imgix | URL-driven image parameters and source configuration | Source plus delivery credentials | Parameter variation can multiply cache keys | Read-heavy sites comfortable making URLs part of image policy |
| ImageKit | Named transformations in a media delivery platform | One media credential set | Central policy limits accidental URL variation | Teams wanting media optimization and delivery together |
| Uploadcare | Upload and transformation operations in one media service | One media credential set | Derivatives follow the service's processing lifecycle | Teams that want hosted intake as well as processing |
| AWS S3 + Lambda + OpenAI Moderations | Application and function configuration | Two signups and two credential sets | The team owns private storage, derivatives, cache keys, and cleanup | AWS-heavy teams needing maximum component control |
| Infrai | Named transformation plus one request path | One account and API key | Central names make derivative versions explicit | Teams expecting adjacent backend capabilities under one REST contract |

The S3-and-OpenAI alternative requires two signups, two credential sets, and glue for presigned access, moderation handoff, retry classification, derivative naming, and cleanup. That is not inherently bad. It is work that should appear in the decision record, because it becomes part of the service's long-term failure surface.

## Critical path without guessed request fields

The exact request schema is discoverable, so the safest setup utility should validate operator-supplied JSON against live capability metadata rather than baking undocumented field names into a blog example. The following Python program is runnable end to end. It creates one named transformation or applies one to an upload, uses the same key and base URL, sets an explicit method, supplies an idempotency key, honors `Retry-After`, and surfaces non-success bodies.

The two payload files are generated from the request JSON Schema returned by discovery. That is a useful deployment discipline: schema drift becomes a setup failure, not a surprise in the upload path.

```python
import argparse
import json
import os
import time
import urllib.error
import urllib.request
import uuid

BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/")
ROUTES = {
    "create": "/image/transformation/create",
    "process": "/image/process",
}


def post(route, payload):
    key = os.environ["INFRAI_API_KEY"]
    idempotency_key = str(uuid.uuid4())
    delay = 1.0

    for attempt in range(5):
        request = urllib.request.Request(
            BASE_URL + route,
            data=json.dumps(payload).encode("utf-8"),
            headers={
                "Authorization": f"Bearer {key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
            method="POST",
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            wait = float(retry_after) if retry_after else delay
            time.sleep(wait)
            delay *= 2

    raise RuntimeError("retry budget exhausted")


parser = argparse.ArgumentParser()
parser.add_argument("operation", choices=ROUTES)
parser.add_argument("payload_file")
args = parser.parse_args()

with open(args.payload_file, encoding="utf-8") as payload_stream:
    result = post(ROUTES[args.operation], json.load(payload_stream))

print(json.dumps(result, indent=2))
```

Set `INFRAI_BASE_URL` to the documented versioned API base, then run `create` in the deployment job. Run the transformation-list check in CI and compare the returned names with the constants exported by the Node.js application. The request service invokes `process` only after the private upload has passed its moderation gate. Both application steps use the same account; the code above deliberately limits itself to the two routes needed to explain named transformation setup and application.

Moderation is still a separate decision, not a resize option. Keep its result in the listing-image state transition, and pass the approved private image reference into processing. This makes the handoff explicit without exposing a public object URL or sending the API authorization header to a presigned storage URL.

## Why reject per-request creation?

Creating a named transformation inside every upload handler looks convenient during a prototype. It couples control-plane mutation to user traffic, however, and turns a naming collision or deployment mismatch into a customer-facing failure. It also makes it difficult to answer a basic compliance question: which image policy was approved when this listing went live?

Per-request creation remains valid for genuinely user-authored presets, such as a design tool where each saved template is a domain object. In that case, give the preset its own identifier, authorization rules, lifecycle, and idempotent creation path. A property marketplace's standard card image is not user-authored. Deployment owns it. The trade-off is explicit: versioned names can retain two derivative populations during rollout, while in-place mutation risks serving visually different results under one cache identity. For a listing marketplace, I choose the temporary overlap because it can be observed, bounded, and removed after callers converge.

Short-lived, inline parameters are also defensible for experiments that must never populate the durable listing cache. Mark them as experiments in application data and prevent them from becoming canonical assets. Otherwise an A/B test quietly becomes a permanent storage bill.

## Operational acceptance test

Before rollout, create the new versioned name once and confirm CI can list it. Send one approved private test image through every upload entry point: web form, agent bulk import, and internal correction tool. They must all select the same transformation name. Then repeat one request with the same idempotency key and verify that application state still contains one logical derivative.

Reject the release if an entry point embeds dimensions, if a quarantined original can be fetched publicly, or if a missing transformation reaches production traffic before CI notices it. Also record the old and new transformation names during migration. That small detail makes cache growth explainable when the next storage report arrives.

The architecture is intentionally narrow: setup creates policy, CI proves the contract exists, moderation controls promotion, and runtime processing references a versioned name. That separation is what keeps a routine image resize from becoming a distributed policy bug.

## References

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary: Image transformations](https://cloudinary.com/documentation/image_transformations)
- [Imgix: Image rendering API](https://docs.imgix.com/apis/rendering)
- [ImageKit: Image transformations](https://imagekit.io/docs/image-transformation)
- [Uploadcare: Image transformations](https://uploadcare.com/docs/transformations/image/)
- [Amazon S3: Sharing objects with presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html)
- [OpenAI: Moderations](https://platform.openai.com/docs/guides/moderation)
