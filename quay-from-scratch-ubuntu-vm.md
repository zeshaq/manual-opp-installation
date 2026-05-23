# Quay From Scratch On Ubuntu VM

This document describes how to install a standalone Project Quay instance on a
fresh Ubuntu VM, connect it to MinIO for registry blob storage, expose it
through TLS, and validate that it works. It also explains how to move existing
registry content to a new object store or reupload image content manually.

The current CompTech lab reference build uses:

- Quay: `quay.io/projectquay/quay:3.17.1`
- Registry VM: Ubuntu 24.04
- Registry hostname: `quay.v7.comptech-lab.com`
- Private registry hostname: `quay-private.v7.comptech-lab.com`
- Object store: MinIO
- MinIO bucket for registry blobs: `quay-storage`
- MinIO bucket for backups: `quay-backups`

If you are building a different environment, keep the same flow and replace the
example hostnames, IPs, and credentials with your own values.

---

## 1. What you are building

This setup is a single Quay VM with local Postgres and Redis, backed by S3
compatible object storage in MinIO.

The important split is:

- Quay application metadata lives in PostgreSQL.
- Quay registry blobs live in MinIO.
- Quay backup archives live in a separate MinIO bucket.

Do not assume you can restore Quay by copying files on the VM alone. A working
restore needs both:

- the Quay database, and
- the blob store contents.

If you change object storage later, you must migrate the bucket contents or
reupload the registry content. If you change only the VM and leave MinIO and
PostgreSQL intact, the registry should continue to work once the VM can reach
those services.

---

## 2. Prerequisites

Before you begin, make sure you have:

1. A fresh Ubuntu 24.04 VM.
2. Root or sudo access on that VM.
3. A reachable MinIO endpoint.
4. A bucket and user for Quay blob storage.
5. A TLS hostname and reverse proxy in front of Quay.
6. DNS records for the public and private Quay names.
7. A place to store secrets, such as Vault or another secret manager.

Recommended VM size for this pattern:

- 4 vCPU
- 16 GiB RAM
- 200 GiB system disk
- one private management network interface

Recommended service layout:

- Quay container on the VM
- PostgreSQL on the VM
- Redis on the VM
- MinIO external
- HAProxy or equivalent TLS terminator external

---

## 3. Decide the storage model first

You need to choose one of two Quay storage models.

### Recommended: S3 compatible object storage

This is the current lab pattern.

- Quay uses MinIO via the S3-compatible `RadosGWStorage` backend.
- Registry blobs are stored in the bucket, not on the VM filesystem.
- The VM can be rebuilt independently as long as the database and bucket are
  preserved.

### Alternate: Local filesystem storage

This is only appropriate if you intentionally want Quay to store blobs on a
mounted disk on the VM.

- This is a different architecture.
- It is not the current production-pattern setup in the lab.
- It is less flexible for migration and disaster recovery.

If your goal is to reproduce the current lab behavior, use S3-compatible
object storage. Do not copy the bucket into a local directory and expect Quay
to keep behaving the same way.

---

## 4. Prepare the Ubuntu VM

Install the base packages first:

```bash
sudo apt update
sudo apt install -y \
  ca-certificates \
  chrony \
  curl \
  fail2ban \
  gnupg \
  jq \
  lsb-release \
  openssh-server \
  podman \
  postgresql-16 \
  postgresql-client-16 \
  python3-yaml \
  redis-server
```

Enable the local services:

```bash
sudo systemctl enable --now chrony
sudo systemctl enable --now fail2ban
sudo systemctl enable --now postgresql
sudo systemctl enable --now redis-server
```

Recommended host directories:

```bash
sudo mkdir -p /etc/quay/config
sudo mkdir -p /var/lib/quay/storage
sudo chmod 0750 /etc/quay/config /var/lib/quay/storage
```

If you are following the current lab pattern, the container will mount:

- `/etc/quay/config` as the Quay configuration directory
- `/var/lib/quay/storage` as the host-side mount visible to the container

The registry blobs themselves still go to MinIO when you use the S3 backend.

---

## 5. Prepare PostgreSQL

Create the Quay database and user. Replace the password with your own value.

```bash
sudo -u postgres psql <<'SQL'
CREATE USER quay WITH PASSWORD 'REPLACE_WITH_STRONG_PASSWORD';
CREATE DATABASE quay OWNER quay;
ALTER ROLE quay SET client_encoding TO 'utf8';
ALTER ROLE quay SET default_transaction_isolation TO 'read committed';
ALTER ROLE quay SET timezone TO 'UTC';
SQL
```

If you are using a separate database host, the same logical steps apply:

- create a dedicated database
- create a dedicated database user
- give that user ownership of the Quay database

Keep PostgreSQL local if you want the simplest recovery story.

---

## 6. Prepare Redis

Quay uses Redis for runtime coordination. Keep Redis local and do not expose it
to the network.

If you want a password on Redis, configure it in `/etc/redis/redis.conf` and
restart Redis. The current lab uses Redis with authentication.

Example baseline:

```bash
sudo sed -i 's/^# \?bind .*/bind 127.0.0.1 ::1/' /etc/redis/redis.conf
sudo systemctl restart redis-server
```

If you add a Redis password, record it in your secret manager and reference it
from the Quay config.

---

## 7. Prepare MinIO

Quay needs a bucket and a user that can read, write, list, and delete within
that bucket.

### 7.1 Create the registry bucket

For the current lab model:

- bucket name: `quay-storage`
- MinIO endpoint: `30.30.200.1:9000`
- access key: stored in secret manager
- secret key: stored in secret manager

If the bucket does not exist, create it first.

### 7.2 Create a dedicated MinIO user

Use a dedicated user for Quay registry storage, not a shared admin key.

Recommended permissions for the registry bucket:

- `s3:ListBucket`
- `s3:GetObject`
- `s3:PutObject`
- `s3:DeleteObject`
- `s3:AbortMultipartUpload`
- `s3:ListBucketMultipartUploads`
- `s3:ListMultipartUploadParts`

Example policy shape:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": ["arn:aws:s3:::quay-storage"]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:AbortMultipartUpload",
        "s3:ListBucketMultipartUploads",
        "s3:ListMultipartUploadParts"
      ],
      "Resource": ["arn:aws:s3:::quay-storage/*"]
    }
  ]
}
```

### 7.3 Store credentials safely

Store the MinIO access key and secret key in a secret manager. Do not place
them in Git.

For the current lab pattern, the Quay storage credential is stored under:

- `secret/greenfield/object-storage/minio/users/quay-storage`

### 7.4 Optional backup bucket

If you want Quay backups, create a separate bucket, for example:

- bucket: `quay-backups`
- user: `quay-backup`

Keep backups separate from blob storage.

---

## 8. Install Quay

The current lab uses a standalone Quay container on the VM. The simplest
pattern is:

- install the Quay container image
- write `/etc/quay/config/config.yaml`
- start Quay under systemd

Install the container image from the current pinned release:

```bash
sudo podman pull quay.io/projectquay/quay:3.17.1
```

Create or render the Quay config directory:

```bash
sudo mkdir -p /etc/quay/config
```

Create `config.yaml` with the correct database, Redis, hostname, and storage
settings.

### 8.1 Core config fields

Use values similar to the following:

```yaml
SERVER_HOSTNAME: quay.example.com
PREFERRED_URL_SCHEME: https
EXTERNAL_TLS_TERMINATION: true
SETUP_COMPLETE: true
FEATURE_USER_INITIALIZE: true
SECRET_KEY: REPLACE_WITH_RANDOM_SECRET

DB_URI: postgresql://quay:REPLACE_WITH_PASSWORD@127.0.0.1:5432/quay

BUILDLOGS_REDIS:
  hostname: 127.0.0.1
  port: 6379
  password: REPLACE_WITH_REDIS_PASSWORD

DISTRIBUTED_STORAGE_PREFERENCE:
  - default
DISTRIBUTED_STORAGE_CONFIG:
  default:
    - RadosGWStorage
    - access_key: quay-storage-v2
      secret_key: REPLACE_WITH_MINIO_SECRET
      bucket_name: quay-storage
      hostname: 30.30.200.1
      is_secure: false
      port: 9000
      storage_path: /datastorage/registry
      signature_version: v4
```

Notes:

- `storage_path` is part of the Quay storage backend configuration. It is not
  a local filesystem path you can copy to and from.
- If your object storage uses TLS, set `is_secure: true` and install the
  correct CA trust chain on the VM and in the container trust path.
- Keep the Quay version pinned while you are doing the initial install and any
  migration work.

### 8.2 Start Quay with systemd

Use a systemd service that launches Quay via Podman in host network mode.

Typical service characteristics:

- container name: `quay`
- image: `quay.io/projectquay/quay:3.17.1`
- network: `host`
- config mount: `/etc/quay/config:/conf/stack:ro`
- storage mount: `/var/lib/quay/storage:/datastorage`
- restart policy: always

If you already have an install script, use it. If not, keep the container
definition minimal and reproducible.

Start and enable the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now quay.service
```

Check status:

```bash
sudo systemctl status quay.service --no-pager
```

---

## 9. Expose Quay through TLS

Do not expose the registry container directly to users. Put it behind a
reverse proxy.

The current lab pattern uses HAProxy:

- public FQDN: `quay.v7.comptech-lab.com`
- private FQDN: `quay-private.v7.comptech-lab.com`
- HAProxy forwards HTTPS to `http://quay-vm:8080`

Recommended behavior:

1. Port 80 redirects to 443.
2. Port 443 terminates TLS.
3. HAProxy forwards to the Quay container on port `8080`.

If you use another proxy, the same behavior applies.

You also need DNS records pointing at the proxy:

- public registry name for end users and robot accounts
- private registry name for internal use

---

## 10. Initialize Quay

On the first start, Quay needs its initial setup completed.

Depending on the installer flow you use, this can mean:

- running the Quay setup helper,
- logging into the Quay admin bootstrap page, or
- using an automation script that writes the initial config and seeds the
  database.

The current lab stores the bootstrap material in Vault and uses a helper script
to render the standalone config. Your new environment should do the same
conceptually, even if you implement it differently.

At minimum, make sure:

- the superuser exists,
- the registry hostname is correct,
- the database URI is correct,
- the object storage backend is correct,
- the Redis host and password are correct.

After initialization, verify the discovery endpoint:

```bash
curl -k https://quay.example.com/api/v1/discovery
```

You should also be able to log in through the UI and push/pull a test image.

---

## 11. Create registry content and test it

After Quay is up, create a small smoke repository and test a push/pull cycle.

Example flow:

```bash
podman login quay.example.com
podman pull registry.access.redhat.com/ubi9/ubi:latest
podman tag registry.access.redhat.com/ubi9/ubi:latest quay.example.com/test/smoke:latest
podman push quay.example.com/test/smoke:latest
podman pull quay.example.com/test/smoke:latest
```

If this fails, check:

- DNS
- TLS certificate
- HAProxy routing
- PostgreSQL
- Redis
- MinIO access
- bucket permissions

---

## 12. Backups

Do not treat the registry as complete until you have a backup path.

The current lab uses a separate backup bucket and a timer-driven backup job.
That backup archive is not the registry blob store. It is a backup of the Quay
application data and related state.

Recommended backup pieces:

1. PostgreSQL dump.
2. Quay config and secret material.
3. Registry bucket copy or replication.
4. The backup archive itself in `quay-backups`.

Example backup approach:

```bash
pg_dump -Fc quay > quay.dump
```

Then encrypt and upload the backup archive to the backup bucket.

The exact script can be whatever you standardize on, but keep it repeatable and
test restore it before you rely on it.

---

## 13. How to migrate an existing registry to a new object store

If you already have a Quay instance and need to move it to a different object
store, do not copy just the VM filesystem. Move the database and the object
store together.

### Option A: Preferred bucket-to-bucket sync

Use this when the source and destination are both S3-compatible stores.

This is the cleanest way to keep the registry blobs intact.

High-level steps:

1. Stop writes to the source registry.
2. Take a PostgreSQL backup.
3. Mirror the source bucket to the destination bucket.
4. Verify object counts and sample digests.
5. Restore or point Quay at the destination database.
6. Update the Quay config for the new bucket or endpoint.
7. Start Quay and validate pulls.

Example with `mc`:

```bash
mc alias set src http://old-minio:9000 SOURCE_ACCESS_KEY SOURCE_SECRET_KEY
mc alias set dst http://new-minio:9000 DEST_ACCESS_KEY DEST_SECRET_KEY
mc mirror --overwrite --preserve src/quay-storage dst/quay-storage
mc ls --recursive --summarize dst/quay-storage
```

If the new object store is reachable under a different hostname, change:

- `hostname`
- `port`
- `is_secure`
- `bucket_name`
- credentials

in `config.yaml`, then restart Quay after the bucket copy is complete.

### Option B: Manual registry-to-registry reupload

Use this when you can still access the source Quay registry, but you do not
want to move the raw bucket directly.

This is usually safer than manipulating bucket objects by hand because it
replays content through the registry API.

Example with `skopeo`:

```bash
skopeo copy --all \
  docker://source-quay.example.com/team/app:tag \
  docker://dest-quay.example.com/team/app:tag
```

Use this for repository-by-repository rehydration.

Pros:

- preserves registry semantics
- does not depend on bucket internals
- works even if the backing object store changes format

Cons:

- slower for large registries
- requires source registry access
- you must enumerate and copy repositories

### Option C: Download and reupload object blobs manually

This is the least preferred option, but it can work for small test registries.

Use it only if you understand that:

- Quay still needs the database metadata.
- The object key layout must be preserved.
- Manual download/upload is slower and riskier than bucket sync.

Example with `mc`:

```bash
mc alias set src http://old-minio:9000 SOURCE_ACCESS_KEY SOURCE_SECRET_KEY
mc alias set dst http://new-minio:9000 DEST_ACCESS_KEY DEST_SECRET_KEY
mc mirror src/quay-storage /tmp/quay-storage-export
mc mirror /tmp/quay-storage-export dst/quay-storage
```

Or with `aws s3 sync`:

```bash
aws s3 sync s3://quay-storage ./quay-storage-export
aws s3 sync ./quay-storage-export s3://quay-storage
```

Important:

- This only works if the object names and paths remain consistent.
- It does not replace the PostgreSQL restore step.
- It should be done with Quay writes stopped, or you will get drift.

### What not to do

Do not do this:

- copy the bucket contents into a random directory on the new VM and hope
  Quay finds them
- restore only the bucket without the database
- restore only the database without the bucket
- change the bucket name without updating Quay config
- mix the source and destination registry writes during the final cutover

---

## 14. If you want to switch from MinIO to another object store

If the new object store is another S3-compatible backend, the migration is the
same in principle:

1. Create the destination bucket.
2. Create the destination credentials.
3. Move the objects.
4. Update Quay config.
5. Restart and validate.

If the destination uses TLS, install the CA chain and set `is_secure: true`.

If the destination is not S3-compatible, Quay storage configuration must be
reworked. That is an architectural change, not just a migration.

---

## 15. Validation checklist

After the install, verify all of the following:

### VM level

- Ubuntu boots cleanly.
- SSH works.
- Time sync is active.
- The VM can reach MinIO.

### Service level

- `postgresql` is active.
- `redis-server` is active.
- `quay.service` is active.

### Registry level

- `https://quay.example.com/api/v1/discovery` returns `200`.
- A `podman login` succeeds.
- A test image can be pushed and pulled.

### Storage level

- The bucket exists.
- The Quay user can list objects.
- Objects are being written to the bucket, not to a local throwaway path.

### Backup level

- The backup timer or backup job runs successfully.
- A test restore is possible.

---

## 16. Operational notes

- Keep Quay version pinned during initial deployment and migration.
- Keep PostgreSQL local unless you have a strong reason to externalize it.
- Keep Redis local unless you have a strong reason to externalize it.
- Keep Quay object storage separate from Quay backup storage.
- Use the same bucket name during migration if you can. It reduces config drift.
- Store all credentials in Vault or a similar secret store.

The most common mistake is treating the registry blob store like a normal
filesystem backup. It is not. You need both the database state and the blob
store content.

---

## 17. Recommended execution order

If you are standing up a brand new environment, do it in this order:

1. Provision the VM.
2. Set DNS and TLS termination.
3. Prepare MinIO bucket and user.
4. Install PostgreSQL and Redis.
5. Install the Quay container.
6. Write the Quay config.
7. Start Quay.
8. Initialize the registry.
9. Push a smoke image.
10. Configure backups.
11. Test restore.

If you are migrating an existing registry:

1. Stop writes on the source registry.
2. Dump PostgreSQL.
3. Sync the blob bucket.
4. Restore on the destination.
5. Update DNS or proxy routing.
6. Start Quay on the new VM.
7. Validate discovery, login, push, and pull.

---

## 18. Practical rule of thumb

If the question is:

> Can I copy the `quay-storage` bucket contents into a directory on the new VM?

The answer is:

- not if you are keeping the current S3-backed Quay architecture
- yes only if you intentionally redesign Quay to use local filesystem storage,
  which is a different install model

For a normal migration, move the bucket contents into the destination object
store and restore the database.

