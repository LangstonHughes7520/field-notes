# Healthtech SMS Alert Service Alternative: Compare Sender Evidence for Password Resets

TL;DR: For a startup sending short-lived password-reset messages in the US and EU, choose an SMS service by the evidence it can return, not by the smallest advertised per-message number. Model the bill as submitted message segments plus sender-registration charges, then add the less visible cost of collecting, normalizing, and retaining delivery receipts. Keep a provider-neutral message ID, timestamps, destination region, segment count, sender identity, status transitions, and policy version; do not keep the reset token or rendered message in the audit record. Poll only while a reset can still be useful, record every transition idempotently, and expire detailed evidence according to a written retention rule.

The dominant variable term is usually straightforward to compare without pretending that one public unit price settles the matter:

`monthly transport units = sum(message segments submitted by destination and sender type)`

That formulation matters because SMS character encoding can turn one visible message into multiple segments. A compact template that stays within one segment can move the largest variable term more reliably than shaving a small amount from a quoted rate. Registration, recurring sender charges, receipt polling, database writes, and evidence retention belong on the same worksheet, but they should remain separate rows; blending them into a single “per SMS” estimate hides which assumption changed.

## How should a startup app compare SMS alert service alternatives?

A delivery receipt does not prove that a patient read a password-reset message, that the handset was in the intended person's possession, or that the reset was secure. It proves a narrower event reported by the delivery chain. Treating `delivered` as identity verification is a category error.

For each attempt, the durable record should answer six questions: what internal event requested the send, which approved template and policy versions were used, which sender identity was selected, when the service accepted the request, which status transitions were observed, and when the reset stopped being actionable. The destination can be stored as a keyed digest when later operations do not require the raw phone number. The token and message body should stay out of this record.

This produces a useful boundary. The transactional store owns the short-lived reset challenge; the evidence store owns an append-only account of state changes. A lookup key connects them without copying the secret. If an investigator later needs to reconstruct the decision, the evidence shows that the application requested a particular template under a particular policy and observed a provider-reported outcome. It cannot overclaim human receipt.

The storage trade-off is blunt: detailed status history improves reconstruction but increases the amount of sensitive metadata under management. Retaining only a terminal status is smaller, yet it erases timing and retry behavior that may explain a complaint. A defensible compromise is to retain immutable transitions for a policy-defined interval, restrict access, then reduce them to coarse aggregates that cannot be traced back to one reset attempt.

## Count transport and retention before comparing services

There is no durable “cheapest” answer without a workload. Build the comparison from the same synthetic month for every candidate: US and EU destination counts, the exact reset template, expected encoding, segments per message, sender types, registration charges, and the proportion of attempts that will be polled more than once. Use current candidate documentation and quotes for prices rather than copying a static figure into an architecture decision.

| Cost or constraint | Unit to measure | Failure hidden by a low headline rate |
|---|---:|---|
| Message transport | Submitted segments by destination | A template change creates multipart messages |
| Sender setup | Registrations and recurring sender identities | A test sender cannot support the production route |
| Receipt collection | Poll requests, worker time, and writes | Aggressive polling costs more and still observes late states |
| Evidence storage | Transition rows multiplied by retention time | Raw bodies or numbers enlarge the compliance boundary |
| Operations | Alerts, reconciliation, and support review | Accepted requests are mistaken for delivered messages |

Do the segment test with the final text, including the URL. GSM-7 and UCS-2 have different single-message and concatenated-message limits, so one substituted character can change the number of billable segments. The correct input to a cost model is the encoded, rendered template, not its character count in an editor.

Price is still only one axis. Sender registration requirements differ by route and sender type, and the evidence needed during registration affects launch time and ownership. Ask each candidate to document the applicable process for the exact US and EU routes in scope. Do not infer production eligibility from a successful development send.

I would stop retaining rendered message text, reset tokens, raw provider payloads after normalization, and per-attempt transitions beyond the approved evidence window. That reduces storage and exposure. The cost is reduced forensic detail: after compaction, an investigation can establish aggregate outcomes and the normalized terminal state, but it cannot replay every provider-specific field or reconstruct the original message byte for byte. That loss should be explicit in the retention decision, not discovered during an audit.

## Poll receipts through a narrow adapter

Polling is reasonable for a small startup workload when inbound webhooks would create a larger public verification surface, but the application should not scatter provider status names across business logic. A narrow adapter returns a normalized observation, while an append-only store makes repeated polls harmless.

The example below deliberately excludes the phone number, token, and message body. It also stops at a deadline supplied by the reset workflow; a terminal delivery event after that deadline may matter to aggregate carrier analysis, but it cannot make an expired reset valid again.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from enum import Enum
from typing import Protocol


class DeliveryState(str, Enum):
    ACCEPTED = "accepted"
    IN_TRANSIT = "in_transit"
    DELIVERED = "delivered"
    FAILED = "failed"


TERMINAL = {DeliveryState.DELIVERED, DeliveryState.FAILED}


@dataclass(frozen=True)
class Observation:
    internal_attempt_id: str
    provider_message_id: str
    state: DeliveryState
    observed_at: datetime


class ReceiptClient(Protocol):
    def fetch(self, provider_message_id: str) -> DeliveryState: ...


class EvidenceStore(Protocol):
    def append_if_new(self, observation: Observation) -> None: ...


def poll_once(
    client: ReceiptClient,
    store: EvidenceStore,
    *,
    internal_attempt_id: str,
    provider_message_id: str,
    reset_expires_at: datetime,
) -> bool:
    now = datetime.now(timezone.utc)
    if now >= reset_expires_at:
        return False

    state = client.fetch(provider_message_id)
    store.append_if_new(
        Observation(
            internal_attempt_id=internal_attempt_id,
            provider_message_id=provider_message_id,
            state=state,
            observed_at=now,
        )
    )
    return state not in TERMINAL
```

`append_if_new` needs a uniqueness rule over the attempt, normalized state, and provider observation identity or timestamp. Without one, a retrying worker manufactures a misleading history and inflates storage. The scheduler should apply bounded backoff and jitter, but its exact intervals belong in configuration derived from the reset expiry and the candidate's published rate limits. Inventing universal polling intervals would be false precision.

Keep the raw response briefly only if it is required to diagnose adapter changes, access-control it as sensitive operational data, and delete it on a separate schedule. Normalization must preserve unknown states rather than silently mapping them to failure. Unknown means the adapter needs attention.

## Failure modes decide the shortlist

Start with ambiguity. An HTTP success from a send request may establish acceptance by the service, not handset delivery. A missing receipt may mean the message is still progressing, the polling request failed, the identifier was lost, or the delivery chain never produced a terminal event. Those cases need distinct internal outcomes.

Then test expiry races. A message can arrive after its reset token expires; the reset endpoint must reject the token based on server-side expiry regardless of the delivery state. Retrying the send can also produce two messages, so the request path needs an idempotency strategy and the user experience must make clear which challenge remains valid.

Encoding drift is quieter. A localized template, typographic punctuation, or a long link can change encoding or segmentation. Put rendered templates through an encoding-and-segment test during deployment, and record the template version with each attempt. This catches a cost and deliverability change before it becomes an unexplained bill movement.

Finally, rehearse evidence gaps: polling credentials revoked, status vocabulary changed, storage unavailable, and a worker restarted between fetching and persisting a receipt. The safe response is not to claim failure or delivery. Record an observation gap, retry within the useful window, and reconcile accepted message IDs against stored transitions. Alert on age and missing terminal observations, not merely on transport exceptions.

A candidate belongs on the shortlist only if its documentation and a controlled test can answer the following without sales-language interpretation:

| Decision test | Evidence to collect | Reject or escalate when |
|---|---|---|
| Exact sender path for each destination | Registration steps and ownership records | Production sender eligibility is unclear |
| Segment accounting | Encoded template and reported segment count | Counts cannot be reconciled |
| Receipt semantics | Documented states plus observed transitions | Acceptance and delivery are conflated |
| Polling behavior | Authentication, limits, and message lookup behavior | Useful receipts cannot be retrieved before expiry |
| Data handling | Fields, regions, deletion controls, and retention terms | The compliance boundary cannot be documented |
| Export and reconciliation | Stable message IDs and timestamped status history | Evidence cannot be joined to an internal attempt |

Run the same test corpus against at least three viable candidates if the organization chooses to compare products. The result should be a boundary comparison, not a ranking: one may have simpler registration for a particular sender type, another may expose richer receipt transitions, and another may fit an existing operational environment. Those observations are only valid for the tested routes and current terms.

The final decision rule is simple: select the service whose applicable sender path is documented, whose segment counts reconcile, whose receipts can be normalized before reset expiry, and whose data handling supports the approved evidence policy. Break a tie with the modeled total cost for the same workload. Do not reverse that order merely because a landing page shows a smaller unit number.

## Further reading

References:

- Twilio, “What Is the SMS Character Limit?”: https://www.twilio.com/docs/glossary/what-sms-character-limit
- Amazon Simple Email Service documentation, useful for contrasting email transport with the SMS channel discussed here: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
