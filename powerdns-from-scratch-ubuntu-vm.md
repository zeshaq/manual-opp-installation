# PowerDNS From Scratch On Ubuntu VM

This guide describes how to build a fresh PowerDNS authoritative server plus
recursor on Ubuntu, configure it for a new environment, and use it as the DNS
foundation for an OpenShift-oriented lab.

The current CompTech greenfield pattern uses:

- PowerDNS Authoritative for the public and internal `v7.comptech-lab.com`
  zone
- PowerDNS Recursor on a private management IP for lab clients
- A single Ubuntu VM for both services
- SQLite zone storage for simplicity

If you are building a production deployment, split the authoritative service
and recursor across separate hosts and add a second authoritative server.
This guide still works if you only have one VM, but the single-host setup is a
lab pattern, not a resilience pattern.

The current lab reference addresses are:

- Authoritative public IP: `59.153.29.101`
- Recursor private IP: `30.30.200.53`
- Zone: `v7.comptech-lab.com`
- Hostnames: `pdns.v7.comptech-lab.com`, `ns1.v7.comptech-lab.com`,
  `ns2.v7.comptech-lab.com`

Replace those values with your own IPs, domain, and hostnames.

---

## 1. What you are building

You are building:

- an authoritative DNS server for your lab zone
- a recursor for VM and operator lookups
- a place to store cluster API, API-int, ingress, and service records
- a DNS source that OpenShift and platform services can depend on before the
  cluster exists

In the current greenfield pattern, the authoritative zone is the control point
for all of these records:

- cluster API and API-int names
- `*.apps` wildcard names
- service hostnames for Quay, Nexus, GitLab, NetBox, MinIO, Vault, and
  similar platform services

The recursor is used so local VMs and operators can resolve external names
without leaving the lab.

Important: do not treat this as a generic BIND install. The steps below are
tailored for PowerDNS with a SQLite authoritative backend plus a recursor.

---

## 2. Prerequisites

Before you begin, make sure you have:

1. A fresh Ubuntu 24.04 VM.
2. Root or sudo access.
3. One static IP for authoritative DNS.
4. One private IP for the recursor, if you want split-horizon lab resolution.
5. A domain you control.
6. A place to store the API key and any zone admin secrets.
7. Time sync working on the VM.

Recommended VM size for this pattern:

- 2 to 4 vCPU
- 4 to 8 GiB RAM
- 20 to 40 GiB disk
- one public or semi-public interface for authoritative DNS
- one private interface for the recursor if you want to keep recursion off the
  public side

Recommended network exposure:

- TCP/UDP 53 reachable from the networks that need DNS
- PowerDNS API on localhost only
- recursor reachable only from trusted lab subnets

---

## 3. Install packages

Install the base packages first:

```bash
sudo apt update
sudo apt install -y \
  ca-certificates \
  chrony \
  curl \
  fail2ban \
  jq \
  openssh-server \
  pdns-server \
  pdns-backend-sqlite3 \
  pdns-tools \
  pdns-recursor \
  sqlite3 \
  unzip
```

If you will use this server from OpenShift build hosts, also install:

```bash
sudo apt install -y dnsutils
```

Enable the local services:

```bash
sudo systemctl enable --now chrony
sudo systemctl enable --now fail2ban
sudo systemctl enable --now pdns
sudo systemctl enable --now pdns-recursor
```

---

## 4. Create the directory layout

Create the PowerDNS directories if they are not already present:

```bash
sudo mkdir -p /etc/powerdns/recursor.d
sudo mkdir -p /var/lib/powerdns
sudo chmod 0750 /etc/powerdns /etc/powerdns/recursor.d /var/lib/powerdns
sudo chown -R pdns:pdns /var/lib/powerdns
```

If you want local notes or backup exports, create a separate working area:

```bash
sudo mkdir -p /root/pdns-backups
sudo chmod 0700 /root/pdns-backups
```

Do not store secrets in Git. Keep the API key and any zone admin credentials
in a secret manager.

---

## 5. Configure the authoritative server

Edit `/etc/powerdns/pdns.conf`.

The current lab pattern uses these key settings:

```ini
launch=gsqlite3
gsqlite3-database=/var/lib/powerdns/pdns.sqlite3
gsqlite3-pragma-synchronous=0

local-address=127.0.0.1,59.153.29.101

webserver=yes
webserver-address=127.0.0.1
webserver-port=8081
webserver-allow-from=127.0.0.1,::1

api=yes
api-key=REPLACE_WITH_RANDOM_KEY
```

Notes:

- `local-address` should include the authoritative address you want to serve.
- Keep the API on localhost unless you have a strong reason not to.
- If you use a different public IP, replace `59.153.29.101`.
- If you want a secondary authoritative server later, add AXFR/IXFR and
  `allow-axfr-ips` after you have the second host.

Generate a strong API key if you do not already have one:

```bash
openssl rand -hex 32
```

Restart PowerDNS:

```bash
sudo systemctl restart pdns
sudo systemctl status pdns --no-pager -l
```

Verify the listener:

```bash
sudo ss -ltnup | grep -E '(:53|:8081)'
```

---

## 6. Configure the recursor

Edit `/etc/powerdns/recursor.conf`.

Keep the base file simple and add site-specific settings in
`/etc/powerdns/recursor.d/`.

Suggested base settings:

```ini
include-dir=/etc/powerdns/recursor.d
local-address=30.30.200.53
allow-from=127.0.0.0/8,30.30.0.0/16
webserver=yes
webserver-address=127.0.0.1
webserver-port=8082
webserver-allow-from=127.0.0.1,::1
lua-config-file=/etc/powerdns/recursor.lua
```

Then create a site-specific forwarder file such as:

```ini
# /etc/powerdns/recursor.d/10-lab-forwarder.conf
local-address=30.30.200.53
allow-from=127.0.0.0/8,30.30.0.0/16
forward-zones=v7.comptech-lab.com=127.0.0.1:53
forward-zones-recurse=.=1.1.1.1;8.8.8.8
```

What this does:

- queries for `v7.comptech-lab.com` are forwarded to the authoritative
  server on the same VM
- all other names recurse to public resolvers

Restart the recursor:

```bash
sudo systemctl restart pdns-recursor
sudo systemctl status pdns-recursor --no-pager -l
```

Verify the listener:

```bash
sudo ss -ltnup | grep -E '(:53|:8082)'
```

If you do not want a recursor at all, stop here and disable the service. The
authoritative server still works without it.

---

## 7. Create the zone

Create the zone with an appropriate primary nameserver and contact address.
Use a real hostmaster mailbox for your environment.

Example:

```bash
sudo pdnsutil create-zone v7.comptech-lab.com ns1.v7.comptech-lab.com hostmaster.v7.comptech-lab.com
```

If you want the lab to behave like the current greenfield setup, you can keep
`ns1` and `ns2` on the same host. In a better production build, use two
different authoritative servers.

Check the new zone:

```bash
sudo pdnsutil list-zone v7.comptech-lab.com
```

---

## 8. Add the core DNS records

Before you install OpenShift, create the records that the installer and later
operators will need.

The minimum OpenShift set is:

- `api.<cluster>`
- `api-int.<cluster>`
- `*.apps.<cluster>`

The current greenfield pattern also adds node names and service names.

Example record plan for a compact v7-style cluster:

```text
api.v7.comptech-lab.com           -> 30.30.210.10
api-int.v7.comptech-lab.com       -> 30.30.210.10
*.apps.v7.comptech-lab.com        -> 30.30.210.11
master-0.v7.comptech-lab.com      -> 30.30.210.13
master-1.v7.comptech-lab.com      -> 30.30.210.14
master-2.v7.comptech-lab.com      -> 30.30.210.15
```

Apply the records with `replace-rrset` or `add-record`.

Example:

```bash
sudo pdnsutil replace-rrset v7.comptech-lab.com api A 300 30.30.210.10
sudo pdnsutil replace-rrset v7.comptech-lab.com api-int A 300 30.30.210.10
sudo pdnsutil replace-rrset v7.comptech-lab.com '*.apps' A 300 30.30.210.11
sudo pdnsutil replace-rrset v7.comptech-lab.com master-0 A 300 30.30.210.13
sudo pdnsutil replace-rrset v7.comptech-lab.com master-1 A 300 30.30.210.14
sudo pdnsutil replace-rrset v7.comptech-lab.com master-2 A 300 30.30.210.15
```

Add platform service names as needed:

```bash
sudo pdnsutil replace-rrset v7.comptech-lab.com quay A 300 30.30.200.40
sudo pdnsutil replace-rrset v7.comptech-lab.com nexus A 300 30.30.200.41
sudo pdnsutil replace-rrset v7.comptech-lab.com minio A 300 59.153.29.102
sudo pdnsutil replace-rrset v7.comptech-lab.com minio-console A 300 59.153.29.102
sudo pdnsutil replace-rrset v7.comptech-lab.com gitlab A 300 59.153.29.102
sudo pdnsutil replace-rrset v7.comptech-lab.com netbox A 300 59.153.29.102
sudo pdnsutil replace-rrset v7.comptech-lab.com vault A 300 30.30.200.35
```

If you use a private split-horizon layout, add matching private hostnames too,
for example `quay-private` or `nexus-private`.

After editing, check the zone:

```bash
sudo pdnsutil check-zone v7.comptech-lab.com
```

---

## 9. Publish delegation in the parent zone

Your parent DNS provider must know where the zone lives.

For a simple lab, you need:

- NS records for `ns1` and `ns2`
- A/AAAA glue records if the nameservers are inside the delegated zone
- public reachability to TCP/UDP 53 on the authoritative server

If you are using a registrar or public DNS service for the parent zone, add:

- `ns1.v7.comptech-lab.com`
- `ns2.v7.comptech-lab.com`
- the zone delegation to those nameservers

If `ns1` and `ns2` are on the same host, be honest about that in the docs.
It is acceptable for a lab, but it is not redundant.

---

## 10. Validate the setup

Use three different checks:

1. local service state
2. direct authoritative queries
3. recursor queries

Service checks:

```bash
sudo systemctl is-active pdns
sudo systemctl is-active pdns-recursor
```

Direct authoritative query:

```bash
dig @127.0.0.1 api.v7.comptech-lab.com A +short
dig @59.153.29.101 api.v7.comptech-lab.com A +short
```

Recursor query:

```bash
dig @30.30.200.53 quay.v7.comptech-lab.com A +short
dig @30.30.200.53 google.com A +short
```

Zone health:

```bash
sudo pdnsutil check-zone v7.comptech-lab.com
```

If the records do not resolve through the recursor but do resolve directly on
the authoritative server, the forwarding file is wrong or the recursor cache
is stale.

If the authoritative server does not answer from the public IP, the listener
or firewall is wrong.

---

## 11. Harden the server

Do not skip the basics:

- keep the API on localhost
- keep the recursor allow list tight
- restrict TCP/UDP 53 to the networks that need it
- keep system time accurate
- keep the API key in a secret manager
- keep backups of the zone database and config files

Example firewall approach:

- allow 53 from the lab networks that need DNS
- allow 22 from your admin network only
- block everything else by default

If you use UFW:

```bash
sudo ufw allow from 30.30.0.0/16 to any port 53 proto udp
sudo ufw allow from 30.30.0.0/16 to any port 53 proto tcp
sudo ufw allow from <admin-network> to any port 22 proto tcp
sudo ufw enable
```

Replace `<admin-network>` with your management subnet.

---

## 12. Back up and restore

Back up both the database and the config files.

Recommended backup set:

- `/var/lib/powerdns/pdns.sqlite3`
- `/etc/powerdns/pdns.conf`
- `/etc/powerdns/recursor.conf`
- `/etc/powerdns/recursor.d/*.conf`
- `/etc/powerdns/recursor.lua`

Safe backup example:

```bash
sudo systemctl stop pdns
sudo cp /var/lib/powerdns/pdns.sqlite3 /root/pdns-backups/pdns.sqlite3.$(date -u +%Y%m%dT%H%M%SZ)
sudo tar -C /etc -czf /root/pdns-backups/powerdns-config.$(date -u +%Y%m%dT%H%M%SZ).tgz powerdns
sudo systemctl start pdns
```

If you prefer not to stop the service, use SQLite's backup tooling, but keep
the process documented and tested before trusting it.

Restore example:

```bash
sudo systemctl stop pdns
sudo cp /root/pdns-backups/pdns.sqlite3.<timestamp> /var/lib/powerdns/pdns.sqlite3
sudo chown pdns:pdns /var/lib/powerdns/pdns.sqlite3
sudo systemctl start pdns
```

After restore, run `pdnsutil check-zone` and a few `dig` checks.

---

## 13. Migration options

If you need to move PowerDNS to a new environment, you have three choices.

### Option 1: Restore the SQLite database

Best when the same VM family is being rebuilt.

- copy the SQLite file
- copy the config files
- restore the same IPs or update them consistently
- restart the services

### Option 2: Recreate the zone from records

Best when you want a clean rebuild.

- export the record list from your inventory or docs
- recreate the zone with `pdnsutil`
- add the records again
- verify with `dig`

### Option 3: Delegate a new authoritative server

Best when you want to move to a new object store or site without reusing the
old VM.

- stand up the new server
- create the new zone there
- update the parent NS delegation
- wait for TTL expiry
- retire the old server only after validation

For OpenShift-related records, do not cut over until both `api` and
`api-int` resolve correctly from the bootstrap and admin hosts.

---

## 14. Common failure modes

Watch for these:

- authoritative listener not bound on the public IP
- recursor forwarding file pointing at the wrong local address
- stale recursor cache after record changes
- zone serial not incremented after edits
- API key exposed in a world-readable file
- parent zone not delegating to the right nameservers

If a record exists in the authoritative zone but does not resolve through the
recursor, reload the zones and clear the recursor cache.

---

## 15. Quick validation checklist

Before you hand the server over, confirm:

- `pdns.service` is active
- `pdns-recursor.service` is active if you use it
- `pdnsutil check-zone` returns zero errors
- `dig` against the authoritative IP answers the zone
- `dig` against the recursor answers both internal and external names
- the parent zone points to the right nameservers
- backups exist and have been tested

