# Vault From Scratch On Ubuntu VM

This guide describes how to build the current greenfield Vault stack on
Ubuntu: the seed Vault, the three-node Raft cluster, transit auto-unseal, and
the backup model used for operator custody.

The current lab reference build uses:

- Seed VM: `gf-ocp-vault-seed-01`
- Seed IP: `30.30.200.30/16`
- Main node 1: `gf-ocp-vault-01` / `30.30.200.31`
- Main node 2: `gf-ocp-vault-02` / `30.30.200.32`
- Main node 3: `gf-ocp-vault-03` / `30.30.200.33`
- Client endpoint: `vault.v7.comptech-lab.com`
- Vault version: `1.21.1`
- Storage model: Integrated Storage / Raft
- Seal model: Shamir on the seed tier, transit auto-unseal on the main tier
- Backup bucket: `vault-raft-snapshots`
- Secret custody root: `/home/ze/codex-opp-agent/secrets/greenfield-vault/`

Vault is private-only. Do not publish it on a public interface.

---

## 1. What You Are Building

You are building two layers:

1. A seed Vault that can be initialized manually and used to issue the narrow
   transit token required by the main cluster.
2. A three-node main Vault Raft cluster that holds platform and application
   secrets for the lab.

The main cluster is what platform automation uses. The seed tier exists so the
main tier can auto-unseal without storing its own seal secret locally.

Do not store recovery keys, root tokens, transit tokens, CA private keys, or
node private keys inside Vault itself. Keep them in local operator custody or a
separate approved secret system.

---

## 2. Prerequisites

Before you begin, make sure you have:

1. A private-only lab network on `br33`.
2. DNS records for the seed host, three Raft nodes, and the client endpoint.
3. MinIO available for snapshot backups.
4. SSH access to the guest VMs as `ze`.
5. A plan for local-only TLS material.
6. A secure place to keep initialization output and recovery material.

Recommended VM size:

- 2 to 4 vCPU
- 4 to 8 GiB RAM
- 20 to 40 GiB disk per node

Network posture:

- SSH on `22`
- Vault API on `8200`
- Vault cluster traffic on `8201`
- nothing exposed directly to the public Internet

---

## 3. DNS And Address Plan

The current lab uses round-robin DNS for the main cluster:

```text
vault.v7.comptech-lab.com A 30.30.200.31
vault.v7.comptech-lab.com A 30.30.200.32
vault.v7.comptech-lab.com A 30.30.200.33
```

The seed host has its own name:

```text
gf-ocp-vault-seed-01.v7.comptech-lab.com A 30.30.200.30
```

Each node is private-only on the same lab network:

```yaml
version: 2
ethernets:
  enp1s0:
    match:
      macaddress: "REPLACE_WITH_VM_MAC"
    set-name: enp1s0
    dhcp4: false
    addresses:
      - REPLACE_WITH_VM_IP/16
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

---

## 4. Provision The Seed And Main VMs

Build the four VMs with cloud-init and a private-only NIC.

The current greenfield repo provides the helper pattern; if you are recreating
the lab manually, use the same shape:

- one NIC on `br33`
- private-only IPs
- no public gateway
- preserved SSH key for the operator
- cloud-init seed ISO

After boot, confirm:

```bash
ssh ze@gf-ocp-vault-seed-01
cloud-init status --wait
hostnamectl
ip -br addr
ip route
```

Repeat the same check for the three main nodes after they are created.

---

## 5. Install The Vault Binary

Install the base packages:

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

Install the Vault binary from HashiCorp release artifacts and verify the
download checksum before placing it in `/usr/local/bin`.

Then create the base directories:

```bash
sudo install -d -m 0750 /etc/vault.d
sudo install -d -m 0750 /opt/vault/data
sudo install -d -m 0750 /var/log/vault
sudo useradd --system --home /etc/vault.d --shell /usr/sbin/nologin vault || true
sudo chown -R vault:vault /opt/vault /var/log/vault /etc/vault.d
```

Enable the local hardening services:

```bash
sudo systemctl enable --now chrony fail2ban ssh
```

Allow only the management subnet to reach Vault:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow from 30.30.0.0/16 to any port 8200 proto tcp
sudo ufw allow from 30.30.0.0/16 to any port 8201 proto tcp
sudo ufw --force enable
```

---

## 6. Create TLS Material

Use local-only TLS material for the seed and main nodes.

The current lab pattern keeps the CA and node certificates local to the
operator workspace and does not publish private keys in Git.

If you are following the repo helpers, the current source tree includes a
Vault TLS render path. Otherwise generate:

- a local CA;
- one server certificate per Vault node;
- SAN entries for the node FQDN and the IP;
- a shared CA certificate for clients.

Every node should be able to present a certificate that matches its DNS name.

---

## 7. Configure The Seed Vault

The seed Vault is a short-lived control plane used to bootstrap the main
cluster.

Representative `vault.hcl`:

```hcl
ui = true
api_addr = "https://gf-ocp-vault-seed-01.v7.comptech-lab.com:8200"
cluster_addr = "https://gf-ocp-vault-seed-01.v7.comptech-lab.com:8201"

storage "raft" {
  path = "/opt/vault/data"
  node_id = "gf-ocp-vault-seed-01"
}

listener "tcp" {
  address         = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"
  tls_cert_file   = "/etc/vault.d/tls/server.crt"
  tls_key_file    = "/etc/vault.d/tls/server.key"
  tls_client_ca_file = "/etc/vault.d/tls/ca.crt"
}

disable_mlock = true
default_lease_ttl = "24h"
max_lease_ttl = "72h"
```

Start the service:

```bash
sudo systemctl enable --now vault
```

Initialize the seed Vault:

```bash
export VAULT_ADDR=https://gf-ocp-vault-seed-01.v7.comptech-lab.com:8200
export VAULT_CACERT=/etc/vault.d/tls/ca.crt
vault operator init -key-shares=5 -key-threshold=3
```

Save the unseal keys and root token in local custody. Do not paste them into a
ticket or chat.

Unseal the seed Vault with three of the five keys:

```bash
vault operator unseal
vault operator unseal
vault operator unseal
```

Then log in with the root token and enable the engines required for the main
cluster bootstrap:

```bash
vault login <root-token>
vault secrets enable -path=transit transit
vault audit enable file file_path=/var/log/vault/audit.log
```

Create a narrow policy for main auto-unseal and issue a scoped token for that
policy. Keep the token local-only.

The seed Vault is not the normal client endpoint. It is a bootstrap control
plane only.

---

## 8. Configure The Main Vault Nodes

Each main node uses Raft storage and transit auto-unseal.

Representative `vault.hcl` for a main node:

```hcl
ui = true
api_addr = "https://vault.v7.comptech-lab.com:8200"
cluster_addr = "https://gf-ocp-vault-01.v7.comptech-lab.com:8201"

storage "raft" {
  path    = "/opt/vault/data"
  node_id = "gf-ocp-vault-01"
  retry_join {
    leader_api_addr = "https://gf-ocp-vault-01.v7.comptech-lab.com:8200"
  }
  retry_join {
    leader_api_addr = "https://gf-ocp-vault-02.v7.comptech-lab.com:8200"
  }
  retry_join {
    leader_api_addr = "https://gf-ocp-vault-03.v7.comptech-lab.com:8200"
  }
}

seal "transit" {
  address = "https://gf-ocp-vault-seed-01.v7.comptech-lab.com:8200"
  token   = "<scoped-transit-token>"
  key_name = "main-vault-auto-unseal"
  mount_path = "transit/"
}

listener "tcp" {
  address         = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"
  tls_cert_file   = "/etc/vault.d/tls/server.crt"
  tls_key_file    = "/etc/vault.d/tls/server.key"
  tls_client_ca_file = "/etc/vault.d/tls/ca.crt"
}

disable_mlock = true
default_lease_ttl = "24h"
max_lease_ttl = "72h"
```

Start the first main node:

```bash
sudo systemctl enable --now vault
```

Initialize the main cluster on the first node:

```bash
export VAULT_ADDR=https://gf-ocp-vault-01.v7.comptech-lab.com:8200
export VAULT_CACERT=/etc/vault.d/tls/ca.crt
vault operator init -recovery-shares=5 -recovery-threshold=3
```

Join the other nodes:

```bash
vault operator raft join https://gf-ocp-vault-01.v7.comptech-lab.com:8200
```

After the join, verify:

```bash
vault status
vault operator raft list-peers
vault operator raft autopilot state
```

The main cluster should show three voters and be unsealed after restarts
without manual intervention.

---

## 9. Enable The Main Secret Engines

The normal operational client endpoint is the round-robin DNS name:

```text
vault.v7.comptech-lab.com
```

Use that endpoint for platform clients and automation.

On the leader, enable the common services:

```bash
vault secrets enable -path=secret kv-v2
vault audit enable file file_path=/var/log/vault/audit.log
```

If you use the current lab convention, application and platform secrets live
under:

- `secret/greenfield/*`
- `secret/apps/*`

Keep platform and app secret paths separate. Use narrow ACL policies. Do not
create a broad shared read policy for all application teams.

---

## 10. Backups To MinIO

The main Vault cluster snapshots to MinIO.

Current backup posture:

- bucket: `vault-raft-snapshots`
- object layout: timestamped snapshot plus checksum
- credential path: MinIO credential stored in Vault

The backup runbook should:

1. create a snapshot with `vault operator raft snapshot save`;
2. upload it to MinIO;
3. store a checksum beside it;
4. keep the snapshot timer enabled on the primary node.

If you change Vault, MinIO, or the backup script, run a restore validation
again. A backup that was never restored is only a claim.

---

## 11. Validation

Check health endpoints:

```bash
for endpoint in \
  gf-ocp-vault-seed-01.v7.comptech-lab.com \
  gf-ocp-vault-01.v7.comptech-lab.com \
  gf-ocp-vault-02.v7.comptech-lab.com \
  gf-ocp-vault-03.v7.comptech-lab.com \
  vault.v7.comptech-lab.com; do
  curl -fsS --cacert /etc/vault.d/tls/ca.crt \
    "https://${endpoint}:8200/v1/sys/health?standbyok=true&sealedcode=200" >/dev/null &&
    echo "${endpoint}: ok"
done
```

Check the main cluster:

```bash
export VAULT_ADDR=https://vault.v7.comptech-lab.com:8200
export VAULT_CACERT=/etc/vault.d/tls/ca.crt
vault status
vault operator raft list-peers
vault operator raft autopilot state
vault audit list
vault secrets list
```

Restart a follower and confirm it auto-unseals:

```bash
sudo systemctl restart vault
sleep 8
vault status
```

Expected posture:

- main cluster is initialized and unsealed;
- seal type is `transit`;
- storage type is `raft`;
- all three voters are present;
- audit logging is enabled;
- `secret/` is mounted;
- a restarted follower comes back unsealed.

---

## 12. Common Gotchas

Watch for these:

- The seed Vault is not the client endpoint.
- The transit token must be narrow and local-only.
- The main node certificates must match their DNS names.
- Do not mix seed paths and main paths.
- Do not store recovery keys or root tokens in Vault itself.
- If a follower restarts sealed, the transit token or seal config is wrong.
- If a node does not join the Raft set, the `retry_join` or TLS settings are
  usually wrong.

---

## 13. Recovery And Drills

Use a restore drill before you trust the cluster.

The current lab pattern validates:

- MinIO snapshot download;
- checksum verification;
- isolated restore into a throwaway Vault process or VM;
- readable restored metadata;
- cleanup of the temporary restore path.

If you need to rebuild the cluster from scratch:

1. Recreate the TLS material.
2. Bring up the seed Vault.
3. Recreate the transit token.
4. Bring up the three main nodes.
5. Re-initialize the main cluster if the data directory was lost.
6. Restore secrets from backup or re-seed them from the source of truth.

---

## 14. Operator Checklist

Before production use:

- [ ] seed Vault initialized and unsealed
- [ ] transit token created and scoped
- [ ] three main nodes joined to Raft
- [ ] main cluster auto-unseals on restart
- [ ] `vault.v7.comptech-lab.com` resolves to the main cluster
- [ ] audit logging enabled
- [ ] snapshot backup to MinIO enabled
- [ ] restore drill passed
- [ ] recovery material stored locally and not in Git

