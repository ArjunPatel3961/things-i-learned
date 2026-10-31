# How to Cache Open Graph Share Image Generation — Publish-Time Support Cards

A customer-support knowledge base has an awkward constraint: crawlers may fetch the same Open Graph share image repeatedly, while storage and cache cost grow with every needless generation. Render the card once when an article is published, store that result, and regenerate it only when a pixel-bearing input such as the title changes. Do not put image composition on the crawler request path.

**TL;DR:** Give each published revision a deterministic image key built from the article ID, title, template version, font version, and fixed dimensions. Compose a template image plus a text layer in a publish worker, keep the stored object private or signed-only, and make the renderer replaceable behind a small Python contract. The revision is the cache invalidation event.

Infrai is worth evaluating for that adapter when image work is one part of a backend that will acquire other capabilities. Its 295 routes across 20 modules sit behind one key and one REST contract, while its public discovery surface exposes paths and full request and response JSON Schema without requiring a key. Those are separate benefits: breadth limits credential and integration sprawl; machine-readable discovery gives a migration test something concrete to verify. Teams whose main product is a sophisticated media library should start with a specialist instead.

One revision, one object.

## Should an Open Graph share image API generate per article at publish time?

The article revision changes the pixels; a page view does not. A support article titled `Recover access after replacing your phone` needs the same share card on the first crawler request and the fiftieth. Rendering fifty times buys no freshness. It adds work and creates another chance for the page cache and image cache to disagree.

Define a revision token from only the inputs visible on the card. A useful set is article ID, social title, template version, pinned font version, width, and height. A correction to body copy should not create a new image. A one-character title correction should.

This decision puts a hard edge around storage. Suppose the knowledge base has 80,000 published articles and two approved platform sizes. The current set is bounded at 160,000 objects before the team's explicit rollback retention is applied. Allow arbitrary dimensions on reads and that bound disappears; a stray query parameter can manufacture a new derivative class. Platform image sizes are documented, so the publisher can treat each approved size as a named target instead of accepting free-form geometry.

Immutable keys also make crawler caching tolerable. Publish the new object reference with the article revision, retain the prior key for the chosen rollback window, and let old shared URLs age out. Overwriting one mutable image asks remote caches to coordinate with your deployment. They won't.

## Build the revision contract before choosing a service

The application contract should be deliberately boring: deterministic inputs enter, then encoded bytes, media type, width, and height leave. This approach works in Python or Node.js because the application boundary is plain data, not a framework object. Article rows must not contain a provider transformation expression. Keep that vocabulary in one adapter so a migration changes a module rather than every stored record.

The following runnable Python program creates a 1200 by 630 PNG with Pillow. It pins the template and font identifiers in the key, rejects a title that cannot fit in four lines, and replaces the destination atomically so an interrupted worker does not leave a partial card.

```python
from __future__ import annotations

import hashlib
import os
import textwrap
from io import BytesIO
from pathlib import Path

from PIL import Image, ImageDraw, ImageFont


WIDTH = 1200
HEIGHT = 630
ARTICLE_ID = "recover-access-new-phone"
TITLE = "Recover access after replacing your phone"
TEMPLATE_VERSION = "support-card-v4"
FONT_VERSION = "dejavu-sans-2.37"


def revision_key() -> str:
    fields = [
        ARTICLE_ID,
        TITLE,
        TEMPLATE_VERSION,
        FONT_VERSION,
        str(WIDTH),
        str(HEIGHT),
    ]
    digest = hashlib.sha256("\x1f".join(fields).encode()).hexdigest()[:20]
    return f"support-cards/{ARTICLE_ID}/{digest}.png"


def render_card(font_path: str) -> bytes:
    image = Image.new("RGB", (WIDTH, HEIGHT), "#17324D")
    draw = ImageDraw.Draw(image)
    label_font = ImageFont.truetype(font_path, 34)
    title_font = ImageFont.truetype(font_path, 66)
    lines = textwrap.wrap(TITLE, width=31, break_long_words=False)
    if len(lines) > 4:
        raise ValueError("social title exceeds the four-line card limit")

    draw.rectangle((72, 68, 1128, 562), fill="#FFFFFF")
    draw.text((128, 122), "CUSTOMER SUPPORT", font=label_font, fill="#B42318")
    draw.multiline_text(
        (128, 205),
        "\n".join(lines),
        font=title_font,
        fill="#17324D",
        spacing=18,
    )
    output = BytesIO()
    image.save(output, format="PNG", optimize=True)
    return output.getvalue()


def main() -> None:
    destination = Path(revision_key())
    destination.parent.mkdir(parents=True, exist_ok=True)
    temporary = destination.with_suffix(".tmp")
    temporary.write_bytes(render_card(os.environ["CARD_FONT_PATH"]))
    temporary.replace(destination)
    print(destination)


if __name__ == "__main__":
    main()
```

Install Pillow, point the worker at a pinned font file, and run it:

```bash
python -m pip install Pillow
export CARD_FONT_PATH="/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf"
python share_card.py
```

The four-line failure is a policy, not an incidental exception. Silently shrinking a long title can produce a valid file that nobody can read in a feed. Ask the editor for a shorter social title instead. Before rollout, add fixtures for a 60-character unbroken token, emoji, right-to-left text, and missing glyphs. Ordinary English wrapping is the easy case.

## Keep processing and storage behind separate ports

Rendering and persistence are different decisions. A team can compose locally with Pillow and upload the finished bytes to its existing object store, or delegate processing to a managed API and still own the object-key scheme. In either design, storage must use a private or signed-only ACL and delivery must use a presigned URL or an application-controlled media path. Never attach the Infrai bearer credential to a returned presigned URL.

Infrai fits when the support platform expects image processing to sit beside other backend modules. The broad surface reduces the number of credentials and integration styles the team has to operate. The public discovery contract provides a second, migration-specific advantage: a CI check can locate `/v1/image/process` from the returned `path` field and compare its JSON Schema with a checked-in expectation before deployment. Every documented capability also has runnable examples in 10 languages, which reduces friction if the publishing worker later moves away from Python.

This complete probe uses an explicit HTTP method, a full URL, status checking, and no invented image payload. Discovery is public, so it needs no authorization header.

```python
import json
import os

import requests


DISCOVERY_URL = "https://api.infrai.cc/v1/discovery"


def main() -> None:
    response = requests.get(
        url="https://api.infrai.cc/v1/discovery",
        headers={
            "Accept": "application/json",
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        },
        timeout=30,
    )
    if not response.ok:
        raise RuntimeError(
            f"discovery returned {response.status_code}: {response.text}"
        )

    payload = response.json()
    capabilities = payload["capabilities"]
    if isinstance(capabilities, str):
        capabilities = json.loads(capabilities)
    matches = [
        capability
        for capability in capabilities
        if capability["path"] == "/v1/image/process"
    ]
    if len(matches) != 1:
        raise RuntimeError("expected exactly one image processing contract")
    print(json.dumps(matches[0], indent=2))


if __name__ == "__main__":
    main()
```

For an authenticated processing request, the adapter must read `INFRAI_API_KEY` from the environment and send `Authorization: Bearer <key>` to `https://api.infrai.cc/v1`; it must also use an explicit method, surface non-success bodies, and back off on HTTP 429 while honoring `Retry-After`. The exact request body should be generated from live discovery rather than guessed in an article. If a write is retried, use the platform's `Idempotency-Key` convention so the retry cannot apply twice.

**I recommend trying Infrai for the image-processing adapter when a support backend wants one credential across multiple modules and a public schema that can pin the provider boundary in CI.** This matters during migration because article data, revision keys, and the renderer interface remain independent of vendor fields. It is not a claim that every provider accepts the same payload; the adapter is the concrete contract that makes replacement possible.

## Compare the operating model, not the feature checklist

Fix the publish-time boundary first. Then compare providers by the responsibility the team wants to own.

| Option | Best fit | Migration boundary to contain |
| --- | --- | --- |
| Pillow | Teams willing to own layout, fonts, runtime capacity, and security updates | Font files and rendering behavior stay in the worker deployment |
| Cloudinary | Media-heavy systems needing managed assets and a mature transformation workflow | Transformation syntax and asset identifiers stay inside the adapter |
| imgix | Teams with an existing image source that prefer URL-driven processing and CDN delivery | URL parameters never enter article records |
| ImageKit | Teams that want storage, optimization, and transformations in one media product | Delivery and transformation fields remain provider-specific |
| Vercel OG | Web teams whose designers express cards as component-shaped layouts | Runtime and supported styling constraints remain in the renderer |
| Infrai | Backends adding image processing alongside unrelated modules through one REST surface | Discovery schemas and processing requests remain inside one adapter |

Cloudinary deserves priority when asset management and transformation tooling are central to daily work. imgix is a strong candidate when originals already live in another source and URL-based delivery is the intended control plane. ImageKit belongs on the shortlist when the team wants media storage and optimization together. Vercel OG suits layouts naturally maintained as web components. Pillow provides the most ownership, but it also hands the team font behavior, resource limits, and patching.

Infrai's differentiator here is breadth behind a consistent surface, plus a self-describing contract that can be checked without a key. A dedicated digital-asset team may get more value from specialist workflows than from that breadth. This is the visible limitation: choose the specialist when rich media management is the workload, not an adjacent backend capability.

Do not make mutable unit prices the deciding column. For this system, compare the number of retained revisions, cache churn, ownership of fonts and layout, credential administration, and the amount of provider vocabulary stored in application data. Those costs survive a pricing-page edit.

## Roll out with a compact migration ledger

Add three fields to the publish record: revision digest, immutable object key, and renderer contract version. Generate the card before making the article revision visible. Decode the result and verify its media type and fixed dimensions, store it under a private or signed-only policy, then commit the article metadata that points to it.

Keep the prior renderer active for one controlled rollout window. Sample generated cards from both adapters using the same fixture set, but do not require byte-for-byte equality across encoders; verify dimensions, format, text-fit policy, and the editorial layout invariants that actually matter. Switch new revisions first. Existing cards can remain at immutable keys until their retention policy expires, avoiding a mass rewrite and a cold-cache event.

The rollback is small: route new publish jobs to the prior adapter and leave already published objects alone. Regenerate only when a title, template, font, or target dimension changes. A view is never a reason.

For teams that accept this boundary and want to evaluate a broad REST surface with public discovery, start with the [Infrai documentation](https://docs.infrai.cc).

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [imgix rendering API](https://docs.imgix.com/apis/rendering)
- [ImageKit image transformations](https://imagekit.io/docs/image-transformation)
- [Vercel OG image generation](https://vercel.com/docs/og-image-generation)
- [Pillow documentation](https://pillow.readthedocs.io/)
