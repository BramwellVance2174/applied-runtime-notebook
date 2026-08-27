# Compatible Speech-to-Text Runtime: One-Key Model List Detection and EU/US Provider Fallback

Speech transcription for a private logistics knowledge base should be routed from a verified capability snapshot, not from an `OpenAI-compatible` label. The practical design is to resolve one logical model key against region-specific model catalogs, reject an unsupported request before audio leaves its boundary, and record usage against the tenant that caused the call. This adds a control-plane step, but it makes fallback behavior and per-tenant cost visible instead of accidental.

Short answer: build a small capability registry from provider model lists and explicit feature flags, select only candidates that support speech-to-text in the tenant's EU or US region, then emit a tenant-scoped usage event for every attempted transcription.

For a logistics team, the flow is concrete. A dispatcher uploads a voice note, the runtime identifies the tenant and required data region, and a policy layer maps a stable key such as `dispatch-transcription` to an eligible model. The resulting text can enter retrieval over that tenant's private manuals, delivery exceptions, and operating procedures. Audio routing and knowledge retrieval remain separate decisions; mixing them makes both access review and cost attribution harder.

## How should feature flags detect unsupported speech models across EU and US providers?

Treat compatibility as a protocol clue, not a capability guarantee. A familiar authentication scheme or response shape does not prove that a deployment accepts audio, exposes a transcription model, or serves the tenant's required region. The registry therefore needs evidence at three levels: the provider configuration says transcription is enabled, the current model catalog contains the configured model, and deployment metadata places that model in an allowed region.

This should be a positive check. Don't send audio to every configured endpoint and use errors as discovery; that turns a routine control-plane question into data-plane traffic and produces noisy fallback metrics. Refresh catalogs on a schedule, retain the last successful snapshot for a bounded period chosen by the team, and fail closed when the snapshot is too old for the application's policy. Your mileage may vary on that freshness window because release frequency and risk tolerance differ. The important part is to make it explicit.

The logical key is what application code depends on. Its candidates are deployment records, not marketing model families, and their order should reflect data residency and operational policy before throughput or unit price. In this design, an EU tenant can never fall through to a US deployment. That's deliberate.

No guesswork.

A minimal registry record needs no vendor-specific types:

| Field | Purpose | Failure prevented |
| --- | --- | --- |
| `logical_key` | Stable application-facing name | Prompt and pipeline changes during provider rotation |
| `capabilities` | Explicit set containing `speech_to_text` | Assuming all compatible endpoints support audio |
| `models` | Last observed model identifiers | Selecting a configured but unavailable model |
| `regions` | Allowed processing locations | Cross-region fallback |
| `catalog_observed_at` | Snapshot age input | Routing forever from stale discovery |
| `metering_mode` | Usage evidence expected from the adapter | Unattributed tenant spend |

Feature flags belong beside this evidence, not scattered through request handlers. One flag can disable transcription for a deployment; another can control whether a newly observed model is eligible for a small evaluation cohort. The flag doesn't establish capability by itself. It gates capability already established by catalog and deployment metadata.

## A runnable registry and routing boundary

The following Python example keeps discovery local so the selection rule is easy to test in a notebook. In production, adapters would populate the same records from authenticated model-list responses and deployment configuration. The example deliberately stops before any network call: provider request shapes differ, while tenant, region, and metering invariants should not.

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, timezone
from decimal import Decimal
from typing import FrozenSet, Iterable


SPEECH_TO_TEXT = "speech_to_text"


@dataclass(frozen=True)
class Deployment:
    deployment_id: str
    logical_key: str
    model: str
    regions: FrozenSet[str]
    capabilities: FrozenSet[str]
    catalog_models: FrozenSet[str]
    transcription_enabled: bool


@dataclass(frozen=True)
class TenantRequest:
    tenant_id: str
    region: str
    logical_key: str
    audio_seconds: Decimal


@dataclass(frozen=True)
class UsageEvent:
    tenant_id: str
    deployment_id: str
    model: str
    audio_seconds: Decimal
    outcome: str
    observed_at: datetime


class NoEligibleDeployment(Exception):
    pass


def eligible(deployment: Deployment, request: TenantRequest) -> bool:
    return all(
        (
            deployment.logical_key == request.logical_key,
            request.region in deployment.regions,
            SPEECH_TO_TEXT in deployment.capabilities,
            deployment.model in deployment.catalog_models,
            deployment.transcription_enabled,
        )
    )


def select_deployment(
    deployments: Iterable[Deployment], request: TenantRequest
) -> Deployment:
    for deployment in deployments:
        if eligible(deployment, request):
            return deployment
    raise NoEligibleDeployment(
        f"No speech deployment is eligible for logical key {request.logical_key!r} "
        f"in region {request.region!r}"
    )


def usage_event(
    request: TenantRequest, deployment: Deployment, outcome: str
) -> UsageEvent:
    return UsageEvent(
        tenant_id=request.tenant_id,
        deployment_id=deployment.deployment_id,
        model=deployment.model,
        audio_seconds=request.audio_seconds,
        outcome=outcome,
        observed_at=datetime.now(timezone.utc),
    )


deployments = [
    Deployment(
        deployment_id="eu-primary",
        logical_key="dispatch-transcription",
        model="stt-eu-1",
        regions=frozenset({"EU"}),
        capabilities=frozenset({SPEECH_TO_TEXT}),
        catalog_models=frozenset({"stt-eu-1", "chat-eu-1"}),
        transcription_enabled=True,
    ),
    Deployment(
        deployment_id="us-primary",
        logical_key="dispatch-transcription",
        model="stt-us-1",
        regions=frozenset({"US"}),
        capabilities=frozenset({SPEECH_TO_TEXT}),
        catalog_models=frozenset({"stt-us-1"}),
        transcription_enabled=True,
    ),
]

request = TenantRequest(
    tenant_id="carrier-042",
    region="EU",
    logical_key="dispatch-transcription",
    audio_seconds=Decimal("37.4"),
)
selected = select_deployment(deployments, request)
event = usage_event(request, selected, outcome="accepted")

assert selected.deployment_id == "eu-primary"
assert event.tenant_id == "carrier-042"
```

The order of `deployments` is the fallback policy. Keep it deterministic and version-controlled. A catalog refresh can add evidence, but it shouldn't silently reorder candidates. Promotion should happen only after the candidate passes the same transcription evaluation set used for the primary: representative accents, warehouse noise, tracking numbers, abbreviations, and the terms that retrieval actually depends on.

One subtle failure is worth calling out. If a model disappears from the latest valid catalog, `eligible` rejects it even though the static configuration still names it; if the catalog gains an unknown model, the runtime does not promote it merely because it is new. Those two checks keep discovery and rollout separate.

## Fallback is a policy state machine, not a retry loop

Fallback should distinguish “no eligible deployment” from a failed transcription attempt. The first is a control-plane outcome: configuration, flags, region, and catalog evidence produced no candidate. The second occurs after an eligible candidate receives a request. Combining them under a generic retry counter can send the same private recording through several processors while hiding which tenant incurred each attempt.

Define states such as `selected`, `submitted`, `accepted`, and `rejected`, and attach the tenant ID, logical key, deployment ID, model ID, region, capability-snapshot version, and request correlation ID at each transition. Never put raw audio or transcript text in ordinary logs. For private knowledge bases, metadata is usually enough to debug selection; content needs its own access controls and retention policy.

This boundary also clarifies streaming. Server-Sent Events are a one-way server-to-client channel with named events and reconnection behavior described by MDN. They can be useful for delivering transcription progress to a browser, but SSE says nothing about provider capability discovery or audio upload. Keep the UI event channel downstream of selection so a reconnect cannot trigger another provider submission.

Be strict here.

If the EU candidates are exhausted, return a typed “no regional transcription capacity” result and preserve the audio according to the application's approved retry policy. Do not treat a US candidate as the next item in a global list. The same rule protects a US-only tenant from an unintended EU route, even if both deployments share a logical model key.

## Per-tenant cost visibility starts before the invoice

An invoice groups charges too late for an AI application team that needs to explain why one carrier's voice-note workflow consumed more resources than another's. Emit an immutable usage event at the runtime boundary for every attempt, including rejected and canceled outcomes. Reconcile accepted events with provider-reported usage later, while retaining the application event as the record of which tenant initiated the work.

Audio duration is a useful workload measure, but it is not universally identical to a billable unit. Store both separately: observed input duration from the application and billable quantity returned by the adapter. I'm not sure a single normalization formula is defensible across every provider contract; the contract's current billing definition and an adapter-level reconciliation test are what resolve that uncertainty. Avoid estimating currency inside the router unless rates are effective-dated and reviewed. Prompt-cost awareness means preserving the raw dimensions first, not forcing every modality into token-shaped accounting.

Consider the accounting path when one EU tenant submits a recording, the primary candidate is selected, and the application later cancels the job. A provider reconciliation record without `tenant_id` can show the deployment's aggregate usage, while an application record that logs only completed transcripts can omit the attempt entirely. The runtime event closes that gap: it records the tenant, selected deployment, input duration, and canceled outcome at the boundary, then allows a later reconciliation process to attach the provider's billable quantity without rewriting the original event. This isn't a claim that every canceled request is billed. It is a way to retain enough evidence to apply the relevant contract rather than guessing from successful transcripts.

For the downstream knowledge-base answer, create a separate usage chain for transcription, embedding, retrieval, and generation. Link those events with one trace ID but retain the tenant on every row. This catches an easy accounting mistake: attributing generation to the tenant while leaving speech preprocessing in a shared bucket. It also lets an eval harness compare end-to-end answer quality against the full per-tenant workload, rather than optimizing word error rate in isolation.

The catch is the extra control-plane machinery. A small, single-region internal tool with one fixed transcription deployment may be better served by a static adapter and direct metering; a catalog cache, rollout flags, and policy state machine can cost more to operate than they prevent. Use the registry when multiple regions, providers, or tenant policies create real routing ambiguity. If processing locality must be legally interpreted rather than merely configured, keep the policy decision with privacy and security owners; architecture alone cannot determine whether a particular workflow meets HIPAA or another regulatory obligation. The HIPAA requirements themselves are codified in 45 CFR Part 164.

## Operational checks before audio reaches retrieval

Start in a notebook with a frozen catalog fixture and a table of tenant policies. Test positive selection, missing speech capability, absent model, disabled flag, stale snapshot, and region mismatch. Then run the same cases in CI against the pure selection function. The valuable assertion is not “fallback happened”; it is “only this ordered set of deployments was eligible under this snapshot and policy version.”

At deployment time, refresh catalogs without placing model-list calls on the user request path. Alert on snapshot age, the share of requests with no eligible deployment, selection changes by policy version, duplicate correlation IDs, and usage events that fail reconciliation. Review flag changes like code because disabling a capability changes production routing immediately. Sample transcript quality through an approved, redacted evaluation set, and keep the evaluator blind to which deployment produced each candidate.

Finally, exercise the handoff into the private knowledge base. Verify that transcript storage inherits tenant isolation, retrieval cannot cross tenant namespaces, and deletion covers audio, transcript, index entries, and evaluation artifacts. A transcription router can make region and cost decisions legible, but it cannot repair a retrieval layer that loses tenant context.

Preserve that context end to end.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164
