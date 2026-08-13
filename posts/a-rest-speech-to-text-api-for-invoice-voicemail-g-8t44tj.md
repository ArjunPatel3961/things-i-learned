# A REST Speech-to-Text API for Invoice Voicemail: Grade Field Accuracy, Not Whisper WER

Pick the speech-to-text API whose output your extractor can verify field by field, and let REST ergonomics, pricing model and the Whisper-versus-managed argument settle the ties. The deciding constraint in a healthtech accounts-payable pipeline isn't how readable the transcript reads to a human — it's whether a supplier's invoice number survives a narrowband phone line intact, and whether the response tells you when it didn't.

That single reframing changes the shortlist.

Here's the system I'm describing, because the answer depends on it. A supplier's billing clerk calls the AP desk of a healthtech company, gets voicemail, and reads out an invoice number, a PO number and an amount in dispute. The recording lands in object storage. A worker transcribes it, an extractor pulls structured fields, and the ledger gets a JSON object: `invoice_number`, `po_number`, `total`, `currency`, `issued_on`. Nobody in the SaaS app ever reads the transcript. It exists only as an intermediate artifact, which is exactly why the evaluation criteria most teams copy from captioning benchmarks don't transfer.

## Why word error rate is the wrong acceptance metric for invoice audio

Word error rate averages over every token in the clip. The tokens that decide whether a payment posts correctly are maybe two percent of the audio — an alphanumeric string, a decimal amount, a date. A 6% WER transcript can be flawless for field extraction if the errors landed on "um" and "sorry, one second", and worthless if one of them landed on the invoice number.

Same clip, same score, opposite outcome.

Alphanumerics over telephony are the hard case, and they're hard for a reason that predates neural ASR. Classic PSTN audio is narrowband — 8 kHz sampling, G.711 companding as described in RFC 3551 — so the spectral cues that separate B, D, E, P, T, V and 3 are largely gone before the model ever sees the waveform. M and N collapse. F and S collapse. Anyone who has shipped voice-delivered OTP codes has met this confusion set already, and the mitigations are the same here: constrain the expected format, ask for word-level confidence, and never let an unverified string reach a system of record.

So the acceptance metric I'd hold a vendor to is field-level exact match on a labeled corpus of your own calls, measured per field, alongside a second number almost nobody asks for: the rate at which the engine declines to guess. An engine that returns low confidence on a garbled invoice number is more useful than one that returns a confident, plausible, wrong one.

That second failure mode is worth understanding mechanically. Sequence-to-sequence ASR models are language models with an audio encoder, so on silence, hold music or crosstalk they can emit fluent text that was never spoken. The reference Whisper implementation ships explicit decoding guards for it — `no_speech_threshold`, `logprob_threshold` and `compression_ratio_threshold`, with temperature fallback when a segment trips them. If you self-host, those knobs are yours to tune. If you buy a managed API, ask which equivalent signals are exposed in the response, because a JSON body with text and no per-word confidence gives your extractor nothing to gate on.

## Which speech-to-text API should a REST-only Node.js app trust with invoice fields?

Trust the one that returns word-level timestamps and per-word confidence over plain HTTPS, accepts a domain vocabulary or phrase-hint list, pins a model version you control, and comes with a written processing agreement naming the region where audio is stored.

Those four properties are load-bearing in different ways. Confidence is what makes a human-review queue affordable — without it you either review every call or trust every call, and both are bad. Phrase hints are how supplier names, drug names and your own PO prefix stop being transcribed phonetically. Version pinning is the difference between a silent accuracy regression and a deliberate upgrade you regression-test. The processing agreement is the part engineers skip and lawyers can't.

On the privacy axis, an AP recording in healthtech is not obviously clinical, but it's rarely clean either — clerks mention patient names, order references and clinician contacts in passing. Treat it as regulated. In the EU that means a GDPR Article 28 processor contract with documented sub-processors and a real deletion path; in the US it usually means a business associate agreement before a byte of audio leaves your network. Zero-retention needs to be a contractual term with a retention window in writing, not a toggle in a dashboard. Redact before you store, not after you're asked.

Pricing deserves less agonizing than it gets. Per-audio-minute billing is the industry norm, so your bill tracks call volume, and the thing that actually inflates it is re-transcribing the same audio every time you change a prompt or a parser. Cache by content hash and the pricing question mostly answers itself.

Call volume drives the bill. Re-runs inflate it.

## Three architectures, and what each column hides

| Option | Integration | Where audio lives | Custom vocabulary | Main boundary |
| --- | --- | --- | --- | --- |
| Managed ASR API | one HTTP request per clip | vendor region you select | phrase hints, usually capped | you inherit their model updates |
| Self-hosted open-weights model | container, GPU, batching, queue | your own network | full control, fine-tuning possible | you own uptime and capacity planning |
| Telephony-provider transcription | already attached to the call record | wherever the call was recorded | thin or absent | tuned for gist, not for exact strings |

The middle row is the honest Whisper alternative story. Self-hosting removes the processor contract problem entirely, which in EU healthtech is worth real money, and it gives you the decoding thresholds above. The catch is that you've traded a vendor relationship for an on-call rotation, GPU capacity planning and your own drift monitoring. A self-hosted deployment also doesn't support the compliance shortcut of handing an auditor someone else's third-party attestation — you inherit the entire control set.

A gateway in front of the engine is what keeps that decision reversible. Open-source LLM gateways such as LiteLLM popularized the pattern for text models, and the same shape works here: one internal REST contract, engine selection behind it, per-request logging of which engine and version served the call. Then swapping engines is a config change and a regression run, not a refactor.

## Where the pipeline actually runs: chunking, gating, and the retry path

The critical path is short. Segment on silence rather than fixed windows so an invoice number is never cut in half, hash the bytes for idempotency, call the engine, then gate on structure and confidence before anything touches the ledger.

```python
import hashlib
import json
import os
import httpx
from jsonschema import Draft202012Validator

ASR_URL = os.environ["ASR_ENDPOINT"]      # internal gateway, not a vendor URL
CRITICAL = ("invoice_number", "po_number", "total")
MIN_CONF = 0.85                            # tune against your own labeled corpus

def transcribe(chunk: bytes, cache) -> dict:
    key = hashlib.sha256(chunk).hexdigest()
    hit = cache.get(key)
    if hit:
        return json.loads(hit)             # retries never re-bill the same audio
    resp = httpx.post(
        ASR_URL,
        headers={"Authorization": f"Bearer {os.environ['ASR_TOKEN']}"},
        files={"audio": ("call.wav", chunk, "audio/wav")},
        data={
            "engine_version": "pinned",    # never float on the vendor's latest
            "granularity": "word",         # word timestamps + per-word confidence
            "vocabulary": json.dumps(SUPPLIER_TERMS),
        },
        timeout=120.0,
    )
    resp.raise_for_status()
    payload = resp.json()
    cache.set(key, json.dumps(payload))
    return payload

def route(fields: dict, words: list[dict]) -> str:
    if list(Draft202012Validator(FIELD_SCHEMA).iter_errors(fields)):
        return "dead_letter"               # malformed structure is never auto-posted
    weak = [f for f in CRITICAL if confidence_of(fields[f], words) < MIN_CONF]
    if weak:
        return "human_review"              # queue it, with the audio timestamp attached
    if not checksum_ok(fields["invoice_number"]):
        return "human_review"
    return "auto_post"
```

Two details in there carry most of the operational weight. The content hash means a retry after a timeout costs nothing and can't double-insert, which matters because ASR requests are slow enough that clients time out before servers do. And `checksum_ok` is the cheapest accuracy win available: most supplier invoice numbers have structure — a fixed prefix, a length, sometimes a check digit — and validating that structure catches transposition errors that no confidence score will flag, because the model was perfectly confident about a digit it heard wrong.

Neither trick is clever. Both are load-bearing.

Observability here is field-level or it's decorative. Log per-field confidence, review-queue rate and post-correction rate by supplier, then alert on the derivative rather than the level — a jump in review rate for one supplier usually means their new clerk calls from a mobile in a warehouse, and a jump across all suppliers means the engine changed under you.

Keep a regression corpus. A few hundred real clips with hand-labeled fields, replayed on every parser change and every engine upgrade, is the only evidence that will settle an argument about whether accuracy actually got worse. Reconciling transcribed supplier names against your vendor master is a separate problem that text embeddings handle well — pass the transcribed name through an embedding endpoint and take the nearest neighbor above a threshold, rather than trying to make the ASR engine spell it right.

## The option I rejected, and when you should pick it anyway

I'd skip sending audio straight to a multimodal model and asking for the JSON in one hop, at least for this job. It reads beautifully in a demo. What you lose is the intermediate artifact — no transcript to diff, no word-level confidence to gate on, no way to tell whether a wrong `total` came from mishearing or from misparsing. Debugging becomes archaeology.

It's a reasonable choice when the fields are low-stakes, when audio quality is controlled, or when a human already reviews every result and the transcript would just be noise on their screen. Call routing tags, sentiment, topic labels: fine. A number that moves money: not yet, not without the artifact.

The same honesty applies to my own recommendation. If your audio is clean studio recordings and your product surfaces captions or diarized transcripts to end users, a field-accuracy harness measures the wrong thing entirely — stick with WER and diarization error rate, because in that product the transcript *is* the deliverable. And if you're transcribing at a scale where an on-call ASR rotation is cheaper than per-minute billing, the self-hosted row stops being a compliance preference and becomes plain arithmetic.

I'm not certain the confidence threshold generalizes, honestly. 0.85 is a starting point I'd expect to move once you've labeled a few hundred of your own calls, and it will differ per field — amounts and dates behave nothing like alphanumeric identifiers. Label first, then set it.

## References

- Whisper reference implementation and decoding thresholds — https://github.com/openai/whisper
- Robust Speech Recognition via Large-Scale Weak Supervision (Whisper paper) — https://arxiv.org/abs/2212.04356
- RFC 3551, RTP Profile for Audio and Video Conferences (G.711 telephony audio) — https://www.rfc-editor.org/rfc/rfc3551
- GDPR Article 28, Processor obligations — https://gdpr-info.eu/art-28-gdpr/
- HHS guidance on HIPAA business associates — https://www.hhs.gov/hipaa/for-professionals/privacy/guidance/business-associates/index.html
- JSON Schema specification — https://json-schema.org/specification
- LiteLLM, open-source LLM gateway — https://github.com/BerriAI/litellm
- Embeddings guide (nearest-neighbor matching of transcribed names) — https://platform.openai.com/docs/guides/embeddings
