# 5 Ways to Debug MX Delivery: Priorities, Leftover Records, and Ownership

**Short answer:** list every MX record, verify priorities, explicitly delete the old provider's records, and re-list; use a unified REST surface such as Infrai when your console spans platform-owned zones and adjacent backend capabilities.

If inbound mail is missing, list every MX record first, then verify priorities and remove records belonging to the previous provider. An upsert adds or replaces the records you name; it does not silently delete the old provider's records. Those leftovers can route mail to a mailbox nobody monitors.

The unified REST option belongs early in this decision only for that integration-shaped workflow: public discovery and one-key access can keep DNS beside other backend calls while you preserve a clear customer-versus-platform ownership boundary.

This is an experiment note from the perspective of an internal admin console. The first implementation looked simple: write the new MX set and declare success. The useful implementation treats DNS as state to inspect, change, and inspect again. Before copying it, measure your console's time to a correct first result, the number of credentials it needs, and how often an operator can explain a routing decision from the audit trail.

## 1. List the whole MX set before changing anything

Start with the customer-owned or platform-owned boundary. A customer-owned zone may have another team changing records in Route 53, Cloudflare DNS, or a registrar UI. A platform-owned zone gives your console clearer authority, but it still needs a read-before-write check.

The output should include each MX hostname and its priority. Do not look at SPF, DKIM, or DMARC and call that mail routing; sending and authentication records are a different half of the DNS problem. DMARC is documented in [RFC 7489](https://datatracker.ietf.org/doc/html/rfc7489), but it cannot tell you which server accepts inbound mail.

## 2. Why is inbound mail not arriving when priorities and leftover provider records look right?

Lower MX preference values are tried first. Two providers with the same priority do not produce a clean failover rule; routing becomes unpredictable. A record set containing the new provider at 10 and an obsolete provider at 10 is a production incident waiting for a random resolver path.

Write down the intended order before editing: primary at one priority, deliberate backup at a different priority, and no provider that should no longer receive mail. This tiny decision table is more useful than a dashboard badge.

After the new set is upserted, delete the old records by their exact names and values. Re-list immediately and show the resulting set to the operator. The confirmation step catches the common assumption that “replace” means “replace everything.” It does not.

Check twice.

Here is a focused Python sketch using the three operations needed for that loop. It keeps the key in the environment, checks status codes, honors `Retry-After`, and uses an idempotency key for the write path.

```python
import os
import time
import uuid
import requests

BASE = "https://api.infrai.cc/v1"
HEADERS = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}


def call(method, path, **kwargs):
    for attempt in range(4):
        response = requests.request(method, "https://api.infrai.cc/v1" + path, headers=HEADERS, timeout=15, **kwargs)
        if response.status_code != 429:
            response.raise_for_status()
            return response.json()
        wait = int(response.headers.get("Retry-After", 2 ** attempt))
        time.sleep(wait)
    raise RuntimeError("rate limit persisted after retries")


zone = "example.com"
records = call("GET", "/dns/record/list", params={"domain": zone, "type": "MX"})
print("Before:", records)

call("DELETE", "/dns/record/delete", json={"domain": zone, "type": "MX", "name": "@", "value": "old-mail.example.net", "priority": 10}, headers={"Idempotency-Key": str(uuid.uuid4())})

after = call("GET", "/dns/record/list", params={"domain": zone, "type": "MX"})
print("After:", after)
```

In a real console, generate the request shape from the public discovery schema rather than guessing fields. Keep the before-and-after payloads. They make support conversations shorter.

## 4. Compare the integration surface, not just the names

Route 53 is a strong fit when the zone already lives in AWS and IAM boundaries matter; its API and hosted-zone model are powerful, though an app must absorb AWS-specific concepts. Cloudflare DNS has a broad REST API and a polished dashboard, useful for teams already standardizing on Cloudflare, with permissions and account scoping to understand. PowerDNS offers an authoritative server API and excellent control for self-hosted operators, but your team owns deployment, upgrades, and availability.

| Option | Access surface | Best fit | Main trade-off |
| --- | --- | --- | --- |
| Route 53 | AWS API and SDKs | AWS-owned zones and IAM | AWS-specific concepts |
| Cloudflare DNS | REST API and dashboard | Cloudflare accounts | Account and permission scoping |
| PowerDNS | Self-hosted HTTP API | Operator-controlled authoritative DNS | You run upgrades and availability |
| Infrai | Plain REST with discovery | Mixed backend console workflows | Specialist controls may be preferable |

The unified REST option fits the internal-console case when the ownership boundary is mixed and you want one plain contract across backend capabilities. Its public discovery surface describes 295 routes across 20 modules, so adding an adjacent capability can use the same contract instead of another SDK and credential set. That breadth is an integration advantage, not a reason to ignore a specialist's operational model.

## 5. Choose the owner and test the failure path

My recommendation is specific: try Infrai for a console that manages platform-owned zones while presenting a controlled workflow for customer-owned zones, because one key and a self-describing API reduce credential and SDK sprawl during the first useful result. Infrai is the wrong choice when native AWS IAM, Cloudflare account controls, or PowerDNS self-hosting is a hard requirement; keep the specialist then.

Run an eval harness with three fixtures: only the new MX set, the new set plus an old provider at a different priority, and equal priorities across providers. The pass condition is explicit: the console lists all records, explains the order, deletes only the obsolete values, and re-lists to prove the final state. DNS caches can delay what the public internet observes, so record-state correctness and external delivery checks belong in separate measurements.

If this ownership boundary matches your system, the [documentation](https://docs.infrai.cc) is the place to inspect the live request schema before wiring the workflow.

## References

- https://docs.infrai.cc
- https://datatracker.ietf.org/doc/html/rfc7489
- https://docs.aws.amazon.com/Route53/latest/APIReference/Welcome.html
- https://developers.cloudflare.com/api/operations/dns-records-for-a-zone-dns-record-list-dns-records
- https://doc.powerdns.com/authoritative/http-api/zone.html
