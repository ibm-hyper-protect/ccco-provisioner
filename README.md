# ccco-provisioner

**GitHub:** https://github.com/ibm-hyper-protect/ccco-provisioner

Ansible automation to install the IBM Confidential Containers for Red Hat OpenShift Container Platform ([CCCO](https://www.ibm.com/docs/en/rhocp-ibm-z?topic=confidential-computing-containers-red-hat-openshift-container-platform)) stack on a bare metal OpenShift cluster, generate an encrypted CCCO contract, and produce pod manifests for the **PayNow** and **Busybox** sample workloads. The final output is a CCCO-ready cluster plus signed, contract-bound pod manifests that you apply with `oc` to run the workloads as confidential containers.

The stack installed on the cluster is: IBM Secure Execution (SE) → Node Feature Discovery (NFD) → OpenShift Sandboxed Containers (OSC) → KataConfig (`kata-cc` RuntimeClass, handler `kata-se`).

All cluster work runs on a **bastion** host over SSH, using `kubernetes.core` modules and the kubeconfig on the bastion.

## Contents

- [Prerequisites](#prerequisites)
- [Push the IBM CCCO image to a registry](#1-push-the-ibm-ccco-image-to-a-registry)
- [Add the required artifacts](#2-add-the-required-artifacts)
- [Set up SSH access](#3-set-up-ssh-access)
- [Configure the inventory](#4-configure-the-inventory)
- [Set up the cluster environment](#5-set-up-the-cluster-environment)
- [Deploy the workloads](#6-deploy-the-workloads)
- [Clean up the environment](#clean-up-the-environment)
- [Reference](#reference)


## Prerequisites

**Controller (your local machine)**

- `ansible-core` 2.14+ and the `kubernetes.core` collection (installed in [step 4](#4-configure-the-inventory))
- Python `kubernetes` and `openshift` packages (required by `kubernetes.core`)
- `sshpass` — the generated hosts file sets `ansible_ssh_pass`, and the jump-host `ProxyCommand` uses it when a jump host password is set

**Bastion**

- Root access (the same password is used as the `become` password)
- A cluster kubeconfig at `<cluster.auth_dir_path>/kubeconfig`
- `oc` CLI, `pip3`, `openssl`, `base64`, `gzip`, and network access to install the `kubernetes` Python library (each role installs it)

**Images**

- The IBM CCCO image downloaded locally, with Podman and GPG installed on the management server (see step 1)


## 1. Push the IBM CCCO image to a registry

After downloading the IBM CCCO image, push it to a remote registry such as `quay.io` or `icr.io` before deploying. The pushed image reference is what you set as `kata.addon_image` in [step 4](#4-configure-the-inventory). For `icr.io` specifics, see [Getting started with IBM Cloud Container Registry](https://cloud.ibm.com/docs/Registry?topic=Registry-getting-started).

First, log in to your registry with `podman login quay.io` or `podman login <region>.icr.io` and enter your credentials at the prompt. See your registry's documentation for how to obtain them (for `icr.io`, the user name is `iamapikey` and the password is an IAM API key).

> **Do not pass the password with `-p` / `--password`.** It is stored in your shell history and is visible to other users in the process list. Use the interactive prompt, or `--password-stdin` with the secret read from a protected source.

Then run the commands for your registry (`quay.io` or `icr.io`):

```bash
# 1. Load the image into the local registry, then verify
podman load -i images/ibm-confidential-computing-container-v1.2.2.2.tar.gz
podman images

# 2. Tag the image for your registry
podman tag docker.io/ibm_ccco/ibm-ccco-podvm-container-image:1.2.2.2 quay.io/<your_namespace>/ibm-ccco-podvm-container-image:1.2.2.2   # quay.io
podman tag docker.io/ibm_ccco/ibm-ccco-podvm-container-image:1.2.2.2 icr.io/<your_namespace>/ibm-ccco-podvm-container-image:1.2.2.2    # icr.io

# 3. Push the image
podman push quay.io/<your_namespace>/ibm-ccco-podvm-container-image:1.2.2.2       # quay.io
podman push icr.io/<your_namespace>/ibm-ccco-podvm-container-image:1.2.2.2        # icr.io
```

Verify the image is in your registry:

- `quay.io`: open `https://quay.io/repository/<namespace>/ibm-ccco-podvm-container-image?tab=tags-list` in a browser
- `icr.io`: run `ibmcloud cr image-list`

> The registry must be reachable from the cluster worker nodes, or set `kata.verify_addon_kernel_service: false` in step 4.


## 2. Add the required artifacts

Place these files in the `artifacts/` directory (git-ignored) before running:

| File | Description |
|---|---|
| `HKD*.crt` | Host Key Document certificate |
| `*.crt` (non-HKD) | IBM CCCO contract encryption certificate |
| `*.bin` | SE header binary |
| `ibm-ccco-sealed-secret-creator-client-*` | CCCO sealed-secret-creator-client binary (s390x) |
| `common_app_cert.pem` | TLS certificate for the PayNow application |
| `common_app_key.pem` | TLS private key for the PayNow application |

- All files except the two `common_app_*.pem` files are auto-discovered by pattern. The first match is used, so keep only one file per pattern.
- `common_app_*.pem` are copied on **every** workload run (PayNow and Busybox), so both must exist even if you only deploy Busybox.

**Don't have a certificate handy?** Generate a self-signed pair for testing from the repository root (use a real certificate for anything beyond testing):

```bash
openssl req -x509 -newkey rsa:4096 -nodes -days 365 \
  -subj "/CN=<your-hostname>" \
  -keyout artifacts/common_app_key.pem \
  -out artifacts/common_app_cert.pem
```

Keep the `.pem` file names. A `.crt` name in `artifacts/` would be picked up as the contract encryption certificate.


## 3. Set up SSH access

Configure SSH access from your local machine to the bastion:

```bash
# 1. Generate an SSH key pair (skip if ~/.ssh/ansible-ocpz already exists)
ssh-keygen -t rsa -b 4096 -f ~/.ssh/ansible-ocpz -N ""

# 2. Copy the public key to the bastion
ssh-copy-id -i ~/.ssh/ansible-ocpz.pub root@<bastion-ip>

# 3. Verify the connection
ssh -i ~/.ssh/ansible-ocpz root@<bastion-ip>
```

> The key name `ansible-ocpz` matches `private_key_file` in `ansible.cfg`. If you use a different name, update `ansible.cfg`.


## 4. Configure the inventory

```bash
# 1. Install the required Ansible collection
ansible-galaxy collection install -r requirements.yml

# 2. Copy the template to all.yaml and fill it in
cp inventories/default/group_vars/all.yaml.template inventories/default/group_vars/all.yaml
vi inventories/default/group_vars/all.yaml
```

The file has eight sections:

```yaml
# Section 1 - Jump host (optional)
jump_host:
  ip: ""                  # leave blank to disable
  user: ""
  pass: ""                # leave blank if an SSH key is set

# Section 2 - Bastion
env:
  bastion:
    vm_name: <bastion-vm-name>
    cluster_name: <ocp-cluster-name>
    networking:
      ip: <bastion-ip>
      hostname: <bastion-fqdn>
    access:
      user: <ssh-user>
      pass: <ssh-password>

# Section 3 - Contract security
contract:
  signing_passphrase: "<passphrase>"
  workload_secret: "<secret>"

# Section 4 - Log router (optional)
log:
  public_ingress_endpoint: <endpoint>
  log_apikey: <api-key>

# Section 5 - NFD operator
nfd_source: "redhat-operators"

# Section 6 - OSC operator
osc_source: "redhat-operators"
osc_version: "v1.13.1"

# Section 7 - KataConfig
kata:
  addon_image: ""         # e.g. quay.io/<your_namespace>/ibm-ccco-podvm-container-image:1.2.2.2
  kernel_path: ""         # e.g. /image/ccco-bm-2.0.11.img
  verify_addon_kernel_service: true

# Section 8 - Cluster auth
cluster:
  auth_dir_path: ""       # e.g. /root/auth
```

| Section | Field | Description |
|---|---|---|
| 1 Jump host | `jump_host.ip` / `user` / `pass` | IP, user and password of a jump host reachable from your machine. Leave `ip` blank to disable |
| 2 Bastion | `vm_name`, `cluster_name` | Logical VM name and OCP cluster name |
| | `networking.ip` / `hostname` | IP the controller SSHes to, and the bastion FQDN |
| | `access.user` / `pass` | SSH login user and password |
| 3 Contract | `signing_passphrase` | Protects the RSA-4096 signing key; used to produce the `envWorkloadSignature` that binds the workload and env contract sections |
| | `workload_secret` | Payload sealed by `ibm-ccco-sealed-secret-creator-client` and deployed as the `spoiler` Secret; unsealed at runtime by the Confidential Data Hub (CDH) in the pod |
| 4 Log router | `public_ingress_endpoint`, `log_apikey` | Log router URL and ingestion API key, written into the encrypted `env` section of the contract |
| 5 NFD | `nfd_source` | OLM catalog source for the NFD Subscription. `redhat-operators` for connected clusters, or your mirror catalog name for disconnected ones |
| 6 OSC | `osc_source` | OLM catalog source for the OSC Subscription (same rule as `nfd_source`) |
| | `osc_version` | Exact CSV version to install (Manual InstallPlan approval). **Required** — the playbook fails if blank |
| 7 KataConfig | `addon_image` | Full reference of the image you pushed in [step 1](#1-push-the-ibm-ccco-image-to-a-registry) |
| | `kernel_path` | Path to the IBM SE kernel image inside the container image |
| | `verify_addon_kernel_service` | `true` checks `kata-addon-kernel.service` on each worker. Set `false` if the registry is not reachable from worker nodes |
| 8 Cluster auth | `auth_dir_path` | Folder on the bastion containing the `kubeconfig` file, **without** the filename. `/kubeconfig` is appended automatically |
| *(not in `all.yaml`)* | `workload_image` | **Not user-configurable.** Fixed at the role default `ghcr.io/ibm-hyper-protect/paynow-website:s390x-v1.0.3` (`roles/ccco_contract_generator/defaults/main.yaml`). It is the only image the generated REGO policy allows, and it is also hardcoded in the PayNow pod manifest template |

`inventories/default/hosts` is generated from `all.yaml` on every run — do not edit it. Both files are git-ignored.


## 5. Set up the cluster environment

Every playbook starts with a play on `localhost` that runs `ccco_setinventory` to regenerate the hosts file, then runs its main play on the `bastion`.

### Run everything in one go

```bash
ansible-playbook playbooks/ccco_baremetal.yaml        # add -vv for verbose output
```

This installs the cluster stack (`ccco_install_cluster.yaml`), then builds the PayNow and Busybox workloads (`ccco_paynow_workload.yaml`, `ccco_busybox_deploy.yaml`). To install the cluster only, run `ccco_install_cluster.yaml`.

### Install playbooks

Install order is **SE → NFD → OSC → Kata**. `ccco_install_cluster.yaml` runs all four; run the individual playbooks to install or re-run one step.

| Playbook | What it does |
|---|---|
| `ccco_baremetal.yaml` | Full end-to-end: cluster install, then PayNow, then Busybox |
| `ccco_install_cluster.yaml` | One-time cluster setup: SE → NFD → OSC → Kata |
| `ccco_enable_se.yaml` | Applies a `99-<role>-enable-secure-execution` MachineConfig with `prot_virt=1` to masters (and worker-only nodes if any), waits for the MachineConfigPool rollout (nodes reboot), and reports the SE label per node |
| `ccco_install_nfd.yaml` | Installs the NFD operator, creates the `NodeFeatureDiscovery` instance and the IBM `ibm-se-rule` `NodeFeatureRule`, then waits for the `ibm.feature.node.kubernetes.io/se=true` label on all SE-capable nodes (Kata requires it) |
| `ccco_install_osc.yaml` | Asserts `osc_version` is set, installs the OSC operator pinned to that version (Manual approval: finds and approves the InstallPlan), and waits for the CSV to reach `Succeeded` |
| `ccco_install_kata.yaml` | Applies the `osc-feature-gates` and `kata-addon-artifacts` ConfigMaps and the `cluster-kataconfig` `KataConfig`, waits for install to finish, and checks that the `kata-cc` RuntimeClass uses handler `kata-se`. Optionally checks `kata-addon-kernel.service` on each worker |


## 6. Deploy the workloads

Run these after the cluster is set up:

| Playbook | What it does |
|---|---|
| `ccco_paynow_workload.yaml` | Copies the artifacts to `/root/ccco-artifacts` on the bastion, generates the keys and a restricted REGO policy (limited to `workload_image`), creates the `spoiler` and `sealed-paynow` secrets, builds and signs the encrypted contract, and writes `paynow-baremetal.yaml` (pod `paynow-ssl` + NodePort service). Re-run when the image, policy or secrets change |
| `ccco_busybox_deploy.yaml` | Same flow with a permissive REGO policy (any image containing `busybox`); writes `busybox-baremetal.yaml` (pod `busybox-cc`) |

> **The playbooks generate manifests; they do not apply them.** Deploy from the bastion:
>
> ```bash
> export KUBECONFIG=<cluster.auth_dir_path>/kubeconfig
> oc apply -f /root/ccco-artifacts/paynow-baremetal.yaml     # PayNow pod + NodePort service
> oc apply -f /root/ccco-artifacts/busybox-baremetal.yaml    # Busybox pod
> ```

> **Only one manifest survives at a time.** Each workload run deletes every previously generated file in `/root/ccco-artifacts` (the copied artifacts are kept), regenerates all keys, and re-creates the `spoiler` and `sealed-paynow` secrets. Running Busybox after PayNow removes `paynow-baremetal.yaml` and invalidates its contract. Apply each manifest right after its playbook finishes, or re-run the PayNow playbook when you need it again.


## Clean up the environment

```bash
ansible-playbook playbooks/ccco_cleanup.yaml
```

`ccco_cleanup.yaml` is the reverse of `ccco_baremetal.yaml` and removes, in order: **workload → Kata → OSC → NFD → SE**. Each cleanup role first checks whether its component exists, so every playbook is safe to re-run or to run when a step was never installed.

> **Warning:** removing SE and Kata deletes MachineConfigs / `KataConfig` and triggers MachineConfig Operator node reboots. Run cleanup only when the cluster is idle.

| Playbook | What it removes |
|---|---|
| `ccco_cleanup.yaml` | Everything below, in order. Workloads go first because Kata depends on them, and SE goes last because it needs the node rollout to finish |
| *(workload step of `ccco_cleanup.yaml`)* | Pods `paynow-ssl` and `busybox-cc`, service `paynow-ssl`, secrets `spoiler` and `sealed-paynow`, then deletes the whole `/root/ccco-artifacts` directory on the bastion. There is no standalone playbook for this step |
| `ccco_clean_kata.yaml` | Deletes the `cluster-kataconfig` `KataConfig`, waits (up to 60 min) for it to be gone and for the MachineConfigPools to settle, and verifies the `kata-cc` RuntimeClass is removed |
| `ccco_clean_osc.yaml` | Deletes the OSC Subscription, CSV, OperatorGroup and the `openshift-sandboxed-containers-operator` namespace |
| `ccco_clean_nfd.yaml` | Deletes the `ibm-se-rule` `NodeFeatureRule`, the `NodeFeatureDiscovery` CR (clearing its finalizer), the NFD Subscription, CSV, OperatorGroup and the `openshift-nfd` namespace, then strips all `feature.node.kubernetes.io/*` and `ibm.feature.node.kubernetes.io/*` labels from every node |
| `ccco_clean_se.yaml` | Deletes the `99-master-enable-secure-execution` and `99-worker-enable-secure-execution` MachineConfigs (each only if present) and waits for the MachineConfigPool rollout |


## Reference

### Cluster topology

Node roles are detected automatically. A *worker-only* node carries the worker label but neither the master nor the control-plane label.

- **Compact / SNO** (all nodes are master + worker): worker-only count is 0, so the worker MachineConfig and worker MCP waits are skipped and the NFD SE-label check covers all nodes.
- **HA** (dedicated workers): the SE MachineConfig is applied to both pools, `KataConfig` is scoped to workers through `kataConfigPoolSelector`, and the NFD SE-label check covers worker-only nodes.

Tested OSC version: `v1.13.1` (`kataconfiguration.openshift.io/v1`).

### Directory structure

```
ccco-provisioner/
├── ansible.cfg
├── requirements.yml                        # Ansible collection dependencies
├── artifacts/                              # Required binaries and certs (git-ignored, see artifacts/README.txt)
├── inventories/
│   └── default/
│       ├── hosts                           # Auto-generated from all.yaml (git-ignored)
│       └── group_vars/
│           ├── all.yaml.template           # Copy to all.yaml and fill values
│           └── all.yaml                    # Your values (git-ignored)
├── playbooks/
│   ├── ccco_baremetal.yaml                 # Full end-to-end (cluster + PayNow + Busybox)
│   ├── ccco_install_cluster.yaml           # SE + NFD + OSC + Kata
│   ├── ccco_enable_se.yaml                 # Standalone install steps
│   ├── ccco_install_nfd.yaml
│   ├── ccco_install_osc.yaml
│   ├── ccco_install_kata.yaml
│   ├── ccco_paynow_workload.yaml           # PayNow contract + manifest
│   ├── ccco_busybox_deploy.yaml            # Busybox contract + manifest
│   ├── ccco_cleanup.yaml                   # Full teardown (workload → Kata → OSC → NFD → SE)
│   ├── ccco_clean_kata.yaml                # Standalone teardown steps
│   ├── ccco_clean_osc.yaml
│   ├── ccco_clean_nfd.yaml
│   └── ccco_clean_se.yaml
└── roles/
    ├── ccco_setinventory/                  # Generates the hosts file from all.yaml (adds ProxyJump if a jump host is set)
    ├── ccco_enable_se/                     # prot_virt=1 MachineConfig + MCP rollout wait
    ├── ccco_install_nfd/                   # NFD operator, NodeFeatureDiscovery, IBM SE NodeFeatureRule
    ├── ccco_install_osc/                   # OSC operator (Manual InstallPlan approval)
    ├── ccco_install_kata/                  # Feature gates, addon ConfigMap, KataConfig, RuntimeClass checks
    ├── ccco_contract_generator/            # Discovers artifacts, copies them to the bastion, generates REGO policy + keys
    ├── ccco_sealed_secrets/                # Creates the spoiler and sealed-paynow secrets
    ├── ccco_paynow_deploy/                 # Builds the encrypted contract, generates paynow-baremetal.yaml
    ├── ccco_busybox_deploy/                # Builds the encrypted contract, generates busybox-baremetal.yaml
    ├── ccco_remove_workload/               # Removes workload pods/service/secrets and the scripts directory
    ├── ccco_remove_kata/                   # Removes KataConfig and Kata runtime
    ├── ccco_remove_osc/                    # Removes the OSC operator
    ├── ccco_remove_nfd/                    # Removes the NFD operator and node labels
    └── ccco_remove_se/                     # Removes SE MachineConfigs
```


## License

This project is licensed under the Apache License 2.0
