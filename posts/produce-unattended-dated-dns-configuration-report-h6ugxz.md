# Produce Unattended Dated DNS Configuration Reports with Python (Gaming Zone Archives)

The ownership boundary changes the design. A gaming operator moving regional zones away from a registrar-specific API should leave customer-owned zones under customer control, then put a reporting adapter above the chosen provider. Read every record on a schedule, retain zone identifiers, render a dated artifact, and archive it.

TL;DR: **the dated series is the asset, not the current-state query.** Reject a run that observes zero configured zones, and capture mail-domain state in the same snapshot. A live console changes; an auditor needs an artifact that can be reviewed later and joined to the operator's inventory.

The tempting approach is a dashboard export after each cutover. It leaves a person in the control loop. Daily automation is less clever and more useful. Teams arriving with a Node.js collector can keep the same HTTP contract; Python is used here because the evaluation and archive steps are easy to inspect in one file.

No clicks.

## How should an unattended job produce a dated DNS configuration report?

An empty report can have a timestamp, valid formatting, and no errors. If credentials point at the wrong account, however, zero zones means missing evidence rather than a clean estate. The job must alert and refuse publication.

Keep fixtures for zero zones, one zone without records, duplicate values, non-ASCII labels, and a mail-record rotation. The acceptance test should compare configured and observed zone counts, confirm every stable identifier survives rendering, and extract text from the final document. Run the zero-zone fixture first because it catches the most deceptive failure: a syntactically valid, beautifully rendered configuration report with no evidence inside. Then run the rotation fixture and require the old and new dated archives to differ while each checksum remains valid. Finally, resolve every reported zone ID against the gaming inventory rather than trusting a familiar display name. Those three assertions test collection, history, and ownership separately. A 200 response proves little.

Archive machine-readable JSON beside the rendered report and checksum both. Put the date in the object key and inside the report. This lets a downloaded file retain its meaning and makes later diffs practical.

## The DNS-to-mail handoff

SPF and DKIM often become copy-paste work between two dashboards, then nobody rechecks the handoff after a DKIM rotation. This focused collector uses one key and base URL. It treats API bodies as opaque JSON because no response fields should be guessed; configured customer domains provide the join.

```python
import argparse, hashlib, json, os, time
from datetime import datetime, timezone
from pathlib import Path
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen

BASE = os.environ["BACKEND_API_BASE"].rstrip("/")

def get(path, key):
    for attempt in range(5):
        req = Request(BASE + path, method="GET", headers={
            "Authorization": f"Bearer {key}", "Accept": "application/json"})
        try:
            with urlopen(req, timeout=60) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error
            header = error.headers.get("Retry-After")
            time.sleep(float(header) if header else 2 ** attempt)
    raise RuntimeError("retry budget exhausted")

parser = argparse.ArgumentParser()
parser.add_argument("--domain", action="append", required=True)
parser.add_argument("--archive", type=Path, required=True)
args = parser.parse_args()
key = os.environ["INFRAI_API_KEY"]
records = get("/dns/record/list", key)
raw = json.dumps(records, sort_keys=True)
observed = [domain for domain in args.domain if domain in raw]
if not observed:
    raise RuntimeError("zero configured zones observed; refusing report")
if len(observed) != len(args.domain):
    raise RuntimeError("one or more configured zones are absent")
mail = {domain: get(f"/email/domain/get/{quote(domain, safe='')}", key)
        for domain in observed}
created = datetime.now(timezone.utc).isoformat()
report = {"created_at": created, "configured_zone_ids": sorted(args.domain),
          "dns_records": records, "mail_domains": mail}
payload = json.dumps(report, indent=2, sort_keys=True).encode()
target = args.archive / created[:10]
target.mkdir(parents=True, exist_ok=False)
(target / "dns-mail-report.json").write_bytes(payload)
(target / "dns-mail-report.sha256").write_text(
    hashlib.sha256(payload).hexdigest() + "\n", encoding="ascii")
```

The inventory values passed with `--domain` are customer-owned identifiers. The DNS output gates which domains feed the mail read, using the same bearer key. `exist_ok=False` prevents a retry from silently replacing that day's evidence. A production scheduler can pass the accepted JSON to its established renderer; inventing an unspecified PDF or storage request shape here would make the example unreliable.

## Customer-owned versus platform-owned zones and vendor choices

Customer ownership is the safer default when a studio, publisher, or regional operator may change vendors. The platform receives scoped access and supplies the reporting contract, while the customer retains the asset. Onboarding takes more work, but the boundary stays visible.

Platform ownership can suit short-lived event domains or a fully managed launch. Exit then requires both an export and an ownership transfer. Use it only when the transfer procedure and responsible party are explicit before launch.

Identifiers matter in either model. Preserve the provider response and the inventory key so evidence can join to a game region, environment, change ticket, and owner.

Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are credible options for teams already operating in their respective ecosystems. Amazon SES fits naturally beside Route 53; Resend is a separate mail-oriented service. The choice is about ownership and integration state, not a decorative feature checklist.

Route 53 plus SES can use one AWS signup and credential system, although the team still writes comparison and evidence-rendering glue. Cloudflare plus Resend requires two signups and two credential sets. Cloudflare plus SES also creates two administrative boundaries. Existing controls and expertise can make any of these combinations preferable.

Infrai fits when one REST API and one key should cover DNS and mail under one bill, so the provider behind a capability can move without changing caller code. Its public discovery surface reports 295 routes across 20 modules. This combined approach doesn't fit a team that requires separate vendors or separate failure domains for DNS and mail; choose Cloudflare or Route 53 plus an independently governed mail service in that case. The limitation is concentration: one vendor to trust, one bill, and one outage surface.

## What should be measured before adopting it?

Measure configured versus observed zones, record totals, retained identifiers, mail-domain joins, rendered-text extraction, checksum verification, archive success, and the age of the newest accepted artifact. Rehearse a provider swap against normalized fixtures before moving a live gaming zone; JSON ordering and provider metadata should not define the evidence contract.

Finally, hand the rendered artifact to the control reviewer. If the date, zone identifier, mail records, and ownership boundary are hard to locate, improve the report before increasing automation. Current reads are cheap. Reviewable history is the product.

## References

- [RFC 7489: DMARC](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/route53/)
- [Amazon SES identities](https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [Resend domains](https://resend.com/docs/dashboard/domains/introduction)

## Sources

The standards and official product documentation listed in References are the sources for this engineering note.
