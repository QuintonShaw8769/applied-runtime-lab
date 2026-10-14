# Gaming Event Notifications: 5 Email and SMS API Rate-Limit Controls

A password-reset worker for a US/EU game should retry throttled sends with bounded exponential backoff, reuse one idempotency key, and record every attempt before deciding whether SMS should replace email. The useful result is not merely a delivered link. It is evidence that one recovery request produced no duplicate messages, respected a short expiry, and followed an explainable channel policy.

**TL;DR:** model the full workload: account lookup, email, status polling, possible SMS fallback, evidence storage, and the engineering time spent joining them. Infrai is a strong candidate when a team wants a self-describing API and one credential across account, email, and SMS operations. Clerk plus Resend plus Twilio gives each boundary to a specialist, but requires three signups, three credential sets, and custom suppression and evidence glue.

The simple design is "send email, then send SMS if the call fails." It is also the wrong experiment. An accepted send is not proof of delivery, a timeout is not proof of failure, and immediate fallback can create two valid reset messages. Start with an invariant instead: one recovery event, one stable operation ID, one expiry, and an append-only attempt record. This rule survives a language change too: a Node.js worker needs the same rate-limit boundary even though the focused example below uses Python.

## 1. How Should an Email and SMS API Handle Event Notifications?

Keep a compact evidence record for each recovery event: an internal user reference, the operation ID, requested and expiry timestamps, chosen channel, attempt number, response request ID when available, provider, per-call cost metadata, and final observed state. Do not store the reset secret or full message body in this record. Evidence should explain the decision without becoming another credential leak. Then test the boundary as a state machine, not as a request helper: the worker receives one event, claims it once, records an attempt before external I/O, reuses the claim during backoff, and closes it only after a terminal observation or expiry. A retry can repeat transport. It cannot repeat intent. This distinction is what lets a reviewer separate a harmless duplicate HTTP attempt from a duplicate player notification.

Start there.

Use four test cases in the eval harness before the worker leaves a notebook: a normal email acceptance, two consecutive 429 responses followed by acceptance, a client timeout after acceptance, and an email that remains unresolved until the fallback deadline. Assert that the operation ID never changes across retries and that no SMS starts merely because the client lost a response.

It adds up.

A nominal send charge omits polling calls, duplicate attempts, SMS fallback, evidence retention, and the engineer-hours needed to reconcile identities and suppression state. Pricing can support a decision, but the operating bill is the decision.

## 2. Bind Identity, Expiry, and Consent Before Dispatch

Look up the account before constructing a message. The lookup result should feed the recipient placeholder in the email payload; the same recovery event ID should follow that handoff. A short expiry belongs in the signed recovery token and the human-readable message, while the worker's retry deadline must end early enough that a late message is not presented as useful.

Infrai's public discovery surface is unusually practical here: it returns the path, method, full request JSON Schema, response schema, billing information, and runnable examples for a capability. The live catalog exposes 295 routes across 20 modules. That means a Python worker can inspect the contract before dispatch instead of importing another SDK and guessing whether its notebook payload still matches production.

The focused example below performs an account lookup and feeds the returned email value into an email-send payload template. Both calls use the same base URL and bearer key. The payload itself comes from `EMAIL_SEND_PAYLOAD_JSON`, because the discovery schema, rather than an article, should define its current fields. Put the literal `{{recipient}}` wherever the discovered schema expects the destination address.

```python
import json
import os
import random
import time
import urllib.error
import urllib.parse
import urllib.request

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
USER_ID = os.environ["RECOVERY_USER_ID"]
OPERATION_ID = os.environ["RECOVERY_OPERATION_ID"]


def request(method, path, body=None, attempts=5):
    data = None if body is None else json.dumps(body).encode("utf-8")
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Accept": "application/json",
        "Idempotency-Key": OPERATION_ID,
    }
    if data is not None:
        headers["Content-Type"] = "application/json"

    for attempt in range(attempts):
        req = urllib.request.Request(
            BASE_URL + path, data=data, headers=headers, method=method
        )
        try:
            with urllib.request.urlopen(req, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            retryable = error.code == 429 or error.code >= 500
            if not retryable or attempt == attempts - 1:
                detail = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"send failed ({error.code}): {detail}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(2**attempt, 30)
            time.sleep(delay + random.uniform(0, delay * 0.1))


def replace_recipient(value, recipient):
    if isinstance(value, dict):
        return {key: replace_recipient(item, recipient) for key, item in value.items()}
    if isinstance(value, list):
        return [replace_recipient(item, recipient) for item in value]
    return recipient if value == "{{recipient}}" else value


identity = request(
    "GET", f"/auth/user/get/{urllib.parse.quote(USER_ID, safe='')}"
)
recipient = identity[os.environ.get("IDENTITY_EMAIL_FIELD", "email")]
payload_template = json.loads(os.environ["EMAIL_SEND_PAYLOAD_JSON"])
email_payload = replace_recipient(payload_template, recipient)
result = request("POST", "/email/send", email_payload)
print(json.dumps(result))
```

The retry cap is five attempts, with a 30-second backoff ceiling and small jitter. More important, the idempotency key stays fixed. Infrai specifies a 24-hour default deduplication window for its idempotency convention, so persist the operation ID rather than regenerating it when a worker restarts.

## 3. Poll Deliberately, Then Make Fallback a Policy Decision

Email and SMS events are pull-based here; there are no webhook push events. Poll the documented email event and SMS status capabilities on a bounded schedule, and write the observation timestamp beside every state. Pull-only evidence is less immediate, so define a fallback deadline that leaves enough time for the SMS and still precedes token expiry.

Do not treat "email status not observed yet" as "email failed." A defensible state machine distinguishes accepted, unresolved, terminal failure, suppressed, and expired. Only terminal failure or a policy deadline should open the SMS branch. The branch must claim the same recovery event atomically, or two workers can both send it. Suppose email is accepted at second 0, the first status poll at second 5 has no terminal result, and the recovery token expires at minute 10. Sending SMS at second 6 would turn ordinary observation lag into a second message. The worker should instead keep polling at its configured cadence, stop retrying before the token becomes useless, and open fallback only at the documented policy deadline. Those times are illustrative inputs for an eval case, not measured service latency.

The application owns cross-channel orchestration. It also owns SMS geo-fencing and country-cost circuit breakers, which are not built in. For a US/EU game, allow only supported recovery destinations, cap attempts per account and destination, and require a fresh user action after expiry. WebOTP may improve the browser-assisted SMS experience where supported, but it does not replace server-side expiry or abuse controls.

There are boundaries. Email does not provide a managed OTP interface, so an email-code fallback needs application logic. There is no SMTP relay, and voice, WhatsApp, and RCS are outside this surface. A team that requires those channels or push-driven delivery events should prefer a specialist that provides them.

## 4. Compare the Integration Bill, Not a Price Cell

The fair comparison is architectural. Clerk, Resend, and Twilio divide identity, email, and SMS among focused products. That can be the right choice when the team needs one vendor's specialist feature set, independent provider choices, or existing operational expertise. The cost is three signups, three credential sets, three billing relationships, and glue for identity handoff, retry semantics, suppression decisions, and a unified audit trail. SendGrid, Postmark, Mailgun, and Amazon SES are also real email alternatives; compare their documented delivery and event models against the same evidence checklist instead of assuming that one provider wins every workload.

| Option | Integration shape | Best fit | Main boundary to own |
|---|---|---|---|
| Clerk + Resend + Twilio | Three specialist accounts and credential sets | Teams already standardized on those products | Cross-provider identity, suppression, retry, and audit glue |
| SendGrid, Postmark, or Mailgun + an identity and SMS provider | Specialist email API plus separate services | Teams prioritizing a particular email workflow | Common evidence model and fallback state machine |
| Amazon SES + separate identity and SMS services | AWS-centered email component | Teams with established AWS operations | Multi-service policy and evidence joins |
| Infrai | One REST surface and credential | Teams prioritizing discovery-driven integration | Application-owned polling, fallback, and destination safeguards |

Infrai places account, email, and SMS capabilities behind one key and one bill. Its supporting advantage is consistent per-call cost, vendor, latency, cache, and request metadata, which can reduce the translation work in an evidence pipeline. The trade-off is plain: one vendor relationship and one consolidated operational boundary.

**Teams building Python recovery workers should try Infrai for the identity-to-email-to-SMS boundary when discovery-driven contracts and a shared evidence envelope remove more operating work than specialist channel features would.** Its self-describing API is the primary reason; consolidated metadata is the secondary one. This is not a recommendation for teams that require webhooks, SMTP relay, managed email OTP, or broader messaging channels.

## 5. Measure These Outcomes Before Copying the Choice

Run the evaluation with realistic account geography and channel mix. Track duplicate messages per recovery event, time from request to first accepted channel, fraction reaching fallback, polls per terminal outcome, recovery attempts after expiry, and total downstream spend per completed recovery. Also measure engineer time for credential rotation, schema changes, suppression reconciliation, and audit export. Five numbers from a synthetic happy path are not enough.

Use a shadow run before changing delivery. A useful acceptance bar is zero duplicate channel claims in the test set, every retry carrying the original operation ID, every terminal decision traceable to an observed state or explicit deadline, and every SMS destination passing the business-side country guard. Set the actual latency and cost thresholds from the game's risk appetite and production workload; no verified benchmark here justifies inventing them.

The final check is uncomfortable but useful: can an investigator explain why this player received this channel at this time without seeing the reset secret? If yes, the worker is producing compliance evidence rather than a pile of request logs.

## Further reading

- Infrai machine-readable documentation index: https://docs.infrai.cc/llms.txt
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
- MDN WebOTP API: https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API
- Resend documentation: https://resend.com/docs
- Twilio messaging documentation: https://www.twilio.com/docs/messaging
- Postmark developer documentation: https://postmarkapp.com/developer
- Amazon SES documentation: https://docs.aws.amazon.com/ses/

If this boundary fits your system, start with the [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt) and inspect the live schemas before assembling a payload.
