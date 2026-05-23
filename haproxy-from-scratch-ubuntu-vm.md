# HAProxy From Scratch On Ubuntu VM

This guide describes how to build a fresh HAProxy edge VM on Ubuntu, expose
services through TLS, and keep the config maintainable enough to survive a
real lab lifecycle.

The current CompTech greenfield pattern uses HAProxy as the public edge for
platform services, not for OpenShift API or ingress. The OpenShift cluster
records live in PowerDNS and resolve directly to the cluster VIPs.

The current lab reference addresses are:

- Public edge IP: `59.153.29.102`
- Private stats IP: `30.30.200.102`
- Hostname: `haproxy.v7.comptech-lab.com`

Replace those values with your own hostnames, IPs, and service targets.

---

## 1. What you are building

You are building:

- a TLS-terminating reverse proxy
- a host-based routing edge for VM-hosted services
- a private stats endpoint for operators
- a place to centralize certificates for public service names

In the current greenfield v7 build, HAProxy fronts:

- GitLab
- NetBox
- MinIO API and console
- Quay
- Nexus
- the insurance-app suite
- a few lab/admin endpoints such as APIC and HPE VME

It does not currently front:

- OpenShift API
- OpenShift API-int
- OpenShift ingress
- OpenShift console

Those cluster endpoints are handled directly by DNS in the current pattern.
If you want HAProxy to front OpenShift in another design, treat that as a
separate architecture and validate it independently.

---

## 2. Prerequisites

Before you begin, make sure you have:

1. A fresh Ubuntu 24.04 VM.
2. Root or sudo access.
3. A public IP or edge network address.
4. A private management address for stats and admin access.
5. DNS records ready for the service hostnames you plan to expose.
6. TLS certificates ready to install, or a plan to issue them.
7. Backend services reachable from the HAProxy VM.

Recommended VM size for this pattern:

- 2 to 4 vCPU
- 2 to 4 GiB RAM
- 20 to 40 GiB disk
- one public interface
- one private interface for stats and management

Recommended network exposure:

- TCP 80 and 443 on the public interface
- stats port only on the private interface
- SSH only from admin networks

---

## 3. Install packages

Install the base packages first:

```bash
sudo apt update
sudo apt install -y \
  ca-certificates \
  certbot \
  curl \
  fail2ban \
  haproxy \
  jq \
  openssh-server \
  openssl \
  unzip
```

If you use a DNS-01 flow through PowerDNS, install the tooling you need for
that method too. The current lab has used both timer-driven certificate
automation and manual PEM deployment at different points, so pick one path and
document it.

Enable the local services:

```bash
sudo systemctl enable --now fail2ban
sudo systemctl enable --now haproxy
```

---

## 4. Create the directory layout

Create the HAProxy directories if they are not already present:

```bash
sudo mkdir -p /etc/haproxy/certs
sudo mkdir -p /etc/haproxy/certs-disabled
sudo mkdir -p /etc/haproxy/errors
sudo chmod 0750 /etc/haproxy/certs /etc/haproxy/certs-disabled
```

If you keep operator notes or backups on the VM, store them separately from
the live config.

---

## 5. Prepare the TLS certificates

The live greenfield pattern uses wildcard certs such as:

- `v7-wildcard.pem`
- `insurance-app-wildcard.pem`

Your choices for cert provisioning are:

### Option A: DNS-01 with a DNS plugin

Best when you control the DNS zone and want wildcard certs.

Typical flow:

1. Create the DNS TXT records required by the ACME challenge.
2. Issue a wildcard certificate for the root lab domain.
3. Combine the certificate and private key into a PEM file.
4. Place the PEM file under `/etc/haproxy/certs/`.
5. Reload HAProxy.

### Option B: Manual PEM deployment

Best when certificate issuance is handled elsewhere.

Typical flow:

1. Obtain the certificate and private key off the VM.
2. Combine them into a single PEM file.
3. Copy the PEM into `/etc/haproxy/certs/`.
4. Reload HAProxy.

The important part is the final shape:

- HAProxy expects PEM files in `/etc/haproxy/certs/`
- the `bind ... ssl crt /etc/haproxy/certs/` directive loads them
- one wildcard PEM can cover several hostnames if the SANs or wildcard match

Check the certificate content:

```bash
openssl x509 -in /etc/haproxy/certs/v7-wildcard.pem -noout -subject -issuer -dates
```

If a cert is old or wrong, move it into `/etc/haproxy/certs-disabled/` until
you fix the issue.

---

## 6. Write the base HAProxy config

Edit `/etc/haproxy/haproxy.cfg`.

Use this structure:

- `global`
- `defaults`
- `frontend fe_http_public`
- `frontend fe_https_public`
- `listen stats_private`
- one `backend` per service

Example base settings:

```haproxy
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy

defaults
    mode http
    log global
    option httplog
    option dontlognull
    timeout connect 5s
    timeout client  50s
    timeout server  50s
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http
```

Do not copy this blindly. Keep the same shape, but match the ports and host
names to your environment.

---

## 7. Build the HTTP frontend

The HTTP frontend should do three things:

1. answer `/healthz`
2. redirect public app traffic to HTTPS
3. send ACME challenge traffic to the certificate helper

Example:

```haproxy
frontend fe_http_public
    bind 59.153.29.102:80
    mode http

    acl path_health path /healthz
    acl path_acme path_beg /.well-known/acme-challenge/

    acl host_gitlab hdr(host) -i gitlab.v7.comptech-lab.com
    acl host_netbox hdr(host) -i netbox.v7.comptech-lab.com
    acl host_quay hdr(host) -i quay.v7.comptech-lab.com
    acl host_nexus hdr(host) -i nexus.v7.comptech-lab.com

    http-request return status 200 content-type text/plain string "haproxy ready\n" if path_health
    use_backend be_certbot_acme if path_acme

    http-request redirect scheme https code 301 if host_gitlab
    http-request redirect scheme https code 301 if host_netbox
    http-request redirect scheme https code 301 if host_quay
    http-request redirect scheme https code 301 if host_nexus

    http-request return status 503 content-type text/plain string "No backend configured yet\n" if !path_health !path_acme !host_gitlab !host_netbox !host_quay !host_nexus
```

If your environment has more hostnames, add them one by one and keep the
health endpoint intact.

---

## 8. Build the HTTPS frontend

The HTTPS frontend should terminate TLS and route on the host header.

Example:

```haproxy
frontend fe_https_public
    bind 59.153.29.102:443 ssl crt /etc/haproxy/certs/
    mode http
    option httplog

    acl host_minio_api hdr(host) -i minio.v7.comptech-lab.com
    acl host_minio_console hdr(host) -i minio-console.v7.comptech-lab.com
    acl host_netbox hdr(host) -i netbox.v7.comptech-lab.com
    acl host_quay hdr(host) -i quay.v7.comptech-lab.com
    acl host_nexus hdr(host) -i nexus.v7.comptech-lab.com

    http-request return status 404 content-type text/plain string "No HTTPS backend configured\n" if !host_minio_api !host_minio_console !host_netbox !host_quay !host_nexus

    use_backend be_minio_api if host_minio_api
    use_backend be_minio_console if host_minio_console
    use_backend be_netbox_http if host_netbox
    use_backend be_quay_http if host_quay
    use_backend be_nexus_http if host_nexus
```

For services that care about forwarded headers, set:

- `X-Forwarded-Proto`
- `X-Forwarded-Ssl`
- `X-Forwarded-Host`

Use those headers consistently so upstream apps generate correct URLs.

---

## 9. Add the backend services

Create one backend block per service.

The current lab patterns look like this:

```haproxy
backend be_gitlab_http
    mode http
    option httpchk
    http-check send meth GET uri /users/sign_in ver HTTP/1.1 hdr Host gitlab.v7.comptech-lab.com
    http-check expect rstatus (2|3)[0-9][0-9]
    server gitlab01 30.30.200.20:80 check

backend be_quay_http
    mode http
    option httpchk
    http-check send meth GET uri /api/v1/discovery ver HTTP/1.1 hdr Host quay.v7.comptech-lab.com
    http-check expect status 200
    http-request set-header X-Forwarded-Proto https
    http-request set-header X-Forwarded-Ssl on
    server quay01 30.30.200.40:8080 check

backend be_nexus_http
    mode http
    option httpchk
    http-check send meth GET uri /service/rest/v1/status ver HTTP/1.1 hdr Host nexus.v7.comptech-lab.com
    http-check expect status 200
    http-request set-header X-Forwarded-Proto https
    http-request set-header X-Forwarded-Ssl on
    server nexus01 30.30.200.41:8081 check
```

For each service:

1. identify the real backend IP and port
2. decide whether the service speaks HTTP or HTTPS
3. define a health check that matches the application
4. add the host header if the app expects virtual hosting
5. reload only after `haproxy -c` succeeds

For backends that are already TLS-enabled upstream, use re-encryption:

```haproxy
backend be_example_tls
    mode http
    server example 10.0.0.50:443 ssl verify none check check-ssl
```

That pattern is useful for WSO2, APIC, and other self-signed appliances.

---

## 10. Add the private stats listener

Expose the statistics page only on the private management address.

Example:

```haproxy
listen stats_private
    bind 30.30.200.102:8404
    mode http
    stats enable
    stats uri /stats
    stats refresh 10s
```

Do not put the stats page on the public edge unless you also add strong
network controls and authentication.

---

## 11. Add ACME helper support if needed

If you use a local ACME helper for certificate automation, wire it into a
dedicated backend:

```haproxy
backend be_certbot_acme
    mode http
    server certbot01 127.0.0.1:8888
```

This backend only matters during issuance or renewal workflows. If your
certificate process does not need it, remove it and keep the config simpler.

---

## 12. Validate the config

Before you reload HAProxy, validate the syntax:

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```

If the syntax fails, do not reload.

Check the service:

```bash
sudo systemctl is-active haproxy
sudo systemctl status haproxy --no-pager -l
```

Check the listeners:

```bash
sudo ss -ltnp | grep haproxy
```

Check the edge:

```bash
curl -I http://59.153.29.102/healthz
curl -I https://quay.v7.comptech-lab.com/
curl -I https://nexus.v7.comptech-lab.com/
```

If you expose the stats page only privately, test it from the private network
or via SSH port forwarding.

---

## 13. Reload safely

Use a backup-before-edit workflow.

Example:

```bash
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak.$(date -u +%Y%m%dT%H%M%SZ)
sudo editor /etc/haproxy/haproxy.cfg
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy
```

If the reload fails, restore the backup and retry.

Keep dated backups. They are useful, but they are not a substitute for a real
source of truth.

---

## 14. Firewall and exposure

The edge should be reachable only where needed.

Recommended baseline:

- public 80 and 443 from the internet or edge network
- private 8404 only from management/admin networks
- SSH only from trusted admin hosts

Example UFW approach:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow from <admin-network> to any port 22 proto tcp
sudo ufw allow from <admin-network> to any port 8404 proto tcp
sudo ufw enable
```

Replace `<admin-network>` with your own management subnet.

---

## 15. Back up and restore

Back up at least:

- `/etc/haproxy/haproxy.cfg`
- `/etc/haproxy/certs/*.pem`
- `/etc/haproxy/errors/*`
- any helper scripts you use for certificate refresh

Backup example:

```bash
sudo tar -C /etc -czf /root/haproxy-backup.$(date -u +%Y%m%dT%H%M%SZ).tgz haproxy
```

Restore example:

```bash
sudo tar -C /etc -xzf /root/haproxy-backup.<timestamp>.tgz
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy
```

If you lose the certs, the config can still be restored, but TLS traffic will
fail until the PEMs are recreated.

---

## 16. Migration options

If you need to move HAProxy to a new VM or new hypervisor, use one of these
paths.

### Option 1: Clone the config and certs

Best when the target environment is almost identical.

- install HAProxy on the new host
- copy the config and certs
- adjust only the IP addresses and backend targets that changed
- validate and reload

### Option 2: Rebuild from a clean config

Best when the old config has a lot of manual drift.

- write a fresh `haproxy.cfg`
- re-add only the required hostnames
- issue fresh certs
- validate each backend one at a time

### Option 3: Move the edge role to another proxy layer

Best when you are redesigning the perimeter.

- define the new edge architecture first
- decide where TLS terminates
- decide whether OpenShift remains direct-to-VIP or moves through the proxy
- cut over the DNS only after backend checks pass

For the current greenfield v7 design, do not assume HAProxy should front
OpenShift unless you explicitly add that role to the new design.

---

## 17. Common failure modes

Watch for these:

- cert files not present in `/etc/haproxy/certs/`
- wrong PEM ordering or broken key/cert pairing
- host ACL missing for a new service
- backend health check mismatched to the app path
- backend service not reachable from the HAProxy VM
- public and private addresses swapped in the bind lines
- stats page exposed on the wrong interface

If a service returns 503 from HAProxy, check:

1. the backend IP and port
2. the host ACL
3. the health check path
4. the upstream app log

---

## 18. Quick validation checklist

Before you hand the edge over, confirm:

- `haproxy -c` passes
- `haproxy.service` is active
- public `80` and `443` listeners are present
- the stats page is only on the private address
- the certificates are valid and current
- each backend returns a healthy response
- DNS points the hostnames to the right edge IPs
- OpenShift records remain direct-to-VIP if that is your chosen design

