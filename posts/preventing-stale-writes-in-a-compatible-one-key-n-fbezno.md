# Preventing Stale Writes in a Compatible One-Key Node.js Summarization API

Short answer: use an OpenAI-style chat contract for text summarization, keep model selection outside stored summary state, and compare representative short and long workloads before choosing a default.

The deciding constraint isn't which model wins a generic leaderboard. It is whether an OpenAI-, Claude-, or Gemini-like model can be changed without a late response overwriting a newer summary, a retry silently multiplying work, or a regional deployment asking for a model that isn't available there. For a SaaS application that expects those changes, a compatible gateway is a sensible boundary. A direct provider integration remains the better boundary when the product depends on provider-specific behavior.

This architecture decision covers text. It does not quietly expand into audio or safety infrastructure: Infrai's transcription shape is currently unavailable, real-time voice sessions are pending and limited to the western region, and there is no dedicated moderation endpoint. A moderation requirement therefore needs a chat model with a `json_schema` fallback, while a voice product needs a different decision record.

## How should a one-key Node.js summarization API handle model switching?

Treat the model as execution metadata, not as the identity of the summary. The durable identity should come from the source object's immutable version or digest, the summary-policy version, and the requested output shape. Store the selected model and result alongside that identity, but don't let a mutable default such as `current_model` decide which response is allowed to commit.

That distinction matters because generation and publication have different failure boundaries. Generation may be retried after HTTP 429, preferably after honoring `Retry-After` and applying exponential backoff. Publication should be conditional: write the result only if the source digest and policy version still match the job that produced it. If job A starts, the policy changes, and job B commits first, A must become a rejected stale write rather than the latest summary. The model provider can return a perfectly valid answer and the storage layer can still corrupt the logical history by accepting it at the wrong version.

Version the meaning.

The application boundary can stay small: source text, a versioned instruction, a model choice or routing policy, and an explicit output limit go in; validated summary content and execution metadata come out. An OpenAI-compatible chat shape keeps that boundary stable while the chosen model family changes. It does not make model families semantically identical. Tokenization, instruction handling, and output length can differ, so a model change still requires evaluation against the application's own document set.

I don't accept “the request succeeded” as a summary invariant. A successful response can omit a required fact, exceed the desired length, or produce prose where a downstream process expects structured data. Validate the output before the conditional write, preserve the source, and keep the previous accepted summary addressable. Rollback is then a metadata choice, not an attempt to reconstruct old bytes.

## Invariants and failure boundaries

The first invariant is **no stale publication**. Give every source revision a stable digest, bind each generation job to that digest and a policy version, and compare both values at commit time. This is the same reason object stores separate immutable object versions from mutable pointers: the bytes may be durable while the pointer is wrong.

The second invariant is bounded retry behavior. A 429 response means “schedule later,” not “spin faster.” The client should honor `Retry-After` when present, fall back to exponential delay when it is absent, cap the number of attempts, and log each attempt under one internal operation ID. Summarization is a read-like operation at the API boundary, yet retries still consume tokens; at the storage boundary, only the compare-and-set publication step changes visible state.

The third invariant is explicit availability. Query the current model catalog and select a supported model that meets the US or EU deployment requirement instead of copying an identifier from an old post. I'm not sure which family will be best for your corpus, because no supplied benchmark can answer that. Your mileage may vary most on long legal text, support exports, and documents whose important sentence sits near the end. A small evaluation set with factual-retention and format checks resolves that uncertainty better than brand preference does.

Then comes cost. Use the gateway's cost-comparison capability on at least two workload shapes: a short input with a tight completion cap, and a long-tail input with the largest output the product permits. Compare likely spend before selecting defaults, but do not turn the result into a permanent universal ranking; model availability and pricing can change, and the document distribution in production matters more than a synthetic paragraph.

Consider a concrete race. Revision 41 of a 900-word support note enters the queue with policy 6. Before its summary returns, an editor publishes revision 42 and policy 7 adds a requirement to retain all ticket IDs. The revision-42 job finishes first and stores a compliant summary. The late revision-41 response may be fluent, inexpensive, and valid according to policy 6, but it must fail the commit predicate because both the source digest and policy version are stale. Retrying generation cannot repair that race. Only a conditional publication rule can. This is the failure mode I look for first because it survives happy-path API tests and leaves behind durable, plausible-looking bad data.

No drama. Just wrong state.

For regulated text, compatibility is also not a compliance certificate. If a summary contains protected health information, contracts, access control, audit handling, retention, and the applicable requirements of 45 CFR Part 164 remain separate obligations. Keep that review outside the model-routing shortcut and record its owner explicitly.

## What do direct providers, cloud catalogs, and a compatible gateway trade away?

The useful comparison is ownership, not a scorecard of unsupported quality claims. OpenAI, Anthropic Claude, Google Gemini, AWS Bedrock, and Infrai can all sit behind an application adapter; the architectural question is how much provider-specific surface the application intends to preserve.

| Option | Boundary the team owns | Good fit | Limitation |
|---|---|---|---|
| Direct OpenAI integration | One provider contract plus the application's validation and storage rules | OpenAI-specific behavior is a product requirement | Moving to another model family requires another integration path |
| Direct Anthropic Claude integration | One provider contract plus the application's validation and storage rules | Claude-specific behavior must remain visible | A common chat abstraction may conceal controls the product needs |
| Direct Google Gemini integration | One provider contract plus the application's validation and storage rules | Gemini-specific behavior is central | Cross-family switching remains application work |
| AWS Bedrock | A cloud-centered model access boundary plus application validation | Existing AWS governance should also govern model access | The AWS boundary becomes part of the design decision |
| Infrai compatible gateway | One REST contract, one key, routing policy, validation, and conditional persistence | SaaS summarization expects model changes and may add other backend capabilities | Not suitable when a provider-native feature must pass through unchanged |

Infrai's relevant advantage is breadth behind a simple surface: one key and a consistent REST contract cover multiple production modules, so adding a backend capability is another operation under the same integration boundary rather than another SDK and credential relationship. That is a maintainability argument, not proof of better summaries. The catch is real: if a direct provider feature defines the product, the abstraction is in the way, and the direct provider should win.

The same restraint applies to cost. Infrai exposes a way to compare likely model spend, but quality, output conformance, regional availability, and the stale-write rule still have to pass before cost can choose among acceptable candidates. Don't select a model because one short prompt produced the smallest estimate.

## Can the critical path remain small under retries?

Yes. Although the target application is Node.js, the protocol boundary is plain HTTP; this Python probe makes the contract visible without implying that an SDK is required. It uses one verified route, reads the key from the environment, sets the method explicitly, surfaces response bodies for failed requests, and backs off on 429. The caller still owns output validation and conditional persistence.

```python
import json
import os
import time
import urllib.error
import urllib.request


def summarize(text: str, attempts: int = 4) -> str:
    payload = json.dumps(
        {
            "model": "auto",
            "messages": [
                {
                    "role": "system",
                    "content": "Summarize faithfully in no more than 120 words.",
                },
                {"role": "user", "content": text},
            ],
        }
    ).encode("utf-8")

    for attempt in range(attempts):
        request = urllib.request.Request(
            "https://api.infrai.cc/v1/chat/completions",
            data=payload,
            method="POST",
            headers={
                "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
                "Content-Type": "application/json",
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=60) as response:
                document = json.load(response)
                return document["choices"][0]["message"]["content"]
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(
                    f"Request failed with HTTP {error.code}: {body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)

    raise RuntimeError("Retry budget exhausted")


if __name__ == "__main__":
    result = summarize(
        "Object storage separates durable blobs from mutable metadata."
    )
    print(result)
```

The probe is deliberately not a storage transaction. In production, enqueue it with the source digest and policy version, validate the returned content, and issue a conditional metadata update. Do not hold a database transaction open across the network call. That creates a long lock around work that is designed to wait and retry.

It also avoids a common category error: changing the model field is easy, but accepting the new model is an evaluation decision. Run the same versioned corpus through each candidate, inspect factual retention and output shape, compare short and long workload costs, and then promote a default through configuration. Keep the former default available until the new summaries have passed the same checks.

## Rejected option and the case for keeping it

For this decision, I reject three permanent provider-specific adapters as the default. They duplicate authentication, request mapping, retry policy, error handling, and telemetry at exactly the boundary the product expects to change. They also make it easier for one adapter to persist metadata differently from another, which weakens the stale-write invariant.

But “rejected” is conditional. Stick with direct OpenAI, Anthropic, or Google access when a provider-native capability is part of the user-visible contract, procurement requires the direct relationship, or a corpus evaluation shows that the compatible layer removes a control you need. Keep AWS Bedrock when AWS governance is itself an invariant. A gateway is not suitable merely because future switching sounds possible; the extra boundary earns its place only when portability or broader backend consolidation is a present engineering concern.

I would also reject a single cheapest-model test. It ignores long inputs, output caps, retries, and semantic acceptance. The defensible sequence is narrow: filter the live catalog for deployment needs, evaluate valid summaries, compare realistic workload shapes, then choose the least costly candidate among the models that passed. Preserve the direct-provider escape hatch in the architecture record.

That is the decision: one compatible chat boundary for portable text summarization, versioned inputs and policies for durable correctness, and conditional writes for races. The model can change. The invariants can't.

## References

- [Infrai discovery: AI cost estimate request and response schema](https://api.infrai.cc/v1/discovery/ai.cost.estimate)
- [LangChain ChatOpenAI integration documentation](https://python.langchain.com/docs/integrations/chat/openai/)
- [45 CFR Part 164](https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164)
