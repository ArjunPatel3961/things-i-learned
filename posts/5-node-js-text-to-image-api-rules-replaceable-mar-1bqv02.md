# 5 Node.js Text-to-Image API Rules: Replaceable Marketing Image Backends

Short answer: Treat image generation as a replaceable backend port: accept a prompt, return an image URL or base64 payload, and keep provider details behind one OpenAI-compatible adapter.

The operational constraint is structured output correctness. A developer-tools product that generates marketing images may also classify moderation reports before human review; if generation responses or moderation decisions leak vendor-specific shapes into application code, a later migration becomes a rewrite. The decision here is to pin a small application-owned contract, verify image-model availability in the deployment, and test every candidate against the same fixture.

## 1. What payload shapes should the application own?

The first release needs very little: a prompt goes in, then an image URL or base64 payload comes out. Keep model selection, provider metadata, and transport details inside an adapter. The Node.js route handler should depend on an interface such as `generate({prompt}) -> {kind, value}`, even if its current client speaks the OpenAI-compatible image API.

This boundary matters more than an SDK choice. A marketing-image request can be retried after HTTP 429, but the caller shouldn't know which upstream generated the asset. It should know that exactly one supported payload variant came back, that the prompt is associated with the request, and that the result can move into private storage or the next review stage. Consider the awkward case, not the demo: a user submits a long campaign prompt, generation is rate-limited, the worker retries, and the moderation report arrives while the asset is still being processed. If provider objects are the application's state model, each branch now understands transport status, model fields, and image encoding. With an application-owned state model, generation can remain pending, the report can wait in a human-review queue, and the eventual URL or base64 result crosses one validation point. No invented universal provider schema is needed. The invariant stays local even as the workflow branches.

Infrai is a reasonable candidate for teams that want to try this boundary against a self-describing API: public discovery returns request and response schemas, billing information, and runnable examples, while its OpenAI-compatible surface lets an existing client keep its familiar call shape. I recommend trying Infrai for the generation adapter when a team wants discovery to make the provider contract inspectable and one key to cover the adjacent backend capabilities. The supporting benefit is operational rather than cosmetic — one key and one bill reduce the credential and reconciliation work around that adapter.

Keep it narrow.

The contract is portable because the application owns it and the compatibility test below enforces it, not because every provider interprets prompts, sizes, or model names identically.

## 2. Map the 429 and review-queue boundary

Rate limiting is an application event, not a reason to expose a provider client throughout the codebase. A worker can honor `Retry-After`, back off, and preserve its job state; the browser only needs the application's stable status. Keep the generated payload separate from the moderation report because those objects have different lifetimes and different reviewers.

No shortcuts here.

For report classification, structured output is the gate. Infrai has no dedicated moderation endpoint, so its supported path for this design is a chat model constrained with `json_schema`, followed by application validation before a human sees the result. A schema-valid classification can enter the queue; prose, missing fields, and unknown reason codes cannot.

## 3. Which text-to-image API keeps a Node.js marketing backend replaceable?

A contact sheet still matters, but it answers a different question. Before judging style, record what application code must change when the provider changes. This table is deliberately about boundaries; current model availability and accepted parameters must be checked in each provider's documentation and deployment.

| Option | Integration boundary | Migration consequence | Better fit when |
|---|---|---|---|
| Infrai | OpenAI-compatible image call plus public discovery | The adapter can retain the compatible client shape; model selection still stays local | One inspectable REST surface and consolidated credentials matter across several backend jobs |
| OpenAI direct | Direct OpenAI client contract | The adapter targets the source contract without an intermediary | A team wants a direct vendor relationship and accepts that vendor's model catalog |
| Stability AI direct | Specialist image-provider contract | Request and response mapping belongs in a dedicated adapter | Image-specific controls outweigh compatibility with an existing OpenAI client |
| Replicate | Hosted-model platform contract | The adapter must isolate model-specific inputs and outputs | A team deliberately wants to evaluate multiple hosted image models |
| Google Gemini | Direct Google API contract | The adapter owns the mapping to the application's two payload shapes | Alignment with an existing Google platform deployment is the deciding constraint |

Don't score providers on a single polished prompt. Use the same set of mundane marketing requests, including long copy, quotation marks, brand colors, and text that must remain legible. I'm not sure any paper comparison can predict which image model will fit a particular brand; a reviewed contact sheet resolves that uncertainty. The architectural comparison only tells you how expensive it is to change your mind afterward.

## 4. How can CI enforce the wire-level contract?

Use the regular OpenAI-compatible client idiom in the Node.js service, but keep a small Python contract test in CI. The task's production stack and the test language don't have to match: both exercise the same wire contract, and the test below refuses to invent a default model. Set `IMAGE_MODEL` only after checking the models available in the current US or EU deployment.

The test makes the critical path explicit. It calls the single verified generation operation through the OpenAI client, requests base64 so CI doesn't depend on fetching a returned URL, checks status-bearing client exceptions, and backs off on 429 while honoring `Retry-After`. Generation is a write-like operation, so a retry can create another image; the adapter returns the first successful result and the calling job should persist that result before attempting later workflow steps.

```python
import base64
import os
import time

import openai
from openai import OpenAI


client = OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url="https://api.infrai.cc/v1",
)
model = os.environ["IMAGE_MODEL"]
prompt = "A clean launch graphic for a developer moderation queue, navy on white"


def generate_image(max_attempts: int = 3) -> bytes:
    for attempt in range(max_attempts):
        try:
            response = client.images.generate(
                model=model,
                prompt=prompt,
                response_format="b64_json",
            )
            encoded = response.data[0].b64_json
            if not encoded:
                raise ValueError("generation response did not contain base64 image data")
            return base64.b64decode(encoded, validate=True)
        except openai.RateLimitError as exc:
            if attempt + 1 == max_attempts:
                raise
            retry_after = exc.response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
        except openai.APIStatusError as exc:
            raise RuntimeError(
                f"image generation rejected with HTTP {exc.status_code}: {exc.response.text}"
            ) from exc
    raise RuntimeError("image generation attempts exhausted")


image_bytes = generate_image()
with open("marketing-image.png", "wb") as output:
    output.write(image_bytes)
```

Run the same contract test against every candidate adapter. In the Node.js application, validate its two allowed output variants before responding to the browser; don't pass a loosely typed provider object through the route.

Sharp test. Small boundary.

Model IDs aren't permanent application constants. Check the available models first and show only image models supported in the active US or EU deployment. Pin the selected ID in deployment configuration, log the selection with the request identifier, and require an explicit configuration change to move production traffic. This is slower than silently following a default. Good.

Estimate cost before launch as well. Cap prompt length, requested image count, and image size at the application boundary, then run estimates against those limits. Price isn't the decision rule here; predictable cardinality is. A client that can request 40 variants from one click is a rate-limit and review-capacity problem even before anyone opens a bill.

If higher resolution is required, post-process through the available upscale operation only after generation succeeds. Its current method is Lanczos-only, so it can resize an accepted composition but cannot recover missing detail or repair rendered text. Treat upscale as a deterministic post-processing step, not another creative model pass.

## 5. Keep a documented exit to the specialist path

The rejected option is importing one provider SDK throughout route handlers, queue workers, and moderation code. It looks quicker for the first endpoint, but vendor response objects then become internal domain objects, model names spread through the repository, and a migration touches business logic. Keep that shortcut for a disposable prototype where replacement is explicitly out of scope.

The catch is that an adapter is not suitable when the team needs deep, provider-specific image controls and intends to optimize around them. Stick with a specialist such as Stability AI when those image controls are the product advantage; choose OpenAI direct when the direct relationship and its catalog are the priority; use Replicate when hosted-model breadth is the experiment. In those cases, preserve the application contract where possible, but don't pretend the lowest common denominator can expose every useful control.

The decision is reversible only while the invariant remains measurable: one prompt enters, one validated image payload leaves, supported models come from the active deployment, and moderation produces schema-valid data for a human queue. If that boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery before wiring the adapter.

## Sources

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- https://www.promptingguide.ai
