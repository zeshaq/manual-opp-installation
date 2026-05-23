# NetBox From Scratch On Ubuntu VM

This guide describes how to install the current greenfield NetBox host on
Ubuntu, configure PostgreSQL and Redis locally, back it up to MinIO, and
expose it through HAProxy once the restore path is proven.

The current lab reference build uses:

- VM name: `gf-ocp-netbox-01`
- Private IP: `30.30.200.23/16`
- Public FQDN: `netbox.v7.comptech-lab.com`
- HAProxy edge IP: `59.153.29.102`
- Bridge: `br33`
- MAC: `52:54:00:70:07:23`
- vCPU: `4`
- RAM: `8 GiB`
- Disk: `100 GiB`
- NetBox version: `v4.6.0`
- Backup bucket: `netbox-backups`
- Backup credential path: `secret/greenfield/object-storage/minio/users/netbox-backup`

NetBox is the lab source of truth for IPAM, DNS intent, racks, devices, VMs,
prefixes, and service ownership.

---

## 1. What You Are Building

You are building a single private NetBox VM with:

- local PostgreSQL;
- local Redis;
- NetBox application code under `/opt/netbox`;
- Nginx as the local HTTP front end;
- HAProxy as the public TLS edge;
- backup and restore validation to MinIO;
- a seed data loader for the initial lab baseline.

Do not expose the VM directly to arbitrary hosts. Use HAProxy for public
access after validation is complete.

---

## 2. Prerequisites

Before you begin, make sure you have:

1. A private-only Ubuntu 24.04 VM.
2. DNS and HAProxy entries for `netbox.v7.comptech-lab.com`.
3. MinIO access for `netbox-backups`.
4. Vault access for the NetBox application secret bundle and backup credential.
5. SSH access to the guest as `ze`.
6. A place to keep backup and restore scripts.

Recommended VM size:

- 4 vCPU
- 8 GiB RAM
- 100 GiB disk
- one private NIC only

---

## 3. Provision The VM

Build the VM with cloud-init and a private address on the lab network.

Current reference network:

```yaml
version: 2
ethernets:
  private0:
    match:
      macaddress: "52:54:00:70:07:23"
    set-name: enp2s0
    dhcp4: false
    addresses:
      - 30.30.200.23/16
    routes:
      - to: default
        via: 30.30.0.1
    nameservers:
      addresses:
        - 30.30.200.53
        - 8.8.8.8
      search:
        - v7.comptech-lab.com
```

Use the current greenfield VM creation pattern:

```bash
./scripts/gfctl.sh prepare-cloud-init --execute gf-ocp-netbox-01
./scripts/gfctl.sh cloud-init-iso --execute gf-ocp-netbox-01
cp scripts/vms/gf-ocp-netbox-01.env.example scripts/vms/gf-ocp-netbox-01.env
./scripts/gfctl.sh create-vm --execute scripts/vms/gf-ocp-netbox-01.env
./scripts/gfctl.sh validate-vm --execute ze@30.30.200.23
```

After first boot:

```bash
ssh ze@30.30.200.23
cloud-init status --wait
hostnamectl
ip -br addr
ip route
```

Expected posture:

- only the private IP is present;
- default route goes to `30.30.0.1`;
- SSH key login works;
- `cloud-init` is `done`.

---

## 4. Install The Base OS And Dependencies

Install the system packages NetBox needs:

```bash
sudo apt update
sudo apt install -y \
  build-essential \
  ca-certificates \
  chrony \
  curl \
  fail2ban \
  git \
  libffi-dev \
  libjpeg-dev \
  libldap2-dev \
  libpq-dev \
  libssl-dev \
  libxml2-dev \
  libxslt1-dev \
  nginx \
  openssh-server \
  postgresql \
  postgresql-contrib \
  python3 \
  python3-dev \
  python3-pip \
  python3-venv \
  redis-server \
  unzip \
  ufw \
  zlib1g-dev
```

Enable the local support services:

```bash
sudo systemctl enable --now chrony fail2ban ssh postgresql redis-server nginx
```

Lock down the firewall:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow from 30.30.200.102 to any port 80 proto tcp
sudo ufw allow from 30.30.200.102 to any port 443 proto tcp
sudo ufw --force enable
```

Direct HTTP access from arbitrary hosts should not be possible once the VM is
in service.

---

## 5. Create PostgreSQL And Redis

Create the NetBox database and user:

```bash
sudo -u postgres psql <<'SQL'
CREATE USER netbox WITH PASSWORD 'REPLACE_WITH_STRONG_PASSWORD';
CREATE DATABASE netbox OWNER netbox;
ALTER ROLE netbox SET client_encoding TO 'utf8';
ALTER ROLE netbox SET default_transaction_isolation TO 'read committed';
ALTER ROLE netbox SET timezone TO 'UTC';
SQL
```

Keep Redis local and private-only. If you use a password, store it in Vault and
reference it from the NetBox config.

---

## 6. Install NetBox

The current lab uses the official NetBox stack layout under `/opt/netbox`.

Representative install flow:

```bash
sudo useradd --system --create-home --home-dir /opt/netbox --shell /bin/bash netbox || true
sudo mkdir -p /opt/netbox
sudo chown -R netbox:netbox /opt/netbox

sudo -u netbox git clone -b v4.6.0 --depth 1 https://github.com/netbox-community/netbox.git /opt/netbox/netbox
sudo -u netbox python3 -m venv /opt/netbox/venv
sudo -u netbox /opt/netbox/venv/bin/pip install --upgrade pip wheel
sudo -u netbox /opt/netbox/venv/bin/pip install -r /opt/netbox/netbox/requirements.txt
```

Copy the local configuration template:

```bash
sudo cp /opt/netbox/netbox/netbox/configuration.example.py /opt/netbox/netbox/netbox/configuration.py
sudo chown root:netbox /opt/netbox/netbox/netbox/configuration.py
sudo chmod 0640 /opt/netbox/netbox/netbox/configuration.py
```

Populate the config from Vault-held values:

- `DATABASE_PASSWORD`
- `REDIS_PASSWORD`
- `SECRET_KEY`
- API token pepper
- admin username
- admin password
- admin email
- installed NetBox version

The current lab keeps those values under:

```text
secret/greenfield/netbox/application/gf-ocp-netbox-01
```

Minimum configuration settings to get right:

- `ALLOWED_HOSTS = ['netbox.v7.comptech-lab.com', '30.30.200.23']`
- `CSRF_TRUSTED_ORIGINS = ['https://netbox.v7.comptech-lab.com']`
- `SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')`
- PostgreSQL database name, user, and password
- Redis connection and password
- local email settings if you want email alerts later

Run the standard NetBox migrations and static asset collection:

```bash
sudo -u netbox /opt/netbox/venv/bin/python /opt/netbox/netbox/manage.py migrate
sudo -u netbox /opt/netbox/venv/bin/python /opt/netbox/netbox/manage.py createsuperuser
sudo -u netbox /opt/netbox/venv/bin/python /opt/netbox/netbox/manage.py collectstatic --no-input
```

Create and enable the service units for `netbox` and `netbox-rq`. The current
lab validates both services plus Nginx:

```text
netbox
netbox-rq
nginx
```

The application config file lives at:

```text
/opt/netbox/netbox/netbox/configuration.py
```

Do not print or commit it.

---

## 7. Validate The Local App

Check service state:

```bash
ssh ze@30.30.200.23 'systemctl is-active postgresql redis-server netbox netbox-rq nginx'
ssh ze@30.30.200.23 'systemctl --failed --no-pager'
ssh ze@30.30.200.23 'sudo nginx -t'
ssh ze@30.30.200.23 'sudo /opt/netbox/venv/bin/python /opt/netbox/netbox/manage.py check'
```

Check the database:

```bash
ssh ze@30.30.200.23 'sudo -u postgres psql -tAc "SELECT datname FROM pg_database WHERE datname = '\''netbox'\'';"'
```

Check the local login page:

```bash
ssh ze@30.30.200.23 'curl -fsS -o /tmp/netbox-login.out -w "%{http_code}\n" http://127.0.0.1/login/'
```

Expected result:

```text
200
```

---

## 8. Backup And Restore

NetBox backups go to the MinIO bucket `netbox-backups`.

Credential custody:

```text
secret/greenfield/object-storage/minio/users/netbox-backup
```

The backup includes:

- PostgreSQL custom-format dump;
- media, reports, scripts, and config when present;
- timestamped manifest;
- encrypted archive and checksum.

Current helper files on the VM:

```text
/etc/netbox/netbox-backup.env
/usr/local/sbin/netbox-backup-to-minio.sh
/usr/local/sbin/netbox-restore-validate.sh
/etc/systemd/system/netbox-backup.service
/etc/systemd/system/netbox-backup.timer
```

Manual backup:

```bash
ssh ze@30.30.200.23 'sudo /usr/local/sbin/netbox-backup-to-minio.sh'
```

Restore validation:

```bash
ssh ze@30.30.200.23 'sudo /usr/local/sbin/netbox-restore-validate.sh'
```

Retention model used in the current lab:

- daily backups;
- keep current versions for 90 days;
- keep noncurrent versions for 90 days;
- expire delete markers;
- run restore validation after significant changes.

Do not trust a backup until you have restored it into a temporary database at
least once.

---

## 9. Expose NetBox Through HAProxy

Once local validation and backup validation pass, point public DNS at HAProxy:

```text
netbox.v7.comptech-lab.com A 59.153.29.102
```

The NetBox backend is the private VM:

```text
30.30.200.23:80
```

NetBox proxy settings should tell the app that the external scheme is HTTPS:

```python
CSRF_TRUSTED_ORIGINS = ['https://netbox.v7.comptech-lab.com']
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
```

The Nginx site must also forward `X-Forwarded-Proto: https` so Nginx, HAProxy,
and Gunicorn agree on the request scheme.

Validation through the edge:

```bash
curl -sS -o /dev/null -w '%{http_code} %{redirect_url}\n' \
  http://netbox.v7.comptech-lab.com/

curl -sS -o /dev/null -w '%{http_code}\n' \
  -L https://netbox.v7.comptech-lab.com/login/
```

Expected result:

```text
301 https://netbox.v7.comptech-lab.com/
200
```

---

## 10. Seed The Baseline Data

The first NetBox source-of-truth baseline is loaded from Git rather than typed
manually in the UI.

Current loader artifacts:

```text
data/netbox/greenfield-baseline.json
scripts/services/netbox/seed-baseline.py
```

The API token used by the loader is stored in Vault:

```text
secret/greenfield/netbox/api-tokens/baseline-seed
```

The baseline typically includes:

- site records;
- VLANs;
- prefixes;
- physical devices;
- libvirt cluster inventory;
- foundation VMs;
- reserved gateways.

Re-running the loader should update the records rather than duplicate them.

---

## 11. Validation

Check DNS:

```bash
dig @30.30.200.53 netbox.v7.comptech-lab.com A +short
```

Expected result:

```text
30.30.200.23
```

Check local service health:

```bash
ssh ze@30.30.200.23 'systemctl is-active postgresql redis-server netbox netbox-rq nginx'
ssh ze@30.30.200.23 'sudo /opt/netbox/venv/bin/python /opt/netbox/netbox/manage.py check'
```

Check the edge route:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' -L https://netbox.v7.comptech-lab.com/login/
```

Check the VM is private-only:

```bash
ssh ze@30.30.200.2 'virsh domiflist gf-ocp-netbox-01'
```

Expected posture:

- NetBox app is healthy locally;
- the login page works through HAProxy;
- direct public access to `30.30.200.23` is blocked;
- backups and restore validation succeed.

---

## 12. Common Gotchas

Watch for these:

- `X-Forwarded-Proto` mismatch between HAProxy, Nginx, and Gunicorn.
- Missing `CSRF_TRUSTED_ORIGINS`.
- Wrong database or Redis password in `configuration.py`.
- Backup restore path that cannot read a root-only archive.
- Seed loader token not loaded from Vault.
- Public exposure before backup validation.

If a restore test creates `502` briefly, wait for the app socket to settle and
recheck before declaring the system broken.

---

## 13. Operator Checklist

Before public exposure:

- [ ] VM private-only
- [ ] PostgreSQL and Redis running locally
- [ ] NetBox app and RQ workers healthy
- [ ] local `manage.py check` passes
- [ ] backup timer or manual backup path works
- [ ] restore validation passed
- [ ] HAProxy route verified
- [ ] baseline data seeded
- [ ] config file and secrets kept out of Git

