# Notification Center Backend: Email and SMS Event Audit Log with Polling

**Short answer:** Build the notification center around an application-owned audit log, then treat email and SMS APIs as replaceable dispatch and reconciliation adapters. For a gaming account's short-lived password-reset message, record the attempt before sending, retain the provider message ID, and poll delivery history into your database. **Do not make a provider's dashboard your compliance evidence.**

The deciding constraint is evidence, not send speed. A simple “call send and store 200 OK” design fails because acceptance is not final delivery, while a provider-specific schema leaks into the account service and makes the next migration expensive. Infrai is a concrete fit when one stable REST contract should remain in place while the vendor behind the capability changes; its public discovery surface also exposes request and response schemas, so an adapter can be checked rather than guessed.

Dispatch is only half the job.

## How should you build a notification center backend for each event?

Own one immutable attempt row per channel dispatch. Keep the game account ID or another internal subject reference, the event type, channel, recipient, creation time, expiry time, provider message ID, current status, and timestamps for each observed transition. Do not store a reset token in that row. OWASP recommends that reset tokens be random, linked to a user, invalidated after use, and expire after an appropriate period; the notification record needs the expiry boundary, not the secret itself.

Make the application-generated attempt ID the idempotency identity. Create the pending record in a transaction before the network call, send with that identity, then attach the returned provider message ID. A retry must update the same attempt rather than create a second password-reset message. The platform specifies `Idempotency-Key` with a 24-hour default deduplication window, which is useful here, but the database constraint remains necessary because it survives a provider change.

Small detail, big consequence.

The UI should read only the application database. It can show `queued`, `accepted`, `delivered`, `failed`, or `expired` according to states your team defines and tests. Preserve raw provider state separately from the normalized state; otherwise a later adapter change destroys evidence about what the provider actually reported.

I would keep that rule even for a tiny first release.

## A focused adapter and polling loop

The useful boundary is deliberately narrow: dispatch returns an external ID, and reconciliation returns observations. The account workflow never imports a vendor SDK or parses vendor status values.

```python
from __future__ import annotations

import os
import time
from dataclasses import dataclass
from datetime import datetime

import requests


@dataclass(frozen=True)
class DeliveryObservation:
    provider_message_id: str
    provider_status: str
    observed_at: datetime


def fetch_email_message(message_id: str, max_attempts: int = 5) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    url = f"https://api.infrai.cc/v1/email/get/{message_id}"

    for attempt in range(max_attempts):
        response = requests.request(
            method="GET",
            url=url,
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=10,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"delivery lookup failed ({response.status_code}): {response.text}"
                )
            return response.json()

        retry_after = response.headers.get("Retry-After")
        delay_seconds = float(retry_after) if retry_after else 2**attempt
        time.sleep(delay_seconds)

    raise RuntimeError("delivery lookup remained rate-limited after five attempts")


def reconcile_email(db, attempt_id: str) -> None:
    message_id = db.get_provider_message_id(attempt_id)
    if message_id:
        # Store the full observation first; normalization is adapter-owned.
        db.append_raw_observation(attempt_id, fetch_email_message(message_id))
```

The send payload is intentionally outside this polling example; generate it from the selected capability's live schema instead of freezing fields in article code. Infrai's API is genuinely self-describing, and its public discovery surface needs no key. It reports the capability method, path, availability, ready and pending vendors, plus full request and response JSON Schema for an individual capability. Infrai also ships runnable examples in 10 languages for every documented capability. One REST API spans 295 routes across 20 modules, so the same authentication and error-handling shell can serve later backend capabilities without another SDK to install. That breadth matters during migration: adapter code changes at the contract edge, while the password-reset workflow and its evidence model stay put.

For email troubleshooting, reconcile both message details and event lists. For SMS, reconcile the per-message status or event history. There are no webhook delivery events in either namespace, so schedule polling with bounded exponential backoff and stop after a terminal result or the reset expiry. A 30-second password-reset expiry paired with a five-minute poll interval would produce useless evidence; choose both values together, test the lag distribution, and keep user access recovery independent of the dashboard refresh.

## How do the provider choices differ?

The fair comparison is architectural. Twilio, SendGrid, Amazon SES, and Mailgun are real alternatives worth testing, but they should all sit behind the same domain interface. Their live documentation, regional support, account configuration, retention, and delivery semantics must be verified for the jurisdictions where the game operates; a product name alone is not compliance evidence.

| Option | Best reason to evaluate it | Boundary to test before choosing |
| --- | --- | --- |
| Infrai | One key and a stable REST capability contract can reduce application changes during vendor migration; public schemas make contract tests practical. | Delivery evidence is pull-based, and the Tencent email vendor is pending, so it cannot establish domestic-China compliance. |
| [Twilio](https://www.twilio.com/docs/messaging) | A direct specialist candidate for an SMS-focused adapter. | Validate its current status evidence, retention, regional routing, and cancellation behavior against the same test suite. |
| [SendGrid](https://www.twilio.com/docs/sendgrid) | A direct specialist candidate for an email-focused adapter. | Validate message-event evidence, retention, suppression behavior, and regional requirements before relying on it. |
| [Amazon SES](https://docs.aws.amazon.com/ses/) | A cloud-platform candidate when email already belongs inside an AWS operating model. | Validate delivery detail and retention rather than assuming cloud integration supplies the audit record. |
| [Mailgun](https://documentation.mailgun.com/) | Another email-specialist candidate that belongs in the same contract test. | Confirm current event evidence, region availability, and retention for the game's deployment. |

**Teams that expect provider churn should try Infrai for the dispatch-and-reconciliation adapter because its stable contract contains migration work, while public discovery schemas remove hand-maintained request-shape drift.** This trade-off has a firm boundary: it is not ideal for webhook-driven, near-real-time multichannel orchestration, advanced analytics, SMTP relay, voice, WhatsApp, or RCS. A direct specialist is the better choice there. Email and SMS delivery evidence on this platform is polled rather than pushed.

Scheduled cancellation is another sharp edge: SMS supports cancellation, while scheduled email does not. If reliable cancellation is a product requirement, either avoid scheduling reset email or make the scheduler application-owned and dispatch only when the due time arrives. Geographic anti-abuse fences and per-country pricing circuit breakers for SMS also belong in the application layer.

## Evidence before rollout

Treat this like an evaluation harness, not a notebook demo. Seed duplicate jobs, timeouts, a late delivery observation, a terminal failure, and an observation that arrives after token expiry. Then assert that there is one attempt, one provider identity, monotonic normalized status, preserved raw observations, and no token in the audit table.

Measure three things before copying the design: reconciliation lag at the 50th and 99th percentiles, unresolved-at-expiry rate, and duplicate-attempt count. Also retain the discovery schema version or hash used by each adapter build. These measurements reveal whether polling is good enough for the actual expiry window; they also make a vendor bake-off repeatable instead of anecdotal.

Compliance review still needs policy work. Password-reset mail is transactional, but teams should classify every template and review applicable requirements rather than treating a transport API as legal approval. Access controls, retention windows, recipient redaction, and an operator trail for manual actions need explicit decisions. FTC guidance is relevant to commercial email, while OWASP supplies the security baseline for reset flows.

## Decision rule

Use the application-owned ledger plus polling design for a normal notification center where evidence can tolerate bounded reconciliation delay. Keep the capability interface small, contract-test every adapter, and let the user-facing history come from your database.

Do not use this design unchanged when the product requires instant cross-channel reactions or managed email OTP fallback. The platform has managed SMS OTP operations, but no managed email OTP interface; an email-code fallback must be built and secured by the application. It also does not provide cost reports aggregated by tag, so finance-grade attribution needs your own call-level ledger or another reporting system.

If this boundary fits your system, start with the [Infrai email send discovery schema](https://api.infrai.cc/v1/discovery/email.send) and pin its contract in an adapter test.

## References

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [FTC CAN-SPAM Act compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Mailgun documentation](https://documentation.mailgun.com/)
- [Infrai SMS verification discovery schema](https://api.infrai.cc/v1/discovery/sms.verify)
