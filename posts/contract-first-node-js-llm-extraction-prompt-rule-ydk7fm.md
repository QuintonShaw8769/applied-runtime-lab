# Contract-First Node.js LLM Extraction: Prompt Rules for JSON Schema Nulls and Enum Values

Short answer: validate the generated object before application code sees it, then classify missing fields, null values, and enum mismatches as separate outcomes instead of asking one broad extraction prompt to “fix the JSON.”

The deciding constraint is evidence. A second model call can correct a shape violation when the source already contains the answer; it cannot recover a fact that was never present. For a Node.js service, the production boundary should therefore be boring and explicit: parse, validate without coercion, record exact paths, and retry only the failure classes that another generation can actually repair. A small Python eval harness can establish that contract before the notebook experiment becomes a deployed endpoint.

## What should a Node.js LLM JSON schema extraction prompt do with null values?

Start with field semantics, not prompt adjectives. “Return valid JSON” says nothing about whether a required property may contain `null`, whether an empty string means unknown, or what happens when the input uses a category outside the enum. Those decisions belong to the application contract. The prompt merely communicates them to the model; the validator enforces them.

A useful contract distinguishes presence from knowledge. If `customer_tier` must always exist but the source may omit the tier, require the property and allow `null`. If `priority` drives automation, require the property and accept only the declared enum literals. If `summary` is required and the evidence contains enough text to summarize, a missing key is a structural failure. These are three different states even though all three can surface as a rejected object.

| Output state | Meaning at the boundary | Default handling |
|---|---|---|
| Required key absent | The object has the wrong shape | Reject; retry only if the prompt declared the key |
| Required key present as `null` | The contract represents unknown explicitly | Accept only for a nullable field |
| Unknown enum literal | The output vocabulary differs from the contract | Reject without guessing a replacement |
| Wrong JSON type | The value cannot be consumed as declared | Reject with its exact property path |

This matters in JavaScript because a truthiness check collapses cases that the data model needs to preserve. An absent property, `null`, `false`, `0`, and `""` are not interchangeable. A Node.js implementation can use any standards-aware JSON Schema validator, but its observable behavior should be tested independently of that library: no silent defaults, no fuzzy enum mapping, and no conversion of an evidence gap into a plausible-looking value.

Keep the prompt narrow. Name every required key, state exactly which fields permit `null`, list the allowed enum strings with their spelling and case, and tell the model not to infer facts absent from the source. Then include only the source passage and the contract needed for this extraction. Replaying a long conversation adds tokens and can reintroduce stale instructions.

Prompt wording helps. It is not enforcement.

## Separate contract failures from evidence gaps

The tempting approach is one repair loop: if validation fails, append the error and regenerate. It is simple, and it fails in a costly way. The loop treats an incomplete source as if it were a formatting problem, so it spends another call asking for information the model does not have. Worse, the retry can produce a valid-looking guess and turn an honest unknown into bad data.

Classify before retrying. Syntax failures mean the response cannot be parsed. Shape failures mean a required key or declared type is wrong. Vocabulary failures mean a value falls outside an enum. Evidence gaps mean the source does not support the requested fact. Only the first three might be repairable by another model call, and even then the retry should preserve the original evidence and contract rather than invite a reinterpretation. An evidence gap should take the representation chosen by the schema, often a required nullable property, or go to review if the workflow cannot represent unknown.

Here is a concrete failure sequence. Suppose a support note says, “Checkout stalled after the address change; the customer did not state urgency.” The requested object requires `summary`, `priority`, and `customer_tier`; `customer_tier` is nullable, while `priority` must be one of `low`, `medium`, or `high`. An output that omits `customer_tier` has a shape error. An output containing `"customer_tier": null` follows that part of the contract. An output containing `"priority": "urgent"` has a vocabulary error. Yet the deeper problem is that the note supplies no urgency at all. Retrying until the model picks an allowed label would satisfy the enum while fabricating evidence. The schema should either permit an explicit unknown state for priority or the record should be withheld from automation.

This is where aggregate “valid JSON rate” becomes misleading. It rewards coercion and repeated sampling even when semantic fidelity gets worse. The eval needs counts by failure class plus a check that every non-null extracted value is supported by the input. I’m not sure a fully automatic entailment check is reliable for every domain; labeled fixtures and review of ambiguous cases are what would resolve that uncertainty for a particular corpus.

## A focused Python harness for the production boundary

The following harness does not call a model or depend on a vendor SDK. That is deliberate. It makes the contract executable in a notebook, and the same fixtures can then be applied to the Node.js validator used in production. The important behavior is language-independent: membership tests detect absent keys, nullable fields are declared explicitly, enum comparisons are exact, and each violation has a stable code and path.

```python
import json
from dataclasses import dataclass
from typing import Any


PRIORITIES = {"low", "medium", "high"}


@dataclass(frozen=True)
class Violation:
    path: str
    code: str
    message: str


def validate_extraction(raw: str) -> tuple[dict[str, Any] | None, list[Violation]]:
    try:
        value = json.loads(raw)
    except json.JSONDecodeError as error:
        return None, [Violation("$", "invalid_json", str(error))]

    if not isinstance(value, dict):
        return None, [Violation("$", "wrong_type", "expected an object")]

    violations: list[Violation] = []
    required = ("summary", "priority", "customer_tier")

    for key in required:
        if key not in value:
            violations.append(
                Violation(f"$.{key}", "missing_field", "required property is absent")
            )

    if "summary" in value and not isinstance(value["summary"], str):
        violations.append(
            Violation("$.summary", "wrong_type", "expected a string")
        )

    if "priority" in value:
        priority = value["priority"]
        if not isinstance(priority, str):
            violations.append(
                Violation("$.priority", "wrong_type", "expected a string")
            )
        elif priority not in PRIORITIES:
            violations.append(
                Violation(
                    "$.priority",
                    "enum_mismatch",
                    "expected low, medium, or high",
                )
            )

    if "customer_tier" in value:
        tier = value["customer_tier"]
        if tier is not None and not isinstance(tier, str):
            violations.append(
                Violation(
                    "$.customer_tier",
                    "wrong_type",
                    "expected a string or null",
                )
            )

    return value, violations
```

Do not pass the validator’s prose directly into an unconstrained conversation. Build a repair request from the stable code, the property path, the relevant rule, and the unchanged source. For `missing_field`, that might mean asking for the same object with all required keys. For `enum_mismatch`, repeat the allowed literals but also check whether the source supports any of them. A loop without a cap is an availability and token-cost problem disguised as error handling.

Cap the retry count.

The Node.js boundary should return the same logical result as this harness: either a typed candidate with no violations or a candidate plus a complete list of violations. Do not throw away the candidate after validation. Retaining it in controlled, redacted telemetry lets the evaluation distinguish a parser failure from one wrong field, while the application still blocks invalid data from reaching business logic.

## Measure prompt repair as a pipeline

Before revising the prompt, freeze a compact fixture set. Include a fully valid object, each required key missing in turn, legal and illegal nulls, every allowed enum literal, an unknown literal, a top-level array, malformed JSON, and source text with no supported answer. The last case is crucial because it tests whether the system prefers an explicit unknown over invention.

Measure exact contract pass rate on the first attempt, recovery rate by violation code, unsupported-value rate after repair, end-to-end latency, and total tokens across all attempts. If a new prompt raises syntactic success but increases unsupported enum selections, it is a regression despite the prettier JSON. Likewise, a retry strategy that rescues a small class of missing keys while repeating every evidence gap may not deserve its extra latency and prompt cost.

Keep the attempts separate.

Run the same fixtures whenever the model, prompt, schema, parsing library, or adapter changes. This is the notebook-to-production bridge: the notebook discovers a candidate instruction; the frozen eval defines acceptable behavior; the service reports the same violation taxonomy in production. Log contract version, model configuration, attempt number, latency, token usage, and violation codes. Avoid logging raw source text by default because extraction inputs often contain sensitive material.

One short rule helps: **never let repair change the meaning of unknown.**

## When strict rejection is the wrong policy

Strict rejection is appropriate when an enum triggers an irreversible or privileged action, when downstream code cannot represent uncertainty, or when a fabricated value would be materially harmful. It is not suitable for every extraction workflow. If a human reviewer sees the source and expects best-effort suggestions, preserve the raw candidate, label invalid fields as unverified, and send the record to review rather than discarding all useful fields.

The catch is operational complexity. Field-level acceptance requires provenance, a UI that exposes uncertainty, and downstream consumers that cannot accidentally treat an unverified suggestion as approved data. Stick with whole-object rejection when those safeguards do not exist. Also skip model retries when a deterministic parser can correct transport-only syntax without changing meaning, but do not use cleanup code to add a missing fact or translate an unknown category.

Before copying this design, decide what “unknown” means for every field, identify which violations are genuinely retryable, and run the frozen cases through both the Python harness and the Node.js production validator. Ship only when they agree on every case and the evaluation shows that repairs improve contract compliance without inventing evidence.

## References

- https://platform.openai.com/docs/guides/embeddings
- https://elevenlabs.io/docs
