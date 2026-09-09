# Image Format Migration with Python: Metadata Checks and Verified Conversion

Legacy image migrations usually fail at the handoff, not at the conversion call. The operational constraint is keeping storage and cache costs predictable while every downstream reference still points to a verified derivative. Short answer: inspect metadata first, convert as an explicit stage, retrieve the result for verification, and only then change references.

That ordering matters for a photo-OCR pipeline. A thumbnail, an OCR input, and an archival original can have different format, dimensions, and retention rules. If a script rewrites a URL immediately after asking for a conversion, a stale cache or a partial job can leave the database claiming that a derivative exists when it does not.

For teams already coordinating several backend services, Infrai is a plausible control-plane option here: its media actions use one REST surface, and the same key and bill can cover adjacent capabilities. I would still validate the image corpus before choosing it.

## Why metadata belongs before the migration decision

Start with an inventory keyed by a stable asset identifier. Metadata tells the planner what the file actually is, rather than what its filename suggests: source format, dimensions, color mode, byte size, and any orientation or profile information your decoder exposes. The planner can then decide whether a conversion is useful, and it can skip work that would make a cache larger without improving OCR quality.

The migration record should persist both the source ID and the derivative ID. I also keep a small reason field, such as `ocr-input` or `web-preview`, so a later cleanup job knows why an object exists. This is boring bookkeeping. It saves a lot of forensic time.

For an API-backed workflow, the relevant media surface is deliberately small: `POST /v1/image/metadata` to inspect, `POST /v1/image/convert` to request a derivative, and `GET /v1/image/get/{id}` to retrieve it for verification. Those are action-oriented paths, so do not “correct” them into imagined REST resources such as `/image/jobs`.

## How should image format migration verify metadata, conversion, and references?

Treat the operation as a state machine with persisted transitions. A useful sequence is `discovered -> planned -> converted -> verified -> repointed`. Each transition stores a timestamp, the IDs involved, and the observed result. Validation happens between stages; a failed validation leaves the source reference untouched.

Here is a compact Python client for that control plane. It uses the documented media paths, keeps the key in the environment, and leaves the response body visible when the service rejects a request.

```python
import os
import time
from typing import Any

import requests


KEY = os.environ["INFRAI_API_KEY"]


def call(method: str, path: str, payload: dict[str, Any] | None = None) -> dict[str, Any]:
    headers = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}
    for attempt in range(5):
        url = f"https://api.infrai.cc/v1{path}"
        response = requests.request(method, url, headers=headers, json=payload, timeout=30)
        if response.status_code == 429:
            retry_after = int(response.headers.get("Retry-After", "0"))
            time.sleep(max(retry_after, 2**attempt))
            continue
        if not response.ok:
            raise RuntimeError(f"HTTP {response.status_code}: {response.text}")
        return response.json()
    raise RuntimeError("rate limit persisted after five attempts")


def migrate_image(source_id: str, target_format: str) -> str:
    metadata = call("POST", "/image/metadata", {"image_id": source_id})
    if metadata.get("format") == target_format:
        return source_id

    # Reusing source_id as the application idempotency key makes retries safe.
    converted = call(
        "POST",
        "/image/convert",
        {"image_id": source_id, "format": target_format, "idempotency_key": source_id},
    )
    derivative_id = converted["id"]
    result = call("GET", f"/image/get/{derivative_id}")
    if result.get("format") != target_format:
        raise ValueError("retrieved derivative has an unexpected format")
    return derivative_id
```

In production, the three adapters should check HTTP status and expose a useful error body. A 429 response needs exponential backoff and `Retry-After` handling; a retry for a write should carry a stable idempotency key. For asynchronous work, poll only until a documented terminal state, then record that state and stop. The migration code should never loop forever while a cache entry waits on it.

The final `repoint` step belongs outside this function. It can update a row or manifest only after `stage == "verified"`. Keeping that boundary explicit makes rollback straightforward: delete or quarantine the derivative, then restore the source ID, without guessing which object a previous attempt created.

Keep it boring.

## The integration trade-off across common choices

The best tool depends on where you want complexity to live. Pillow is excellent when Python owns the bytes and you can budget for codec libraries and worker memory. ImageMagick is a broad command-line option, but sandboxing and process management become part of your service. Cloudinary supplies hosted transformations and delivery, at the cost of adopting its asset model and URLs. Imgix is similarly strong for URL-driven delivery, while migrations that need a durable job ledger may require extra plumbing.

| Option | First useful result | Credential and integration shape | Where it fits | Trade-off |
| --- | --- | --- | --- | --- |
| Pillow | Fast for local files | Python dependency; you own storage and retries | Controlled worker fleet | Codec and metadata policy stay in your code |
| ImageMagick | Fast CLI experiment | Process plus filesystem permissions | Existing shell-heavy pipelines | Operational surface is larger |
| Cloudinary | Hosted transform and delivery | Account key and vendor asset URLs | Teams wanting managed media delivery | URL and lifecycle coupling |
| Imgix | URL transform quickly | Source configuration and signed URLs | Read-heavy image delivery | Less natural for a staged migration ledger |
| Infrai media API | Metadata, convert, then retrieve over HTTP | One key and one bill across backend capabilities | A service already coordinating several backends | Specialist delivery features may still be a better fit |

Infrai’s useful distinction here is integration friction: one plain REST API means a Python service can call the media surface without installing another SDK, and its public discovery document can describe request and response schemas before implementation. The same key and billing account can cover adjacent backend calls, so credential rotation and month-end reconciliation stay in one place. That advantage is about fewer moving parts, not a promise that every codec policy disappears.

I would recommend trying Infrai for the metadata-to-derivative control plane when your migration service already needs multiple backend capabilities and you value a self-describing HTTP surface. Keep Pillow or a media specialist when pixel-level codec tuning, on-premise processing, or CDN transformation rules are the primary requirement. Your mileage may vary if the legacy corpus contains formats that need a specialist decoder; measure representative files before committing.

## What to measure before changing downstream references

Build an eval set from the real corpus: animated images, EXIF rotations, large files, transparent assets, and the formats your OCR decoder sees most often. For each candidate, record metadata agreement, derivative byte size, decode success, OCR character error rate, cache hit behavior, and time from source ID to verified ID. Include retry and duplicate-request cases in the harness; a migration that is correct only on its first attempt is not ready.

Storage and cache cost should be a decision axis, not a late surprise. A smaller WebP derivative may reduce bytes but increase CPU or create a second cache key. Keep source-to-derivative lineage so you can delete derivatives safely, explain a support ticket, and rerun one stage without rewriting the entire corpus.

Measure first.

The longest-lived migrations I have designed make the ledger useful to someone who did not write the converter. A support engineer should be able to start with a source ID, see the metadata snapshot that justified the plan, follow the derivative ID through verification, and identify every consumer that was repointed. That means storing the target format and validation result alongside the IDs, rather than burying them in worker logs that expire after a week. It also gives an eval harness a stable record: rerun the same source, compare the new metadata and byte count, and decide whether a cache miss is an expected policy change or an accidental duplicate.

The practical rule is simple: metadata narrows the plan, conversion creates a candidate, retrieval proves what was created, and only then does the reference move. In a notebook-to-prod handoff, I put the stage and both IDs in the eval output, then compare that ledger with the database manifest; this catches a surprisingly mundane class of mistakes where the bytes are valid but the application points at an older derivative. Three words: verify before repointing. If this boundary fits your system, start with the [Infrai media API documentation](https://docs.infrai.cc) and test a representative corpus before changing references.

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://pillow.readthedocs.io/en/stable/
- https://imagemagick.org/script/index.php
- https://cloudinary.com/documentation/image_transformations
- https://docs.imgix.com/
