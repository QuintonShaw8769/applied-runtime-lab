# Vanity Asset Hostname CNAME: Per-Tenant Signed URL Generation for Fintech

Short answer: keep each customer's DNS zone customer-owned, ask it to publish one CNAME for the asset hostname, and bind every signed request to the tenant and exact public hostname. Keep mail separate. The customer's MX records point mail at its chosen provider, while `statements.customer.example` aliases only the asset delivery name. A platform-owned zone is useful for a fallback hostname, but it should not become the source of truth for a customer's brand or mail routing.

The critical boundary is the mapping among hostname, tenant, object key, and expiry. Resolve it before minting a URL, sign one canonical representation, and reject a request if its host does not match. This blocks a valid link for one fintech tenant from being replayed through another tenant's vanity hostname.

## How should a vanity asset hostname CNAME plus signed URL work?

Suppose Northstar Treasury owns `northstar.example`. Its mail administrator keeps the apex MX records under company control. For assets, the administrator publishes a CNAME from `statements.northstar.example` to a platform-operated delivery name such as `tenant-edge.assets.example.net`. These are different record sets with different jobs.

That separation limits the scope of change: an asset rollout should not require a mail-routing edit. DMARC is another reason to keep mail in its own control plane; RFC 7489 defines policy and reporting around domain-aligned email authentication, not authorization for CDN objects.

The platform needs a verified registry entry containing `tenant_id`, `hostname`, status, and delivery target before serving the custom name. DNS directs traffic. It does not prove that a caller may read an object.

Routing is not authorization.

Use a platform-owned fallback such as `northstar.assets.example.net` during onboarding and recovery. Two names add certificate and monitoring work, but they let operators test delivery without changing customer DNS. That is a worthwhile trade-off for statement delivery.

Consider the full Northstar transition rather than the isolated DNS record. The application first emits a statement link on the fallback host, with `tenant_northstar`, the statement key, and expiry covered by its signature. After the certificate and CNAME are ready, generation switches to `statements.northstar.example`; the verifier sees the same tenant and key but a different signed host. An old fallback link remains valid only on the fallback name, and a new vanity link remains valid only on the vanity name. Copying either query string to the other host changes the canonical payload and fails verification. This concrete boundary is why host belongs inside the signature rather than merely in the registry lookup. It also gives rollback a clean meaning: stop issuing vanity links, retain the old mapping for the bounded lifetime of links already issued, and change DNS later. Mail never enters that sequence. Northstar's MX records and DMARC policy remain untouched because the asset label has its own narrow purpose.

Keep those planes separate.

## A focused signing implementation

This Python example signs the tenant, normalized host, object key, and Unix expiry. The path carries tenant identity explicitly, so verification does not infer it from an unrelated lookup.

```python
import base64
import hashlib
import hmac
import time
from urllib.parse import quote, urlencode


def b64url(raw: bytes) -> str:
    return base64.urlsafe_b64encode(raw).rstrip(b"=").decode("ascii")


def canonical(tenant: str, host: str, key: str, expires: int) -> bytes:
    fields = (tenant, host.rstrip(".").lower(), key, str(expires))
    return "\n".join(fields).encode("utf-8")


def create_signed_url(
    tenant: str, host: str, key: str, secret: bytes, ttl: int = 300
) -> str:
    if ttl <= 0 or not key or key.startswith("/"):
        raise ValueError("invalid ttl or object key")
    host = host.rstrip(".").lower()
    expires = int(time.time()) + ttl
    signature = b64url(
        hmac.new(secret, canonical(tenant, host, key, expires), hashlib.sha256).digest()
    )
    query = urlencode({"tenant": tenant, "expires": expires, "signature": signature})
    return f"https://{host}/{quote(key, safe='/')}?{query}"


def verify(
    tenant: str,
    request_host: str,
    key: str,
    expires: int,
    supplied: str,
    secret: bytes,
    now: int,
) -> bool:
    if expires < now:
        return False
    expected = b64url(
        hmac.new(
            secret,
            canonical(tenant, request_host, key, expires),
            hashlib.sha256,
        ).digest()
    )
    return hmac.compare_digest(expected, supplied)


url = create_signed_url(
    "tenant_northstar",
    "statements.northstar.example",
    "reports/2026/q3-summary.pdf",
    b"load-this-from-a-secret-manager",
)
print(url)
```

Before object retrieval, verification must also confirm that the normalized request host is active and assigned to the supplied tenant. Decode the request path in exactly one place; signing one representation but authorizing a differently decoded form creates ambiguity around escaped characters.

The 300-second default makes tests concrete, but it is not universal. Short expiry reduces the useful replay window while making clock skew and slow retrieval more visible. Long expiry makes emailed links forgiving but extends exposure after forwarding. Choose from document sensitivity and workflow, then make the chosen duration an evaluation case.

## Where does tenant isolation fail?

Signing only the object path is the sharpest failure. Two tenants may both request `reports/latest.pdf`; a signature can cross the boundary unless tenant and host are covered by the MAC. Sign all four fields. No exceptions.

The mismatch must fail.

Do not accept an arbitrary host from a browser or job payload during generation. Load the active hostname from the tenant registry. During verification, normalize case and a single trailing dot consistently, then reject unknown, pending, disabled, or mismatched mappings.

Secret scope is a real choice. One global secret is easy to cache, but disclosure affects every tenant and rotation becomes coordinated. Per-tenant secrets narrow that scope and allow staggered rotation, at the cost of another lookup and more lifecycle state. For financial documents, that isolation usually earns its complexity.

Do not treat DNS as the authorization database. Resolver caches serve naming; access revocation needs application state with explicit semantics.

## How do you test a DNS-dependent path without flaky evaluations?

Split the harness at the network boundary. Unit tests inject fixed timestamps and cover correct host, wrong tenant, altered key, expired timestamp, mixed-case host, and trailing-dot host. Both allow and deny cases matter. A suite that checks only URL creation measures serialization, not isolation.

Run a smaller integration suite against controlled DNS names. Confirm that the vanity name resolves to the intended target, its certificate covers the requested hostname, and the request reaches the expected tenant mapping. Put live resolution checks in deployment validation and periodic monitoring, where retries and DNS timing are explicit, rather than every pull request.

This is the notebook-to-production gap. A notebook can demonstrate one HMAC. Production needs fixed canonicalization, injected time, negative cases, key identifiers, structured rejection reasons, and registry states. Keep test vectors small as well; prompt-assisted operational analysis should not spend tokens ingesting full documents.

Log tenant ID, normalized hostname, key identifier, expiry outcome, and a reason code. Never log the secret or full signature. Object keys may reveal account context, so use a stable digest or redacted path in general logs. Separate metrics for unknown host, tenant mismatch, expired link, and invalid signature; one authorization-failure counter obscures the failure mode.

## Cutover and operational checks

Begin on the fallback hostname with the tenant entry pending. Verify control, provision a certificate for the exact asset hostname, then request the customer CNAME. Activate only after the expected DNS target and certificate are ready.

Generate new links with the vanity hostname after activation, while honoring fallback links until their maximum expiry passes. For rollback, reverse link generation first; DNS follows according to its cached lifetime. This avoids creating URLs for a name operators are withdrawing.

The final checklist works better as prose than a wall of boxes. Confirm that MX and mail-authentication records are outside the asset change, verify the customer-owned CNAME and certificate, activate one host-to-tenant mapping, run cross-tenant negative tests, and inspect reason-coded metrics. Record who owns DNS, the registry, and key rotation. Rehearse disabling a hostname without deleting its audit history.

The durable design is customer-owned DNS for the branded contract, a platform-owned fallback for controlled testing, and authorization bound to tenant, host, object, and time. The CNAME routes the request. It never grants access.

## Sources

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
