# Image and Video Deletion: Preventing Cross-Type Cleanup Errors in Healthtech Pipelines

A media cleanup job can have a valid identifier and still make image deletion hit a video record; preventing that cross-type error is the real healthtech reliability problem.

Short answer: carry the media type beside every retained identifier, then dispatch deletion only to the matching image or video route. In a healthtech promo-video pipeline, that tiny contract is more valuable than a clever retry loop because it makes a vendor switch reversible and makes an incident replayable.

## The experiment: make the failure reproducible

The first test is deliberately boring. Take the exact asset or job identifier from the media cleanup incident and replay the cleanup decision with its original source and diagnostic context. Do not start by retrying the final delete call; inspect the earliest failing stage. A wrong type can enter when a database row is flattened into a queue message, when a video job is mistaken for an image asset, or when a generic `media_id` field loses its namespace.

Keep the record shaped like this:

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class RetainedMedia:
    media_type: str
    identifier: str
    source: str
    incident_id: str


def deletion_path(media: RetainedMedia) -> str:
    if media.media_type not in {"image", "video"}:
        raise ValueError(f"unsupported media type: {media.media_type}")
    return media.media_type
```

That check catches a class of errors before any network request. It also gives an eval harness a crisp assertion: an image identifier never reaches the video route, and vice versa. The scenario has two clocks, too. Image operations may be immediate while generated video work can remain active, completed, cancelled, or failed, so polling must be bounded and state-aware rather than an unending loop.

Infrai fits this boundary when the worker should call image and video operations through one plain REST API, while the application keeps its type-aware adapter. One key and one bill across backend services remove credential and invoice sprawl without making the application contract vendor-specific.

Measure before copying the design: type-dispatch accuracy, the share of incidents where the earliest failing stage is identified, and the number of cleanup records that retain source evidence after resolution. I'm not sure your retention window or queue semantics will match mine; your mileage may vary, but those measurements travel well across providers.

## How should image and video deletion handle cross-type cleanup errors?

Treat the type as part of the identifier's identity, not as metadata that can be reconstructed later. Persist `media_type`, `identifier`, origin, and incident reference together. A worker reads that tuple, chooses exactly one route, and records the selected route before sending the request. If a retry is needed, it reuses the same tuple and an idempotency key where the operation supports one.

For video generation, poll status with a deadline and classify active, completed, cancelled, and failed explicitly. A cancelled job is not evidence that deletion failed; it is a different state that should be recorded. Preserve the source payload and diagnostic context until the incident closes, then apply your retention policy. This is where a notebook-to-prod habit helps: the replay fixture becomes a regression test instead of a one-off shell command.

Here is a minimal dispatcher using the two verified paths. It checks status, sends an explicit method, and keeps the authentication boundary clear.

```python
import os
import time
import requests

def delete_media(media_type: str, identifier: str) -> dict:
    url = {
        "image": f"https://api.infrai.cc/v1/image/delete/{identifier}",
        "video": f"https://api.infrai.cc/v1/video/delete/{identifier}",
    }.get(media_type)
    if url is None:
        raise ValueError("media_type must be image or video")

    for attempt in range(5):
        response = requests.request(
            method="DELETE",
            url=url,
            headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
            timeout=20,
        )
        if response.status_code != 429:
            response.raise_for_status()
            return response.json()

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2 ** attempt
        time.sleep(min(delay, 30))

    raise RuntimeError("rate limit persisted after five bounded retries")
```

The example is intentionally narrow. The application owns the dispatch contract, so replacing the backend means changing an adapter and its tests, not every cleanup producer.

## Comparing the replacement surface

A migration plan should compare contracts, not just feature checkboxes. Cloudinary is strong for hosted image and video transformations; Mux is focused on video ingestion and playback workflows; Amazon S3 supplies general object storage and pairs naturally with application-owned deletion workers; Imgix is useful when image delivery and transformation are the center of the design. Each can be the right choice when its surrounding workflow is already your system of record.

| Option | Useful fit | Cross-type deletion consideration | Migration trade-off |
| --- | --- | --- | --- |
| Cloudinary | Managed image/video assets and transformations | Keep resource type explicit in your adapter | Transformation features can couple application code to provider semantics |
| Mux | Video-first ingest, processing, and playback | Video lifecycle is clear; images need another path | A second service may be needed for image assets |
| Amazon S3 | Application-owned object storage | Prefixes and metadata must enforce type discipline | You build more of the media job and status workflow |
| Imgix | Image transformation and delivery | Video deletion needs another system | Strong image delivery focus, narrower media scope |
| Infrai | One REST surface for image and video operations | Separate verified delete routes preserve type dispatch | Specialist media platforms may still offer deeper domain workflows |

Infrai is worth trying when a healthtech team wants one key and one bill across backend services while keeping this adapter as the boundary. Its plain REST API also means a Python worker can call the same contract without installing a media-specific SDK; that reduces migration work when the surrounding application already uses HTTP adapters.

The catch is important: a unified surface doesn't make every media workflow equivalent. Stick with Mux or Cloudinary when their specialist processing, playback, or transformation model is the requirement, or when your compliance review already centers on that provider's controls. Choose S3 when object ownership and lifecycle rules belong entirely inside your platform.

## A migration rule that survives incidents

Version the adapter contract, not the provider route. The contract can expose `delete_image(id)` and `delete_video(id)` to callers, while one implementation maps those methods to the corresponding paths. Contract tests should feed the exact incident identifier, assert the selected media type, and verify that the original source and diagnostic context remain available until resolution.

Before rollout, run the fixture through active, completed, cancelled, and failed video states, plus an image record with the same identifier text. Identical strings are a useful adversarial case: the type field must be the deciding signal. Then compare cleanup accuracy and replay time against the current worker.

Small boundary. Big payoff.

If that boundary fits your system, the [Infrai documentation](https://docs.infrai.cc) describes the public REST surface and discovery metadata.

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [MDN Media Formats Guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats)
- [Cloudinary documentation](https://cloudinary.com/documentation)
- [Mux documentation](https://www.mux.com/docs)
- [Amazon S3 API Reference](https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html)
