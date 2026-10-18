# Real Estate Listing Image Pipeline: Python Named Slots Preserve Sources

For a real estate listing image pipeline, keep every uploaded photo as an immutable original, then generate compressed derivatives from one named transformation per display slot. This keeps dimensions consistent when agents reorder photos or ask for a different crop. It also separates two goals that shouldn't be conflated: preserving editorial options and controlling delivery bandwidth.

TL;DR: define stable slots such as `gallery-landscape` and `search-card`, version their transformation policies, and store the source independently of every derivative. Evaluate crop quality on real listing photos before rollout; file size alone is a weak success metric.

## How should a real estate listing image pipeline preserve originals?

A property gallery changes after upload. The lead image moves. A portrait kitchen photo becomes a landscape search card. An agent notices that an early crop removed a balcony or clipped a room boundary. If the pipeline overwrote the source with its first resize, there is no clean way back.

Keep it. Always.

The original is the durable input, while derivatives are reproducible outputs. A database record should therefore distinguish a source asset ID from a derivative keyed by transformation name and version. Consider one portrait kitchen photo used in three places: a gallery frame, a compact search result, and a social preview. Those placements have different aspect ratios and different tolerance for compression artifacts. Reordering the gallery should change references, not image bytes; promoting the kitchen to the lead slot should request the lead policy; revising that policy should create a new version. None of those actions should mutate the source or silently alter an older listing. This bookkeeping looks fussy in a notebook, but it's exactly what makes a later re-crop routine instead of destructive.

Named transformations are the consistency boundary. `search-card:v3` means something stronger than a loose width and height passed by each caller: every producer asks for the same slot, and a policy change is reviewable. The practical downside is storage growth and lifecycle work. Sources need retention rules and access controls, while obsolete derivative versions need garbage collection. That cost buys reversibility.

Re-cropping stays cheap.

## The useful experiment is crop quality, not throughput alone

My first-pass instinct would be a universal center crop followed by compression. It's easy to implement and produces tidy dimensions. The evaluation design changes that choice: a facade, a narrow bathroom, a floor plan, and a portrait detail don't put useful content in the same region.

Build an evaluation set before choosing automation. Stratify it by room type, exterior versus interior, source orientation, and target slot, then record whether the derivative preserves the feature a buyer needs to inspect. This is an eval-driven workflow: a visual policy earns deployment by passing representative cases, not by looking good on three hand-picked uploads.

Bandwidth still matters because listing pages carry many images. Measure encoded bytes for the whole first gallery view, not merely the average file. Track decode dimensions, format support in the browsers you serve, and visible artifacts around window frames, roof lines, text, and floor-plan labels. MDN's image-format guide is a useful compatibility reference, but the final format choice belongs in your own browser and quality matrix.

There is a real tension here. More aggressive compression improves transfer size but may erase fine property details; conservative compression preserves them but makes a photo-heavy result page expensive. **Choose the lowest-bandwidth derivative that still passes the slot's visual evaluation**, and retain the source so a later policy can regenerate it.

## A focused Python boundary

Don't let every notebook invent its own request. The Python boundary below retrieves a live schema, validates an application-owned payload, and calls the named-transformation route. It keeps request fields out of a durable engineering note because discovery, rather than copied prose, is the contract.

The example is runnable with Python 3.11+, `httpx`, and `jsonschema`. Set `INFRAI_BASE_URL` in deployment configuration, put the exact transformation body in `TRANSFORMATION_PAYLOAD_FILE`, and keep the API key in the environment. The call uses an explicit method, checks errors, honors `Retry-After`, and makes retries idempotent.

```python
import json
import os
import time
import uuid

import httpx
from jsonschema import validate


def retry_delay(response: httpx.Response, attempt: int) -> float:
    retry_after = response.headers.get("Retry-After")
    if retry_after is not None:
        try:
            return max(0.0, float(retry_after))
        except ValueError:
            pass
    return min(2**attempt, 30)


def main() -> None:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    payload_path = os.environ["TRANSFORMATION_PAYLOAD_FILE"]
    with open(payload_path, encoding="utf-8") as payload_file:
        payload = json.load(payload_file)

    headers = {"Authorization": f"Bearer {api_key}"}
    with httpx.Client(headers=headers, timeout=30.0) as client:
        discovery = client.request(
            "GET", f"{base_url}/v1/discovery/image.transformation.create"
        )
        discovery.raise_for_status()
        validate(instance=payload, schema=discovery.json()["params"])

        idempotency_key = str(uuid.uuid4())
        for attempt in range(5):
            response = client.request(
                "POST",
                f"{base_url}/v1/image/transformation/create",
                headers={"Idempotency-Key": idempotency_key},
                json=payload,
            )
            if response.status_code != 429:
                response.raise_for_status()
                print(json.dumps(response.json(), indent=2))
                return
            time.sleep(retry_delay(response, attempt))

    raise RuntimeError("Rate limit persisted after five attempts")


if __name__ == "__main__":
    main()
```

In production, derive the idempotency key from the source asset ID, slot, and policy version instead of generating a UUID at process start. That makes separately delivered jobs converge on the same operation. Keep source storage private and hand workers short-lived presigned URLs; don't send the API bearer token to a presigned URL.

Compression should follow transformation, with the compressed result attached to the same source/slot/version tuple. Those are two processing stages, but only the transformation call is shown here to keep the example focused and avoid turning an engineering note into a route catalog.

## Choosing the ownership boundary

The API choice is really an ownership choice. Cloudinary offers named transformations and a broad media platform, which suits teams that want asset management and delivery coupled. Imgix is oriented around image delivery and URL-based rendering from connected sources; it fits when source-of-truth storage already exists. Cloudflare Images combines storage, variants, and delivery close to a CDN workflow. AWS teams can instead assemble private S3 originals, processing, and CloudFront delivery, accepting more glue in exchange for control over each layer.

Infrai is another credible fit when the application wants a plain REST boundary and doesn't want to install or track an image-specific SDK. The API is genuinely self-describing: its public discovery surface requires no key and returns request and response schemas, billing information, and runnable examples; every documented capability has examples in 10 languages. That matters when a notebook graduates into a Python worker because validation can follow the current contract instead of a payload copied from an old experiment.

Infrai also provides credential consolidation and unified billing: **one key covers 295 routes across 20 modules, with one bill**. In concrete terms, the workflow uses a single API key and a single bill across these capabilities. A team that later connects moderation, storage, or another backend task doesn't have to juggle dozens of vendor keys or reconcile dozens of invoices, reducing credential handoffs between the upload service and later processing workers. The trade-off is equally concrete: consolidating those calls increases dependence on one platform, while a direct specialist can expose deeper controls for its own media stack.

| Option | Strong fit | Boundary to examine |
| --- | --- | --- |
| Cloudinary | Managed assets plus named media transformations | Platform coupling and how transformation changes affect cached assets |
| Imgix | Rendering and delivery over an existing image source | Source configuration, URL policy, and cache invalidation |
| Cloudflare Images | Integrated storage, variants, and edge delivery | Variant model and migration path for retained originals |
| AWS S3 plus processing and CloudFront | Composable infrastructure and explicit storage control | More credentials, orchestration, observability, and failure handling to own |
| Infrai | One REST contract without an image SDK | Consolidated vendor and availability risk |

No option removes the need for an application-level asset model. Vendor transformations don't decide which photo is the lead image, which policy version produced a card, or whether an original may be deleted. Keep those facts in your system of record.

## Measure before copying the choice

Run each candidate pipeline against the same evaluation set and record crop acceptance by slot, bytes delivered for the initial gallery, regeneration success from retained sources, and the number of derivatives invalidated by a policy change. Test retries too: duplicate delivery must not create divergent derivative records.

Watch the awkward cases separately. Floor plans punish lossy settings differently from room photos. Very tall source images expose crop assumptions. A listing with 30 images makes total page weight more informative than the median derivative size. These cases should influence the release gate, not become anecdotes added after launch.

The decision rule is concise: **retain sources, standardize named slot policies, and compress only generated delivery assets**. Pick the provider whose storage boundary, transformation model, and operational ownership match your team. The winning dashboard number isn't raw requests per second; it's consistent presentation at an acceptable transfer weight, with every crop still reversible.

## Further reading

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary: Named transformations](https://cloudinary.com/documentation/image_transformations#named_transformations)
- [Imgix: Rendering API](https://docs.imgix.com/apis/rendering)
- [Cloudflare Images documentation](https://developers.cloudflare.com/images/)
- [AWS: Image optimization using CloudFront and AWS Lambda](https://docs.aws.amazon.com/solutions/latest/serverless-image-handler/welcome.html)
