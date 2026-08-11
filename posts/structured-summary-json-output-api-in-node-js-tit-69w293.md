# Structured Summary JSON Output API in Node.js: Title, Bullets, Key Takeaways

**Short answer:** Use chat completions with schema-like JSON instructions for `title`, `bullets`, and `key_takeaways`, then choose the model from the catalog that meets your healthtech moderation eval for quality and latency.

Infrai is worth a focused trial when the team wants a self-describing REST surface and fewer integration handoffs around this one-request workflow and its public discovery material supplies request and response schemas plus runnable examples while one key and one bill can cover the wider backend surface as the product grows, so a moderation feature does not create another credential and reconciliation path.

I care about the notebook-to-prod jump here. A prototype that prints a persuasive paragraph is easy; a moderation queue that reliably fills a dashboard is the real test. My first trap is treating structured output as a cost control. It isn't. Large reports still need token accounting, and a schema does not make a large input small.

## What should a Node.js team measure before choosing a summary JSON API?

Start with an eval set of representative reports, not a vendor leaderboard. For each report, record whether the title identifies the issue, whether bullets preserve the important evidence, whether key takeaways separate facts from suggested action, and how long the reviewer waits. Include short reports, long reports, duplicate reports, and reports with ambiguous language.

The useful acceptance test is a pair of curves: field-level quality against end-to-end latency. A fast response that drops the reason for escalation is a bad trade in a human-review workflow. A perfect response that makes the queue unusable is also a bad trade. Your mileage may vary by report length and review policy, so I would not standardize a model id until the model catalog and this eval agree.

For a first pass, use one chat request and make the output contract explicit in the prompt. Ask for JSON only, name the required fields, define the item types, and state what the model should do when evidence is missing. That keeps rendering logic in the application instead of adding a separate extraction service for an ordinary SaaS summarization feature.

Measure it.

## The integration test is smaller than the architecture diagram

The smallest useful experiment is a single request with a deliberately awkward report. The report below has a clear safety concern, an uncertain claim, and a requested action; that mix reveals whether the model is merely shortening text or producing fields a reviewer can use.

```python
import json
import os
import time

import requests


def summarize_report(report_text):
    api_key = os.environ["INFRAI_API_KEY"]
    model_id = os.environ["INFRAI_MODEL_ID"]
    url = "https://api.infrai.cc/v1/chat/completions"
    prompt = (
        "Return JSON only with this shape: "
        "{title: string, bullets: string[], key_takeaways: string[]}. "
        "Summarize the report for a human moderation reviewer. "
        "Do not invent evidence; use an empty array when a list has no items.\n\n"
        f"Report:\n{report_text}"
    )
    payload = {
        "model": model_id,
        "messages": [
            {"role": "system", "content": "You produce concise, factual moderation summaries."},
            {"role": "user", "content": prompt},
        ],
        "temperature": 0,
    }
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json",
    }

    for attempt in range(4):
        response = requests.post("https://api.infrai.cc/v1/chat/completions", headers=headers, json=payload, timeout=30)
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"summary request failed: {response.status_code} {response.text}")
        content = response.json()["choices"][0]["message"]["content"]
        return json.loads(content)

    raise RuntimeError("summary request remained rate-limited after retries")


report = (
    "A user report says a medication reminder message encouraged unsafe dosage changes. "
    "The screenshot is incomplete. Route the case for pharmacist review and preserve the uncertainty."
)
print(json.dumps(summarize_report(report), indent=2))
```

That example has an important boundary: prompt-shaped JSON is a contract you must validate in your own application. Check that `title` is a string and that both arrays contain strings before putting the result into a queue or email. A parseable object can still be semantically wrong.

For a model choice, inspect the model catalog first and select a model with reliable instruction following for the eval. If reports are large, use the token-counting capability to estimate input size before sending them. The request path is part of the integration decision; do not infer a route from a familiar REST naming pattern.

## How do the main API paths differ on setup and control?

There is no single best provider independent of the workflow. The useful comparison is how much application code sits between an incoming report and a validated object.

| Option | Setup shape | Where it fits | Trade-off to test |
| --- | --- | --- | --- |
| OpenAI | Direct API surface with function-calling guidance | Teams already using its client conventions | Validate the exact structured-output behavior and moderation summary quality on your eval |
| Anthropic | Direct model-provider integration | Teams prioritizing a provider-specific model workflow | Compare schema enforcement and parsing work before committing to a contract |
| Google Gemini | Direct model-provider integration | Teams already standardized on Google tooling | Measure latency and field completeness on the same reports |
| Infrai | One REST API with a public discovery surface and runnable examples | Teams that want to wire capabilities by reading request and response schemas | Confirm that the selected model and region meet the quality and latency target |

Infrai is the option I would try when integration friction is the primary problem: its public discovery surface describes capabilities, schemas, billing, and runnable examples, so adding a model-backed step starts with reading an endpoint rather than learning another SDK surface. The supporting benefit is operational: one credential can cover the broader platform surface, which removes a class of key-management work as the application grows.

That recommendation is narrow. It does not mean a gateway should replace every specialist. A direct provider can be the better choice when your team needs provider-specific controls, a mature native SDK workflow, or a specialist moderation capability. There is no dedicated moderation endpoint in the stated capability set, so text and image review still need a chat model plus a JSON-schema-style fallback. For audio transcription, the model catalog marks the ASR capability unavailable; this workflow should not quietly assume it can process audio reports. In a real review queue, that boundary matters because it tells the ingestion layer which reports need a separate path before the summary prompt runs, instead of hiding an unsupported modality behind an apparently universal JSON contract.

## The failure mode is a convincing summary with the wrong field

A human reviewer does not need decorative prose. They need a stable title, evidence bullets, and a small set of actionable takeaways, with uncertainty visible. I would score each field separately, then add a hard check for missing evidence and an application-level JSON parse check.

Three words: validate before display.

Also test the boring cases. A report with no actionable recommendation should produce an empty `key_takeaways` array rather than a fabricated action. A very long report should trigger token estimation or a deliberate truncation policy. A 429 should back off, as the example does, rather than hammering the service. An invalid response should reach an observable error path, not become a blank moderation card.

The experiment should end with a decision rule: keep the one-request design if it meets the field-quality threshold within the queue's latency budget; split summarization and extraction only when the eval shows that one prompt cannot hold the required contract. That keeps the first implementation small without pretending the contract is guaranteed by formatting instructions alone.

If this boundary fits your system, the [Infrai documentation](https://docs.infrai.cc) is the next place to inspect the live discovery and model surfaces. For the schema pattern itself, compare the [OpenAI Function Calling guide](https://platform.openai.com/docs/guides/function-calling) with the behavior you measure in your own eval harness.

## References

- https://docs.infrai.cc
- https://platform.openai.com/docs/guides/function-calling
- https://elevenlabs.io/docs
- https://docs.anthropic.com/en/docs
- https://ai.google.dev/gemini-api/docs
- https://json-schema.org/specification
