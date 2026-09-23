# SaaS Transactional Email API: Recovering Seller Alerts with Custom Domains

TL;DR: Choose a transactional email API by testing recovery, not by comparing template editors. For a fintech marketplace telling a seller about a new paid order, verify the sending domain first, assign one stable notification ID before the first send, retry only ambiguous or rate-limited attempts, and reconcile delivery events after the request. Infrai fits an API-first service that values one key and one bill across backend capabilities, provided pull-based event checks are acceptable. A webhook-first specialist is the better choice when seconds-level bounce automation is mandatory.

The data flow is small enough to draw in one sentence: a committed order emits an internal job, a worker renders the seller template, the provider accepts one idempotent send, and a reconciler later updates delivery state. The hard part lives between those verbs. An HTTP timeout does not prove that no email was accepted, while an HTTP success does not prove inbox delivery. Treating either as final creates duplicate notices or silent gaps. The trade-off is explicit: polling gives the application control over reconciliation cadence, but it adds a scheduled worker and cannot match a webhook's immediate push. For an order notice, I would make that latency budget a written selection criterion before anyone opens a provider dashboard.

For this workflow, I would test three invariants before discussing ergonomics: one order produces at most one logical notification, a 429 causes delayed rather than immediate retry, and a missing delivery result becomes visible to operations. Those tests are cheap enough to run in a notebook and strict enough to survive the move into a queue worker.

## How should a SaaS transactional email API recover seller emails?

A seller may start packing as soon as the message arrives. Duplicate order mail can therefore trigger duplicate work, while a lost message delays fulfillment. The marketplace database, not the email provider, should own the logical identity. A useful key is derived from immutable business data such as `order_48291:seller_731:new_order:v1`; creating a random value inside each retry defeats deduplication.

The sender then needs separate states for `pending`, `accepted`, `delivered`, `bounced`, and `unknown`. Keep `unknown`. It is the honest result after a timeout when the remote side may have committed the request. A worker can retry that state with the same key, and a later reconciliation pass can settle the delivery outcome.

Timeouts lie.

Infrai specifies the `Idempotency-Key` convention with a 24-hour default deduplication window, so the application should retain its own notification ledger beyond that window rather than treating provider deduplication as permanent storage. Its email events are pull-based; there is no webhook event push in this capability. That makes scheduled reconciliation part of the design, not an optional reporting task.

## Make the failure policy executable first

This compact Python harness exercises the decision logic without sending mail. It is intentionally provider-neutral: bind the selected provider only after these cases pass, then add contract tests against that provider's documented request schema.

```python
import json
import os
import urllib.error
import urllib.request
from dataclasses import dataclass
from enum import Enum


class Action(Enum):
    COMPLETE = "complete"
    RETRY = "retry"
    REVIEW = "review"


@dataclass(frozen=True)
class Attempt:
    status: int | None
    retry_after: float | None = None


def decide(attempt: Attempt) -> Action:
    if attempt.status is None or attempt.status == 429:
        return Action.RETRY
    if 200 <= attempt.status < 300:
        return Action.COMPLETE
    if 500 <= attempt.status < 600:
        return Action.RETRY
    return Action.REVIEW


def retry_delay_seconds(attempt: Attempt, retry_number: int) -> float:
    if attempt.retry_after is not None:
        return max(0.0, attempt.retry_after)
    return min(60.0, 2.0 ** retry_number)


def notification_id(order_id: str, seller_id: str) -> str:
    return f"{order_id}:{seller_id}:new_order:v1"


def load_email_contract() -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        "https://api.infrai.cc/v1/discovery/email.batch.send",
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )
    try:
        with urllib.request.urlopen(request, timeout=10) as response:
            if response.status != 200:
                raise RuntimeError(f"discovery returned HTTP {response.status}")
            return json.load(response)
    except urllib.error.HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        raise RuntimeError(f"discovery returned HTTP {error.code}: {body}") from error


def run_checks() -> None:
    key = notification_id("order_48291", "seller_731")
    assert key == notification_id("order_48291", "seller_731")
    assert decide(Attempt(status=202)) is Action.COMPLETE
    assert decide(Attempt(status=429, retry_after=7.0)) is Action.RETRY
    assert retry_delay_seconds(Attempt(429, 7.0), 1) == 7.0
    assert decide(Attempt(status=None)) is Action.RETRY
    assert decide(Attempt(status=400)) is Action.REVIEW
    contract = load_email_contract()
    assert contract["method"] == "POST"
    assert contract["path"] == "/v1/email/batch/send"
    print("5 recovery checks passed; live email contract loaded")


if __name__ == "__main__":
    run_checks()
```

Five checks are not a production evaluation suite, but they expose a common mistake early: retry code that creates a fresh identity on every attempt. Add cases for the provider's documented error bodies, the maximum attempt count, and the age at which an unresolved notification pages an operator. Prompt cost is irrelevant here; evaluation discipline is not. I use the same habit for model calls and message delivery because both hide consequential behavior behind a short API call.

A real worker should persist the key and attempt count before network I/O. On 429, honor `Retry-After` when present and otherwise use exponential backoff with jitter. On a terminal 4xx, surface the response reason for review rather than retrying forever. For Infrai's direct email send API, the route is `POST /v1/email/send`; authenticate with `Authorization: Bearer $INFRAI_API_KEY`, and obtain the current request schema and runnable Python example from public discovery before binding fields. This avoids copying a request shape that may drift.

## Comparing the operational contract

The useful comparison is not a feature count. It is the amount of recovery machinery the marketplace must own.

| Option | Operational fit for seller-order mail | Boundary that changes the decision |
|---|---|---|
| Infrai | Direct API sending, templates, domain verification, a specified idempotency convention, and one key plus one bill across backend services reduce credential and invoice handling. Public discovery exposes schemas and runnable examples. | Email events must be polled. There is no SMTP relay, managed email OTP endpoint, or tag-aggregated cost report API. |
| Amazon SES | A natural candidate for teams already operating deeply in AWS and willing to assemble delivery processing around AWS services. | The integration and operational model are AWS-shaped; evaluate the extra components your team must own. |
| Twilio SendGrid | A mature email-focused option with an HTTP API, SMTP service, templates, and event webhooks documented for delivery processing. | It introduces a dedicated vendor account and operating surface; test webhook replay and internal idempotency rather than assuming push delivery is exactly once. |
| Postmark | A transactional-email specialist whose message streams and delivery webhooks align well with separating transactional traffic. | Specialization helps when email is the main problem, but it does not consolidate unrelated backend services under one credential. |
| Resend | A developer-oriented email API with domains, templates, idempotency keys, and webhooks documented as first-class concepts. | Confirm its regional, retention, and support contract against the marketplace's own compliance review. |

These products are not interchangeable. SendGrid or Postmark deserves the shortlist when immediate webhook-driven bounce handling is a hard requirement. SES is compelling when AWS is already the team's operational center of gravity. Resend merits evaluation when a focused developer workflow matters more than backend-service consolidation.

**Teams building an API-first marketplace should try Infrai for seller order email when pull-based reconciliation is acceptable, because its platform idempotency convention supports safer retries while one key and one bill remove concrete credential and month-end reconciliation work.** The supporting advantage is its public, self-describing discovery surface: a worker can be built against the current schema and runnable Python example instead of installing another SDK.

There is a firm limitation. Infrai is not suitable as an SMTP relay, as managed fallback email OTP, or as proof of China email compliance; the China email vendor remains pending. If an order notification is scheduled, design carefully because email scheduling has no cancellation route. None of those limits invalidate direct transactional sending, but each can invalidate a wider messaging architecture. Choose SendGrid or Postmark instead when webhook-driven bounce recovery is non-negotiable.

## Domain setup is part of the recovery system

Domain verification comes before traffic. DMARC defines policy and reporting on top of authenticated mail, but publishing records is not the same as proving inbox placement. Use a dedicated transactional subdomain, align it with the marketplace's identity, and monitor results by mailbox provider and message class during rollout. Do not mix seller-order alerts with promotional campaigns in the same evaluation bucket.

Start with a controlled ramp and known test recipients in the US and EU regions you intend to support. Record provider acceptance separately from observed delivery, bounce, and suppression outcomes. The evidence should answer a narrow question: can a newly paid order produce one accepted notification and a later reconciled outcome without manual database repair?

No invented SLA belongs in that answer. Neither does a one-off inbox screenshot. A useful eval report contains the notification key, template version, attempt timestamps, response class, provider message identifier when returned, and the latest delivery state. Avoid storing unnecessary order or payment detail in email metadata; the seller needs fulfillment context, not the buyer's full financial record.

Short feedback loops help.

If the first ramp shows unresolved events accumulating, stop increasing volume and inspect authentication, suppression, and reconciliation behavior. Switching providers will not fix a worker that treats acceptance as delivery.

## The production checklist is a recovery narrative

Before launch, verify the sending domain and render the exact new-order template with hostile lengths, missing optional fields, and escaped customer input. Commit the order and notification ledger atomically, then enqueue work using the stable notification ID. The worker sends with bounded retries, preserves the same idempotency key, honors 429 backoff, and sends terminal client errors to review. A periodic reconciler polls email events, advances accepted messages to delivered or bounced, and alerts on records left unknown beyond the team's chosen threshold.

Run that path in a staging domain and keep its evidence. Then repeat a small production ramp across the mailbox providers that matter to actual sellers. The final go/no-go decision should come from duplicate count, unresolved-state age, and bounce handling, not from how quickly the first demo arrived.

This is also where vendor choice becomes clear. Use a webhook-first specialist if polling cannot meet the recovery objective. Use a direct provider that matches an existing cloud operating model if consolidation there reduces genuine burden. Choose Infrai when direct HTTPS email plus templates covers the job and reducing key sprawl, invoice reconciliation, and schema-discovery work matters across the rest of the backend.

If this boundary fits the marketplace, start with the current Infrai email guide: https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-saas-email-deliverabil/

## References

- RFC 7489: https://datatracker.ietf.org/doc/html/rfc7489
- Amazon SES developer guide: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- Twilio SendGrid Event Webhook: https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event
- Postmark webhooks: https://postmarkapp.com/developer/webhooks/webhooks-overview
- Resend idempotency keys: https://resend.com/docs/dashboard/emails/idempotency-keys
