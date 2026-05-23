# gf-ocp-bootstrap-01 Operator Guide

This guide describes how to build, harden, and use the greenfield bootstrap
host `gf-ocp-bootstrap-01` as the execution point for offline OpenShift
deployments in the v7 lab.

The host is not a generic utility VM. It is the control point for:

- render inputs for new OpenShift clusters;
- build and validate agent ISO media;
- hold cluster installation artifacts and local custody files;
- run read-only `oc` validation against installed clusters;
- stage disconnected mirror resources and install-time trust material.

If you are recreating this in another environment, replace the example
hostnames, IPs, and credentials with your own values. Keep the workflow
structure the same.

---

## 1. Reference Build

Current lab reference values:

| Item | Value |
| --- | --- |
| Hostname | `gf-ocp-bootstrap-01.v7.comptech-lab.com` |
| Jump path | `dl385-2 -> gf-ocp-bootstrap-01` |
| Guest OS | Red Hat Enterprise Linux 9.6 (Plow) |
| Virtualization | KVM guest |
| IP address | `30.30.200.60/16` |
| SSH user | `ze` |
| Nginx root | `/usr/share/nginx/html` |
| Artifact workdir | `/home/ze/ocp-greenfield-deployment/artifacts/openshift/` |
| Toolchain version | OpenShift `4.20.18` |
| Local binaries | `/usr/local/bin/oc`, `/usr/local/bin/openshift-install` |

The host currently runs:

- `nginx`
- `sshd`
- `fail2ban`
- `chronyd`
- `qemu-guest-agent`

Use the bootstrap host for OpenShift-related work only. Do not treat it as a
general application server.

---

## 2. What You Need Before You Start

Before building or reusing the host, make sure you have:

1. A working hypervisor, currently `dl385-2`.
2. SSH access to the hypervisor as `ze`.
3. A preserved or reproducible RHEL 9.6 guest image.
4. A plan for FIPS, vTPM, and static network configuration.
5. A reachable PowerDNS instance for the lab zone.
6. A reachable HAProxy or equivalent edge path if your environment uses one.
7. A Quay or other disconnected image source.
8. Vault or equivalent secret custody for installer pull secrets.
9. A place to keep generated reports and session notes.

Do not start install media creation until you have the DNS, mirror content,
and pull-secret inputs reviewed.

---

## 3. Build the VM From Scratch

The source tree contains the dedicated bootstrap VM provisioning helper:

```text
scripts/rebuild/vm-provisioning/create-ocp-bootstrap-vm.sh
```

The current reference build was produced from the preserved RHEL guest image
with:

- FIPS enabled before first boot;
- TPM 2.0 attached;
- static IP `30.30.200.60/16`;
- public key SSH access only;
- password authentication disabled;
- cloud-init based first boot configuration.

### 3.1 Provision the host

On the hypervisor, create the VM using the approved bootstrap VM helper or the
equivalent local libvirt workflow.

The key requirements are:

- one boot disk sized for the guest image;
- a MAC-address-stable NIC on `br33`;
- UEFI secure boot;
- TPM 2.0 emulator;
- cloud-init seed ISO for first boot;
- a persistent hostname and IP that match the DNS records.

If you are reusing the current greenfield scripts, the bootstrap workflow is:

```bash
./scripts/gfctl.sh prepare-cloud-init --execute gf-ocp-bootstrap-01
./scripts/gfctl.sh cloud-init-iso --execute gf-ocp-bootstrap-01
./scripts/gfctl.sh create-vm --execute scripts/vms/gf-ocp-bootstrap-01.env
```

### 3.2 First-boot network rules

The RHEL cloud-init renderer in this environment expects the route syntax to
be explicit. Use `0.0.0.0/0` for the default route if you are rendering
sysconfig-style network config.

Prefer MAC-based network matching over interface-name matching. The interface
name may change; the MAC does not.

### 3.3 First-boot validation

After the VM boots, verify:

```bash
ssh ze@gf-ocp-bootstrap-01
hostnamectl
cloud-init status --wait
cat /proc/sys/crypto/fips_enabled
fips-mode-setup --check
systemctl status sshd chronyd fail2ban qemu-guest-agent nginx --no-pager
```

Expected state:

- SSH works with key auth;
- `cloud-init` is `done`;
- FIPS is enabled;
- the support services are active.

---

## 4. Install the Host Toolchain

The host uses local copies of the pinned OpenShift tools. The current
reference versions are:

- `oc` `4.20.18`
- `kubectl` `4.20.18`
- `openshift-install` `4.20.18`
- `oc-mirror` from the pinned client bundle
- `butane`
- `jq`
- `podman`
- `skopeo`
- `git`
- `nmstatectl`

If you are following the current lab flow, the repo-provided install helpers
are:

```text
scripts/services/bootstrap/install-openshift-tools.sh
scripts/services/bootstrap/install-bootstrap-hardening-tools.sh
```

The host should be able to run the pinned OpenShift tools without depending on
random package drift.

### 4.1 Recommended package posture

Use approved package sources only:

- public UBI repos for general-purpose tools when allowed;
- a local RPM mirror if you need a fully disconnected bootstrap host;
- no Red Hat credentials should remain on the host once provisioning is done.

If you need to verify the installed toolchain:

```bash
oc version --client --short
openshift-install version
podman --version
git --version
nmstatectl version
```

---

## 5. Directory Layout And Custody

The bootstrap host stores all active work under:

```text
/home/ze/ocp-greenfield-deployment/artifacts/openshift/
```

Current cluster workdirs observed on the host:

- `hub-dc-v7`
- `hub-dr-v7`
- `spoke-dc-v7`
- `spoke-dc-v7-failed-20260516-082628`
- `spoke-dr-v7`

Each cluster workdir normally contains:

- `cluster.env`
- `install-config.yaml`
- `agent-config.yaml`
- `mirror-resources/`
- `auth/kubeconfig`
- `auth/kubeadmin-password`
- `agent.x86_64.iso`
- `.openshift_install.log`
- `.openshift_install_state.json`
- `rendezvousIP`

Treat the following as sensitive custody material:

- `auth/kubeconfig`
- `auth/kubeadmin-password`
- installer state files
- pull secrets

Never print those values into a shared terminal log or commit them to Git.

---

## 6. How The Host Is Used In Practice

The bootstrap host is the execution point for the offline install pipeline.
The current pattern is:

1. Prepare or refresh disconnected mirror content.
2. Build the cluster-specific `cluster.env`.
3. Render `install-config.yaml`, `agent-config.yaml`, and `mirror-resources/`.
4. Build the installer pull secret from Vault.
5. Validate the rendered inputs.
6. Generate the agent ISO.
7. Publish the ISO for the hypervisor to consume.
8. Boot or re-boot the cluster nodes.
9. Run `openshift-install agent wait-for bootstrap-complete`.
10. Run `openshift-install agent wait-for install-complete`.
11. Validate the cluster with read-only `oc` checks.

The host also acts as the read-only validation point after the cluster is up.
That means it is used for:

- `oc get nodes`
- `oc get co`
- `oc get clusterversion`
- `oc get mcp`
- route reachability checks
- catalog and mirror verification

---

## 7. Offline OpenShift Deployment Workflow

This is the operator sequence for a disconnected install.

### 7.1 Refresh mirror content first

The bootstrap host consumes the mirror resources. It does not replace the
mirror workflow itself.

Make sure the disconnected source bundle exists before rendering install
inputs. In this lab, `mirror-resources/` is generated by the mirror workflow
and then placed into the cluster workdir.

If you are refreshing the content, use the mirror worker and the pinned
`oc mirror --v2` flow, then copy the resulting manifests into the cluster
workdir.

### 7.2 Build the cluster workdir

Create or update the cluster directory under the artifact tree:

```bash
install -d /home/ze/ocp-greenfield-deployment/artifacts/openshift/<cluster>
cp bootstrap/openshift-install/cluster.env.example \
  /home/ze/ocp-greenfield-deployment/artifacts/openshift/<cluster>/cluster.env
```

Edit `cluster.env` for:

- cluster name
- base domain
- VIPs
- node IPs
- node MACs
- rendezvous IP
- root disk hints
- SSH key path
- mirror path
- FIPS and topology settings

### 7.3 Assemble the pull secret

Build the install pull secret from Vault or your equivalent secret store.

Keep the secret local to the host and do not print it in shell history.

The important property is that the pull secret must authenticate to the
disconnected registry path that hosts:

- the OpenShift release image;
- the operator catalog images;
- any platform support images that the install path requires.

### 7.4 Render install inputs

Render the install configuration and agent configuration:

```bash
cd /home/ze/ocp-greenfield-deployment
./scripts/gfctl.sh render-openshift-install-inputs --execute \
  --env artifacts/openshift/<cluster>/cluster.env \
  --output-dir artifacts/openshift/<cluster>
```

The rendered workdir should contain:

```text
cluster.env
install-config.yaml
agent-config.yaml
mirror-resources/
```

### 7.5 Validate the rendered inputs

Run the disconnected preflight:

```bash
./scripts/gfctl.sh validate-openshift-install-preflight --execute \
  --env artifacts/openshift/<cluster>/cluster.env \
  --input-dir artifacts/openshift/<cluster>
```

The preflight should confirm:

- the rendered YAML is valid;
- the trust bundle is present and correct;
- release and operator content resolve from the disconnected registry;
- FIPS is enabled in the rendered install inputs;
- no generated auth material was staged into Git.

Do not generate the ISO until this preflight passes.

### 7.6 Generate the agent ISO

Once the render and preflight are clean:

```bash
cd /home/ze/ocp-greenfield-deployment/artifacts/openshift/<cluster>
openshift-install agent create image --log-level=info
```

The resulting ISO is typically named:

```text
agent.x86_64.iso
```

### 7.7 Publish the ISO for the hypervisor

The host serves static media from:

```text
/usr/share/nginx/html
```

Put the published ISO somewhere predictable under `/usr/share/nginx/html/ocp/`
and keep the path consistent for operators.

The current lab has used both file-per-cluster and directory-per-cluster
layouts. The important thing is to publish the exact URL you intend to boot
from and verify it with `curl -I` before attaching the ISO to any VM.

Example validation:

```bash
curl -I http://gf-ocp-bootstrap-01/ocp/<cluster>-agent.x86_64.iso
```

If the URL returns `404`, the ISO was not published to the nginx tree yet.

### 7.8 Boot and monitor the cluster

From the bootstrap host, monitor installer progress:

```bash
openshift-install agent wait-for bootstrap-complete --log-level=info
openshift-install agent wait-for install-complete --log-level=info
```

Then validate the cluster:

```bash
export KUBECONFIG=/home/ze/ocp-greenfield-deployment/artifacts/openshift/<cluster>/auth/kubeconfig
oc get nodes -o wide
oc get clusterversion
oc get co
oc get mcp
```

Expected base-state result:

- all intended control-plane nodes are `Ready`;
- `ClusterVersion` is available;
- core operators are available and not degraded;
- the master MCP is updated.

---

## 8. FIPS, Compliance, And PCI-DSS Considerations

This section is about control posture, not certification claims.

### 8.1 FIPS

The bootstrap host and the cluster install should both be treated as FIPS
assets from the start.

Checks:

```bash
cat /proc/sys/crypto/fips_enabled
fips-mode-setup --check
```

What matters operationally:

- enable FIPS before first boot where practical;
- render `fips: true` into the OpenShift install inputs;
- do not mix a FIPS install path with non-FIPS assumptions later.

### 8.2 Secrets handling

Treat the bootstrap host as a custody point, not a secret store.

Rules:

- keep pull secrets, kubeconfigs, kubeadmin passwords, and token material out
  of Git;
- fetch secret values from Vault or an equivalent system only when needed;
- never paste secret values into shared issue comments or logs;
- clean up temporary files after use.

### 8.3 Access control

Recommended controls:

- SSH key authentication only;
- no password authentication;
- a single operator account with sudo;
- no shared admin logins;
- jump-host access through `dl385-2` only.

### 8.4 Logging and evidence

Keep operational evidence in the repo:

- generated reports under `reports/`;
- session notes under `reports/sessions/`;
- no secret values in reports.

For a PCI-DSS-minded deployment, this host should also have:

- change control for every live action;
- clear operator identity;
- time sync;
- patch cadence;
- service hardening;
- documented evidence for validation and rollback.

This bootstrap host does not make the environment PCI compliant by itself.
It only provides a controllable execution point for the OpenShift install
workflow.

---

## 9. Monitoring And Routine Checks

Use these checks during normal operations.

### 9.1 Host health

```bash
hostnamectl
uptime
df -h
systemctl status nginx sshd fail2ban chronyd qemu-guest-agent --no-pager
ss -ltnp
```

### 9.2 Artifact hygiene

```bash
du -sh /home/ze/ocp-greenfield-deployment/artifacts/openshift
find /home/ze/ocp-greenfield-deployment/artifacts/openshift -maxdepth 2 -name 'agent.x86_64.iso'
```

### 9.3 Nginx publication checks

```bash
curl -I http://gf-ocp-bootstrap-01/
curl -I http://gf-ocp-bootstrap-01/ocp/<cluster>-agent.x86_64.iso
```

### 9.4 Install log checks

```bash
tail -f /home/ze/ocp-greenfield-deployment/artifacts/openshift/<cluster>/.openshift_install.log
```

### 9.5 Cluster checks after install

```bash
oc get nodes -o wide
oc get clusterversion
oc get co
oc get mcp
oc get ingresscontroller -A
oc get route -A | head
```

### 9.6 Hypervisor checks

On `dl385-2`, verify the cluster VM state:

```bash
virsh domstate <vm-name>
virsh domiflist <vm-name>
virsh domblklist <vm-name>
```

Use the hypervisor to confirm the ISO is attached only when needed.

---

## 10. Troubleshooting

### 10.1 SSH to the bootstrap host fails

Check:

- `dl385-2` DNS and SSH config;
- the `gf-ocp-bootstrap-01` host key;
- the bootstrap VM is running;
- the public key is authorized for `ze`.

Quick test:

```bash
ssh ze@dl385-2
ssh gf-ocp-bootstrap-01
```

### 10.2 The ISO URL returns 404

This means the ISO was generated but not published to the nginx tree, or the
published path does not match the URL the operator is using.

Fix:

- confirm the ISO exists in the cluster workdir;
- copy or symlink it into `/usr/share/nginx/html/ocp/`;
- validate with `curl -I` again.

The current lab once had a path mismatch where a `spoke-dc-v7` ISO was
published but the `spoke-dr-v7` URL returned `404`. Do not assume the file
name or path is correct without checking.

### 10.3 `openshift-install` or `oc` is missing

The host uses local binaries under `/usr/local/bin`. Do not assume the tools
are installed as RPM packages.

Check:

```bash
which oc
which openshift-install
oc version --client --short
openshift-install version
```

### 10.4 Preflight fails on disconnected image resolution

This usually means one of the following:

- mirror content was not refreshed;
- the trust bundle is stale;
- the pull secret does not authenticate to the mirror registry;
- the cluster workdir points at the wrong image source.

Re-run the mirror flow, then re-render the cluster inputs and preflight.

### 10.5 The install loops back into the agent ISO

After the nodes write their disks, remove the ISO from persistent boot order.
If a VM keeps returning to the install media:

- detach the ISO;
- reboot from disk only;
- confirm the VM is not still pointing at the install image.

### 10.6 Installer waits never complete

Check:

- DNS records for API, API-int, and apps;
- the node IP plan and MAC addresses;
- the hypervisor VM boot state;
- the mirror content;
- the bootstrap log.

Useful commands:

```bash
oc get nodes
oc get co
oc get clusterversion
tail -f /home/ze/ocp-greenfield-deployment/artifacts/openshift/<cluster>/.openshift_install.log
```

### 10.7 Sensitive files show up in history

If kubeconfig, kubeadmin password, or pull secret content appears in a log or
commit:

- stop and clean the leak immediately;
- rotate the affected credential if needed;
- never leave the leaked value in reports or issue comments.

---

## 11. Recovery And Migration Notes

If you need to rebuild the host:

- rebuild the VM from the preserved image or the approved provisioning flow;
- restore the toolchain;
- restore the artifact tree from Git and backups;
- revalidate DNS, mirror access, and nginx publication;
- confirm the current cluster workdirs are intact before touching live
  clusters.

If you need to move the workflow to another hypervisor:

- recreate the VM shape;
- recreate the SSH trust path;
- copy the artifact tree or restore it from the source-controlled inputs;
- verify the nginx publication path before booting any cluster nodes.

The bootstrap host is recoverable, but the generated artifacts and local
credentials are the real operational dependency. Protect them accordingly.

---

## 12. Quick Operator Checklist

Before a new install:

- [ ] `gf-ocp-bootstrap-01` reachable from `dl385-2`
- [ ] FIPS enabled on the host
- [ ] toolchain present in `/usr/local/bin`
- [ ] mirror resources refreshed
- [ ] pull secret assembled from Vault
- [ ] cluster workdir rendered
- [ ] preflight passed
- [ ] ISO generated
- [ ] ISO published under nginx and verified with `curl -I`
- [ ] VMs booted from the ISO
- [ ] `bootstrap-complete` reached
- [ ] `install-complete` reached
- [ ] cluster validated with `oc`

After the install:

- [ ] kubeconfig kept in secure local custody
- [ ] kubeadmin password not printed or committed
- [ ] nginx path checked for the next run
- [ ] logs and reports saved under `reports/`

