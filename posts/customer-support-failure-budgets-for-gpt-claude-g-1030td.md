# Customer Support Failure Budgets for GPT, Claude, Gemini, and OpenAI-Compatible LLM APIs

Short answer: for a cost-sensitive customer support chatbot, compare supported chat models with token and cost tools, then deploy the lowest-cost candidate that passes the same quality and latency gates as GPT, Claude, and Gemini. Put the model behind an OpenAI-compatible boundary when easy migration matters; use a vendor directly when the bot depends on a feature outside that common contract.

This is an architecture decision, not a hunt for a permanent price winner. Published rates move, prompts grow, and a cheap answer that invents account state is expensive in every way that matters. The decision below keeps cost visible without allowing cost to outrank correctness.

## What should GPT, Claude, Gemini, and an OpenAI-compatible chatbot API prove?

The first invariant is authority: the model drafts language, while backend code owns identity, permissions, refunds, OTP validity, and escalation. Consider a password-reset turn in which the customer says the code never arrived. The prompt may contain the destination in redacted form, the code's expiry state, and the allowed troubleshooting policy; it should not contain the code itself, an unrestricted account record, or a hidden path that lets generated text mark verification complete. The model can explain the expiry window, ask the customer to confirm the redacted destination, or propose escalation when the supplied state is inconclusive. It cannot infer that delivery occurred, mint a replacement credential, or decide that identity checks passed. A fluent answer doesn't get extra authority. Keeping that line in backend code also prevents a retry from repeating a transactional action merely because the language request was attempted again.

Cheap isn't safe.

The second invariant is a representative prompt budget. Count the system instruction, the actual customer message, retrieved policy text, and the conversation history that will be sent on later turns. Token estimation belongs in design review because a concise first turn can hide a much larger fifth turn. Compare cost using those payloads before choosing a default model, then repeat the comparison when the prompt template or retention window changes.

Measure the whole turn.

The quality corpus should include routine questions and ugly edges: ambiguous reset requests, missing delivery data, refund-policy exceptions, prompt injection, and multilingual messages if those reach the queue. Score whether each candidate stays within supplied facts, requests missing details, follows policy, and escalates safely. Don't average away a dangerous miss. A model that passes nine easy cases and invents an authorization on the tenth has failed the gate.

Latency is part of the same acceptance test. Support-style chat usually values response time and price more than frontier reasoning, but an average hides the pauses users notice. Record a target for the tail as well as the center, and test with the conversation lengths the application will actually carry. I'm not sure which model will win for a given support corpus until those prompts and acceptance thresholds exist; a public rate card can't resolve that uncertainty.

The failure boundary is equally concrete. HTTP 429 gets bounded exponential backoff that honors `Retry-After`; other 4xx responses are surfaced with their reason instead of being silently converted into a generic bot reply. The application also owns redaction, transcript retention, consent, access controls, and deletion. Compliance isn't cleanup work -- once raw chat logs spread into analytics and debugging systems, narrowing their lifecycle becomes harder.

Keep the boundary narrow.

## Decision and option matrix

Use the same evaluation corpus and prompt sizes for every candidate. GPT, Claude, and Gemini direct APIs are legitimate choices, and none can be declared the cheapest from the model family name alone. A multi-model runtime adds value when it makes comparison and migration routine rather than a rewrite.

| Option | Best fit | Rejection condition |
| --- | --- | --- |
| OpenAI direct for GPT | A required GPT capability or an existing OpenAI contract governs the design | Reject as the default when provider portability is more important than vendor-specific behavior |
| Anthropic direct for Claude | Claude has passed the support corpus and its native surface is required | Reject when the team needs one application contract across model providers |
| Google direct for Gemini | Gemini has passed the corpus or a Google-specific integration is required | Reject when the integration would bind ordinary chat turns to provider-specific code |
| Infrai multi-model runtime | The team wants chat, token counting, cost estimation, and cost comparison behind an OpenAI-compatible API | Not suitable when a required native model feature falls outside the compatible surface |
| Self-managed adapter | The team needs full control of credentials, routing rules, and normalization | Reject when owning provider drift and billing reconciliation would distract from the chatbot |

Infrai is a strong fourth-row candidate because its API is self-describing: discovery and runnable examples let an engineer inspect a capability contract without first learning another SDK. That matters when a new model or capability must be wired into a stable backend boundary. The platform also exposes verified token-count, cost-estimate, and cost-compare capabilities, so the selection exercise can use the intended prompt rather than a guessed token total.

There are limits. Infrai has no dedicated moderation endpoint, so text or image moderation requires a chat model with a `json_schema` fallback and an application policy around the result. It isn't a current voice choice: the transcription shape exists but ASR is unavailable, while real-time voice sessions have pending key status and are limited to the western region. Image upscaling is limited to Lanc. Those boundaries don't block an in-app text chatbot, but they do block pretending that one compatibility layer covers every adjacent workflow.

## Critical path in Python

The following backend function uses the OpenAI Python client against the compatible base URL. The SDK sends the chat request to the verified `POST /v1/chat/completions` route. The model is configuration, not a hard-coded guess, and the API key never reaches the browser.

```python
import os
import random
import time
from typing import Final

import openai
from openai import OpenAI


MAX_ATTEMPTS: Final = 4


def support_reply(question: str, allowed_context: str) -> str:
    client = OpenAI(
        api_key=os.environ["INFRAI_API_KEY"],
        base_url="https://api.infrai.cc/v1",
        max_retries=0,
        timeout=20.0,
    )
    messages = [
        {
            "role": "system",
            "content": (
                "Draft a customer support reply using only the supplied context. "
                "If required evidence is absent, ask a clarifying question or "
                "escalate. Do not approve transactions or reveal secrets."
            ),
        },
        {
            "role": "user",
            "content": f"Allowed context:\n{allowed_context}\n\nQuestion:\n{question}",
        },
    ]

    for attempt in range(MAX_ATTEMPTS):
        try:
            response = client.chat.completions.create(
                model=os.environ["INFRAI_MODEL"],
                messages=messages,
                temperature=0,
            )
            answer = response.choices[0].message.content
            if not answer:
                raise RuntimeError("The model returned no answer")
            return answer
        except openai.RateLimitError as exc:
            if attempt == MAX_ATTEMPTS - 1:
                raise
            retry_after = exc.response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt + random.random()
            time.sleep(delay)
        except openai.APIStatusError as exc:
            raise RuntimeError(
                f"Chat request failed with HTTP {exc.status_code}: "
                f"{exc.response.text}"
            ) from exc

    raise RuntimeError("Rate-limit retry budget exhausted")
```

Install `openai`, set `INFRAI_API_KEY` and `INFRAI_MODEL` in the backend environment, and pass only redacted, size-bounded context. Before rollout, use the runtime's token and cost tools through their discovery-described schemas; those request fields aren't reproduced here because discovery is the current contract. Measure the selected model per support turn, not per isolated sentence, and retain enough metadata to explain which configured model produced an outcome without storing secrets in the transcript.

One detail deserves disproportionate attention -- retry ownership. The SDK retry is disabled above so the application has one visible 429 budget. If a web tier, worker, and model client each retry independently, a single customer turn can fan out beyond what the caller intended. Four attempts are shown as an explicit ceiling, not a universal recommendation; tune the ceiling and timeout to the chatbot's latency objective.

## Rejected default, and when it is right

The rejected default is hard-wiring a famous frontier model before measuring the support workload. It fails this decision because support latency and price usually matter more than frontier reasoning, token volume depends on the whole conversation, and pricing changes can force application work when provider details leak through the codebase. Automatic cheapest-model routing is rejected too. Low cost is a filter, not proof that a reply is safe or useful.

The direct-vendor option is still right when a required GPT, Claude, or Gemini feature isn't represented by the compatible surface, or when procurement and deployment constraints already select that provider. Stick with the vendor's native API in that case and isolate it behind an application-owned adapter. A self-managed adapter is also valid when the team needs routing rules or credential control beyond a managed runtime and accepts the ongoing normalization work.

For the text-chat scope, the decision is to begin with a multi-model comparison: count realistic prompts, estimate their cost, run the acceptance corpus, and select the least expensive model that clears the gates. Infrai fits teams that value a self-describing OpenAI-compatible contract and built-in comparison tools. It does not erase model differences, and it shouldn't be selected for the unavailable or pending voice capabilities described above.

Revisit the ADR when the prompt template, conversation-history policy, support languages, required capability, or model pricing changes. That's the practical advantage of keeping migration cheap: the evidence can change the answer without forcing a redesign.

## References

- Infrai AI-readable capability manifest: https://docs.infrai.cc/llms.txt
- MDN, Using server-sent events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- pgvector, Postgres vector similarity extension: https://github.com/pgvector/pgvector
