# Why Customer Support Chatbot LLM API Costs Start in the Data Layer

Short answer: the lowest-cost LLM API for a customer support chatbot is the compatible option that meets a fixed resolution-quality threshold on your own conversations after retrieval, retries, and human escalation are counted. A token price cannot answer that question by itself.

This changes the order of work. Before comparing model families or an OpenAI-compatible interface, define the unit that the application buys: a resolved conversation with an acceptable answer, delivered within a latency target and without violating retention rules. Then freeze an evaluation set, constrain the context pipeline, and replay exactly the same work against every candidate. Don't let a polished answer to an easy password-reset question stand in for a system test.

The storage layer matters because a support bot repeatedly reads and rewrites state. It owns conversation history, retrieved documents, embedding versions, evaluation labels, and the evidence needed to explain why one run cost more than another. Ignore that state and the API comparison becomes a comparison of invoices whose workloads were never equal.

## What constraint determines chatbot cost?

Start with a per-conversation budget, not a per-request budget. A conversation can require several model calls, retrieval queries, retries, and a human handoff; optimizing one call while increasing any of the others is accounting theater. The useful numerator is total runtime cost for the evaluated conversations. The denominator is the number that satisfy the resolution rubric.

That rubric must be settled before the candidates are run. For customer support, it might require the answer to cite the applicable policy passage, avoid unsupported actions, ask for missing account context, and escalate when the evidence is insufficient. Those are testable behaviors. “Sounds helpful” isn't.

Context is the first architectural constraint. Sending the full transcript on every turn makes later requests carry earlier material again, while retrieval can add irrelevant passages if the search threshold and result count are chosen by intuition. Keep the raw transcript as durable data, but assemble a bounded prompt from a compact conversation state and a small set of relevant passages. Record the identifiers and versions of those passages beside the response. If an answer changes next week, you need to distinguish a model change from a corpus change.

The same discipline applies to embeddings. pgvector supports exact and approximate nearest-neighbor search within Postgres, including HNSW and IVFFlat indexes. That makes it a reasonable component to evaluate when the team already operates Postgres and wants ticket metadata, document versions, and vectors under one operational model. It isn't an automatic choice: index build behavior, recall targets, write volume, backup time, and corpus growth still belong in the capacity review.

Measure first.

## How should a customer support chatbot compare LLM API compatibility?

Treat compatibility as a contract you test, not a label on a landing page. An OpenAI-compatible LLM API may make the request envelope familiar, but an in-app chatbot also depends on streaming semantics, cancellation, usage accounting, error classification, tool-call representation, and the limits applied to requests. A portable application owns an internal interface and places a thin adapter at each external boundary.

Server-Sent Events are useful for one-way streaming from a server to a browser. MDN describes the browser-facing `EventSource` interface and the `text/event-stream` media type, along with named events and reconnection behavior. That establishes the transport shape; it does not make every model API stream interchangeable. Normalize provider events inside the backend, then expose one application event vocabulary to the browser, such as `delta`, `citation`, `done`, and `error`. The browser should never need to know which upstream produced a token.

The comparison fixture should be provider-neutral too. Store the request, normalized output, usage values reported by the adapter, elapsed time, retry count, retrieval manifest, rubric result, and escalation decision. I don't trust a derived “winner” row unless the underlying records remain queryable — averages erase the long conversations that usually expose context and throttling problems.

The following Python reads normalized JSON Lines produced by candidate adapters. It deliberately does no network work and assumes no vendor route. Prices are supplied as configuration at evaluation time, so a changing rate card does not alter historical usage records.

```python
from dataclasses import dataclass
from decimal import Decimal
import json
from pathlib import Path
from statistics import median


@dataclass(frozen=True)
class Rate:
    input_per_token: Decimal
    output_per_token: Decimal


def load_runs(path: Path) -> list[dict]:
    with path.open(encoding="utf-8") as source:
        return [json.loads(line) for line in source if line.strip()]


def percentile(values: list[float], fraction: float) -> float:
    ordered = sorted(values)
    index = min(len(ordered) - 1, int((len(ordered) - 1) * fraction))
    return ordered[index]


def summarize(path: Path, rate: Rate) -> dict:
    runs = load_runs(path)
    accepted = [run for run in runs if run["rubric_passed"]]
    spend = sum(
        Decimal(run["input_tokens"]) * rate.input_per_token
        + Decimal(run["output_tokens"]) * rate.output_per_token
        for run in runs
    )

    return {
        "conversations": len(runs),
        "resolution_rate": Decimal(len(accepted)) / Decimal(len(runs)),
        "cost_per_accepted_resolution": (
            spend / Decimal(len(accepted)) if accepted else None
        ),
        "median_first_event_ms": median(
            run["first_event_ms"] for run in runs
        ),
        "p95_total_ms": percentile(
            [run["total_ms"] for run in runs], 0.95
        ),
        "retries": sum(run["retry_count"] for run in runs),
        "escalations": sum(run["escalated"] for run in runs),
    }
```

There is a sharp edge in that small program: zero accepted resolutions produces `None`, not a cheap-looking zero. Keep it. A candidate that fails the quality floor is ineligible, however attractive its token rate appears.

## Failure modes that distort the comparison

The most common distortion is fixture leakage. If engineers tune prompts while looking at every expected answer, the evaluation set becomes training material. Keep a development split for prompt work and a sealed acceptance split for the decision. Version both. Access to the sealed labels should be narrow enough that an accidental peek is an event, not a habit.

Context drift comes next. Two candidates have not seen the same workload if one receives different retrieved passages, a newer policy snapshot, or a longer transcript. Persist a retrieval manifest containing document IDs, chunk IDs, content hashes, and embedding-version identifiers. Replay from that manifest during the API comparison. Later, run a separate retrieval experiment; mixing the two experiments makes causality impossible to recover.

Consider a synthetic three-turn refund case. Turn one asks whether an opened item can be returned, turn two supplies the purchase channel, and turn three reveals that the purchase occurred outside the ordinary window. Candidate A receives policy chunks `returns-7` and `exceptions-3` from corpus version 12; candidate B runs a day later and receives a newly edited `returns-8` plus a shipping FAQ because the index was rebuilt between runs. If B gives the correct exception and A does not, the model comparison has no defensible result: the candidates were given different evidence. Replaying the stored text rather than merely repeating the search query fixes that ambiguity, while retaining the search scores and original chunk identifiers still lets an auditor inspect how the production retrieval path behaved. The evaluation record should therefore separate `retrieved_at` from `evaluated_at`, identify the exact prompt assembly version, and bind every cited passage to a content hash. It looks fussy until a policy edit lands halfway through a week-long trial. Then it is the difference between an explainable result and two plausible stories about why the output changed.

Silent retries are another trap. HTTP 429 is operational evidence even when the next attempt succeeds. Count attempts, classify retry reasons, preserve any server-provided delay signal, and measure both first-event and total latency from outside the adapter. The application may tolerate a retry for a background summary but reject the same delay while an agent waits for a suggested reply. One aggregate latency objective cannot represent both paths.

Streaming can also flatter the wrong metric. A fast first event improves perceived responsiveness, yet the complete answer may still arrive too late, terminate early, or contain no usable support action. Record time to first event and time to completion separately, then grade only the completed normalized output. If the connection closes, the client needs an explicit terminal state; otherwise a partial sentence can look like success in the UI and failure only in the transcript review.

Finally, beware denominators. Cost per conversation rewards candidates that end conversations quickly, including candidates that escalate prematurely. Cost per model call rewards extra calls. Cost per accepted resolution is harder to game, but it still needs a separately reported escalation rate and a stable rubric. I'm not sure one rubric can cover both policy questions and account-specific troubleshooting; if those intents produce different error costs, split them into strata and publish the result for each.

## The architecture comparison comes after measurement

Once the replay harness is credible, compare system shapes. This is where the real trade-offs appear.

| Architecture | Useful when | Failure mode to test | Limitation |
| --- | --- | --- | --- |
| One general model for every turn | Traffic is modest and operational simplicity dominates | Large contexts quietly become the default | Not suitable when easy, repetitive intents dominate spend |
| Small-model first, then escalation | Intents separate cleanly and escalation can be labeled | The router is confidently wrong | Requires two quality surfaces and continuing threshold review |
| Retrieval plus a generation model | Answers must stay grounded in a changing support corpus | Stale or irrelevant chunks outrank the applicable policy | Adds indexing, document-version, and citation integrity work |
| Self-hosted inference with an external fallback | Workload is predictable and the team can own capacity | Saturation couples latency to traffic bursts | A poor fit without accelerator operations and an on-call owner |
| Managed end-to-end support stack | Integration speed matters more than component control | Evaluation and transcript data become hard to export | Avoid when retention, residency, or migration terms require direct control |

No row wins generally. Stick with one model when the traffic level does not justify a router and the quality floor is comfortably met. Consider a cascade only when the evaluation set demonstrates a separable class of low-risk requests and the team will inspect routing mistakes. Choose self-hosting only after staffing and capacity are included in the same cost model; accelerator time without engineering time is not a complete number.

This is also why published price tables should remain inputs rather than conclusions. Rates change, workloads differ, and output length can reverse a comparison that considered input rates alone. Your mileage may vary — especially when long policy context dominates short answers — so keep the rate snapshot, currency, effective date, and billing unit beside each evaluation result.

## Roll out without surrendering the evidence

Begin with a shadow run against a frozen, redacted fixture. Promote a candidate only after it passes the quality floor and the team has reviewed failures by intent, not just as a total. Then use a small production cohort with an immediate configuration rollback, while keeping transcript retention and consent rules identical to the existing path.

Store the durable evidence under your control: normalized requests and responses, retrieval manifests, rubric versions, adapter versions, usage records, and rollout decisions. Keep raw customer data only as long as the declared policy permits, and make deletion reach derived artifacts such as summaries and embeddings. A cheap runtime that creates an unauditable data-retention problem is expensive in a different ledger.

The catch is organizational. A portable adapter, a sealed fixture, and a versioned corpus all need owners. If the team cannot maintain those assets, choose the simpler architecture and accept a narrower comparison. The sensible target is not maximum optionality; it is enough replaceability that a model change remains an evaluation and configuration exercise instead of an application rewrite.

## References

- MDN, “Using server-sent events”: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- pgvector, “Open-source vector similarity search for Postgres”: https://github.com/pgvector/pgvector
