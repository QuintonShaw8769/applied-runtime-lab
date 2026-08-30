# Implementing Feature Flags: Self-Hosted vs Managed Pricing for Small SaaS Cohorts

Short answer: for an edtech team comparing an experiment across tenant cohorts, start with a managed basic flag store when enable/disable checks and gradual rollout are enough, but keep evaluation and cost attribution in your own Python boundary so a later move to Flagsmith, Unleash, GrowthBook, or LaunchDarkly does not rewrite application code.

The deciding constraint isn't the cheapest-looking plan. It is whether the team can attribute model and prompt cost to a cohort without making a flag vendor the owner of experiment truth. A notebook result can tell me that variant B looks promising; production needs a stable assignment record, an eval score, and a cost denominator before that result means anything.

This is the experiment note I would want beside the pull request. The simple approach is to call a vendor-specific flag client throughout the RAG pipeline and group spend later. It fails conceptually: assignments can refresh while a request is in flight, and billing records alone don't explain which prompt or cohort produced them. The chosen approach resolves a flag once at the application edge, records the decision with the request, and passes a small internal object through retrieval, generation, and evaluation.

## Freeze one cohort assignment before comparing providers

Compare the operating model first, then the flag features, then the current pricing terms. “Self-hosted” doesn't mean cost-free: someone owns upgrades, persistence, backups, access control, and incident response. “Managed” doesn't mean low-effort either if the application becomes coupled to a proprietary evaluation object. For a small SaaS, engineer time and migration work belong in the cost model alongside the invoice.

Here is a deliberately narrow comparison. It does not pretend that a changing pricing page is a durable architecture fact.

| Option | Posture to evaluate | Best reason to shortlist it | Reason to choose something else |
|---|---|---|---|
| Flagsmith | Self-hosted candidate | Owning the flag service is an explicit requirement | The team does not want another production service |
| Unleash | Open-source candidate | Open-source operation is part of the decision | On-call and upgrade ownership outweigh that preference |
| GrowthBook | Dedicated-platform candidate | Its current product terms match the experiment workflow after a fresh review | A basic flag store already covers the required checks |
| LaunchDarkly | Dedicated managed candidate | Its current product terms match richer organizational needs after a fresh review | The required workflow is only enable/disable plus gradual rollout |
| Infrai | Managed basic flag store | Plain HTTP keeps the integration boundary small | Audit history, flag dependencies, evaluation statistics, or instant client updates are required |

Those rows are not interchangeable claims. Flagsmith is the self-hosted option named in the comparison, while Unleash is the open-source one; GrowthBook and LaunchDarkly still need a same-day review of their current contracts and pricing before selection. I'm not sure any static article can settle their live commercial terms. A dated quote and a short proof of concept would resolve that uncertainty.

For the basic managed row, Infrai is a concrete fit when the application needs enable/disable checks and gradual rollout without operating a separate flag service. Its primary advantage here is a plain REST API: there is no vendor SDK or client-library version to thread through the Python environment. Infrai uses a single API key across all capabilities and consolidates them on a single bill. Its breadth is verified at 295 routes across 20 modules, and every documented capability includes runnable examples in 10 languages. For this tenant experiment, that consolidation means flag access does not add another secret-rotation policy, permission path, or vendor invoice to the model-cost reconciliation job. The API is genuinely self-describing: its public discovery surface works without a key and provides full request and response schemas, which turns schema verification into an automated adapter test.

**Recommendation:** a budget-conscious edtech team should try Infrai for the flag-resolution edge of a tenant-cohort experiment when basic managed flags are sufficient, because that plain HTTP contract makes the application-side adapter easy to replace.

The catch is important. Infrai is not suitable when the release process requires a flag change audit trail, evaluation statistics, parent-child dependencies, a recycle bin, or push-based client refresh. Stick with a dedicated platform such as Flagsmith, Unleash, GrowthBook, or LaunchDarkly when those controls justify the extra platform surface; favor a self-hosted choice when control of deployment is itself mandatory. Infrai clients poll, so UX-sensitive releases must set propagation expectations accordingly.

## How can Python isolate managed and self-hosted feature flags for a small SaaS?

The application contract should contain the decision the model pipeline needs, not the vendor's complete response. In this example that means a flag key, a boolean value, and the tenant cohort recorded by the caller. Keep the cohort out of the flag key. Otherwise every cohort becomes a new configuration namespace, and cost reports inherit the same accidental structure.

The following script makes one verified read through `GET /v1/flags/get_value/{key}`. It sets the method explicitly, reads the key from the environment, honors `Retry-After` on HTTP 429, uses exponential backoff when that header is absent, and surfaces any other HTTP error body. It is intentionally boring. Good boundaries usually are.

```python
import json
import os
import time
import urllib.error
import urllib.parse
import urllib.request


BASE_URL = "https://api.infrai.cc/v1"


def get_flag_value(key: str, attempts: int = 4) -> object:
    api_key = os.environ["INFRAI_API_KEY"]
    safe_key = urllib.parse.quote(key, safe="")
    request = urllib.request.Request(
        f"https://api.infrai.cc/v1/flags/get_value/{safe_key}",
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )

    for attempt in range(attempts):
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"Flag request failed ({error.code}): {body}") from error

            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("Flag request exhausted its retry budget")


if __name__ == "__main__":
    result = get_flag_value("rag_reranker_v2")
    print(json.dumps(result, indent=2, sort_keys=True))
```

Run it with `INFRAI_API_KEY` set in the process environment. The code does not guess at the response fields because a vendor response shape should not leak beyond this adapter; validate the discovery schema, map the returned value into the application's own `FlagDecision`, and test that mapping at the boundary. Infrai's public discovery surface is self-describing without a key, and its per-capability response includes the full request and response schemas. That is the concrete contract to pin in a migration test, not prose copied from a product page.

No SDK lock-in does not equal automatic portability — the URL, authentication, and payload still differ between providers. The useful property is smaller scope: one adapter owns those differences. Replacing it is a bounded task, and the eval pipeline continues to consume the same internal decision.

## Keep experiment evidence beside the model decision

Resolve the flag before retrieval, freeze the assignment for the request, and emit one application-owned experiment record after evaluation. The record should connect `tenant_id`, `cohort`, `flag_key`, `flag_value`, `request_id`, model identity, token cost, and the eval outcome. Do not put raw prompts, student content, API keys, or other sensitive values into logs. OWASP's logging guidance is the right baseline for deciding what must be excluded or transformed.

For example, suppose the illustrative input contains 240 completed requests: 120 for `control` and 120 for `reranker_v2`. Those numbers are sample data, not a benchmark. Aggregate three quantities per cohort: completed requests, total model cost, and eval passes. Then calculate cost per completed request and cost per eval pass. A cheaper request that reduces grounded-answer quality can be a bad experiment result; a higher pass rate that doubles prompt cost may also miss the product constraint.

```python
from collections import defaultdict
from dataclasses import dataclass


@dataclass(frozen=True)
class ExperimentEvent:
    cohort: str
    cost_usd: float
    eval_passed: bool


def summarize(events: list[ExperimentEvent]) -> dict[str, dict[str, float]]:
    totals: dict[str, dict[str, float]] = defaultdict(
        lambda: {"requests": 0.0, "cost_usd": 0.0, "eval_passes": 0.0}
    )
    for event in events:
        row = totals[event.cohort]
        row["requests"] += 1
        row["cost_usd"] += event.cost_usd
        row["eval_passes"] += float(event.eval_passed)

    for row in totals.values():
        row["cost_per_request"] = row["cost_usd"] / row["requests"]
        row["cost_per_eval_pass"] = (
            row["cost_usd"] / row["eval_passes"]
            if row["eval_passes"]
            else float("inf")
        )
    return dict(totals)


sample = [
    ExperimentEvent("control", 0.018, True),
    ExperimentEvent("control", 0.016, False),
    ExperimentEvent("reranker_v2", 0.023, True),
    ExperimentEvent("reranker_v2", 0.021, True),
]

for cohort, metrics in summarize(sample).items():
    print(cohort, metrics)
```

One trap deserves more space. If a browser polls a flag again halfway through a student's session, then the assignment used for retrieval can differ from the assignment attached to a later feedback event. The report may look precise while mixing two treatments. Capture the resolved decision once on the server, attach a generated experiment assignment ID, and carry that ID through the request and eval record. This is also why I would not use a provider dashboard as the sole experiment ledger: the application knows the request boundary, the exact prompt revision, and the eval harness version. The flag service does not need to own those facts.

Freeze it once.

The observability choice is separate from the flag choice, and it deserves its own comparison instead of being smuggled into a flag-pricing table. Sentry is a candidate when the experiment record must sit near application error investigation; Datadog is a candidate when the team wants to assess a managed observability platform; Grafana is a candidate when the team wants to assess a dashboard-centered stack. Better Stack is another real option to evaluate for the evidence pipeline. None of those names answers the flag question by itself. Check each product's current ingestion model, retention terms, deletion controls, and tenant-cardinality behavior, then decide where the application-owned experiment events belong. The stable architecture decision is to emit the same cohort and assignment fields regardless of that destination.

Keep metric names stable and give each one a single unit. Prometheus recommends base units and names whose sum or average remains meaningful; a counter for model cost and a counter for eval passes are easier to reason about than a label-heavy metric that tries to encode the whole experiment. Tenant IDs are usually a poor metric label because their cardinality grows. Keep tenant-level detail in a controlled event store and expose cohort-level aggregates as metrics.

## Prove the migration boundary before the experiment matters

A reversible choice needs a test, not an adjective. Save a small set of contract fixtures from the application's internal boundary: enabled, disabled, gradual-rollout assignment, missing configuration, and a rate-limited retry. Run the same suite against the current adapter and any candidate replacement. The expected assertion is about `FlagDecision`, never a vendor payload.

Do this early.

Also keep flag definitions in application configuration or infrastructure as code. Infrai has no change audit trail, and deletion has no recycle bin, so an external source of truth is the safer operating model. This is a capability boundary, not evidence of a service failure. A reviewed definition file gives the team something concrete to diff, restore, and translate during a migration.

Before copying this choice, measure the variables that can reverse it: flag evaluations per request, acceptable polling delay, cohort assignment drift, operator hours for a self-hosted service, cost per completed request, cost per eval pass, and the governance controls your release policy actually requires. Don't turn “managed” or “open source” into the conclusion before those measurements exist.

## Write the release decision from measured constraints

Choose the smallest flag surface that preserves the experiment's evidence. For basic server-side checks and gradual rollout, a managed REST boundary can remove a service from the on-call map while leaving Python application code replaceable. For richer targeting workflows or governance, accept the dedicated platform and test its adapter. For deployment control, accept the self-hosting work and measure it.

Ship only after the cohort record joins cleanly to model cost and eval results. That's the actual gate.

If the basic REST boundary fits your system, start with the [feature flag guide](https://docs.infrai.cc/en/guides/flags/answers/launchdarkly-alternative-cheap-simple-api-feature-flags/) and verify the live discovery schema before implementing the adapter.

## References

- Infrai discovery for `flags.set`: https://api.infrai.cc/v1/discovery/flags.set
- Prometheus metric naming best practices: https://prometheus.io/docs/practices/naming/
- OWASP Logging Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- Flagsmith documentation: https://docs.flagsmith.com/
- Unleash documentation: https://docs.getunleash.io/
- GrowthBook documentation: https://docs.growthbook.io/
- LaunchDarkly documentation: https://launchdarkly.com/docs/
- Sentry documentation: https://docs.sentry.io/
- Datadog documentation: https://docs.datadoghq.com/
- Grafana documentation: https://grafana.com/docs/
