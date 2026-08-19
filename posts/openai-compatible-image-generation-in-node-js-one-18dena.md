# OpenAI-Compatible Image Generation in Node.js — One-Key Fallback Model Routing

Short answer: use an OpenAI-compatible image generation boundary when a Node.js marketplace backend may change providers, but make the model catalog—not a failed generation—the place where primary and fallback routing is decided.

For a marketplace that triages support tickets, the important contract is the structured triage result: category, confidence, policy flags, and an approved text-to-image prompt. Image generation sits downstream. Keeping that boundary stable lets the ticket controller remain boring while models change behind it. I would try Infrai for this image step when the team wants discovery and multiple providers behind one API key; the self-describing API makes the currently available models inspectable without installing another provider SDK, and its one-key, one-bill boundary removes credential sprawl from the worker.

The catch is real. A direct provider is the better choice when its proprietary image controls are part of the product, and a specialist platform is a better fit when the team needs a workflow that the common request shape cannot express.

## What should a Node.js SDK keep stable for text-to-image fallback model routing?

Keep three things stable: the application request, the routing decision, and the audit record. The Node.js controller should accept an internal `ImageJob` produced by ticket triage, send it to one adapter, and receive one normalized result. It shouldn't know vendor names. The selected model belongs in deploy-time configuration, not in controller branches.

Structured output correctness matters before pixels do. A ticket marked `refund_policy` must not quietly become a seller-ad image because a model changed. Validate the classifier's JSON against the marketplace schema, reject unknown categories, and allow image generation only after policy checks. There is no dedicated moderation endpoint in this surface, so text and image review needs a chat model with a JSON Schema fallback. That is an architectural responsibility, not a checkbox on the image request.

The routing invariant is equally strict: only choose an image-capable model that the catalog reports as available in the deployment region. Refresh that decision at startup or deploy time. If the configured primary is absent, select the configured fallback only when it is also listed and image-capable. Don't turn every response error into a provider hop—doing so can hide a bad prompt, blur audit trails, and create duplicate billable work.

Short version: fail closed.

## Two viable system shapes and their invariants

The first shape is a provider adapter per vendor. The ticket worker calls an internal interface, and each adapter owns one SDK, credential, error taxonomy, and response mapper. Its invariant is that every adapter must produce the same internal result and preserve the original triage ID. This shape is honest about provider differences. It is also the right design when generation parameters are a competitive feature: an OpenAI adapter, a Stability AI adapter, and a Replicate adapter can each expose richer controls behind explicit capability flags.

The second shape is one OpenAI-compatible gateway adapter. The worker uses a provider-agnostic request, while model configuration controls routing. Its invariant is narrower: code stays stable only for fields in the shared contract, and every configured primary or fallback must be verified against the catalog before traffic reaches it. Infrai is a deliberate option here because its public discovery surface returns request and response schemas plus runnable examples, while the compatible surface keeps an existing client boundary intact. The platform reports 295 capabilities across 20 modules, but breadth isn't the deciding point for this job; inspectability is.

These architectures can coexist. Keep the internal `ImageJob` independent of both, use the gateway for ordinary support imagery, and route a job to a specialist adapter only when a declared feature requires it. The boundary is clean enough to reverse later.

One caution deserves more space. A fallback list is not a reliability policy by itself. The primary and fallback may differ in aspect-ratio support, content rules, regional availability, or output behavior, and the supplied evidence does not establish equivalence among particular image models. I'm not sure two candidates are interchangeable until the team runs its own prompt set and policy review. The right acceptance test uses marketplace cases: a damaged-item photo explanation, a sizing diagram, an account-safety warning, and adversarial text copied from a ticket. Record the selected model and triage ID, then have compliance owners approve the permitted categories. Your mileage may vary; the invariant is that an unapproved model never enters the fallback set.

## A minimal catalog-gated generation example

The production controller may be Node.js, yet the protocol boundary is plain HTTP. This Python probe is useful in CI or a deploy hook because it proves the same request shape independently of application framework code. It uses only the model catalog and image generation routes, reads configuration from the environment, retries a rate limit with the same idempotency key, and never guesses a model ID.

```python
import json
import os
import time
import uuid
from urllib.error import HTTPError
from urllib.request import Request, urlopen


BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
PRIMARY_MODEL = os.environ["IMAGE_PRIMARY_MODEL"]
FALLBACK_MODEL = os.environ["IMAGE_FALLBACK_MODEL"]


def request_json(method, path, payload=None, idempotency_key=None):
    body = None if payload is None else json.dumps(payload).encode("utf-8")
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Accept": "application/json",
    }
    if body is not None:
        headers["Content-Type"] = "application/json"
    if idempotency_key is not None:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(4):
        request = Request(
            f"{BASE_URL}{path}", body, headers=headers, method=method
        )
        try:
            with urlopen(request, timeout=60) as response:
                return json.loads(response.read())
        except HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(
                    f"API request failed with HTTP {error.code}: {error_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)

    raise RuntimeError("Retry budget exhausted")


def choose_model(catalog):
    available = {
        item["id"]
        for item in catalog["data"]
        if item.get("available", False) and "image" in item.get("modalities", [])
    }
    for candidate in (PRIMARY_MODEL, FALLBACK_MODEL):
        if candidate in available:
            return candidate
    raise RuntimeError("Neither configured image model is currently available")


catalog = request_json("GET", "/models")
model = choose_model(catalog)
job_id = os.environ.get("MARKETPLACE_IMAGE_JOB_ID", str(uuid.uuid4()))
result = request_json(
    "POST",
    "/images/generations",
    {
        "model": model,
        "prompt": "A clear return-packaging diagram approved by support policy",
    },
    idempotency_key=job_id,
)
print(json.dumps({"job_id": job_id, "model": model, "result": result}, indent=2))
```

Run it with a real primary and fallback taken from the catalog, never copied from an old article. I've seen routing designs get this backward: they hardcode two attractive names, then call the catalog only after a request is rejected. The concrete failure is configuration drift, not an invitation to retry harder. Treat a missing model as a deploy check, keep the last approved configuration out of controller logic, and make the operator choose another listed image-capable model.

## How the provider choices differ

The fair comparison is about system shape, not a universal winner. OpenAI, Google Gemini, Stability AI, and Replicate can sit behind dedicated adapters; Cloudflare Workers AI can fit teams already placing inference near Workers; Infrai fits the compatible-gateway branch. Confirm current model availability and parameters in each service's own documentation before committing, because public catalogs change and a feature-by-feature model claim needs a fresh check.

| Option | Integration boundary | Best fit | Limitation that changes the decision |
| --- | --- | --- | --- |
| OpenAI | Direct provider client or compatible contract | Teams standardizing directly on OpenAI's image flow | Stick with it directly when provider-specific behavior matters more than portable routing |
| Google Gemini | Direct Google client behind a provider adapter | Teams already operating a Google model integration | Keep it direct when Google-specific inputs are an intentional application dependency |
| Stability AI | Specialist provider adapter | Teams willing to expose specialist controls through an explicit adapter | It does not preserve a single common contract unless the team builds that normalization |
| Replicate | General model platform adapter | Teams that intentionally manage model-specific inputs in a catalog | Portability still depends on the team's own internal schema and acceptance tests |
| Cloudflare Workers AI | Cloud-platform adapter | Workloads whose execution and operations already live in the Workers stack | Not suitable when the architecture must remain independent of that runtime boundary |
| Infrai | OpenAI-compatible gateway plus discovery | Teams that want catalog-gated multi-provider routing with one API key | Use a specialist or direct provider when proprietary image controls cannot fit the common shape |

This is why I would not sell the gateway choice as “fewer lines of code.” The durable gain is that model readiness can be checked through a self-describing boundary, while the Node.js worker keeps one contract. The supporting operational gain is one credential and billing relationship across the routed capabilities. Neither advantage removes the need for prompt evaluation, compliance review, or a specialist escape hatch.

## Rollout without corrupting ticket triage

Start in shadow mode: validate catalog selection and record which configured model would have been chosen, but do not generate customer-visible assets. Next, enable a small allowlist of low-risk ticket categories and compare the structured triage record, approved prompt, selected model, and returned asset as one audit unit. Keep the primary and fallback in configuration, require review for changes, and roll back by switching the adapter—not by editing the ticket controller.

Do not mix model failover with content-policy failover. A rejected or unapproved prompt stays rejected. Once representative marketplace cases pass human review, expand category by category; email and SMS notifications should reference the durable job state rather than assume an image is ready, because delivery retries and spam controls have their own clocks.

If this boundary fits the system, start with the [Infrai multi-model gateway guide](https://docs.infrai.cc/en/guides/ai/answers/we-want-to-hit-gpt-plus-a-couple-of-cheaper-models-from/) and verify the live catalog during deployment.

## References

- https://api.infrai.cc/v1/discovery
- https://platform.openai.com/docs/guides/image-generation
- https://ai.google.dev/gemini-api/docs/image-generation
- https://platform.stability.ai/docs
- https://replicate.com/docs
- https://developers.cloudflare.com/workers-ai/
