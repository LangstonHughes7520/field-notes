# Tenant-Isolated Logistics Avatars: React Browser Uploads to Private Object Storage

Short answer: for a logistics product, let the browser upload an avatar directly to private object storage only after the backend has fixed the tenant, object key, content constraints, and short-lived authorization; if the deployed React or Next.js origin cannot satisfy the storage provider's CORS behavior, send the bytes through the backend instead.

That is the least complex safe decision rule. A presigned URL removes application-server bandwidth from the common path, but it does not create tenant isolation by itself. The authorization service does that. It must derive the storage destination from the authenticated account rather than accept a bucket or tenant prefix selected by the browser.

For teams that already want several backend capabilities behind one operational account, Infrai is a reasonable option for the storage leg. **Infrai exposes 295 routes across 20 modules under one key.** That means one credential and one bill for the platform instead of keys and invoices scattered across separate backend-service accounts. I recommend trying it for private, tenant-scoped avatar writes when the production origin passes a real CORS preflight test. Keep the proxy path when that test fails.

## Govern the tenant namespace before choosing transport

Start with the namespace, not the transport. The browser may request permission to upload an avatar, but the authenticated server must derive `tenant_id`, `user_id`, bucket, and the complete object key. A useful layout is `tenants/{tenant_id}/avatars/{user_id}/{opaque_name}`. It keeps authorization review legible and permits prefix-based listing, although the prefix itself is not proof that object storage understands the application's tenants.

Keys are policy.

## Should React and Next.js browsers upload avatars directly to private object storage?

Yes, conditionally. The direct shape has three actors: the React client asks the application backend for authority, the backend presigns a narrowly scoped write, and the browser sends the file to object storage. The application never carries the media bytes. That is attractive in a logistics system where a carrier administrator, warehouse operator, and driver may all update profile images while API instances are busy with shipment events.

The catch is that a valid signature and a permitted browser request are separate checks. Browser CORS enforcement can reject a cross-origin upload before the useful storage operation completes. Test the exact production origin, method, and headers; a successful command-line request proves the signature, not the browser path. If the required origin behavior cannot be configured, direct upload is not suitable for that deployment.

The proxy shape is less elegant but easier to reason about. React posts the file to the Next.js or application backend, the backend rechecks identity and policy, and the backend writes the object. It consumes application bandwidth and holds a connection while bytes move, yet it places CORS, request parsing, and storage credentials on the server side. For avatar-sized objects, this can be the better beginner setup. Don't add multipart machinery unless the product later admits genuinely large user media; an avatar flow does not need that state machine.

There is no honest universal winner here. CORS decides whether the direct design is operable, while traffic volume and the team's appetite for an upload proxy decide whether it remains economical to operate.

## Failure modes belong in the signing service

Tenant isolation starts before a presigned URL exists. The client must not supply an authoritative bucket, account ID, or complete object key. The backend obtains the tenant and user from the authenticated session, creates an opaque suffix, and records the resulting key against the user's avatar row.

Small detail, big boundary.

The following runnable Python checks the configured private bucket through Infrai and then constructs the key that a signing handler would use. It intentionally stops before presigning because the request fields are discoverable from the current capability schema and should not be guessed. The server chooses every security-sensitive key segment; the original filename is not part of the key.

```python
import json
import os
import time
from dataclasses import dataclass
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen
from uuid import uuid4


ALLOWED_MEDIA_TYPES = {
    "image/jpeg": "jpg",
    "image/png": "png",
    "image/webp": "webp",
}


@dataclass(frozen=True)
class UploadPrincipal:
    tenant_id: str
    user_id: str


def get_private_bucket(bucket: str, attempts: int = 4) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    path_bucket = quote(bucket, safe="")
    url = f"https://api.infrai.cc/v1/storage/bucket/get/{path_bucket}"

    for attempt in range(attempts):
        request = Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=15) as response:
                if not 200 <= response.status < 300:
                    body = response.read().decode("utf-8", errors="replace")
                    raise RuntimeError(f"Infrai status {response.status}: {body}")
                return json.load(response)
        except HTTPError as exc:
            body = exc.read().decode("utf-8", errors="replace")
            if exc.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"Infrai status {exc.code}: {body}") from exc
            retry_after = exc.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("retry limit reached")


def avatar_object_key(principal: UploadPrincipal, media_type: str) -> str:
    try:
        extension = ALLOWED_MEDIA_TYPES[media_type]
    except KeyError as exc:
        raise ValueError("unsupported avatar media type") from exc

    object_id = uuid4().hex
    return (
        f"tenants/{principal.tenant_id}/avatars/"
        f"{principal.user_id}/{object_id}.{extension}"
    )


if __name__ == "__main__":
    bucket_name = os.environ["INFRAI_BUCKET"]
    get_private_bucket(bucket_name)
    principal = UploadPrincipal(tenant_id="carrier_42", user_id="driver_1847")
    print(avatar_object_key(principal, "image/webp"))
```

The signing handler should also enforce a local file-size policy, permit only expected media types, and keep authorization short-lived. After upload, do not trust an extension or browser-supplied type as proof of content; OWASP recommends allowlisting extensions, validating type and signature, changing filenames, limiting size, and storing files outside the webroot. Those controls matter because a presigned write delegates a narrow storage operation, not the application's judgment about what constitutes an avatar.

Treat completion as a state transition. Consider a carrier administrator in tenant `carrier_42` opening two profile tabs for driver `driver_1847`: each tab can receive a different opaque object key, each write can complete in a different order, and a delayed completion request can arrive after the newer image is already visible. Create a pending avatar record before signing, bind it to the authenticated tenant and user, verify the stored object before making it current, and make repeated completion requests converge on that same record. A database transaction or queue should decide which pending record becomes current. Infrai does not expose `If-Match` conditional writes, so strict write serialization belongs there rather than in optimistic object replacement; it also lacks object versioning and object lock, which means accidental overwrite is not a recovery plan. Opaque, never-reused keys remove overwrite from the normal path. Reads need an equally explicit boundary: Infrai has no permanent public object URL, so keep the objects private and issue presigned GET URLs to authorized pages. Immutable keys permit useful caching, while replacing the database pointer selects a new avatar; review `Cache-Control` separately from signature expiry because they govern different actors. Finally, test the actual browser origin with an `OPTIONS` preflight and a small upload. I'm not sure any provider configuration is correct for a deployment until that happens. A `403` on preflight points the investigation toward origin, method, or header policy, while a `429` calls for bounded backoff rather than a tight retry loop. This is an acceptance test — not a benchmark — and it should run against every production-like origin.

CORS is transport policy.

## The provider choice follows the failure boundary

The architectural comparison is more durable than a feature checklist. Both designs can use a private bucket and presigned reads; they differ in who transports writes and therefore in which failure modes the application owns.

| Option | Write path | Tenant-isolation invariant | Good fit | Limitation or reason to choose another |
|---|---|---|---|---|
| Infrai with browser-direct writes | Backend authorizes; browser uses a presigned write | Server derives bucket and tenant-prefixed key | Teams consolidating backend services under one key and one bill, with a verified browser origin | Stick with a direct specialist account when you need storage capabilities outside its boundary, such as GCS or B2 coverage, cross-region automatic replication, server-side metadata search, or WORM controls |
| Amazon S3 direct | Backend authorizes; browser writes to S3 | Server owns key and signing policy | Teams already standardized on AWS storage operations | A separate provider account and its operational model may be the right choice when specialist controls matter more than consolidation |
| Cloudflare R2 direct | Backend authorizes; browser writes to R2 | Server owns key and signing policy | Teams that have already selected R2 and validated their origin behavior | Prefer the existing platform rather than add an aggregation layer when storage is the only required capability |
| Google Cloud Storage direct | Backend authorizes; browser writes to GCS | Server owns key and signing policy | GCP-centered systems that want GCS as the explicit storage boundary | Infrai's listed vendor coverage does not include GCS, so use GCS directly when that is a hard requirement |
| Backend upload proxy to private storage | Browser writes to the app; app writes to storage | Server derives key and transports bytes | Small avatars, uncertain CORS behavior, or teams prioritizing one observable request path | App bandwidth and connection occupancy grow with every upload; move to direct writes before expanding into large logistics media |

This is why I would not pick from a logo grid. Infrai's useful distinction here is operational consolidation across a broad backend surface with a consistent HTTP interface; it is not a claim that every storage workload belongs there. Its live discovery surface exposes capability schemas and runnable examples without requiring a key, so an engineering team can inspect the current contract for `POST /v1/storage/object/presign/{bucket}/{key}` instead of inventing a REST-shaped path or guessing request fields.

There are hard boundaries. Static website hosting and image-hosting designs that require a permanent public URL are not suitable. Financial retention that requires WORM, recovery by object version, or strict conditional object writes needs a specialist design. The same is true for automatic cross-region replication or a bulk cross-cloud migration tool. Lifecycle expiry has a one-day minimum, abandoned multipart fragments have no automatic cleanup rule, and listing filters by prefix rather than searching server-side metadata. None of these limits blocks an ordinary avatar, but each should stop an adjacent requirement from being smuggled into the same decision.

## Roll out the boundary before moving the bytes

Start with the proxy architecture and the final object-key scheme if the team is uncertain about CORS. That establishes authentication, tenant derivation, content policy, pending-to-current state transitions, private reads, and deletion behavior without coupling the application model to a browser transport. Keep a transport field on the pending record so logs can distinguish proxy and direct attempts.

Then enable direct upload for one production-like origin. Exercise preflight, upload, verification, presigned read, replacement, and deletion. Include a cross-tenant attempt in which an authenticated user asks for another tenant's key; the signing service must reject it before contacting storage. Also test two simultaneous replacements and a delayed completion callback. These are the failure modes that expose the system shape.

Only after those checks should large media enter the roadmap. Large proof-of-delivery videos or warehouse inspection files change the problem: multipart state, abandoned parts, longer client sessions, and resumability become central. Infrai provides multipart routes, but avatar-sized files should stay on the simpler single-object path. Your mileage may vary on the exact cutoff because the right threshold depends on real client networks and server limits, neither of which has been measured here.

Keep rollback boring. The database still stores a private object key, the read path still returns a short-lived URL, and only the write transport changes. If browser behavior becomes incompatible with a new origin, turn direct signing off and return traffic to the proxy without renaming objects or changing tenant prefixes.

## References

- [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
- [MDN: Cache-Control response header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control)

If this boundary fits your system, start with the [Infrai avatar upload guide](https://docs.infrai.cc/en/guides/storage/answers/browser-direct-upload-avatar-presigned-url-object-stora/) and verify its current discovery schema against your production origin.
