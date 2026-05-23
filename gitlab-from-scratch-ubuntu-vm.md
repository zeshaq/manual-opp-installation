# GitLab From Scratch On Ubuntu VM

This guide describes how to build the current greenfield GitLab VM on Ubuntu,
wire it behind HAProxy, back it up to MinIO, and recover it cleanly when
needed.

The current lab reference build uses:

- GitLab VM: `gf-ocp-gitlab-01`
- Private IP: `30.30.200.20/16`
- Public route: `gitlab.v7.comptech-lab.com`
- Edge IP: `59.153.29.102`
- GitLab package: `gitlab-ce`
- Pinned GitLab version: `18.11.3-ce.0`
- Backup bucket: `gitlab-backups`
- Backup credential path: `secret/greenfield/object-storage/minio/users/gitlab-backup`

GitLab is private-only. It has no public NIC. Humans reach it through HAProxy,
and automation reaches it through the private LAN endpoint.

---

## 1. What You Are Building

You are building a single GitLab CE VM with:

- local PostgreSQL;
- GitLab-managed Redis/Gitaly/Sidekiq components;
- an HAProxy front door on the lab edge;
- object-store backups to MinIO;
- config and secrets custody in Vault;
- a separate GitLab runner VM for CI execution.

Do not expose the GitLab guest directly to the public network. The public name
must point to HAProxy, not the VM.

---

## 2. Prerequisites

Before starting, make sure you have:

1. A working hypervisor and bridge network, currently `dl385-2` and `br33`.
2. DNS entries for `gitlab.v7.comptech-lab.com`.
3. HAProxy with a backend slot for the GitLab VM.
4. A MinIO endpoint and a `gitlab-backups` bucket.
5. Vault access for the initial root password, backup credentials, and config
   custody archive.
6. A fresh Ubuntu 24.04 VM image or equivalent cloud image.
7. SSH access to the guest as `ze`.

Recommended VM size:

- 4 vCPU
- 8 to 16 GiB RAM
- 100 GiB disk
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
      macaddress: "52:54:00:70:07:11"
    set-name: enp2s0
    dhcp4: false
    addresses:
      - 30.30.200.20/16
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

Bootstrap the VM from the current greenfield helper pattern:

```bash
./scripts/gfctl.sh prepare-cloud-init --execute gf-ocp-gitlab-01
./scripts/gfctl.sh cloud-init-iso --execute gf-ocp-gitlab-01
cp scripts/vms/gf-ocp-gitlab-01.env.example scripts/vms/gf-ocp-gitlab-01.env
./scripts/gfctl.sh create-vm --execute scripts/vms/gf-ocp-gitlab-01.env
```

After first boot:

```bash
ssh ze@30.30.200.20
cloud-init status --wait
hostnamectl
ip -br addr
ip route
```

Expected posture:

- only the private address is present;
- default egress is via `30.30.0.1`;
- `cloud-init` is `done`.

---

## 4. Install The Base OS Packages

Install the baseline packages you need for GitLab and the operator workflow:

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
  ufw
```

Enable the services:

```bash
sudo systemctl enable --now chrony fail2ban ssh
```

Lock the firewall down before installing the application:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow from 30.30.200.102 to any port 80 proto tcp
sudo ufw allow from 30.30.200.102 to any port 443 proto tcp
sudo ufw --force enable
```

GitLab should be reachable from HAProxy only, not from arbitrary hosts on the
LAN.

---

## 5. Install GitLab CE

The current lab pins GitLab CE to `18.11.3-ce.0`.

Before installing the package, preseed `/etc/gitlab/gitlab.rb` with the
external URL and alertmanager setting. That avoids the first converge failure
seen during the restore drill.

```bash
sudo install -d -m 0755 /etc/gitlab
sudo tee /etc/gitlab/gitlab.rb >/dev/null <<'EOF'
external_url "http://gitlab.v7.comptech-lab.com"
alertmanager["enable"] = false
EOF
```

Install the GitLab package:

```bash
curl -fsSL https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
sudo apt-get install -y gitlab-ce=18.11.3-ce.0
sudo gitlab-ctl reconfigure
```

If you need to re-run convergence after editing config:

```bash
sudo gitlab-ctl reconfigure
```

Do not treat the first root password as a permanent credential. Rotate it
after the first operational login and write the replacement into Vault.

---

## 6. Configure HAProxy

The public GitLab hostname points to HAProxy, not to the VM.

Expected DNS:

```text
gitlab.v7.comptech-lab.com A 59.153.29.102
```

Representative HAProxy backend:

```haproxy
frontend fe_http_public
    bind 59.153.29.102:80
    mode http
    acl host_gitlab hdr(host) -i gitlab.v7.comptech-lab.com
    acl path_health path /healthz
    http-request return status 200 content-type text/plain string "gf-ocp-haproxy-01 ready\n" if path_health
    http-request return status 503 content-type text/plain string "No backend configured yet\n" if !path_health !host_gitlab
    use_backend be_gitlab_http if host_gitlab

backend be_gitlab_http
    mode http
    option httpchk
    http-check send meth GET uri /users/sign_in ver HTTP/1.1 hdr Host gitlab.v7.comptech-lab.com
    http-check expect rstatus (2|3)[0-9][0-9]
    http-request set-header X-Forwarded-Proto http
    http-request set-header X-Forwarded-Ssl off
    server gitlab01 30.30.200.20:80 check
```

The host header in the health check matters. If it is wrong, HAProxy may mark
the backend down even when the VM itself is healthy.

If you later enable TLS on the edge, keep the same backend and update the
public route to HTTPS.

---

## 7. Backups And Restore Custody

GitLab backups go to MinIO bucket `gitlab-backups`.

Credential custody:

```text
secret/greenfield/object-storage/minio/users/gitlab-backup
```

The GitLab backup configuration lives in a root-only include file:

```text
/etc/gitlab/gitlab-backup-object-storage.rb
```

The main GitLab config includes it:

```ruby
from_file "/etc/gitlab/gitlab-backup-object-storage.rb"
```

The backup archive itself is not enough for a full restore. You also need:

- `/etc/gitlab/gitlab.rb`
- `/etc/gitlab/gitlab-secrets.json`
- the backup object-storage include file

Those files are preserved separately in an encrypted config-custody archive.

Manual backup:

```bash
ssh ze@30.30.200.20 'sudo gitlab-backup create STRATEGY=copy'
```

Restore drill outline:

```bash
sudo gitlab-ctl stop puma
sudo gitlab-ctl stop sidekiq
sudo gitlab-backup restore BACKUP=<timestamped_backup> force=yes
sudo gitlab-ctl restart
sudo gitlab-rake gitlab:check SANITIZE=true
```

Restore caveats observed in the lab:

- PostgreSQL extension ownership warnings for `pg_trgm`, `btree_gist`, and
  `amcheck` were non-fatal.
- HTTP can briefly return `502` while Puma recreates its socket after restore.
- The restore VM should remain private-only.

---

## 8. Validation

Check DNS:

```bash
dig @59.153.29.101 gitlab.v7.comptech-lab.com A +short
dig @30.30.200.53 gitlab.v7.comptech-lab.com A +short
```

Expected result:

```text
59.153.29.102
```

Check the sign-in route:

```bash
curl -sS -o /tmp/gitlab-signin.out -w '%{http_code}\n' \
  http://gitlab.v7.comptech-lab.com/users/sign_in
curl -sS -o /tmp/gitlab-root.out -w '%{http_code} %{redirect_url}\n' \
  http://gitlab.v7.comptech-lab.com/
```

Expected result:

```text
200
302 http://gitlab.v7.comptech-lab.com/users/sign_in
```

Check GitLab internals:

```bash
ssh ze@30.30.200.20 'sudo gitlab-rake gitlab:check SANITIZE=true'
```

Check firewall posture:

```bash
ssh ze@30.30.200.20 'sudo ufw status verbose'
```

Check that the VM is private-only:

```bash
ssh ze@30.30.200.2 'virsh domiflist gf-ocp-gitlab-01'
```

Expected posture:

- default incoming deny;
- SSH allowed;
- HTTP/HTTPS only from HAProxy;
- no public NIC attached.

---

## 9. Operational Setup

After the base install, create the first operator structures in GitLab:

- rotate the root password and store it in Vault;
- create the first human owner account;
- create the platform groups;
- create the platform repositories;
- register the dedicated GitLab runner VM separately;
- keep CI runner tokens in Vault.

The current lab model keeps the runner on a separate VM:

- `gf-ocp-gitlab-runner-01`
- private IP `30.30.200.22`

Do not register CI runners directly on the GitLab VM.

---

## 10. Troubleshooting

Common issues:

- Package install fails because the repo was not added before `apt-get`.
- First converge fails because `alertmanager["enable"] = false` was not in
  place before install.
- HAProxy returns `503` because the `http-check` Host header does not match
  the GitLab FQDN.
- GitLab sign-in returns `502` for a short time after restore while Puma
  recreates its socket.
- Direct Internet access is not available; plan package updates around the
  private egress design.

Quick recovery steps:

1. Confirm the VM still has only the private NIC.
2. Confirm DNS points to HAProxy, not the guest.
3. Confirm `gitlab.rb` contains the expected `external_url`.
4. Re-run `gitlab-ctl reconfigure`.
5. Re-run `gitlab-rake gitlab:check SANITIZE=true`.

---

## 11. Operator Checklist

Before first use:

- [ ] VM private-only
- [ ] static IP and DNS correct
- [ ] GitLab CE installed and reconfigured
- [ ] HAProxy route healthy
- [ ] root password rotated
- [ ] backup target configured
- [ ] config/secrets custody archived
- [ ] restore drill completed
- [ ] runner VM kept separate

