# Healthtech In-App Chatbot Moderation: Portable LLM JSON-Schema APIs

Short answer: for a low-risk healthtech catalog assistant, choose the API that can return a strictly validated JSON safety decision through a replaceable chat interface, then prove it on a product-shaped evaluation set. A dedicated moderation endpoint is preferable when the policy is specialized or the cost of a false decision is high. Provider portability is the decision axis: the safety controller should survive a model or API change without rewriting catalog enrichment.

The experiment starts with messy descriptions, not a clean benchmark. A supplier might write “cold chain pack, keep away from children,” while another description contains a dosage claim, a copied warning, or half a sentence in another language. The chatbot can enrich fields such as normalized product name, storage condition, and searchable attributes, but it must not quietly turn an unsafe request into catalog data.

I would measure the gate before measuring the prose. A notebook can show that a response parses as JSON. Production needs to show that the decision is the right one for the product.

## What should a portable in-app chatbot API require for LLM JSON-schema moderation?

The application should own the policy. Give a classifier a small schema such as `allow`, `category`, and `reason`; validate it in Python; and stop enrichment when the result is invalid, denied, or uncertain. Run a second check on the proposed catalog fields before they are shown to a user or written to a system of record. This is a basic control, not a claim that structured output makes a model a safety expert.

The provider boundary matters more than the provider name. Keep the classifier prompt, schema, policy version, and retry budget in application code. Ask the selected API for a typed response through the same adapter used by the evaluation harness. If the team changes model access later, the fixture set and controller contract should remain intact. That separation also makes a notebook useful after the first afternoon: the notebook can replay the same records against a candidate, while the application still owns the final allow-or-review decision.

Keep it boring.

The simple alternative is to put “be safe” in the enrichment prompt and inspect the assistant's prose. That is easy to demo and difficult to enforce. A refusal can be polite while still lacking a machine-readable reason, and a valid-looking catalog field can carry a claim the product policy forbids. Two narrow calls add latency and tokens, but they create decisions that can be scored, logged, and reviewed.

Keep categories stable and boring. For this scenario, `allowed`, `medical_claim`, `unsafe_instruction`, `privacy`, and `other` may be a useful starting point, but the product policy owner must define them. JSON Schema constrains shape; it does not define acceptable health content. A description saying “stop the process before it consumes memory” is a benign software instruction, not evidence of violent intent. Conversely, an instruction to falsify a clinical claim should not pass merely because every field is syntactically valid.

## The catalog fixture that changed the design

The most useful fixture family is a set of near-neighbors. Put a harmless storage warning beside a dosage promise, a quoted unsafe sentence beside a direct request, and an obfuscated version beside its plain equivalent. Add the languages and abbreviations the catalog actually receives. Label the expected action before running a model, and record the policy version with every result. For example, take four records that all mention sleep: a refrigerator instruction for a diagnostic kit, a claim that a supplement will cure insomnia, a quotation from a customer complaint, and a request to fabricate evidence for the claim. The words overlap, but the actions are different. The first may be allowed, the second may require review or denial, the third needs quotation-aware handling, and the fourth is a clear policy case. A keyword filter collapses those distinctions; an unconstrained chat reply hides them. A typed verdict makes the disagreement visible, but the reviewers still have to decide whether partial catalog extraction is allowed and what happens to the fields that were not approved.

One particularly important case is a description that mixes useful facts with a prohibited claim: “refrigerate after opening; guaranteed to cure insomnia.” The enrichment job should preserve the first fact only if policy permits partial extraction; otherwise it should send the whole item to review. That choice belongs in the expected label. It should not be invented by whichever model happens to be selected for the bake-off.

This is where notebook-to-prod work gets real. When the model disagrees with a reviewer, inspect whether the category is ambiguous, whether the prompt exposed unnecessary context, and whether the expected action is consistent across similar records. Don't patch one example by adding a keyword. Freeze the whole family, update the policy deliberately, and rerun it.

The boundary is the product.

OWASP's Top 10 for Large Language Model Applications is useful for threat modeling prompt injection and unsafe output handling. It is not a complete healthtech moderation taxonomy. I am not sure which model will give the best precision-recall balance for a particular supplier mix; your mileage may vary by language, formatting, and the proportion of copied text. Only a labeled fixture set can answer that.

## A Python adapter that keeps the API replaceable

The example below deliberately contains no vendor-specific route or SDK. The adapter can wrap an HTTP client, a self-hosted gateway, or another compatible service, while the controller and tests keep the same contract. That makes portability testable rather than aspirational.

```python
import json
from dataclasses import dataclass
from typing import Any, Protocol


class ChatProvider(Protocol):
    def complete(self, *, prompt: str, schema: dict[str, Any]) -> str:
        """Return one response constrained by the supplied schema."""


@dataclass(frozen=True)
class SafetyDecision:
    allow: bool
    category: str
    reason: str


def parse_decision(raw: str) -> SafetyDecision | None:
    try:
        value = json.loads(raw)
    except json.JSONDecodeError:
        return None

    if not isinstance(value, dict):
        return None
    if not isinstance(value.get("allow"), bool):
        return None
    if not isinstance(value.get("category"), str):
        return None
    if not isinstance(value.get("reason"), str):
        return None
    return SafetyDecision(
        allow=value["allow"],
        category=value["category"],
        reason=value["reason"],
    )


DECISION_SCHEMA = {
    "type": "object",
    "additionalProperties": False,
    "required": ["allow", "category", "reason"],
    "properties": {
        "allow": {"type": "boolean"},
        "category": {"type": "string"},
        "reason": {"type": "string"},
    },
}


def screen_description(provider: ChatProvider, description: str) -> SafetyDecision:
    prompt = (
        "Classify this healthtech catalog description under the application policy. "
        "Return only the requested object. If uncertain, set allow to false and "
        f"category to other.\n\nDescription: {description}"
    )
    decision = parse_decision(
        provider.complete(prompt=prompt, schema=DECISION_SCHEMA)
    )
    if decision is None:
        return SafetyDecision(False, "other", "Invalid structured safety response")
    return decision


def may_enrich(provider: ChatProvider, description: str) -> bool:
    decision = screen_description(provider, description)
    return decision.allow and decision.category == "allowed"
```

The fallback is intentionally conservative. A malformed response is not silently treated as approval, and the controller has no dependency on a particular client library. In a real service I would also validate the schema with a JSON Schema implementation, cap retries, attach a request identifier, and keep the raw decision in a restricted audit store. I would never log sensitive free text by default.

The same adapter should serve the eval harness. Feed it the frozen fixture set, collect category-level recall, false positives on benign warnings, parse success, p50 and p95 latency, and tokens per classification. Then run the exact prompt and schema against each candidate. Provider portability is not “all APIs look similar”; it is the ability to compare candidates without changing the test or policy layer.

## Where this pattern should lose

The catch is that a general chat API plus a JSON schema is not suitable as the sole control for a child-facing product, imminent self-harm escalation, regulated clinical advice, or a workflow with legal reporting duties. Use a specialized safety service, qualified review, and domain escalation for those cases. A dedicated moderation interface may also be the better choice when its taxonomy and operational controls map directly to the product's obligations.

The two-pass version is also a poor fit for a very tight latency budget: one classification call before generation and another after it are real work. A product can narrow the post-check to fields that leave the trust boundary, but that is a policy decision to test, not a shortcut to assume. Multimodal inputs need their own evaluation and controls; a text schema does not prove that audio, images, or documents are covered.

Basic moderation has a failure mode that dashboards often hide: a transport success can coexist with a wrong decision. Track blocked unsafe cases, allowed benign cases, ambiguous cases sent to review, schema failures, model changes, and user-visible latency. A high parse rate is useful. It is not the safety metric.

## The decision rule for a healthtech catalog team

Before adopting an API, write the go/no-go rule. Require a valid typed decision, a category-level threshold agreed with policy owners, an explicit false-positive budget for ordinary catalog language, and an end-to-end latency budget that includes both checks. Store the model identifier, prompt version, schema version, and fixture-set version with every evaluation run.

Then compare providers through the adapter using identical fixtures and retry behavior. Keep the decision reversible until the results show that portability is real and the error profile is acceptable. For a low-risk catalog assistant, a general chat API with typed moderation can be a defensible starting point. It is not a universal safety architecture, and it should not be selected because a demo produced tidy JSON.

## References

- OWASP Top 10 for Large Language Model Applications: https://owasp.org/www-project-top-10-for-large-language-model-applications/
- OpenRouter documentation: https://openrouter.ai/docs

## Further reading

- OWASP Top 10 for Large Language Model Applications: https://owasp.org/www-project-top-10-for-large-language-model-applications/
- OpenRouter documentation: https://openrouter.ai/docs
