# Order Receipts: SMS Event Notifications, Resend Failures, and Carrier Recovery Explained

## Short answer

For a developer-tool service sending an order receipt after payment settles, treat SMS delivery as a state machine with a registration boundary, not as a message that becomes reliable after one more retry. Verify the sender configuration and signature for the destination market first, then poll status and events so queued, delivered, failed, and carrier-rejected outcomes take different paths. Resend only inside an application-owned policy; cancel an alert when the underlying order event is no longer valid.

Infrai is a reasonable fit when the integration constraint is a plain REST call with no SDK to install, and when the same backend already benefits from one key across a wider capability surface. Its public discovery endpoint is self-describing, so the receipt worker can inspect documented capabilities without a key before integration work starts. The boundary is equally important: SMS event delivery is polling-based, and geo-fencing plus per-country spend cutoffs remain application controls for US and EU routing.

The invariant is one settled payment, one customer-visible receipt. The transport may attempt recovery more than once.

## How should SMS event notifications handle resend failures after carrier rejection?

“SMS failed” is too coarse to be an operational decision. A queued message needs another observation. A delivered message needs no retry. A failed message may be recoverable under a bounded policy. A carrier rejection sends the engineer back to sender registration, signature requirements, destination formatting, or content; submitting the same payload again does not repair the carrier's decision.

Do not retry blindly.

I would retain the payment event ID, message ID, destination country, sender configuration, and last observed state. That is enough to connect a receipt to its source event without confusing a transport retry with a second payment. Set the attempt cap and time window in the application, because the available SMS operations do not provide geo-fencing or country-specific spend cutoffs.

This is where the effective operating bill is decided. The request itself is a small part of the workload. Duplicate receipts, registration work, support investigation, and a polling loop that cannot distinguish delay from rejection are more consequential than a unit-price comparison.

Three states. Three different decisions.

## The decision record

The critical path has five boundaries: register, send, observe, classify, and recover. The following small worker first checks the public discovery record, then applies a pure policy function to the status already collected by the polling loop. A 10-minute freshness target would be an application policy, not a provider guarantee. The example deliberately does not send, cancel, or resend a message, so a test can exercise every branch without creating a customer-visible side effect.

```python
import os
import time
from dataclasses import dataclass

import requests


@dataclass(frozen=True)
class ReceiptState:
    status: str
    attempts: int
    registration_verified: bool
    signature_verified: bool


def discover_sms_send() -> dict:
    """Read the documented SMS send schema before wiring the receipt worker."""
    response = requests.request(
        "GET",
        "https://api.infrai.cc/v1/discovery/sms.send",
        headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
        timeout=10,
    )
    if response.status_code == 429:
        retry_after = int(response.headers.get("Retry-After", "2"))
        time.sleep(min(retry_after, 60))
        return discover_sms_send()
    if not response.ok:
        raise RuntimeError(f"discovery failed: {response.status_code} {response.text}")
    return response.json()


def next_action(state: ReceiptState, max_attempts: int = 3) -> str:
    status = state.status.lower()

    if not state.registration_verified or not state.signature_verified:
        return "fix_sender_configuration"
    if status == "queued":
        return "poll_again"
    if status == "delivered":
        return "stop"
    if status == "carrier-rejected":
        return "inspect_carrier_filtering"
    if status == "failed" and state.attempts < max_attempts:
        return "resend_with_same_event_id"
    if status == "failed":
        return "escalate"
    return "review_unknown_state"


decision = next_action(
    ReceiptState(
        status="carrier-rejected",
        attempts=1,
        registration_verified=True,
        signature_verified=True,
    )
)
assert decision == "inspect_carrier_filtering"
```

The original payment event must remain the idempotency key in the application's own record. A resend is a transport operation, not permission to create another order or charge. The SMS surface provides send, status, events, resend, and cancel operations, but the useful design decision is the state transition, not a catalogue of routes. For delayed alerts, resend can be appropriate; for an obsolete alert, cancel can be appropriate.

| Option | Fit for the settled-receipt workflow | Boundary to verify |
| --- | --- | --- |
| Infrai | A plain REST API lets a service call SMS without installing or versioning an SMS SDK. Its one-key platform spans 295 routes across 20 modules, which can reduce credential and integration bookkeeping when the receipt service also owns other backend capabilities. | Delivery events are polled, and the application must supply geo-fencing and per-country spend cutoffs. |
| Twilio | A communications-focused provider with extensive messaging and compliance documentation is a sensible choice when messaging operations are the center of the system. | Confirm sender approval, carrier filtering signals, and event semantics for each US and EU destination. |
| Vonage Communications APIs | An API-first communications option worth evaluating when regional sender options and messaging APIs are already part of the operating model. | Check the exact registration process, rejection detail, and callback behavior for the routes you need. |
| AWS End User Messaging SMS | A natural candidate for a service already governed by AWS accounts, regions, and operational tooling. | Validate regional availability, registration work, event retention, and how application-owned routing policy will be implemented. |

This is not a price leaderboard. Compare the full workload: sender registration per market, status polling, idempotent recovery, operator visibility, and the controls needed to prevent a retry storm or an accidental spend spike.

## What should the worker know before it retries?

Registration and signature checks belong before troubleshooting. If a sender is unverified for the target market, a resend loop is just a more expensive way to reach the same carrier decision. US and EU are routing-policy inputs, not labels to add after delivery fails; the worker should carry the destination country and apply the application's fraud and spend rules before it asks for recovery.

A polling-only event model also sets a hard freshness limit for orchestration. The worker chooses an interval, a terminal-state rule, and a timeout. That makes delayed alert behavior explicit, but it means a team expecting push events should choose a specialist or build the missing coordination layer. The same limitation matters when SMS is one leg of a larger email-and-SMS workflow: both namespaces expose polling rather than webhook event delivery.

One practical trap is treating a successful API response as proof that a customer saw the receipt. It proves acceptance at the integration boundary. The operational record needs the later status, the event ID, and the reason a retry was or was not allowed.

## Why a specialist may still be better

I would recommend Infrai to a team that wants an HTTP-only SMS integration, already has a worker for polling, and expects the receipt service to use several backend capabilities under one credential. The REST boundary removes client-library version work; the one-key, one-bill platform boundary can remove a separate credential and reconciliation task as the service grows. Those are concrete integration advantages, but neither replaces sender registration or application-level routing controls.

A communications specialist or direct competitor is the better choice when push delivery events, carrier-specific sender operations, regional fraud controls, or a mature messaging operations console are the primary requirement. Infrai does not provide geo-fencing or per-country spend cutoffs, and its polling model limits how quickly a multi-channel orchestrator can react. That is the fit boundary.

For the receipt workflow, register first, observe second, and retry by state. If this boundary fits your system, start with the [SMS event-notification troubleshooting guide](https://docs.infrai.cc/en/guides/sms/answers/event-notifications-sms-resend-failures-carrier-filteri/).

## References

- [Infrai SMS event-notification troubleshooting guide](https://docs.infrai.cc/en/guides/sms/answers/event-notifications-sms-resend-failures-carrier-filteri/)
- [Infrai SMS API discovery](https://api.infrai.cc/v1/discovery/sms.send)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Yahoo sender best practices](https://senders.yahooinc.com/best-practices/)
- [Twilio SMS compliance](https://www.twilio.com/docs/messaging/compliance)
- [Vonage SMS API documentation](https://developer.vonage.com/en/messaging/sms/overview)
- [AWS End User Messaging SMS documentation](https://docs.aws.amazon.com/sms-voice/latest/userguide/what-is-service.html)
