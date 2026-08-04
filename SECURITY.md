<!--
Copyright (c) 2026 IBM Corp.
All rights reserved.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# Security Policy

## Supported Versions

| Version | Supported |
|---|---|
| latest (`main`) | Supported |
| older branches | Not Supported |

We recommend always using the latest version from `main` to ensure you have the latest security updates.


## Reporting a Vulnerability

The `ccco-provisioner` team takes security vulnerabilities seriously. We appreciate your efforts to responsibly disclose your findings.

### Please DO NOT

- Open a public GitHub issue for security vulnerabilities
- Disclose the vulnerability publicly before it has been addressed

### Instead, Please

**Report via GitHub Security Advisories:**

1. Go to the [Security tab](https://github.com/ibm-hyper-protect/ccco-provisioner/security) of this repository
2. Click **"Report a vulnerability"**
3. Fill out the form with details about the vulnerability

**Or contact the Security Lead directly:**

See [MAINTAINERS.md](MAINTAINERS.md) for the Security Lead contact.

### What to Include

- **Description** — A clear description of the vulnerability
- **Impact** — What an attacker could achieve by exploiting it
- **Reproduction Steps** — Detailed steps to reproduce
- **Affected files/roles** — Which playbook, role or task is affected
- **Proposed Fix** — If you have suggestions (optional)

---

## What to Expect

| Action | Timeframe |
|---|---|
| Acknowledgement of report | Within 3 business days |
| Initial assessment | Within 5 business days |
| Fix for high-severity issues | Within 30 days |
| Fix for medium/low-severity issues | Within 90 days |
| Public disclosure | After fix is released |

When a fix is released we will:
- Publish a security advisory
- Credit you for the discovery (unless you prefer to remain anonymous)


## Security Best Practices for Users

### Protecting Sensitive Files

- **Never commit `all.yaml`** — it contains passwords, API keys and cluster credentials
- **Never commit files from `artifacts/`** — they contain certificates and binaries
- **Use `.gitignore`** to exclude sensitive files (see `artifacts/README.txt`)
- **Use Ansible Vault** to encrypt `all.yaml` at rest:
  ```bash
  ansible-vault encrypt inventories/default/group_vars/all.yaml
  ansible-playbook playbooks/ccco_baremetal.yaml --ask-vault-pass
  ```

### Secret Management

- Rotate `contract.signing_passphrase` and `contract.workload_secret` after any suspected exposure
- Perform periodic secret rotation in accordance with your organisation's security policies — do not wait for a suspected exposure to rotate credentials
- Restrict `cluster.auth_dir_path` — ensure the kubeconfig is readable only by the bastion user
- Use environment variables or secure vaults for sensitive configuration — never hardcode values in playbooks

### Certificate and Artifact Security

- Obtain contract encryption certificates only from trusted IBM Hyper Protect sources
- Verify artifact integrity before use — always validate the checksum or signature of downloaded certificates and binaries against the values published by IBM to mitigate supply-chain risks
- Verify HKD certificates match your target bare metal node
- Store sealed secret client binary in a secure location — restrict execute permissions

### Dependency Security

- Keep `kubernetes.core` collection updated: `ansible-galaxy collection install -r requirements.yml --upgrade`
- Keep Python `kubernetes` library updated: `pip3 install --upgrade kubernetes`
- Monitor security advisories for Ansible and OpenSSL


## Security Best Practices for Contributors

1. **No secrets in code** — Never hardcode passwords, API keys or tokens in any task, template or var file
2. **`no_log: true`** — Add to every task that processes secrets, passphrases or tokens.
   Explicitly, the following must never be written to logs:
   - Passwords and passphrases
   - Private keys and certificates
   - kubeconfig files and bearer tokens
   - Sealed secret values and workload secrets
   Continue using `no_log: true` on all secret-handling tasks to prevent accidental exposure in Ansible output
3. **Code review** — All changes require maintainer review before merging — pay special attention to shell tasks that handle cryptographic operations
4. **Test data only** — Never use real credentials, certificates or cluster endpoints in examples or tests
5. **Dependency audits** — Regularly check collection and Python library dependencies for known vulnerabilities


## Known Security Considerations

### Cryptographic Operations

This playbook performs the following cryptographic operations on the bastion host:

| Operation | Tool | Purpose |
|---|---|---|
| RSA key generation | `openssl genrsa` | Contract signing and sealing keys |
| RSA encryption (PKCS#1) | `openssl pkeyutl` | Encrypting workload + env sections |
| AES-256-CBC encryption | `openssl enc` | Contract data encryption |
| SHA-256 signing | `openssl dgst` | Contract signature |
| gzip + base64 | shell | initdata annotation encoding |

These operations rely on:
- A standard, up-to-date OpenSSL installation on the bastion
- Secure key generation (keys are generated fresh each run)
- Trusted contract encryption certificates from IBM Hyper Protect

### Sealed Secrets

- The sealed secret client binary creates a cryptographically sealed workload secret
- The `spoiler` Kubernetes secret is applied to the OCP cluster — ensure RBAC limits who can read it
- Keys generated in `/root/ccco-artifacts/` should be cleaned up after use


## Vulnerability Disclosure Policy

We follow coordinated vulnerability disclosure:

1. **Report** — Security researchers report vulnerabilities privately
2. **Fix** — We develop and test a fix
3. **Release** — We release the patched version
4. **Disclose** — We publish a security advisory with credit to the researcher
5. **Public** — Full details disclosed after users have had time to update

We will not take legal action against security researchers who:
- Make a good faith effort to avoid privacy violations
- Report vulnerabilities privately and give us reasonable time to respond
- Do not exploit vulnerabilities beyond what is necessary to demonstrate the issue


## Security Updates

Stay informed about security updates:

- **Releases** — Check the [Releases page](https://github.com/ibm-hyper-protect/ccco-provisioner/releases) for security patches


## Questions?

1. Review this document thoroughly
2. Contact the maintainers listed in [MAINTAINERS.md](MAINTAINERS.md)


Thank you for helping keep `ccco-provisioner` and our users safe!
