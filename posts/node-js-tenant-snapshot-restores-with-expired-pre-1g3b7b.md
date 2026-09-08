# Node.js Tenant Snapshot Restores with Expired Presigned Download URL for Generated Images

Short answer: when a presigned download URL has expired and generated images stop loading, persist the tenant-scoped object key and an immutable object revision in each snapshot, then authorize the current user against the restored tenant and mint a fresh, short-lived URL at read time.

An expired link is usually doing exactly what it was designed to do. The architectural mistake is treating a temporary bearer capability as durable image identity. In a SaaS that stores AI-generated images, that mistake stays quiet until a backup is restored, an old browser tab wakes up, or a queued job tries to use yesterday's serialized response. Then the database points at a perfectly valid-looking string whose authority has ended.

The URL is disposable.

The deciding constraint is tenant isolation. This is an authorization-boundary problem, not a link-lifetime tuning problem: a restore must not turn an old snapshot into a way to sign another tenant's object, even if the restored row contains a syntactically valid key.

## How should expired presigned download URLs preserve SaaS tenant isolation?

The decision is to store a stable object reference in the database and snapshots, while generating download authority only after a current authorization check. For an image row, the durable fields are the tenant identifier, bucket or logical store identifier, object key, optional immutable revision identifier, content type, byte length, and a digest computed when the object is accepted. A field named `download_url` does not belong in the backup contract if its value contains an expiry and signature.

Three invariants matter. First, the authenticated tenant must equal the tenant recorded on both the image and the selected snapshot. Second, the object key must be derived from trusted stored metadata, not accepted from a client during the download request. Third, a successful database restore is not proof that the referenced bytes exist or are the intended revision. Those checks cross different systems, so pretending they form one atomic transaction hides the actual failure boundary.

Keep the states explicit — for example, `requested`, `metadata_restored`, `objects_verified`, and `ready`. A worker may resume an interrupted verification pass by snapshot ID and object key. It must not publish the restored snapshot as ready merely because every SQL statement committed. This is where a digest and immutable revision earn their keep: the verifier can distinguish “the key resolves” from “the restored record resolves to the bytes selected by this snapshot.”

Be strict here.

If the object store does not expose immutable revisions, the design has to supply immutability another way, such as write-once keys that incorporate a generated object identifier. I'm not sure a mutable key can support a defensible point-in-time restore without additional history; the evidence needed to settle that question is a test showing what bytes the key returns before and after overwrite, deletion, and restoration under the exact storage configuration.

## Snapshot migration acceptance starts with an authorization trace

Treat this diagnostic as a migration acceptance test, not as an isolated frontend repair. A selected tenant snapshot is acceptable only if its durable references survive the move while its old access capabilities do not acquire new life.

Start by separating an expiration failure from an object, signature, browser, and authorization failure. Capture the request time, the server time that issued the URL, the declared expiry, the HTTP method, and a redacted request identifier. Do not copy the signature-bearing query string into ordinary logs. OWASP's secrets guidance treats logs and monitoring as part of a secret's lifecycle; a signed URL should receive the same operational caution because possession can grant access until its authority ends.

Use a controlled request outside the browser to inspect the response status and headers without exposing the full URL in a ticket. A response indicating that the signature window has elapsed points to renewal behavior. A missing object points to backup completeness, retention, or the wrong stable key. A signature mismatch points toward method, canonical request inputs, credentials, region or endpoint configuration, or signed headers. A browser-only failure, while the same request succeeds in a controlled client, shifts attention to cross-origin policy, referrer behavior, content security policy, and cached application state. HTTP status codes narrow the search, but they are not a portable diagnosis by themselves; providers do not have to use identical error bodies.

Then inspect the restored record. It should look more like `{tenant_id, snapshot_id, object_key, revision, sha256}` than `{tenant_id, image_url}`. Confirm that the snapshot belongs to the active tenant before touching the signer, and confirm that the key belongs to the tenant namespace after reading it from storage. The second check is deliberate defense in depth. A row can be malformed through import, a bad migration, or an operator mistake even when the request-level authorization is correct.

Clock skew is worth checking, but it shouldn't become the universal explanation. Compare hosts against the same time source and record issuance and expiry as absolute timestamps in structured diagnostics. Don't “fix” a suspected clock problem by making every link last for days; that expands the exposure window and leaves stale serialized URLs embedded in backups, queues, and caches.

Finally, reproduce the whole lifecycle with a deliberately short expiry in a non-production test: create an image reference for tenant A, obtain a URL, verify access before expiry, wait until it expires, obtain a new URL from the application, and verify that tenant B cannot obtain one for the same stable reference. Restore a selected snapshot and repeat the isolation test. Include one image created before the snapshot, one created after it, one whose object was deliberately omitted from the test backup, and one mutable-key fixture whose bytes changed after the snapshot. The expected result is not “all four images load.” The pre-snapshot immutable object should verify and receive a new URL; the post-snapshot row should be absent from the restored metadata; the omitted object should keep the restore out of `ready`; and the changed mutable object should fail digest or revision verification. Run the same matrix with a stale browser URL and a current application session. This exposes confusion between metadata time, object time, signing time, and authorization time far better than asserting that the signing library returned a string.

## Operational cost follows capability placement

The relevant comparison is not which signer has the nicest helper method. It is where authorization happens, what crosses the trust boundary, whether a restored snapshot remains meaningful after credentials and URLs have aged out, and which component pays the bandwidth, connection, cache, and renewal costs.

| Option | Tenant-isolation property | Restore behavior | Operational cost | Valid boundary |
|---|---|---|---|---|
| Store presigned URLs in image rows | Authorization is frozen into a copyable string | Restored links may already be expired | Low at first, high during failures | Suitable only as an ephemeral cache, never as snapshot identity |
| Store keys; sign after each authorized read | Current tenant policy gates every new capability | Restored keys can receive fresh URLs | Requires a signing service and careful cache rules | Default for private images served directly from object storage |
| Proxy image bytes through the application | Application can enforce policy on every byte request | Restore uses the same stable key | Adds bandwidth, connection, and backpressure load | Useful when immediate centralized control matters more than direct delivery |
| Publish immutable objects without signatures | No per-request tenant boundary exists | Stable across restores | Simple delivery and caching | Suitable only for content intentionally public to anyone with the address |

The on-demand signing choice has limits. It is not suitable when a client must remain offline and reuse the same link beyond its expiry; that workflow needs an online renewal step or a different, explicitly designed transfer mechanism. It also does not provide instant revocation of a URL already issued in every object-storage implementation. Keep the proxy option when policy requires each request to be reevaluated centrally, and keep public delivery for assets whose disclosure is genuinely acceptable. “Hard to guess” is not a public-access policy.

There is another trade-off: signing on every thumbnail render can create needless work and churn in browser caches. A bounded cache of newly minted URLs can be reasonable if its key includes tenant, object identity, revision, and authorization context, and if its lifetime never exceeds the signed URL's remaining validity. The catch is that application authorization changes can outlive a cached capability. Set the lifetime from that risk, not from a round number copied from an example.

## Developer experience should make cross-tenant signing difficult

The following Python models the critical path even when the production handler is Node.js. The language is incidental; the ordering is the contract. Repositories return typed records, the authorization check happens before signing, and the signer never accepts a client-supplied key. A good internal interface makes the secure path shorter for developers: there is no general `sign(any_key)` helper available to a request handler, only a function that loads a restored image, proves tenant ownership, checks the namespace, and then delegates to a narrow signer.

```python
from dataclasses import dataclass
from datetime import timedelta
from typing import Protocol


@dataclass(frozen=True)
class ImageRef:
    image_id: str
    tenant_id: str
    snapshot_id: str
    object_key: str
    revision: str | None
    sha256: str


class ImageRepository(Protocol):
    def restored_image(self, image_id: str) -> ImageRef: ...


class ObjectSigner(Protocol):
    def sign_download(
        self, *, object_key: str, revision: str | None, expires_in: timedelta
    ) -> str: ...


class AccessDenied(Exception):
    pass


def issue_image_download(
    *,
    authenticated_tenant: str,
    image_id: str,
    repository: ImageRepository,
    signer: ObjectSigner,
) -> dict[str, str]:
    image = repository.restored_image(image_id)

    if image.tenant_id != authenticated_tenant:
        raise AccessDenied("image does not belong to the authenticated tenant")

    tenant_prefix = f"tenants/{authenticated_tenant}/"
    if not image.object_key.startswith(tenant_prefix):
        raise AccessDenied("stored object key violates tenant namespace")

    url = signer.sign_download(
        object_key=image.object_key,
        revision=image.revision,
        expires_in=timedelta(minutes=10),
    )
    return {"url": url}
```

The ten-minute value is an example policy, not a universal recommendation. Measure enough real transfer duration to choose a window that lets an authorized download start and finish under the provider's documented semantics, then test slow clients and retries. Longer is not automatically safer operationally. Shorter is not automatically safer either if aggressive renewal creates a thundering herd or causes clients to retry partial transfers without bounds.

For larger generated artifacts or bundled tenant exports, multipart upload is a separate concern from signed downloads. Multipart upload lets parts be uploaded independently and requires a completion step; an initiated upload should also have an abort and cleanup policy. It does not make an expired download URL durable, and its upload identifier is not a substitute for the restored object's stable identity. Mixing those concepts is how temporary transfer state leaks into backup schemas.

Observability should preserve the distinction. Count signing denials by reason, renewal attempts, post-restore verification failures, and downloads that fail before any bytes are transferred. Use tenant-safe correlation identifiers rather than raw object keys when operators do not need the key. Never attach full signed URLs to traces. Alert on a rise in renewals followed by failures, because that pattern distinguishes “an old URL expired and was replaced” from “the application cannot produce a usable current capability.”

## Trade-offs behind rejecting serialized authority

The rejected design stores the generated URL beside the image and copies that field into every tenant snapshot. It looks convenient because rendering becomes a direct property read, but it couples durable metadata to the credential, endpoint configuration, signing time, and expiration policy that existed during one earlier request. Restoring the row restores none of that authority. Extending the expiry only delays the same mismatch while increasing the period in which a copied link may be used.

There is a narrow valid use for the field: an in-memory or disposable cache whose entry cannot outlive the URL and is never treated as backup data. Label it accordingly. A client may also hold the URL long enough to complete its immediate authorized download, then ask the application for another one after expiry. That is renewal, not restoration.

The resulting ADR is intentionally dull: back up identity, verify bytes, authorize the tenant now, and issue authority late. Dull survives restores.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://www.rfc-editor.org/rfc/rfc9110
- https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
