# Generated Image Object Storage: Choosing User Downloads, Exports, and US/EU Recovery

Short answer: for a marketplace accepting private, large image uploads, choose object storage by the slowest required operation: sustained upload and download throughput, expiring authorization, regional placement, and a restore path you can rehearse. Keep the asset catalog in a transactional database, use immutable object keys, and make a signed URL the short-lived delivery token. A bucket that can hold the bytes but cannot support your recovery evidence is not a complete design.

The marketplace scenario matters. A seller may upload a high-resolution generated image, attach it to a listing, and later request an export containing many images. Buyers and sellers need downloads; operators need retention and backups; legal or contractual rules may constrain US and EU placement. Large-file throughput is the primary axis here, so latency on a single metadata call is less important than how the system behaves under concurrent multipart work and a busy export queue.

## The first failed export is usually a contract failure

Start with a written asset contract. Give every object a logical ID, an immutable key, an owning marketplace record, a retention class, an allowed region, and a deletion state. The database is the authority for ownership and authorization. The bucket holds bytes. Treating a prefix as the catalog creates a painful question later: which object is the current image, and was it approved for this user?

For a download, the application checks the authenticated user against the listing or project record, then issues a narrowly scoped signed URL for the private object. The URL should expire according to the product action, and it should not be treated as a permanent public address. Do the same for an export: create the export in a worker, write it under a separate immutable key, record its retention class, and mint a fresh link only after an authorization check. A download token is an authorization artifact. Small detail.

Choose the region before the write. Store the tenant or listing's allowed geography beside the asset record, route the upload to an approved region, and reject a request that does not have a valid placement decision. A region label added after upload is an audit field, not a residency control. For US/EU deployments, define whether a backup is allowed to cross that boundary; “primary region” alone does not answer the question.

Picture the failure sequence. A seller uploads a large image, the worker records the database row early, and the connection drops after several parts arrive. A retry starts a second upload under a new key, while the first key remains untracked. Later, the export job sees one catalog row but two byte sets; a deletion worker removes the row and only one object; and the signed link issued before validation points at an artifact the application no longer considers current. Nothing here requires an exotic storage outage. It follows from allowing byte movement, catalog state, authorization, and cleanup to have different owners and different definitions of “done.” The contract should make each transition explicit: upload parts, validate and complete, commit the catalog row, authorize a link, and clean abandoned work under a measured policy.

## How do large generated-image exports affect object-storage throughput?

Throughput is a pipeline property. Measure the client-to-storage path, the worker's multipart concurrency, the queue, the catalog transaction, and the download path separately. A fast provider cannot rescue a worker that serializes every part, a database transaction that stays open while bytes move, or an export job that retries the whole archive after one part fails. The bytes are not the bottleneck.

Use bounded concurrency and resumable work. A part should have a stable identity, a checksum or equivalent validation decision, and a recorded completion state. The catalog transaction should describe the object only after the upload is complete and validated. Keep the final key immutable so a retry cannot silently replace a different image. If an upload is abandoned, the cleanup policy must be explicit; otherwise incomplete multipart state becomes an operational leak.

Here is the shape of a provider-neutral worker. It deliberately leaves the storage adapter behind an interface: the point is to make retry and authorization behavior testable without pretending that every backend has the same controls.

```python
from dataclasses import dataclass
from time import sleep
from typing import Protocol


@dataclass(frozen=True)
class PartResult:
    number: int
    checksum: str


class ObjectStore(Protocol):
    def upload_part(self, key: str, number: int, data: bytes) -> PartResult: ...
    def complete(self, key: str, parts: list[PartResult]) -> None: ...
    def signed_download(self, key: str, expires_seconds: int) -> str: ...


def upload_with_bounded_retry(
    store: ObjectStore, key: str, parts: list[bytes], max_attempts: int = 4
) -> str:
    completed: list[PartResult] = []

    for number, data in enumerate(parts, start=1):
        for attempt in range(max_attempts):
            try:
                completed.append(store.upload_part(key, number, data))
                break
            except TimeoutError:
                if attempt == max_attempts - 1:
                    raise
                sleep(2 ** attempt)

    store.complete(key, completed)
    return store.signed_download(key, expires_seconds=900)
```

This is not a throughput benchmark. It is a boundary sketch. In production, limit the in-memory part set, make retries idempotent, and record enough state to resume without re-reading every byte. An HTTP 429 response should feed a backoff policy, not a tight loop that adds pressure to an already busy system. Capture upload duration by size band, completion rate, retry count, and time to first byte for downloads.

A common failure is measuring only the happy path with one 20 MB image. The marketplace's real workload may be a hundred concurrent 500 MB exports, a burst after a model release, or a seller reconnecting over a poor network. Test those cases with synthetic objects. Do not use customer images to discover whether a retention rule or recovery copy is wrong.

Measure twice.

## Retention is a promise, while backup is a separate operation

Retention answers how long the product should keep an asset. Backup answers how the team will recover it after deletion, corruption, credential loss, or a regional event. They overlap in policy but differ in mechanism. A lifecycle rule that expires an object is not a backup, and a copied object without a catalog record is not a reliable restore.

Define classes such as preview, accepted listing image, and generated export. For each class, write down minimum lifetime, deletion authority, backup destination, restore priority, and evidence required at deletion. Keep deletion events auditable. If an image must be immutable for a period, verify that the selected storage contract supplies the required immutability control; otherwise enforce ownership and state transitions outside the bucket and say exactly what guarantee remains.

The restore drill should be boring and specific. Create a test listing with several large synthetic images, its catalog rows, a US or EU placement decision, and an export manifest. Copy the approved backup set, remove only the primary test objects that the drill permits, restore them, and authorize a download as the test user. Compare object identity, size, validation result, region metadata, catalog state, and user-visible access. Record elapsed time and operator actions. If the team only verifies that bytes were copied, it has tested copying, not recovery.

Your mileage may vary on the right recovery objective; the answer depends on listing value, contractual retention, and the size of the export queue. What resolves that uncertainty is a timed restore exercise against the same worker and catalog path used in production.

## Compare storage choices by hard boundaries

A fair comparison starts after the architecture is explicit. Ask each candidate the same questions: Can large uploads resume? How are multipart fragments cleaned up? Can the application issue scoped expiring links for private objects? What regional choices and backup destinations are available? What happens to an object after deletion? Can the team enumerate, validate, and restore a bounded export? Which limits are enforced by the provider, and which must be implemented by the application?

| Decision area | Evidence to collect | Failure if ignored |
|---|---|---|
| Large-file throughput | Concurrent upload and download results by size band | Export queues miss their completion target |
| Private access | Expiry, scope, logging, and revocation behavior | A leaked link remains useful too long |
| US/EU placement | Write-time routing and backup geography | A backup violates the stated residency rule |
| Retention and deletion | Lifecycle semantics, audit evidence, and immutability controls | Operators cannot prove why an image disappeared |
| Recovery | Restore time, catalog reconciliation, and validation steps | A copied object cannot be served to its owner |
| Exit | Inventory format, bulk transfer process, and integrity checks | A provider change becomes an unplanned rewrite |

The catch is that no single object-storage feature removes the application contract. Replication may not match the geography you are permitted to use; a retention control may not be the immutability guarantee your compliance team expects; and a familiar API shape does not prove identical semantics across implementations. Use native controls when identity, regional, or governance behavior is essential to the deployment. Choose a thinner abstraction when portability is the requirement, and test the capabilities it actually exposes.

For regulated workloads, treat an authorization such as FedRAMP as a boundary and service assessment, not a universal claim about every configuration. The deployment, region, data path, and operational procedures still need review. A compliance logo is not a restore test.

## Roll out with a restore-first gate

Build the first slice around one private upload, one authorized download, one export, and one restore. Separate production and test data. Generate immutable keys that do not contain personal information. Keep signed URLs out of ordinary application logs, and make the audit record say who requested the link, for which object, with what expiry.

Before increasing concurrency, run the same synthetic workload against each required US and EU placement. Set an explicit throughput target, an acceptable retry rate, a restore objective, and a maximum link lifetime. Then fail the gate if the provider or adapter cannot show evidence for one of them.

No restore evidence, no retention claim.

## References

- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)
- [FedRAMP: Federal Risk and Authorization Management Program](https://www.fedramp.gov/)
