# Nexus From Scratch On Ubuntu VM

This guide describes how to build the current greenfield Nexus VM on Ubuntu,
wire it behind HAProxy, back it up safely, and recover it cleanly when
needed.

The current lab reference build uses:

- Nexus VM: `gf-ocp-nexus-01`
- Private IP: `30.30.200.41/16`
- Public route: `nexus.v7.comptech-lab.com`
- Edge IP: `59.153.29.102`
- Nexus version: `3.92.0`
- Local data: `/var/lib/nexus-data`
- Credential custody: `secret/greenfield/nexus/application/gf-ocp-nexus-01`

Nexus is private-only. Keep the guest off the public Internet. Humans reach it
through HAProxy, and automation reaches it through the private LAN endpoint.

Do not use Nexus as the OpenShift release, operator, or disconnected image
mirror. Those paths are Quay-owned in the current lab design. Nexus is for
non-OpenShift dependency caching, application artifact storage, and build
support.

---

## 1. What You Are Building

You are building a single Ubuntu VM with:

- a local Nexus Repository OSS service;
- local persistent data under `/var/lib/nexus-data`;
- a systemd-managed service called `nexus.service`;
- a private-only address on the lab network;
- an HAProxy front door on the lab edge;
- optional MinIO-backed backup archives for recovery.

This guide follows the current lab pattern for `gf-ocp-nexus-01`. If your
environment uses a different hostname or edge IP, substitute those values
consistently everywhere.

---

## 2. Prerequisites

Before starting, make sure you have:

1. A working hypervisor and bridge network, currently `dl385-2` and `br33`.
2. DNS entries for `nexus.v7.comptech-lab.com`.
3. HAProxy with a backend slot for the Nexus VM.
4. A MinIO endpoint or other object store for backups.
5. Vault access for the Nexus application secret bundle and any backup
   credential.
6. SSH access to the guest as `ze`.
7. A fresh Ubuntu 24.04 VM image or equivalent cloud image.

Recommended VM size:

- 4 to 8 vCPU
- 8 to 16 GiB RAM
- 100 to 250 GiB disk
- one private NIC only

---

## 3. Provision The VM

Create the VM with cloud-init, a private-only NIC, and a static address on the
lab network.

The current reference network shape is:

```yaml
version: 2
ethernets:
  private0:
    match:
      macaddress: "52:54:00:70:07:41"
    set-name: enp2s0
    dhcp4: false
    addresses:
      - 30.30.200.41/16
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

If you are using the greenfield helper pattern, render and create the VM from
the bootstrap host. If you are doing it manually, build the same shape with
your libvirt or cloud-init tooling.

After first boot:

```bash
ssh ze@30.30.200.41
cloud-init status --wait
hostnamectl
ip -br addr
ip route
```

Expected posture:

- only the private address is present;
- default egress is via `30.30.0.1`;
- `cloud-init` is `done`;
- the host name resolves as `gf-ocp-nexus-01`.

---

## 4. Install The Base OS Packages

Install the baseline packages you need for Nexus and the operator workflow:

```bash
sudo apt update
sudo apt install -y \
  ca-certificates \
  chrony \
  curl \
  fail2ban \
  gnupg \
  jq \
  openssh-server \
  ufw \
  unzip
```

Install the container runtime you will use for the Nexus service. The current
lab service is a single container wrapped by systemd. If your base image
standardizes on Docker, use Docker. If your site standardizes on Podman, adapt
the systemd unit accordingly.

For a Docker-based install:

```bash
sudo apt install -y docker.io
sudo systemctl enable --now docker
```

Enable the support services:

```bash
sudo systemctl enable --now chrony fail2ban ssh
```

Lock the firewall down before exposing the application:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow from 30.30.200.102 to any port 8081 proto tcp
sudo ufw --force enable
```

Keep port `8081` private. Public users should only see the HAProxy edge.

---

## 5. Create Nexus Data And Service Paths

Create the local directories that the container and bootstrap scripts need:

```bash
sudo install -d -m 0750 /etc/nexus
sudo install -d -m 0750 /var/lib/nexus-data
sudo install -d -m 0755 /var/log/nexus
```

If your container expects the Nexus container user to own the data tree,
adjust ownership before first boot:

```bash
sudo chown -R 200:200 /var/lib/nexus-data
```

The exact UID/GID may vary if you change the image or container runtime. The
rule is simple: the Nexus process must be able to write to its data dir before
first startup.

---

## 6. Install The Nexus Service

The current lab uses a systemd unit called `nexus.service`. The unit should
start the official Sonatype Nexus image and keep the data directory mounted
locally.

Representative systemd unit:

```ini
[Unit]
Description=Sonatype Nexus Repository
Wants=network-online.target
After=network-online.target docker.service
Requires=docker.service

[Service]
Type=simple
Restart=on-failure
RestartSec=15
TimeoutStartSec=300
ExecStartPre=-/usr/bin/docker rm -f nexus
ExecStart=/usr/bin/docker run --rm --name nexus \
  -p 8081:8081 \
  -v /var/lib/nexus-data:/nexus-data \
  sonatype/nexus3:3.92.0
ExecStop=/usr/bin/docker stop nexus

[Install]
WantedBy=multi-user.target
```

Install it:

```bash
sudo tee /etc/systemd/system/nexus.service >/dev/null <<'EOF'
[Unit]
Description=Sonatype Nexus Repository
Wants=network-online.target
After=network-online.target docker.service
Requires=docker.service

[Service]
Type=simple
Restart=on-failure
RestartSec=15
TimeoutStartSec=300
ExecStartPre=-/usr/bin/docker rm -f nexus
ExecStart=/usr/bin/docker run --rm --name nexus \
  -p 8081:8081 \
  -v /var/lib/nexus-data:/nexus-data \
  sonatype/nexus3:3.92.0
ExecStop=/usr/bin/docker stop nexus

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now nexus.service
```

If you prefer Podman, keep the same service shape and translate the `ExecStart`
and `ExecStop` commands to the Podman equivalent. The important parts are the
same:

- one long-lived container;
- port `8081`;
- bind mount to `/var/lib/nexus-data`;
- systemd restart policy;
- no public exposure from the guest itself.

---

## 7. Bootstrap The First Admin Login

On a fresh Nexus volume, the container writes an initial admin password into
the data directory. Use that password once, then rotate it immediately.

Wait for Nexus to become reachable locally:

```bash
until curl -fsS http://127.0.0.1:8081/service/rest/v1/status >/dev/null; do
  sleep 5
done
```

Read the bootstrap password:

```bash
sudo cat /var/lib/nexus-data/admin.password
```

Use it to change the admin password and create your real operator account. Do
not keep using the bootstrap password after first login.

Representative REST flow:

```bash
export NEXUS_URL=http://127.0.0.1:8081
export NEXUS_BOOTSTRAP_PASSWORD='<bootstrap-password>'
export NEXUS_ADMIN_PASSWORD='<long-random-password>'
export NEXUS_OPERATOR_USER='<operator-user>'
export NEXUS_OPERATOR_EMAIL='<operator@example.com>'

curl -fsS -u "admin:${NEXUS_BOOTSTRAP_PASSWORD}" -X PUT \
  -H "Content-Type: text/plain" \
  --data "${NEXUS_ADMIN_PASSWORD}" \
  "${NEXUS_URL}/service/rest/v1/security/users/admin/change-password"

curl -fsS -u "admin:${NEXUS_ADMIN_PASSWORD}" -X POST \
  -H "Content-Type: application/json" \
  -d "{
        \"userId\": \"${NEXUS_OPERATOR_USER}\",
        \"firstName\": \"${NEXUS_OPERATOR_USER}\",
        \"lastName\": \"operator\",
        \"emailAddress\": \"${NEXUS_OPERATOR_EMAIL}\",
        \"password\": \"${NEXUS_ADMIN_PASSWORD}\",
        \"status\": \"active\",
        \"roles\": [\"nx-admin\"]
      }" \
  "${NEXUS_URL}/service/rest/v1/security/users"
```

Store the real password and any automation credential in Vault under the
current greenfield Nexus custody path. Do not keep it in shell history, tickets
or Git.

If you are using the current greenfield helper scripts, this bootstrap step is
what the first-boot setup service performs automatically.

---

## 8. Configure The Repository Set

Create only the repository types your environment actually needs. In the
current lab, Nexus is for non-OpenShift dependency caching and artifact
support, not for OpenShift release or operator content.

Typical repo families:

- proxy repositories for upstream package managers;
- hosted repositories for internal artifacts;
- group repositories for client convenience;
- raw repositories for tarballs, scripts, or metadata files.

Recommended policy:

- keep OpenShift release images, operator catalogs, and disconnected mirror
  payloads in Quay;
- keep package-manager caches and build artifacts in Nexus;
- do not mix the two roles unless a later ADR explicitly changes that rule.

If you are doing this manually in the UI:

1. Log in as the operator/admin user.
2. Go to `Administration -> Repositories`.
3. Create the proxy repositories you need.
4. Set the remote URL, blob store, and cleanup policy for each one.
5. Create any hosted repos needed for internal artifacts.
6. Create a group repository that lists the repos in consumer order.
7. Save and verify each repo with a test fetch.

Do not invent repo names for this guide if your lab has not standardized them.
Use the exact names your environment documents and keep them consistent across
build jobs and consuming applications.

---

## 9. Configure HAProxy

The public Nexus hostname points to HAProxy, not directly to the guest.

Expected DNS:

```text
nexus.v7.comptech-lab.com A 59.153.29.102
nexus-private.v7.comptech-lab.com A 30.30.200.41
```

Representative HAProxy backend:

```haproxy
frontend fe_http_public
    bind 59.153.29.102:80
    mode http
    acl host_nexus hdr(host) -i nexus.v7.comptech-lab.com
    http-request redirect scheme https code 301 if host_nexus

frontend fe_https_public
    bind 59.153.29.102:443 ssl crt /etc/haproxy/certs/v7-wildcard.pem
    mode http
    acl host_nexus hdr(host) -i nexus.v7.comptech-lab.com
    use_backend be_nexus if host_nexus

backend be_nexus
    mode http
    option httpchk GET /service/rest/v1/status
    http-check expect status 200
    http-request set-header X-Forwarded-Proto https
    http-request set-header X-Forwarded-Ssl on
    server nexus01 30.30.200.41:8081 check
```

The public certificate is the shared wildcard certificate for
`*.v7.comptech-lab.com`.

---

## 10. Validation

Run local validation on the VM:

```bash
ssh ze@30.30.200.41 'hostname; whoami; systemctl is-active nexus.service'
ssh ze@30.30.200.41 'sudo /usr/local/sbin/validate-nexus-standalone.sh /etc/nexus/nexus.env'
```

Run edge validation from an operator host:

```bash
curl -sS -o /tmp/nexus-status.out -w '%{http_code}\n' \
  https://nexus.v7.comptech-lab.com/service/rest/v1/status

curl -sSI http://nexus.v7.comptech-lab.com/ | head -1
curl -sSI https://nexus.v7.comptech-lab.com/ | head -1
```

Expected results:

- `nexus.service` is active;
- the status endpoint returns `200`;
- HTTP redirects to HTTPS;
- the UI loads over the public HTTPS route;
- DNS resolves the public route to `59.153.29.102`;
- the private route resolves to `30.30.200.41`.

Also confirm the edge path:

```bash
ssh ze@dl385-2 'ssh gf-ocp-nexus-01 "curl -fsS http://127.0.0.1:8081/service/rest/v1/status"'
```

If the public route works but the direct private route does not, the issue is
usually local firewall or SSH pathing. If the direct route works but public
access does not, the issue is usually DNS or HAProxy.

---

## 11. Backup And Restore

Nexus stores its live state locally in `/var/lib/nexus-data`. Back up that
directory with the service stopped or with a filesystem snapshot. Do not copy
the live directory while the service is actively writing unless you know your
snapshot method is consistent.

Minimum backup set:

- `/var/lib/nexus-data`
- `/etc/nexus/nexus.env`
- `/etc/systemd/system/nexus.service`
- any HAProxy/Nexus route notes you need for replay

Example archive flow:

```bash
sudo systemctl stop nexus.service
sudo tar -C / -czf /var/backups/nexus-$(date -u +%Y%m%d-%H%M%SZ).tgz \
  var/lib/nexus-data \
  etc/nexus \
  etc/systemd/system/nexus.service
sudo systemctl start nexus.service
```

Upload the archive to your backup store with the object-store tool your lab
uses, for example `mc cp` or `aws s3 cp`. If you migrate to a different object
store, sync the archive there first, then restore from the archive on the new
host.

Restore flow:

1. Stop `nexus.service`.
2. Restore the archived data directory and config files.
3. Fix ownership so the container can write to `/var/lib/nexus-data`.
4. Start `nexus.service`.
5. Verify the status endpoint.
6. Re-check the UI and any repository endpoints.

If the restored instance shows a stale admin password, re-read the bootstrap
password file only if the volume is genuinely a fresh bootstrap. Do not rely on
an old bootstrap file after the service has already been configured.

---

## 12. Common Gotchas

1. Do not expose `8081` publicly. The guest should stay private-only.
2. Do not route OpenShift images through Nexus. Quay owns that path.
3. Do not leave the bootstrap admin password in shell history or notes.
4. Do not let the data directory remain owned by the wrong UID/GID.
5. Do not forget to restart HAProxy after changing the backend.
6. Do not assume an empty Nexus means a broken install. Fresh installs start
   with a nearly empty repository list by design.
7. Do not mix the current `gf-ocp-nexus-01` service model with the older
   registry-heavy `nexus-mirror` pattern. They are different builds.
8. If you see a 502 at the edge, verify the backend process first, then the
   HAProxy config, then DNS.
9. If you see a 401 on the repository UI or API, that is usually healthy
   authentication behavior, not a failure.
10. If the container starts but the UI never comes up, check disk space first.

Typical diagnostics:

```bash
systemctl status nexus.service
journalctl -u nexus.service -b --no-pager
curl -fsS http://127.0.0.1:8081/service/rest/v1/status
df -h /var/lib/nexus-data
```

---

## 13. Operator Checklist

Before handing the system off, verify:

- the VM boots cleanly after a reboot;
- `nexus.service` starts automatically;
- `curl http://127.0.0.1:8081/service/rest/v1/status` returns `200`;
- the public hostname resolves and redirects to HTTPS;
- the UI is reachable from the edge route;
- the admin credential is stored in Vault;
- backup output exists in your object store or backup system;
- restore instructions have been tested at least once;
- the host is documented in NetBox or the lab source of truth;
- the service is not accidentally sharing the OpenShift image supply path.

If all of those are true, the Nexus install is ready for normal use.
