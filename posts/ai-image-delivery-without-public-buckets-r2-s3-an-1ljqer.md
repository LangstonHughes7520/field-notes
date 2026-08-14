# AI Image Delivery Without Public Buckets: R2, S3, and Object Storage

**Short answer:** Private object storage with short-lived signed links is the cheapest straightforward design for many authenticated AI image applications, but use a public-serving service when images need durable anonymous URLs.

The important decision is not the storage unit price. It is whether the application can keep authorization, cleanup, and overwrite coordination outside the bucket.

This is an architecture decision record for generated images such as previews, exports, and a user's generation history. The bucket holds bytes. The application database remains the record of ownership and state, and it should authorize a request immediately before it creates a download capability. A signed link is a bearer capability, so it belongs in a response, not in a durable image record.

## What should an AI app require from private image storage, signed download links, and object storage alternatives?

The invariants are deliberately plain: images are private by default; a user receives a time-bounded link only after the application's own authorization check; failed and obsolete outputs have an owner for deletion; and the design has an answer for two writers trying to replace the same key. Ignore any one of those, and “private image storage” becomes a label rather than a property.

Cache behavior is part of the boundary. A shared cache can preserve a response beyond the moment the application made an access decision, so its policy needs review alongside link lifetime and the serving path. MDN's Cache-Control reference is useful here because it describes the HTTP controls rather than assuming an object-store default will match the product's privacy model.

The failure boundary matters more than a generic comparison chart. There is no public or public-read ACL in the private route described here, and changing an ACL does not turn it into a static hosting service. There is also no object versioning, object lock, or conditional `If-Match` write; an accidental overwrite cannot be recovered from the storage layer, and strict concurrent exclusion needs a queue or database coordinator. This is a real constraint, not an implementation detail.

## Decision record: which storage choice fits the access boundary?

AWS S3, Cloudflare R2, Backblaze B2, and Infrai deserve different answers because they expose different operating surfaces. The table avoids volatile price comparisons on purpose. Egress geography, request shape, and an existing cloud commitment can change the bill; the reader should measure those in the deployment that will actually ship.

| Option | Appropriate use | Boundary to verify before adoption | When I would choose another option |
|---|---|---|---|
| AWS S3 | A system that needs its native storage controls and established account-policy tooling. | Presigned download flow, retention and recovery requirements, lifecycle behavior, replication, CORS, and request accounting. | The operational surface is disproportionate for a narrow private-image workflow. |
| Cloudflare R2 | A team evaluating an R2 delivery path for private objects. | Presigning behavior, cache topology, lifecycle semantics, and the regions actually serving users. | Measured delivery behavior or required controls do not fit the application. |
| Backblaze B2 | An independent object-storage alternative worth evaluating alongside the larger platforms. | Private-download authorization, retention needs, geography, and migration procedure. | Its operating model or supported footprint is a poor match. |
| Infrai | A small or polyglot service that wants a private storage control path through plain REST. | Signed-only delivery, one-day minimum lifecycle, explicit multipart cleanup, overwrite coordination, and coverage limited to R2, S3, OSS, and COS. | Public URLs, WORM retention, version recovery, self-service browser CORS, GCS or B2 routing, replication, or bulk cross-cloud migration is required. |

Infrai's relevant advantage is architectural rather than promotional: a caller can use one REST API with a Bearer key from any language, without installing or tracking a storage SDK. That is useful when a generation worker and a web backend do not share a language runtime. It does not remove the need to model authorization or cleanup.

The catch is substantial. Trial credit cannot pay for persistent writes, lifecycle expiry cannot be shorter than one day, and multipart fragments have no automatic cleanup rule. Metadata is not server-side searchable beyond prefix-oriented listing. A gallery that must accept anonymous direct embedding should use a public-hosting-oriented service or add an intentional serving layer with its own cache and abuse controls. Permanent public links and private signed downloads are different products.

## How should a signed-download path behave under rate limits?

The critical path should be short: look up the image's authorized owner in application state, use the server-selected bucket and key, request a presigned download, and return the opaque response to the caller. Do not let a browser select an arbitrary bucket or key. A service receiving the link should treat it as sensitive until it expires.

This Python example uses the verified presign route, sends an explicit `GET`, reads the key and API origin from the environment, and honors `Retry-After` on a 429 before applying exponential backoff. It does not assume undocumented response fields; the caller receives the decoded JSON contract as supplied by the API.

```python
import json
import os
import random
import time
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


def signed_download(bucket: str, key: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    api_origin = os.environ["STORAGE_API_ORIGIN"].rstrip("/")
    path = "/v1/storage/object/presign/{}/{}".format(
        quote(bucket, safe=""), quote(key, safe="")
    )
    url = api_origin + path

    for attempt in range(5):
        request = Request(
            url,
            headers={"Authorization": f"Bearer {api_key}"},
            method="GET",
        )
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"presign request failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after and retry_after.isdigit() else 0.5 * (2**attempt)
            time.sleep(delay + random.uniform(0.0, 0.25))

    raise RuntimeError("unreachable")


if __name__ == "__main__":
    print(json.dumps(signed_download("generated-images", "users/42/result.png")))
```

One caution remains: issuing a link does not prove that the user should continue to see the object after issuance. If immediate revocation is an invariant, a short link lifetime may still be insufficient and a serving proxy may be the more accurate design. Your mileage may vary with the threat model.

Writes deserve separate discipline. A retry after a lost response must carry a client operation ID or idempotency key that survives process restarts, then the application should persist its result before it acknowledges the generation job. For failed or superseded images, reconcile database state with stored objects, use the verified delete route for the intended key, observe bucket usage as an audit signal, and explicitly abort abandoned multipart uploads. The cleanup worker is mundane, but it is the part that prevents storage from accumulating output nobody can reach. Treat it as a state machine rather than a periodic deletion script: a generation can be pending, successfully stored, associated with a product record, superseded, or rejected; each transition must leave an operation identifier and the object key in durable application state so a worker can decide whether it is repairing a known transition or discovering an unowned byte. The object store's prefix listing can help narrow a sweep, but it cannot answer a metadata query on the application's behalf. A daily lifecycle rule is only a coarse backstop, since its minimum expiry is one day and it does not reclaim multipart fragments automatically. The worker therefore needs a record of upload IDs, a terminal state for failed work, and an explicit abort for incomplete multipart uploads. This design is less elegant than assuming failures disappear, but it assigns the deletion decision to the system that knows whether an image still belongs to a user.

Cleanup has an owner.

## Rejected option: public delivery and storage-native retention controls

For an open-web portfolio, documentation site, or shareable public gallery, reject this signed-only pattern. Re-signing every embedded asset complicates cache behavior and link sharing, while placing a permanent token in a URL weakens the access model. Use a public-serving layer designed for that exposure, with deliberate invalidation and abuse policy, or select a service whose public-hosting model is the requirement rather than an accidental side effect.

Keep a native cloud service when the storage control plane must provide version recovery, object lock for immutable retention, automatic cross-region copies, a self-managed browser CORS policy, or a migration mechanism across the relevant providers. A database can serialize writers; it cannot create WORM guarantees underneath an object store. Likewise, a one-day lifecycle floor cannot meet an hourly deletion promise.

The ADR outcome is narrow but useful: choose private object storage and signed downloads for authenticated AI-generated images, with authorization and garbage collection owned by the application. Choose the provider after its limits pass the invariants. For public reach, immutability, replication, or browser-upload policy, select the option that supplies that capability directly.

## Further reading

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- https://developers.cloudflare.com/r2/api/s3/presigned-urls/
- https://www.backblaze.com/apidocs/b2-g
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest_API/Using_XMLHttpRequest
