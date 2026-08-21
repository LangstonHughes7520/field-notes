# Operational Recovery for Failed Daily Report Email Retries and Queued Sends

Short answer: use cron to start a report run, then put one email job per active customer on a queue; retry individual failed sends there, move exhausted jobs to a dead-letter queue, and never rerun the whole batch merely to recover a few recipients.

The bill is made of report generation, email-provider calls, queue operations, and retained recovery state. For a weekly digest sent to `N` active customers, the dominant retry term is not the single cron trigger but the `N` independently delivered messages and any repeated provider calls. A cron rerun repeats work at batch granularity. A queue changes the recovery unit to one customer and one digest edition, which is the useful unit for both cost control and diagnosis.

Keep the business ledger longer than the transport message. Queue retention is bounded, and an acknowledgement deletes the message, so a delivery record keyed by customer and digest edition must live outside the queue. Without that record, at-least-once delivery turns an ordinary retry into a duplicate-email risk.

## What does a failed daily report email actually cost?

Consider an edition for 20,000 active customers where 37 submissions need another attempt. A cron replay makes the application re-enumerate 20,000 customers and then depend on its ledger to suppress 19,963 duplicates. Per-recipient queue recovery touches the 37 failed units. Those figures are an arithmetic example, not a throughput benchmark, but they expose the architectural difference: the batch replay has a large blast radius even when its duplicate guard is correct.

The queue does add storage and operations. Each job carries an identifier, attempt count, and enough immutable context to reconstruct the send, and each retry is another queue and provider operation. Keep the body below the 256 KB message limit; a digest that exceeds it should be stored elsewhere and referenced by an immutable identifier. Delay can be no longer than seven days, so delays are retry spacing, not an archive policy.

Retention deserves the same skepticism as price. Transport data can be retained for at most 30 days, and acknowledged messages disappear, which makes the queue a poor audit log. Persist compact send outcomes in the application database, including the idempotency key, edition, recipient identity, provider reference when available, attempt count, and final classification. That ledger is the expensive-looking extra table that prevents the genuinely expensive event: mailing the same active customer twice and then having no durable evidence of why.

The change that matters is small: stop paying the recovery cost at batch granularity. Generate the report once, reference immutable content from each job, and retry only the failed customer-delivery unit. This doesn't make failure free, but it confines repeated provider calls and investigation to the affected records.

Cron is a good daily trigger. It is not an individual-send recovery system, and asking it to become one usually creates a second, poorly specified scheduler inside application code. The cron task should calculate an edition identifier and enqueue work; it should not wait for every email. This boundary also respects the 900-second execution ceiling, which a large customer list and downstream rate limits can easily exceed.

No single transport wins every recovery problem. This comparison uses controlled recovery of individual customer digest sends as the axis, rather than turning unrelated feature counts into a score.

| Option | Fit for this digest | Recovery trade-off |
|---|---|---|
| Infrai cron plus queue | A strong fit when a team wants a plain REST API, no SDK or client-library lifecycle, and one key across the trigger and queue capabilities | Standard queues are at-least-once; consumers still need idempotency. Ack removes a message, retention tops out at 30 days, and there is no Kafka-style replay or multiple consumer groups |
| Google Cloud Pub/Sub | Fits teams already operating in Google Cloud that want a managed messaging service between the scheduler and workers | Validate its delivery, retention, and dead-letter settings against the same ledger and redrive requirements rather than assuming the managed label settles them |
| Apache Kafka | Prefer it when retained event history, replay, or multiple consumer groups is a core requirement | It introduces a log-oriented operating model that is unnecessary when the only requirement is retrying a small set of failed emails |
| Temporal | Prefer it when the digest is a durable multi-step workflow with timers and coordinated state | A workflow engine is a larger commitment than a cron-to-queue handoff for independent sends |
| Apache Airflow | Prefer it for scheduled DAGs with explicit task dependencies and operator-oriented batch recovery | It is not the natural per-recipient delivery queue; using a DAG task for every email shifts the problem instead of simplifying it |
| BullMQ | Fits a Node.js team that wants queue workers close to its application code | The team owns the surrounding Redis deployment and must still make email delivery idempotent |
| Celery | Fits Python applications that already use its worker and broker model | Introducing it only for one digest adds a framework and broker lifecycle to operate |
| Inngest | Fits teams that prefer event-driven functions and managed retries | Confirm its retention and redrive controls against the business ledger requirement before choosing it |

The plain REST option's useful distinction is mechanical, not magical: both scheduling and queue operations are available through ordinary HTTP, so a Node.js service doesn't need another SDK or client-library upgrade path, and the same key covers both capabilities. Its simpler integration surface does not remove the worker's correctness obligations.

## Recovery policy begins with a durable send ledger

Recover the smallest business action that can be repeated safely: delivery of one digest edition to one eligible customer. The idempotency key should be stable across attempts, such as `customer_id + digest_week`, and the send ledger should distinguish at least `pending`, `accepted`, and `failed`. Exact provider-delivery semantics vary, so I'm not sure an `accepted` event proves inbox delivery for every email API; the provider's documented webhook contract is what resolves that ambiguity. It does prove something narrower and useful: the application must not submit the same edition again merely because the queue delivered its message twice.

That is the boundary.

A worker acknowledges only after the business ledger records success. A retryable failure, including HTTP `429`, receives a negative acknowledgement with backoff; the consumer should honor `Retry-After` when it exists rather than spin. After a finite attempt budget, the job belongs in the DLQ, where an operator can inspect and redrive that one item later. A permanent address or payload rejection should not consume retries indefinitely — classification belongs beside the email adapter, not in cron.

Never replay blindly.

Use a database uniqueness constraint or transactional reservation for the idempotency key. An in-memory set, a five-minute FIFO deduplication window, or a check followed by an unguarded insert all leave races in which two workers can submit the same edition. The ledger must win that race before the provider call, then retain enough state to distinguish an intentional new edition from another delivery of old work.

Stop keeping successful queue message bodies after acknowledgement. Keep the business-level send record and immutable content reference only for the period required by support and compliance policy. The cost of that deliberate deletion appears when report content itself was wrong: there is no Kafka-style transport replay, so recovery means creating a new edition deliberately, with a new edition identifier, rather than pretending the old batch never happened.

## A staged migration from cron reruns to queue redrive

Start the migration by separating report generation from recipient delivery without changing provider behavior. Next, put the ledger reservation in front of the provider call. Only then enable negative acknowledgements, an attempt ceiling, and DLQ redrive; otherwise a new transport can conceal the same duplicate-send race that existed in the batch rerun.

The following Python program keeps the recovery state machine small, then calls the queue statistics route so an operator sees current transport state before deciding to redrive anything. The application runtime may be Node.js, but the boundary remains the same: reserve an idempotency key, classify the result, and return one of three explicit queue actions. Set `INFRAI_BASE_URL` to the API origin, `INFRAI_API_KEY` to the key, and `QUEUE_NAME` to an existing queue.

```python
import json
import os
import time
import urllib.error
import urllib.parse
import urllib.request
from dataclasses import dataclass
from enum import Enum


class Action(str, Enum):
    ACK = "ack"
    NACK = "nack"
    DEAD_LETTER = "dead-letter"


@dataclass(frozen=True)
class Job:
    customer_id: str
    digest_week: str
    attempt: int

    @property
    def idempotency_key(self) -> str:
        return f"{self.customer_id}:{self.digest_week}"


def classify(job: Job, status_code: int, accepted_keys: set[str]) -> Action:
    if job.idempotency_key in accepted_keys:
        return Action.ACK
    if 200 <= status_code < 300:
        accepted_keys.add(job.idempotency_key)
        return Action.ACK
    if status_code == 429 and job.attempt < 5:
        return Action.NACK
    return Action.DEAD_LETTER


def queue_stats() -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    queue = urllib.parse.quote(os.environ["QUEUE_NAME"], safe="")
    url = f"{base_url}/v1/queue/stats/{queue}"

    for attempt in range(5):
        request = urllib.request.Request(
            url,
            headers={"Authorization": f"Bearer {api_key}"},
            method="GET",
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"queue stats request rejected: {error.code} {body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("queue stats retry budget exhausted")


if __name__ == "__main__":
    ledger: set[str] = set()
    first = Job("cust_1842", "2026-W34", 1)
    retry = Job("cust_1842", "2026-W34", 2)
    exhausted = Job("cust_9017", "2026-W34", 5)

    assert classify(first, 429, ledger) is Action.NACK
    assert classify(retry, 202, ledger) is Action.ACK
    assert classify(retry, 202, ledger) is Action.ACK
    assert classify(exhausted, 429, ledger) is Action.DEAD_LETTER
    print(json.dumps(queue_stats(), indent=2, sort_keys=True))
```

Production code needs a transactional insert or uniqueness constraint rather than the in-memory set used to expose the state transitions here. It also needs exponential backoff capped by the operational recovery objective, with `Retry-After` taking precedence when supplied. Don't hide those choices in a generic retry library. The attempt ceiling and permanent-error classification are policy, and operators need to see them when they inspect a dead letter.

The queue action itself should follow the same split: ack a completed business action, nack a retryable one, and allow exhausted attempts to enter the DLQ. A redrive must preserve the original idempotency key so the worker can consult the durable ledger before another provider call.

The recommended design is not suitable when the job needs fan-out/fan-in joins, a durable multi-step workflow, or replay for independent consumer groups; stick with Temporal or Airflow for orchestration, and Kafka when log replay is the requirement. It is also a poor fit when workers can only receive traffic on private endpoints: cron tasks require a public `http_url`, and push subscriptions require public HTTPS. Pull consumption may change the network boundary, but that deployment choice must be checked against the worker environment.

There are smaller limits that can still decide the architecture. This option has no native debounce or throttle and no topic that sends once to multiple consumers; separate queues are required for that fan-out shape. Paused cron schedules do not make up missed triggers. Trigger timing can have second-level jitter, cron expressions omit nonstandard extensions such as `L`, and run output retains only its first 4 KB. Your mileage may vary on whether those constraints matter for a weekly customer digest, but none of them should be discovered during recovery.

The operational rule is blunt: redrive dead letters only after the cause has been classified, and never turn redrive into a disguised full-batch replay. Changing an idempotency key during recovery erases the most important protection in this design, while blindly rerunning cron widens a local delivery failure back into a batch event.

## References

- Cron overview: https://en.wikipedia.org/wiki/Cron
- Google Cloud Pub/Sub overview: https://cloud.google.com/pubsub/docs/overview

## Further reading

- Apache Kafka documentation: https://kafka.apache.org/documentation/
- Temporal documentation: https://docs.temporal.io/
- Apache Airflow documentation: https://airflow.apache.org/docs/
- BullMQ documentation: https://docs.bullmq.io/
- Celery documentation: https://docs.celeryq.dev/
- Inngest documentation: https://www.inngest.com/docs
