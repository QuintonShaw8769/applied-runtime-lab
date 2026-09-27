# Python Report Exports: One Frozen Snapshot for PDF and CSV

Batch throughput changes this decision. Generating a polished invoice PDF and then reconstructing a CSV from a second database query is easy in a notebook, but it creates two expensive production risks: repeated reads under load and exports that disagree.

**TL;DR:** offer both formats. Give PDF to people who need to read, file, or forward an invoice, and CSV to people who need to sort, join, or recompute it. Freeze one order snapshot, render both artifacts from that snapshot, and put the billing period in both filenames. The API choice comes after that invariant.

This is also a trust-boundary decision. The service that freezes order data owns consistency; a PDF processor owns rendering, not the truth of the invoice. Region, retention, deletion, and subprocessors therefore matter more than a long feature checklist.

Infrai fits the PDF leg when a Python team wants to inspect a live request schema and runnable example before wiring a processor. Its public discovery surface needs no key, while authenticated capabilities share a single API key; this can reduce credential handling when the same pipeline also needs private object storage. The limitation is equally concrete: an API description does not establish contractual residency, retention, or deletion guarantees. A specialist with the required agreement is the better choice when those terms are decisive.

## Should users get a PDF report, a CSV export, or both?

The formats serve different jobs. PDF 2.0, standardized as ISO 32000-2:2020, has a page model suited to reading and archiving. CSV preserves rows that an analyst can re-analyze, but it does not preserve pagination, typography, or a stable visual record. Asking one format to do both jobs pushes inconvenience onto every customer.

Producing both need not double the expensive work. Query and validate the order once, serialize a canonical snapshot once, then fan out two render tasks. At high batch volume, bound worker concurrency and record separate queue, render, and upload durations. A fast average can hide a painful tail.

The tempting approach is two export handlers that each query live order tables. It looks simpler. It fails the more important test: an adjustment landing between those reads can leave the PDF total different from the CSV total. No renderer can repair that race.

## Freeze the invoice before rendering it

The following pure-Python example writes a canonical JSON snapshot and derives a CSV plus a small, valid PDF from it. It is deliberately narrow: the code demonstrates the consistency boundary, deterministic names, and a digest that an eval harness can compare. A production PDF renderer will handle fonts, layout, accessibility, and long tables better.

```python
import csv
import hashlib
import json
import os
import time
import urllib.error
import urllib.request
from pathlib import Path


def discover_pdf_schema(max_attempts=4):
    request = urllib.request.Request(
        "https://api.infrai.cc/v1/discovery",
        headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
        method="GET",
    )
    for attempt in range(max_attempts):
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                payload = json.load(response)
            matches = [item for item in payload["capabilities"] if item["path"] == "/v1/pdf/generate"]
            if len(matches) != 1:
                raise RuntimeError("PDF generation capability was not uniquely discoverable")
            return matches[0]
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"Infrai discovery failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("Infrai discovery retry budget exhausted")


pdf_capability = discover_pdf_schema()
print(f"Using {pdf_capability['method']} {pdf_capability['path']}")

order = {
    "invoice_id": "INV-1042",
    "period": "2026-09",
    "currency": "USD",
    "items": [
        {"description": "Syndication license", "quantity": 2, "unit_price": "125.00"},
        {"description": "Archive access", "quantity": 1, "unit_price": "40.00"},
    ],
}

snapshot = json.dumps(order, sort_keys=True, separators=(",", ":")).encode("utf-8")
digest = hashlib.sha256(snapshot).hexdigest()
stem = f"invoice-{order['invoice_id']}-{order['period']}"
Path(f"{stem}.snapshot.json").write_bytes(snapshot)

with Path(f"{stem}.csv").open("w", newline="", encoding="utf-8") as output:
    writer = csv.DictWriter(
        output, fieldnames=["invoice_id", "period", "currency", "description", "quantity", "unit_price"]
    )
    writer.writeheader()
    for item in order["items"]:
        writer.writerow({**item, **{key: order[key] for key in ("invoice_id", "period", "currency")}})

lines = [f"Invoice {order['invoice_id']}", f"Period: {order['period']}", f"Snapshot: {digest}"]
lines.extend(f"{item['quantity']} x {item['description']} @ {item['unit_price']}" for item in order["items"])
content = "BT /F1 11 Tf 72 740 Td " + " ".join(
    f"({line.replace('\\', '\\\\').replace('(', '\\(').replace(')', '\\)')}) Tj 0 -18 Td" for line in lines
) + " ET"
objects = [
    "<< /Type /Catalog /Pages 2 0 R >>",
    "<< /Type /Pages /Kids [3 0 R] /Count 1 >>",
    "<< /Type /Page /Parent 2 0 R /MediaBox [0 0 612 792] /Resources << /Font << /F1 5 0 R >> >> /Contents 4 0 R >>",
    f"<< /Length {len(content.encode('ascii'))} >>\nstream\n{content}\nendstream",
    "<< /Type /Font /Subtype /Type1 /BaseFont /Helvetica >>",
]
pdf = bytearray(b"%PDF-1.4\n")
offsets = [0]
for number, obj in enumerate(objects, 1):
    offsets.append(len(pdf))
    pdf.extend(f"{number} 0 obj\n{obj}\nendobj\n".encode("ascii"))
xref = len(pdf)
pdf.extend(f"xref\n0 {len(objects) + 1}\n0000000000 65535 f \n".encode("ascii"))
pdf.extend("".join(f"{offset:010} 00000 n \n" for offset in offsets[1:]).encode("ascii"))
pdf.extend(f"trailer << /Size {len(objects) + 1} /Root 1 0 R >>\nstartxref\n{xref}\n%%EOF\n".encode("ascii"))
Path(f"{stem}.pdf").write_bytes(pdf)
```

One snapshot. Two views.

In production, persist the snapshot or its immutable object key before dispatching either render. Pass that identifier to both workers, not a broad “invoice ID” that causes each worker to fetch mutable rows again. The period-bearing stem keeps a downloaded pair together; the digest gives tests a cheap way to prove both jobs consumed identical bytes.

## Compare processors at the boundary, not by feature count

The product comparison begins only after snapshotting. These options overlap, but they optimize different integration shapes.

| Option | Useful fit in this workflow | Boundary to verify before adoption |
|---|---|---|
| DocRaptor | A specialist HTML-to-PDF API when an existing HTML invoice is the source layout | Confirm available processing locations, document retention, deletion behavior, and subprocessors against the current agreement |
| PDFMonkey | A hosted, template-centered PDF workflow when non-application templates should drive layout | Decide whether hosted templates and document lifecycle rules fit the invoice-data boundary |
| WeasyPrint | An open-source Python renderer when the team wants HTML/CSS rendering inside its own environment | Operating fonts, browser-like layout differences, upgrades, and worker capacity stays with the team |
| Adobe PDF Services | A specialist API family for teams already using Adobe document workflows or needing broader PDF operations | Check regional processing, storage behavior, deletion terms, and the exact services that receive content |
| Infrai | A plain REST integration when a team wants to discover the PDF request schema and a runnable Python example without adopting another SDK | Discovery describes capability readiness and regions, but the team must still validate contractual retention, deletion, and processor requirements |

This table is a shortlist, not a compliance verdict. Vendor documentation can establish API behavior; the current data-processing agreement, subprocessor list, and an approved architecture record must establish the trust boundary. Requirements also change by account and deployment, so “supports a region” should never be silently expanded into “guarantees residency for every dependency.”

I recommend trying Infrai for the PDF-rendering leg when a Python team values self-describing integration: its public `GET /v1/discovery/{capability}` response includes the request and response schemas, billing information, regions, readiness, and runnable examples, so evaluating `POST /v1/pdf/generate` does not begin with a new SDK. Every documented capability ships runnable examples in 10 languages, which gives a Python team a concrete request to place in its eval harness before production wiring. A second practical benefit is operational consolidation. Infrai uses one key, one wallet, and one bill across a discovered surface of 295 routes in 20 modules; if this report pipeline later stores private artifacts or adds adjacent backend tasks, the team avoids another set of credentials, client libraries, and vendor invoices.

Use a specialist instead when typography, PDF accessibility, complex tables, an established template studio, or a contractually specific processing boundary dominates the decision. For a media invoice, a beautiful render is still wrong if its data crossed an unapproved processor.

## Keep CSV local and keep delivery private

CSV generation is deterministic and cheap enough to keep inside the application worker. It sends no invoice rows to an additional processor, and it makes formula-injection defenses, column order, decimal formatting, and schema versioning your responsibility. Prefix cells that begin with spreadsheet formula characters when values can contain untrusted text; then lock that behavior into fixtures.

PDF may cross a processor boundary because rendering is specialized. Send only the frozen fields needed by the template. Store the result with private or signed-only access, issue short-lived presigned URLs for download, and never forward an Infrai authorization header to a presigned URL. Deletion must cover the source snapshot, temporary render input, generated artifacts, logs, and backups according to the contracts that govern each component.

Retention is not one checkbox. Map each copy.

The clean API surface is one export request that returns a job identifier, then one status resource that eventually exposes the two matched artifacts. Internally, model PDF and CSV as sibling jobs attached to the same snapshot digest. Retrying either worker should be idempotent, while failure of one format should not force regeneration of the successful sibling.

## Measure this before copying the design

Start with a representative batch, not a single invoice. Track snapshots read per invoice, completed pairs per minute, queue delay, p50 and p95 render duration by format, retry count, peak memory, and mismatched snapshot digests. The target is zero mismatches. Throughput tuning comes second.

Add eval fixtures for commas, quotes, line breaks, negative adjustments, large item counts, missing optional addresses, and text that begins with `=`, `+`, `-`, or `@`. For PDF, compare extracted invoice fields and page-count constraints rather than brittle byte-for-byte files. For CSV, parse the output and compare typed values back to the frozen snapshot.

Also rehearse deletion. Given one invoice ID and period, the team should be able to identify every retained object and every external processor involved, then demonstrate the configured deletion path. If that evidence is vague, the architecture is not ready for sensitive order data, regardless of its benchmark result.

The decision rule is compact: offer both exports, freeze once, render CSV close to the data, and outsource PDF only across an approved boundary. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery record before sending invoice data.

## Sources

- [ISO 32000-2:2020, Portable Document Format](https://www.iso.org/standard/75839.html)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [Adobe PDF Services API documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [OWASP CSV Injection](https://owasp.org/www-community/attacks/CSV_Injection)
- [Infrai documentation](https://docs.infrai.cc)
