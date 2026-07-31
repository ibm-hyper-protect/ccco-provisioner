# Maintainers

This document lists the maintainers of the `ccco-provisioner` project, their roles, and responsibilities.


## Current Maintainers

| Name | GitHub Handle | Role | Focus Areas |
|---|---|---|---|
| Pavithiran P | [@pavithiran34](https://github.com/pavithiran34) | Maintainer | Overall project direction, releases, core playbooks |
| Lokesh Puthalapattu | [@Lokesh-Puthalapattu](https://github.com/Lokesh-Puthalapattu) | Security Lead | Overall Security |
| Ajay Victor | [@ajaypvictor](https://github.com/ajaypvictor) | Maintainer | Code review, playbook contributions |
| Rohit Singh | [@Rohit-Singh43-1](https://github.com/Rohit-Singh43-1) | Maintainer | Code review, playbook contributions |
| Siddaram Sonnagi | [@siddu-os](https://github.com/siddu-os) | Maintainer | Code review, playbook contributions |


## Roles and Responsibilities

### Lead Maintainer

The Lead Maintainer is responsible for:

- **Project Direction** — Setting technical direction, making final decisions on major changes, coordinating with IBM Hyper Protect team
- **Release Management** — Managing release schedules, versioning, and publishing releases
- **Code Review** — Reviewing and approving pull requests, ensuring quality and consistency
- **Community Management** — Responding to issues and discussions, enforcing the Code of Conduct
- **Security** — Responding to security reports, coordinating fixes and disclosures

### All Maintainer Responsibilities

1. **Be Responsive** — Respond to issues and PRs in a timely manner; acknowledge security reports within 3 business days
2. **Maintain Quality** — Review code thoroughly, ensure `ansible-lint` passes, follow coding standards
3. **Support Contributors** — Provide constructive feedback, help contributors improve their PRs
4. **Uphold Standards** — Enforce the Code of Conduct, maintain professional communication
5. **Stay Informed** — Monitor security advisories, stay current with Ansible and IBM Hyper Protect ecosystem changes


## Decision Making

### Regular Decisions
*(features, bug fixes, documentation)*

1. Proposed via GitHub issue or pull request
2. Discussed in issue/PR comments
3. Maintainers review and provide feedback
4. Consensus preferred — Lead Maintainer makes final call if needed

### Major Decisions
*(breaking changes, architectural changes, new roles)*

1. Proposed via GitHub issue with detailed RFC (Request for Comments)
2. Community discussion period — minimum 1 week
3. Maintainer review and discussion
4. Final decision by consensus
5. Lead Maintainer makes final call if consensus cannot be reached


## Merging Pull Requests

Before merging a PR, verify:

- [ ] All CI checks pass
- [ ] At least one maintainer approval
- [ ] No unresolved review comments
- [ ] Commit messages follow Conventional Commits format with `Signed-off-by`
- [ ] Commits are GPG/SSH signed
- [ ] No hardcoded secrets — sensitive tasks have `no_log: true`
- [ ] New roles have `defaults/main.yaml`
- [ ] Manifests in `templates/*.j2` not inline in tasks
- [ ] `all.yaml.template` updated if new variables introduced
- [ ] `ansible-lint` passes with zero errors


## Becoming a Maintainer

We welcome community members who demonstrate consistent, high-quality contributions.

### Path to Maintainership

1. **Consistent Contributions** — Multiple merged PRs, high-quality code and documentation
2. **Community Engagement** — Active participation in discussions, helping other contributors
3. **Project Alignment** — Following the Code of Conduct, supporting the project's goals

### Nomination Process

1. An existing maintainer nominates a contributor
2. Nomination is discussed among current maintainers
3. Consensus is reached (all maintainers must agree)
4. Nominee is invited to become a maintainer
5. Upon acceptance, added to this document


## Emeritus Maintainers

Maintainers who have stepped down are recognised here for their contributions:

*(none yet)*


## Contact

### General Questions
- Open an issue on [GitHub](https://github.com/ibm-hyper-protect/ccco-provisioner/issues)
- Start a discussion in [GitHub Discussions](https://github.com/ibm-hyper-protect/ccco-provisioner/discussions)

### Security Issues
- Follow the [Security Policy](SECURITY.md)
- **Do not** open public issues for security vulnerabilities
- Report via [GitHub Security Advisories](https://github.com/ibm-hyper-protect/ccco-provisioner/security)

### Code of Conduct Issues
- Contact maintainers directly via GitHub
