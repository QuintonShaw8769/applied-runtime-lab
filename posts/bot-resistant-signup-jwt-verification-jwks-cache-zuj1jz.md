# Bot-Resistant Signup JWT Verification: JWKS Cache Windows and Session Introspection

Short answer: for an e-commerce signup gate, verify JWTs locally with a bounded JWKS cache, then use session introspection only at high-risk boundaries such as captcha enrollment and account recovery. That split keeps the API gateway fast without pretending that a cached key can tell you a session was revoked.

The bot problem starts before a customer has a useful session. A signup request presents a captcha result, receives a short-lived access token, and then reaches an API gateway that must decide whether to create an account. Attackers replay old tokens, automate the captcha step, or fan requests across many IPs. JWT verification answers “was this token signed by a trusted issuer?” It does not answer “is this particular session still allowed right now?”

I build RAG and agent services in Python, so I test this boundary in a notebook before wiring it into the gateway. The test data is deliberately unglamorous: a token with a stale `kid`, a token whose `exp` is 30 seconds in the past, and a valid token whose session was revoked after a suspicious signup. Those three rows expose most design mistakes.

## What should a JWT verification architecture do for captcha-protected signup?

Use two paths. The hot path parses the token header, obtains the issuer's JWKS document from a cache, verifies the signature and registered claims, and applies a local risk policy. The cold path calls an introspection endpoint when the operation is expensive, irreversible, or recently challenged. Account creation belongs in the cold path when abuse signals are high; catalog reads usually do not.

The cache needs explicit limits. Cache a successful JWKS response for a short, documented window; retain the previous key during a rotation overlap; and refresh on an unknown `kid` with request coalescing so a bot burst cannot stampede the issuer. Never extend token lifetime just because the key document is cached. Clock skew, issuer, audience, and algorithm must be checked on every request.

Here is a compact Python sketch. The `jwt_library` and `jwks_client` names stand for the libraries already approved in your service; the important part is the decision order and the bounded cache, not a vendor-specific SDK.

```python
import time


class JwksCache:
    def __init__(self, client, ttl_seconds=300):
        self.client = client
        self.ttl_seconds = ttl_seconds
        self.keys = {}
        self.expires_at = 0

    def key_for(self, kid):
        now = time.time()
        if now >= self.expires_at or kid not in self.keys:
            document = self.client.fetch()
            self.keys = {item["kid"]: item for item in document["keys"]}
            self.expires_at = now + self.ttl_seconds
        return self.keys.get(kid)


def verify_signup(token, cache, jwt_library, issuer, audience, introspect):
    header = jwt_library.get_unverified_header(token)
    jwk = cache.key_for(header["kid"])
    if jwk is None:
        return {"allow": False, "reason": "unknown_key"}

    claims = jwt_library.decode(
        token,
        jwk,
        algorithms=["RS256"],
        issuer=issuer,
        audience=audience,
        options={"require": ["exp", "iat", "iss", "aud"]},
    )
    if claims.get("signup_risk") == "high":
        live = introspect(token)
        if not live.get("active", False):
            return {"allow": False, "reason": "inactive_session"}
    return {"allow": True, "subject": claims["sub"]}
```

The code intentionally fails closed for an unknown key and for an inactive session. In production, map those outcomes to a generic response and record a reason code internally; returning “your key rotated” gives an attacker useful timing information.

## How do JWKS caching and session introspection trade off bot resistance?

Think in terms of two clocks. The JWKS clock controls how quickly the gateway learns a signing-key change. The session clock controls how quickly it learns a user was revoked. A five-minute key cache can be reasonable when the issuer publishes overlapping keys, while a five-minute revocation delay may be unacceptable for a signup token that just triggered a fraud alert.

| Decision | Local JWT plus JWKS cache | Live session introspection |
| --- | --- | --- |
| Network dependency | Only on cache miss or rotation | Every protected decision, unless separately cached |
| Revocation visibility | Delayed until token expiry or a deny-list check | Near real time, subject to endpoint latency |
| Bot burst behavior | Predictable CPU and cache load | Extra upstream calls can become an amplification point |
| Failure policy | Gateway can use a documented stale-key window | Must choose fail-open or fail-closed for the issuer call |
| Best fit | Read-heavy APIs and low-risk token use | Signup, password reset, and fraud-reviewed actions |

The catch is that introspection is not automatically safer. If the introspection service has a broad timeout and no circuit breaker, an attacker can turn signup traffic into a dependency outage. Set a small deadline, cap concurrent calls, and make the fallback explicit. For a high-risk signup, a timeout should hold the request for a challenge or review rather than silently create an account.

Measure twice.

Consider a launch-day replay that looks harmless in aggregate. At 09:00, the issuer rotates from `kid=blue` to `kid=green`; both keys are published for ten minutes. At 09:02, an automated client submits 40,000 signup attempts carrying tokens signed with `blue`, each with a captcha result copied from the same browser session. A gateway that refreshes JWKS only when its five-minute cache expires will still verify those signatures, which is correct key behavior, while a gateway that introspects every request will spend its capacity on sessions that should have been rate-limited earlier. The useful control is a risk transition: once the captcha verifier, IP reputation, or account-velocity counter marks the cohort high risk, route only that cohort to introspection, require a fresh challenge, and keep the normal cache path for low-risk traffic. Record the decision with a request ID so an analyst can reconstruct why one token was accepted locally and another was checked live. This is a policy boundary, not a claim that either mechanism catches bots by itself.

I once treated a cache hit as proof that a session was healthy. That produced a green verification metric while a revoked session kept passing the gateway. The fix was a separate metric for “signature valid” and “session active,” plus a test that revokes a token between those assertions. Two metrics. Different questions.

## What data and tests reveal a safe verification boundary?

Build an eval harness before tuning TTLs. Generate cases for an expired token, a wrong audience, an unexpected algorithm, an unknown `kid`, a rotated key, a revoked session, and a captcha result reused from another signup. Measure acceptance, rejection reason, p95 verification time, introspection timeout rate, and upstream calls per signup. Keep these fixtures in version control so a policy change has a visible before-and-after.

The token itself should carry only what the gateway needs: subject, issuer, audience, issued-at, expiry, and a risk or challenge reference when your policy requires it. Store captcha evidence and abuse counters server-side. A JWT is readable by its holder; signing protects integrity, not confidentiality.

Use property-based tests for claim combinations, then run a small replay test against the real gateway configuration. I'm not sure a universal five-minute TTL exists; your mileage may vary with key-rotation practice, token lifetime, and the cost of a fraudulent account. The answer should come from the measured revocation window your business accepts, not from a copied default.

Operationally, document the issuer URL, allowed algorithms, cache TTL, overlap period, introspection timeout, and rollback switch. Alert on unknown-key spikes, clock-skew failures, and a sudden rise in valid-signature/inactive-session mismatches. During key rotation, observe both old and new `kid` values before retiring the old one. During an abuse event, shorten the high-risk introspection cache or require a fresh challenge; do not change the global JWT expiry as an emergency knob.

This architecture is not suitable when every request must reflect revocation instantly or when the issuer cannot provide stable key-rotation semantics. In that case, keep the protected operation behind live introspection and accept the latency budget, or choose a session design with server-side state. For ordinary catalog traffic, stick with local verification and spend the network calls where account creation can hurt.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://www.rfc-editor.org/rfc/rfc7519
- https://www.rfc-editor.org/rfc/rfc7517
- https://www.rfc-editor.org/rfc/rfc7662
