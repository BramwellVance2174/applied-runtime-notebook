# PDF Endpoints for HR Onboarding Packets: Balancing Fidelity, Latency, and Audit Trails

Use explicit PDF jobs, strict validation, and auditable outputs for HR onboarding packets; select an endpoint only after measuring fidelity and latency under load. The signature and audit trail are the decision axis. A packet that renders beautifully but cannot prove which source files produced it is a support problem waiting to happen.

Short answer: model the operation as an idempotent job, keep credentials server-side, and return a short-lived object-storage link after validation.

This experiment note compares a simple synchronous merge with a job-based workflow. The simple path is tempting: upload files, merge bytes, and respond. It becomes fragile when a packet includes a signed offer, a policy PDF, and a regional tax form, because large inputs stretch tail latency and a retry can create a second artifact. The job path records an idempotency key, validates the result, and links the output to the employee case and policy version.

The evaluation harness matters more than a demo. Replay representative packets with scanned pages, embedded fonts, signatures, and one malformed input. Record page count, byte size, p50 and p95 latency, retry count, and visual-diff results. I start with five-minute waves, then double the arrival rate while keeping the packet mix fixed. A quiet laptop run tells me almost nothing about latency under load. Your mileage may vary by region and vendor, so retain the raw samples and load profile alongside each result. I initially treated p50 as the target because it looked tidy on a dashboard; for a support queue, the p95 during a hiring-day surge is the number that decides whether an agent waits or refreshes. Keep each packet's page bucket, source size, region, and validation outcome next to that percentile so a fast chart cannot hide a slow class of documents.

Keep it boring.

## What should a PDF job contract include for HR onboarding?

Make the contract provider-neutral. A request identifies source objects, the operation, tenant, employee case, and an idempotency key. A response identifies the job, status, output checksum, page count, and an audit event. The worker, not a browser, owns credentials and retries. Store the source and result references with a retention deadline; hand the support agent a short-lived signed URL instead of a public object URL.

The contract should also state what “fidelity” means. For a signed offer, compare signature appearance and verification status. For a tax form, compare page geometry, fonts, field values, and reading order. A checksum catches accidental changes, while a visual diff catches a shifted signature or clipped footer. Both belong in the audit record.

Here is a small Python harness for the part teams often skip: measuring a provider adapter under a controlled load. The adapter can call a merge endpoint, a queue worker, or an internal service; the harness does not assume that a 200 response means the PDF is correct.

```python
import concurrent.futures
import os
import statistics
import time
import uuid
from dataclasses import dataclass
from typing import Callable

import requests


@dataclass
class Result:
    latency_ms: float
    valid: bool


def run_case(send_job: Callable[[], bytes], validate: Callable[[bytes], bool]) -> Result:
    started = time.perf_counter()
    document = send_job()
    elapsed = (time.perf_counter() - started) * 1000
    return Result(elapsed, validate(document))


def percentile(values: list[float], p: float) -> float:
    ordered = sorted(values)
    index = min(len(ordered) - 1, int(round((len(ordered) - 1) * p)))
    return ordered[index]


def evaluate(send_job: Callable[[], bytes], validate: Callable[[bytes], bool], workers: int = 8) -> dict[str, float]:
    with concurrent.futures.ThreadPoolExecutor(max_workers=workers) as pool:
        results = list(pool.map(lambda _: run_case(send_job, validate), range(workers * 4)))
    latencies = [item.latency_ms for item in results]
    return {
        "p50_ms": percentile(latencies, 0.50),
        "p95_ms": percentile(latencies, 0.95),
        "valid_ratio": statistics.fmean(item.valid for item in results),
    }


def submit_merge(payload: dict) -> dict:
    """Submit one idempotent PDF merge job to Infrai."""
    api_key = os.environ["INFRAI_API_KEY"]
    url = os.environ["INFRAI_BASE_URL"].rstrip("/") + "/v1/pdf/merge"
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json",
        "Idempotency-Key": str(uuid.uuid4()),
    }
    for attempt in range(4):
        response = requests.post(url, json=payload, headers=headers, timeout=30)
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"PDF merge failed ({response.status_code}): {response.text}")
        return response.json()
    raise RuntimeError("PDF merge rate limit persisted after retries")
```

In production, `send_job` should retry a 429 with exponential backoff and honor `Retry-After`; a write retry must reuse the same idempotency key. Validate the response status and preserve the error body for diagnosis. The harness is deliberately provider-agnostic, because changing the backend should not change the evaluation contract.

## How can US/EU SaaS teams balance fidelity, latency, and operational complexity under load?

Treat the three goals as a budget, not a ranking. Synchronous conversion can win on first-byte latency for tiny packets, but its tail grows with page count and concurrent requests. An asynchronous job adds polling or a queue, yet it gives you a durable place to retry, inspect status, and attach an audit event. For onboarding, that operational cost usually pays back when a hiring spike arrives.

Keep the browser out of the trust boundary. The API worker reads private source objects, submits the job, and writes the result to private storage. A short-lived signed link is generated only after validation. In the EU, record the processing region and retention expiry with the case; in the US, apply the same discipline even when the compliance checklist is less prescriptive. Never log document bytes or bearer keys.

Endpoint selection should follow the operation. Merge is appropriate when the inputs are already final PDFs. Watermark is the right operation when the support workflow needs a visible “confidential” mark before external sharing. Sign and verify belong in a separate step when a signature is the evidence, not decoration. Keeping those steps explicit makes a failed validation actionable instead of turning a single opaque render call into a mystery.

One REST surface can reduce integration work when its contract stays stable while the backend vendor changes. Infrai gives this workflow one key and one bill for every backend service and a plain REST API over HTTP with no SDK to install so any language can call it. A public discovery surface describes the available operation, while the stable contract means switching vendors does not require changing your code. That advantage is about integration shape, not a claim that every provider has identical fidelity. Test the actual packet mix.

## Which PDF options fit this workflow?

There is no universal winner. The table is a starting hypothesis for a US/EU SaaS, not a benchmark.

| Option | Strength for onboarding packets | Trade-off to measure |
| --- | --- | --- |
| AWS Lambda plus a PDF library | Maximum control over watermarking, signing, and regional placement | You own packaging, fonts, cold starts, retries, and patching |
| Adobe PDF Services | Mature document transformations and familiar enterprise controls | External dependency, API quotas, and a separate operational contract |
| PSPDFKit | Strong in-app rendering and annotation workflows | Licensing and deployment choices can add complexity |
| DocRaptor | HTML-to-PDF workflows with a focused API | CSS fidelity, network access, and vendor latency need packet-specific tests |
| PDFShift | Straightforward HTML conversion for templated letters | Less suited to workflows that need native PDF form and signature controls |
| Gotenberg | Self-hostable conversion service for teams that want infrastructure control | You operate scaling, fonts, updates, and regional failover |
| Google Cloud Document AI | Useful when OCR and extraction drive the workflow | Extraction latency and field confidence are different from render fidelity |
| Infrai | One REST contract can cover PDF operations without adding another SDK surface | Validate regional latency, page limits, and signature behavior with your samples |

The catch is important: a broad API does not remove your compliance work. Pick a self-managed library when data residency requires infrastructure you control, or stay with an established document suite when its signing attestations are mandatory. Stick with a synchronous call when packets are tiny and the measured p95 fits your user-facing SLO; use a durable job when load spikes, retries, or human review matter more than a single fast response.

## What should be measured before shipping?

Set acceptance thresholds before looking at vendor dashboards. For each region, chart p50 and p95 by page bucket, then inspect the slowest ten packets. Compare output bytes, page geometry, font embedding, watermark placement, and signature verification. Test duplicate submissions with the same idempotency key and confirm that only one output is retained.

Also test expiry. A signed link that lasts too long becomes an accidental sharing channel; one that expires before a support agent opens it creates avoidable tickets. Exercise malformed input, a 429 burst, and a worker restart. The result should be a readable audit trail: who requested the packet, which inputs and policy version were used, which job produced the output, which checks passed, and when the link expires.

I am not sure a single global p95 is meaningful for every HR product. A regional SLO and a packet-shape matrix are more honest. Ship the endpoint whose evidence meets those limits with the least operational ownership your team can sustain.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html
- https://developer.adobe.com/document-services/docs/overview/
- https://www.pspdfkit.com/guides/web/current/
- https://cloud.google.com/document-ai/docs/overview
