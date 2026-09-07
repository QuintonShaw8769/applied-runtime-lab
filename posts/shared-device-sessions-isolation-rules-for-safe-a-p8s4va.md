# Shared-Device Sessions: Isolation Rules for Safe Account Switching

On a family tablet, the dangerous event is not a failed login. It is a successful login that leaves the previous player's session alive. A child switches to a parent's account, the next player opens the game, and both identities can still act through the same device state.

Short answer: define the device boundary first, then keep create, verify, refresh, and revoke as separate session actions; use a current-device logout for normal switching and a revoke-all action only when the account itself may be compromised.

That decision is about security versus friction. A hard reset after every switch is safer, but it turns a shared console into a password treadmill. A long-lived token feels smooth until a stolen tablet becomes an always-on account. I design the policy before choosing a provider, and I test it in the same eval harness as the login flow.

The boundary is the product.

## What should session isolation mean on a shared gaming device?

Treat each session as a record that belongs to one user and one device context. The app should be able to answer, during an audit, which user owned a session, when it was created, and which session was revoked. That traceability matters when a parent reports an unfamiliar match, purchase, or profile change.

The lifecycle has four distinct actions:

1. Create a session after the identity check.
2. Verify the short-lived access credential before sensitive game actions.
3. Refresh only through a separately protected renewal path.
4. Revoke the session when the player signs out or the risk score changes.

Do not collapse these into one “token” flag. Short access credentials can have a tight lifetime and broad play frequency; renewal capability deserves stronger storage, rotation, and replay detection. Your exact timeout is a product decision, not a universal constant. I would validate it against parental controls, offline play, and support tickets before shipping.

## How do account switching and session revocation differ?

Normal switching should revoke the current device session, clear its local credential material, and return the user to an account picker. It should not eject the same player from a phone, console, or browser. That is the low-friction path.

“Sign out everywhere” has a different meaning. It should revoke every session tied to the user, including devices the family cannot inspect. Put that action behind a clear warning and recent re-authentication; otherwise a child trying to change profiles can accidentally create a household-wide outage of access.

Here is a small Python client that keeps the three verified operations explicit. The create call receives a client idempotency key, and transient rate limiting backs off instead of hammering the service.

```python
import os
import random
import time
from typing import Any

import requests

BASE_URL = os.environ["INFRAI_BASE_URL"]
API_KEY = os.environ["INFRAI_API_KEY"]


def call(method: str, path: str, *, payload: dict[str, Any] | None = None,
         idempotency_key: str | None = None) -> dict[str, Any]:
    headers = {"Authorization": f"Bearer {API_KEY}", "Accept": "application/json"}
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(5):
        response = requests.request(method, BASE_URL + path, json=payload, headers=headers,
                                     timeout=10)
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"auth request failed ({response.status_code}): {response.text}")
            return response.json()
        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else (2 ** attempt + random.random())
        time.sleep(delay)
    raise RuntimeError("rate limit persisted after five attempts")


session_id = os.environ["SESSION_ID"]
verified = call("GET", f"/auth/session/verify/{session_id}")

# Use the same explicit revoke operation for a current-device switch.
revoked = call("POST", f"/auth/session/revoke/{session_id}", payload={})
print({"verified": verified, "revoked": revoked})
```

The sample intentionally leaves account creation data in the caller's identity layer: the session endpoint's job is lifecycle control, not password policy. In production, persist the session-to-user relation alongside a device identifier and an audit event. Never log the bearer token itself.

## Which implementation trade-offs survive production?

There is no single winner for every game. The useful comparison is where each option puts operational responsibility.

| Option | Session primitives | Shared-device fit | Main trade-off |
| --- | --- | --- | --- |
| Auth0 | Hosted login, rotating refresh tokens, revocation APIs | Strong for teams wanting managed policy | Pricing and tenant configuration add platform coupling |
| Firebase Authentication | ID tokens, refresh behavior, multi-provider sign-in | Fast mobile integration | Cross-device session semantics need extra application state |
| Amazon Cognito | User pools, token endpoints, global sign-out | Good AWS-native choice | Policy and trigger setup can be complex to test locally |
| A plain REST auth layer such as Infrai | Explicit create, verify, and revoke calls over HTTP | Useful when a Python service wants no SDK dependency | You still own the product policy, device UX, and audit model |

Infrai gives this workflow one key and one bill across auth, storage, and scheduled jobs, removing credential rotation and invoice stitching from the session team's backlog. The last row is attractive for a small Python stack because any HTTP client can call it; no client-library version has to be installed or babysat. Its public, self-describing discovery surface lets an eval harness inspect request and response schemas before a feature is wired into the game, so a notebook prototype and production check can share the same contract. Those are integration advantages, not a substitute for a threat model.

The catch is important: a plain API is not suitable when your organization needs a fully managed consent UI, enterprise federation operations, or a compliance team that expects a vendor-owned policy console. Stick with Auth0, Cognito, or Firebase when that operational ownership is the requirement. Your mileage may vary with offline-first games; measure re-auth prompts and recovery volume instead of assuming a shorter token is automatically better.

## What should the eval harness measure before rollout?

I would run a table-driven test for four events: switch accounts on one device, open the old game tab, report a stolen session, and sign out everywhere. The expected result is concrete: the old device session cannot verify after revocation, unrelated devices remain active after a local sign-out, and a global revoke invalidates every recorded session for that user.

Track friction (time to switch, re-auth prompts, failed recovery) beside security signals (stale-session use, replay attempts, and audit completeness). A notebook-to-prod prototype often passes the happy path and misses the second tab. That is where the bug-shaped assumption usually hides.

Start with the smallest interface that preserves those semantics. Expand only when the measurements show a real need.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation
- https://firebase.google.com/docs/auth/admin/manage-sessions
- https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-using-the-access-token.html
