# ccco-provisioner

Ansible automation to generate a CCCO contract and deploy the **PayNow** workload on a bare metal OpenShift cluster using IBM Confidential Computing Containers for Red Hat OpenShift Container Platform (CCCO).


## Prerequisites

- Ansible `2.12+` installed on the controller (your local machine)
- SSH access configured to the bastion host (`~/.ssh/ansible-ocpz`)
- `kubernetes.core` Ansible collection (installed via `requirements.yml`)

### Required artifacts

Place the following files in the `artifacts/` directory before running:

| File pattern | Description |
|---|---|
| `HKD*.crt` | Host Key Document certificate |
| `*.crt` (non-HKD) | IBM CCCO contract encryption certificate |
| `*.bin` | SE header binary |
| `*client*` | CCCO sealed-secret-creator-client binary (s390x) |

> Files are auto-discovered by pattern — no need to specify filenames in config.


## SSH Setup

Configure passwordless SSH access from your local machine to the bastion before running the playbook.

```bash
# 1. Generate SSH key pair (skip if ~/.ssh/ansible-ocpz already exists)
ssh-keygen -t rsa -b 4096 -f ~/.ssh/ansible-ocpz -N ""

# 2. Copy public key to bastion
ssh-copy-id -i ~/.ssh/ansible-ocpz.pub root@<bastion-ip>

# 3. Verify connection
ssh -i ~/.ssh/ansible-ocpz root@<bastion-ip>
```

> The key name `ansible-ocpz` matches the `private_key_file` in `ansible.cfg`. If you use a different key name, update `ansible.cfg` accordingly.


## Setup

```bash
# 1. Install required Ansible collections
ansible-galaxy collection install -r requirements.yml

# 2. Copy and fill in inventory variables
cp inventories/default/group_vars/all.yaml.template inventories/default/group_vars/all.yaml
vi inventories/default/group_vars/all.yaml

# 3. Place required artifacts
cp /path/to/your/artifacts/* artifacts/
```


## Run

```bash
ansible-playbook playbooks/ccco_baremetal.yaml
```

This will:
1. Auto-generate `inventories/default/hosts` from `all.yaml` (no manual hosts file needed)
2. Copy baremetal artifacts to `/root/ccco-artifacts` on the bastion
3. Generate CCCO contract keys and REGO attestation policy
4. Create and apply the sealed secret (`spoiler`) to the `default` namespace
5. Build the encrypted contract (workload + env sections)
6. Generate `paynow-baremetal.yaml` — the PayNow Pod/Service/Route manifest


## Directory structure

```
ccco-provisioner/
├── ansible.cfg
├── requirements.yml                        # Ansible collection dependencies
├── artifacts/                              # Place required binaries and certs here (git-ignored)
├── inventories/
│   └── default/
│       └── group_vars/
│           └── all.yaml.template           # Copy to all.yaml and fill values
├── playbooks/
│   └── ccco_baremetal.yaml                 # Single entry-point playbook
└── roles/
    ├── ccco_setinventory/                  # Auto-generates hosts file from all.yaml
    ├── ccco_contract_generator/            # Discovers artifacts, generates REGO policy + keys
    ├── ccco_sealed_secrets/                # Creates and applies sealed secret to cluster
    └── ccco_paynow_deploy/                 # Builds encrypted contract, generates paynow-baremetal.yaml
```


## License

This project is licensed under the Apache License 2.0
