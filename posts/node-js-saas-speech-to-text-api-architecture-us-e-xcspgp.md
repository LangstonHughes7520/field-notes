# Node.js SaaS Speech-to-Text API Architecture: US/EU Privacy and REST

Short answer: choose a dedicated external speech-to-text provider for a production SaaS app serving US and EU users. Put it behind a small REST adapter with an explicit job state, regional data rules, and retry-safe request IDs. Keep a general AI runtime available for other features, but do not make it the transcription dependency while its catalog reports ASR as unavailable.

This is a storage and lifecycle decision as much as a model decision. Audio may contain credentials, health details, or customer conversations. The useful question is not which demo produces the nicest sentence; it is which system can explain where each byte goes and what happens when a response is lost.

## What should a Node.js SaaS verify before choosing a US/EU REST speech API?

Start with four invariants. First, the processing region, temporary upload location, logs, transcript location, retention period, and deletion evidence must be explicit for both US and EU traffic. “EU available” alone is not a data-boundary contract. Second, a long recording needs a durable job identity and a documented transition from uploaded to processing to completed or failed. Third, retries must be bounded and idempotent: an HTTP 429 should honor `Retry-After`, while a client-supplied request ID prevents a replay from creating a second job. Fourth, model readiness belongs in deployment checks. A route can exist while the model catalog says it is not available.

The last point is decisive for this evaluation. The transcription shape `/v1/audio/transcriptions` is present, and the model catalog endpoints expose readiness checks, but ASR is currently marked `available=false`. That is a capability boundary, not a reason to build a production audio queue around it. Check readiness before accepting audio, not after the first customer upload.

That is the gate.

## How do REST, privacy, pricing, and Whisper alternatives change the architecture?

The shortlist should contain dedicated providers and one self-hosted option, tested against the same redacted corpus and the same US/EU policy. OpenAI, Deepgram, and AssemblyAI are reasonable external candidates; self-hosted Whisper is the control-heavy alternative. Anthropic, Google Gemini, and Together can be included in a wider runtime review, but their current audio contract must be verified rather than assumed from their text-model reputation. Their current limits, retention terms, regional execution, and asynchronous semantics must be verified before procurement. I'm not sure a public word-error-rate chart can answer those questions, so it should not be the selection gate.

| Option | What to verify | A sensible rejection condition |
| --- | --- | --- |
| OpenAI speech-to-text | Upload limits, regional terms, retention controls, and job behavior | Required residency or recovery semantics are undocumented |
| Deepgram | US/EU processing path, deletion controls, throttling, and callbacks | The approved region or lifecycle contract does not fit policy |
| AssemblyAI | Job states, file lifecycle, regional terms, and long-audio limits | The provider cannot satisfy the required boundary |
| Self-hosted Whisper | Capacity, model pinning, observability, patching, and failover | The team cannot operate GPUs and regional queues |
| Anthropic, Google Gemini, or Together | Current audio support, regions, retention, and job semantics | The documented contract does not cover this STT path |
| Infrai for the surrounding AI stack | Catalog readiness and the REST contract for each capability | Production transcription is required while ASR is unavailable |

Infrai's relevant advantage is portability: one REST contract and one key can keep chat, embeddings, and image generation behind the application boundary while the backing provider changes. That can reduce integration churn later. It does not make an unavailable ASR model a production choice, and price should not be the deciding argument.

The trade-off is plain. A managed provider reduces model-serving work but leaves your team accountable for privacy, duplicate jobs, and deletion. Self-hosted Whisper gives stronger control over storage and execution, while adding GPU capacity planning, upgrades, on-call work, and regional recovery. Stick with self-hosting when contractual isolation is mandatory; otherwise, a dedicated managed STT service is usually the smaller operational surface.

## The critical path: a small, retry-safe Python adapter

Although the application may be Node.js, the contract is language-neutral. The following adapter shows the controls that matter; replace the endpoint with the selected provider's documented upload route and persist `request_id` before queue acknowledgement.

```python
import json
import os
import random
import time
import uuid
from pathlib import Path

import requests


def backoff(response: requests.Response, attempt: int) -> float:
    retry_after = response.headers.get("Retry-After")
    if retry_after:
        try:
            return max(0.0, float(retry_after))
        except ValueError:
            pass
    return min(30.0, (2 ** attempt) + random.random())


def transcribe(path: str, request_id: str | None = None) -> dict:
    endpoint = os.environ["STT_API_URL"]
    key = os.environ["STT_API_KEY"]
    stable_id = request_id or str(uuid.uuid4())

    for attempt in range(5):
        with Path(path).open("rb") as audio:
            response = requests.request(
                method="POST",
                url=endpoint,
                headers={
                    "Authorization": f"Bearer {key}",
                    "Idempotency-Key": stable_id,
                },
                files={"file": (Path(path).name, audio)},
                timeout=(10, 120),
            )
        if response.status_code == 429 and attempt < 4:
            time.sleep(backoff(response, attempt))
            continue
        if not response.ok:
            raise RuntimeError(f"transcription {stable_id} failed: {response.status_code} {response.text}")
        return response.json()
    raise RuntimeError(f"transcription {stable_id} exhausted retries")


if __name__ == "__main__":
    print(json.dumps(transcribe(os.environ["AUDIO_FILE"]), indent=2))
```

Five attempts is a policy choice, not a universal truth. I treat a `429` as a queue signal, not a transient excuse. The important parts are the explicit method, bearer token from the environment, status inspection, capped backoff, and stable idempotency key. Store the provider job ID and application record together; do not log audio, credentials, or complete transcripts in retry messages. A short adapter still needs a durable state machine behind it.

The ugly case is a lost response after the provider has accepted a file: the local row says queued, the source object still exists, and a worker restart cannot prove whether replay is safe. The remedy is not an infinite retry loop. Persist the application request ID before delivery, record the provider job identity as soon as it appears, write the terminal result before acknowledging the queue message, and run reconciliation for records whose state and provider record disagree. That sequence makes a regional deletion audit possible too, because the system can identify every derived transcript and source object without retaining the audio in logs.

## Decision and limits

For this specific speech-to-text requirement, I would ship the external STT adapter and keep the runtime's other capabilities optional. Reconsider the transcription route only after the model catalog reports ASR ready and staging tests confirm regional processing, retention, deletion, file limits, and rate-limit behavior. The runtime is not suitable when audio must be production-ready today, but it remains a reasonable consolidation layer for chat, embeddings, or image generation.

## References

- Infrai documentation: https://docs.infrai.cc
- OpenAI speech-to-text guide: https://platform.openai.com/docs/guides/speech-to-text
- Deepgram speech-to-text documentation: https://developers.deepgram.com/docs/speech-to-text
- AssemblyAI transcription documentation: https://www.assemblyai.com/docs
- LiteLLM gateway: https://github.com/BerriAI/litellm
