# Contributing to ccco-provisioner

Thank you for your interest in contributing to `ccco-provisioner`!
This guide covers everything you need to know to contribute effectively.


## Table of Contents

- [Branch Strategy](#branch-strategy)
- [Raising a Pull Request](#raising-a-pull-request)
- [Commit Message Format](#commit-message-format)
- [Commit Signing](#commit-signing)
- [Coding Standards](#coding-standards)
- [Review Process](#review-process)
- [Reporting Bugs and Feature Requests](#reporting-bugs-and-feature-requests)
- [License](#license)


## Branch Strategy

```
main                  → stable, protected — no direct commits
feature/<short-name>  → new features       e.g. feature/add-cleanup-role
fix/<short-name>      → bug fixes          e.g. fix/inventory-whitespace
chore/<short-name>    → non-code changes   e.g. chore/update-readme
```

- All changes go through a **Pull Request** — never commit directly to `main`
- Feature branches are created from `main` and merged back via PR
- Releases are tagged on `main` using semantic versioning: `v1.0.0`, `v1.1.0`


## Raising a Pull Request

> **Open an issue first** before submitting a PR — describe what bug you are fixing
> or what feature you are adding. PRs without a linked issue may be closed.

1. Fork the repository and create your branch from `main`
2. Open an issue and get maintainer feedback before starting work
3. Make your changes following the [Coding Standards](#coding-standards)
4. Commit with the correct [message format](#commit-message-format) and [sign your commits](#commit-signing)
5. Open a PR against `main` using the template below
6. Ensure all checklist items are complete before requesting review

### PR Template

```
## Summary
<!-- One paragraph describing what this PR does and why -->

## Changes
<!-- Bullet list of what was added/changed/removed -->

## Testing
<!-- How was this tested? Which cluster/environment? -->

## Checklist
- [ ] Commit messages follow the conventional format with Signed-off-by
- [ ] No hardcoded secrets or passwords in any task
- [ ] Sensitive tasks have `no_log: true`
- [ ] New roles have `defaults/main.yaml`
- [ ] Manifests are in `templates/` not inline in tasks
- [ ] No `oc` shell commands — use `kubernetes.core.k8s` module
- [ ] `ansible-lint` passes with no errors
- [ ] `all.yaml.template` updated if new variables are introduced
- [ ] README updated if behaviour changes
```


## Commit Message Format

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>: <short description>

- <bullet point detail>
- <bullet point detail>

Signed-off-by: Your Name <your@email.com>
```

### Types

| Type | When to use |
|---|---|
| `feat` | New feature or role |
| `fix` | Bug fix |
| `refactor` | Code restructure with no behaviour change |
| `chore` | Config, docs, dependency updates |
| `docs` | README or documentation only |
| `test` | Adding or fixing tests |

### Examples

```
feat: Add ccco_cleanup role to remove PayNow workload

- Add tasks to delete Pod, Service, Route and sealed secret
- Add defaults/main.yaml with namespace and scripts_dir

Signed-off-by: Your Name <your@email.com>
```

```
fix: Correct inventory path whitespace trimming in ccco_setinventory

- Replace grep inventory= with anchored grep ^inventory=
- Add tr -d [:space:] to strip whitespace from cut output

Signed-off-by: Your Name <your@email.com>
```

> **Sign-off is required** on every commit. Use `git commit -s` to add it automatically.

---

## Coding Standards

### Ansible Best Practices

- Use **fully qualified collection names** (FQCN) for all modules:
  ```yaml
  # correct
  ansible.builtin.shell:
  kubernetes.core.k8s:

  # avoid
  shell:
  k8s:
  ```

- Use `kubernetes.core.k8s` — **never** shell out to `oc` or `kubectl`:
  ```yaml
  # correct
  kubernetes.core.k8s:
    kubeconfig: "{{ cluster.auth_dir_path }}/kubeconfig"
    state: present

  # avoid
  ansible.builtin.shell:
    cmd: oc apply -f manifest.yaml
  ```

- Add `no_log: true` to any task that handles secrets, passphrases or tokens

### Role Structure

Every role must follow this structure:

```
roles/<role_name>/
├── defaults/
│   └── main.yaml       # Default variable values (required)
├── tasks/
│   └── main.yaml       # Task definitions
└── templates/          # Jinja2 templates for manifests (if needed)
    └── *.j2
```

### Variable Naming

| Variable type | Convention | Example |
|---|---|---|
| Inventory vars | `env.<section>.<key>` | `env.bastion.networking.ip` |
| Cluster auth | `cluster.auth_dir_path` | `cluster.auth_dir_path` |
| Role defaults | `snake_case` | `scripts_dir`, `namespace` |
| Discovered facts | `artifact_<name>` | `artifact_contract_key` |

### Manifest Files

- All Kubernetes/OpenShift manifests go in `templates/*.j2` — **not inline** in tasks
- Use `ansible.builtin.template` to render and copy them to the target host

### Lint Rules

Run `ansible-lint` before submitting a PR:

```bash
pip3 install ansible-lint
ansible-lint roles/ playbooks/
```

Expected: zero errors, zero warnings.


## Commit Signing

**All commits must be signed** with GPG or SSH. This proves authenticity and is required by CI.

### GPG Signing (Recommended)

```bash
# 1. Generate GPG key
gpg --full-generate-key
# Choose: RSA 4096, 1 year expiry, use your GitHub email

# 2. Get key ID
gpg --list-secret-keys --keyid-format=long
# Key ID is after rsa4096/ e.g. 3AA5C34371567BD2

# 3. Export public key and add to GitHub → Settings → GPG keys
gpg --armor --export YOUR_KEY_ID

# 4. Configure git for this repo
git config --local commit.gpgsign true
git config --local user.signingkey YOUR_KEY_ID
git config --local user.email "your-github-email@example.com"

# 5. Fix GPG TTY issue (macOS/Linux)
echo 'export GPG_TTY=$(tty)' >> ~/.zshrc && source ~/.zshrc
```

### SSH Signing (Git 2.34+)

```bash
# 1. Add SSH key to GitHub → Settings → SSH keys → type: Signing Key

# 2. Configure git for this repo
git config --local gpg.format ssh
git config --local user.signingkey ~/.ssh/id_ed25519.pub
git config --local commit.gpgsign true
```

### Verify a commit is signed

```bash
git log --show-signature -1
# Should show: gpg: Good signature from "Your Name <email>"
```

### Sign a previous commit

```bash
git commit --amend --no-edit -S
git push --force-with-lease
```


## Review Process

- Every PR requires **at least 1 approving review** before merge
- Tag maintainers listed in [MAINTAINERS.md](MAINTAINERS.md) as reviewers
- The reviewer checks:
  - No hardcoded secrets
  - `no_log: true` on sensitive tasks
  - FQCN used for all modules
  - Role has `defaults/main.yaml`
  - Manifests in `templates/` not inline
  - Commit messages follow the format with sign-off
  - Commits are GPG/SSH signed
- Maintainers merge using **squash merge** to keep `main` history clean


## Reporting Bugs and Feature Requests

Use [GitHub Issues](https://github.com/ibm-hyper-protect/ccco-provisioner/issues).

### Bug Report

Include:
- Ansible version (`ansible --version`)
- Python version (`python3 --version`)
- OCP cluster version (`oc version`)
- The full error message and task name
- Steps to reproduce

### Feature Request

Include:
- What problem it solves
- Which role(s) would be affected
- Whether it changes existing behaviour


## Questions

Open a [GitHub Discussion](https://github.com/ibm-hyper-protect/ccco-provisioner/discussions)
or raise an issue with the label `question`.


## License

By contributing to `ccco-provisioner`, you agree that your contributions will be
licensed under the [Apache License 2.0](LICENSE).
