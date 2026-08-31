# Tenant Isolation in Node.js Object Storage: Backup Prefixes, Uploads, and Database Dumps

Short answer: treat tenant isolation as the acceptance test for a full app backup, then make every nightly artifact addressable by an immutable backup id. Keep browser uploads off the application data path, pair the committed upload set with a database dump, and prove the pair can be restored before expanding the schedule.

The least complicated design is a worker that writes a tenant-scoped manifest, not a scheduler that merely copies a bucket. That distinction matters in a B2B SaaS: the database says who owns an object, while object storage only holds bytes and keys. A successful upload request is not evidence that those two facts still agree.

The governance boundary is easy to state and easy to miss in implementation. A backup operator needs enough authority to restore a tenant, while an application request should have no path to enumerate the backup estate. That means the backup identity, the live application identity, and the restore identity should be reviewed as separate roles, with the manifest acting as an auditable record of what each role is allowed to select. It also means testing a deliberate cross-tenant denial, checking that logs do not leak keys, and retaining the policy version beside the artifact set. A prefix can make the intended boundary legible to a human reviewer, but only authorization rules and a restore drill can demonstrate that the boundary holds when a key is guessed, a job is retried, or a tenant is deleted.

## Which governance records should cross a tenant boundary?

Start with the restore operator's question: can I select one tenant, one completed backup, and nothing belonging to another tenant? A prefix helps narrow the search, but it is not an authorization boundary and it is not a transaction. The manifest has to carry the backup id, tenant id, database dump key, upload root, object count, byte count, checksums, creation window, schema version, and completion state.

Use immutable names that expose partial work rather than hiding it:

```text
backups/prod/tenant-042/2026-08-11T020000Z/db.dump
backups/prod/tenant-042/2026-08-11T020000Z/uploads/asset-91/original.mp4
backups/prod/tenant-042/2026-08-11T020000Z/uploads/asset-91/thumbnail.jpg
backups/prod/tenant-042/2026-08-11T020000Z/manifest.json
```

The environment comes first, followed by the tenant and backup id. Artifact type comes after that. A separate metadata row may point to the latest complete manifest, but a mutable `latest` object should never be the only record of what happened. Concurrent jobs and half-written trees become much easier to spot when a run cannot silently overwrite yesterday's evidence.

The application still owns authorization. A caller allowed to list `backups/prod/tenant-042/` must not thereby gain access to every production prefix, and the backup bucket should not double as a public media bucket. Encrypt database dumps, restrict their readers, and keep secrets out of object names and logs.

## What should a restore drill measure for a full app backup?

Separate the scheduler from the worker. The scheduler takes a distributed lock, creates a run id, enumerates eligible tenants, and enqueues work. The worker snapshots the committed upload set, creates the database dump, uploads both artifact classes, calculates checksums while reading, and writes the manifest last. It should publish `complete` only after returned metadata and the expected object set have been checked.

The following Python example keeps the storage adapter generic. That is intentional: the durable contract is the key layout, manifest, checksum, and completion rule, not a vendor-specific call hidden in the application.

```python
import hashlib
import json
import subprocess
import tempfile
from datetime import datetime, timezone
from pathlib import Path


def sha256_file(path: Path) -> str:
    digest = hashlib.sha256()
    with path.open("rb") as stream:
        for block in iter(lambda: stream.read(1024 * 1024), b""):
            digest.update(block)
    return digest.hexdigest()


def create_database_dump(database_url: str, output: Path) -> None:
    subprocess.run(
        ["pg_dump", "--format=custom", "--file", str(output), database_url],
        check=True,
    )


def backup_tenant(tenant_id: str, storage, database_url: str,
                  upload_root: Path) -> dict:
    stamp = datetime.now(timezone.utc).replace(microsecond=0).isoformat()
    stamp = stamp.replace("+00:00", "Z")
    prefix = f"backups/prod/{tenant_id}/{stamp}"

    with tempfile.TemporaryDirectory() as workdir:
        dump = Path(workdir) / "db.dump"
        create_database_dump(database_url, dump)
        dump_hash = sha256_file(dump)
        db_key = f"{prefix}/db.dump"
        storage.put_file(db_key, dump, checksum=dump_hash)

        objects = []
        for path in sorted(upload_root.rglob("*")):
            if not path.is_file():
                continue
            relative = path.relative_to(upload_root).as_posix()
            key = f"{prefix}/uploads/{relative}"
            checksum = sha256_file(path)
            storage.put_file(key, path, checksum=checksum)
            objects.append({
                "key": key,
                "sha256": checksum,
                "bytes": path.stat().st_size,
            })

        manifest = {
            "backup_id": stamp,
            "tenant_id": tenant_id,
            "database": {"key": db_key, "sha256": dump_hash},
            "uploads": objects,
            "state": "complete",
        }
        manifest_path = Path(workdir) / "manifest.json"
        manifest_path.write_text(json.dumps(manifest, separators=(",", ":")))
        storage.put_file(
            f"{prefix}/manifest.json",
            manifest_path,
            checksum=sha256_file(manifest_path),
        )
        return manifest
```

This is a control shape, not a claim that the loop is a complete snapshot protocol. A browser-direct upload can finish while the database dump is being produced. If strict point-in-time semantics matter, briefly quiesce writes or record a cutoff and make the bounded consistency window explicit. Otherwise, validate missing references during restoration. I'm not sure one policy fits every collaborative media workflow; your mileage may vary when files are edited while a backup runs.

One small failure is enough to invalidate the run. I once treated a zero-byte artifact as a successful upload because the HTTP request succeeded. The restore failed later. That was a 0-byte, 1-line lesson: transport success is not backup validity.

## How can a rollout expose reliability failures before migration?

The useful comparison is between evidence and control, not between storage logos. Prefixes give operators a selection convention. They do not give you relational consistency, tenant authorization, immutability, cross-region disaster recovery, or a transaction across a database and object storage.

| Failure mode | Evidence to inspect | Control to add |
| --- | --- | --- |
| A row names a missing upload | Restore manifest and validate references | Complete-state gate plus reference checks |
| Two jobs reuse a backup id | Duplicate manifests and overlapping locks | Durable scheduler lock and deterministic ids |
| A tenant crosses an object boundary | Authorization test using a second tenant | Policy checks and tenant-scoped queries |
| A dump exists but cannot be restored | Database restore exit status and smoke test | Scheduled isolated restore drills |
| Retention removes required evidence | Manifest audit against policy version | Separate retention classes and review |
| Primary storage is unavailable | Recovery exercise from another copy | An independently operated copy and RPO |

Checksums catch corruption in transit; they do not prove that the application can interpret restored rows. A lifecycle rule can remove old prefixes; it does not know whether an incident investigation or legal hold needs one. Put the retention policy version in the manifest and make its reviewer visible.

For direct uploads, record the object key, tenant id, media type, and committed state in the application before the nightly worker considers the object eligible. The worker should enumerate committed objects, not sweep an unowned prefix and infer ownership from filenames. That is the point where tenant isolation becomes an operational rule instead of a naming preference.

## How should a Node.js nightly job connect uploads, database dumps, and a restore checklist?

Restore is the acceptance test. Select a complete manifest and confirm its environment, tenant id, retention class, and timestamp. List the declared prefix, compare object count and total bytes with the manifest, verify the recorded checksums used by the drill, and restore the database dump into an empty database. Then reconstruct the tenant's upload tree and query restored rows for missing objects, wrong content types, and unexpected cross-tenant keys.

Start the application against that isolated pair. Exercise login, media listing, download authorization, and one write path. Record restore duration, recovered object count, data-loss window, operator, and every deviation from the runbook.

Do not call a backup usable because its objects exist.

The catch is operational scope: this design is not suitable when the requirement is a provider-enforced WORM archive, instant multi-region failover, or a fully managed cross-cloud recovery service. Add those controls when the threat model requires them. Stick with a managed recovery system when the team cannot operate the independent copy and the restore drill it depends on.

## When should a prefix strategy stop being the chosen design?

Roll out one tenant first. Keep the existing application upload path and write a manifest beside the new backup artifacts; do not make a live migration depend on a single successful nightly run. Compare counts and checksums, perform an isolated restore, and only then expand the tenant set. The scheduler can increase coverage after the worker's completion and alerting rules have survived a real drill.

This also keeps provider changes bounded. Object storage systems expose different lifecycle, retention, replication, and compatibility details, so record those assumptions in the adapter and test the restore against the actual target. The source URLs below are useful starting points for pricing, product capability, retention, and dump behavior, but they do not replace a rehearsal of your own tenant data.

The decision rule is compact: choose the least complex storage layout that can prove tenant boundaries, preserve the database-to-upload relationship, and reproduce a working tenant in isolation. A bucket prefix is part of that evidence. It is never the evidence by itself.

## References

- https://aws.amazon.com/s3/pricing/
- https://docs.digitalocean.com/products/spaces/
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html
- https://developers.cloudflare.com/r2/
- https://www.backblaze.com/docs/cloud-storage-s3-compatible-api
- https://min.io/docs/minio/linux/administration/object-management/object-retention.html
- https://www.postgresql.org/docs/current/app-pgdump.html
