# Actions Native Egress Firewall — Early Access

> [!IMPORTANT]
> **Status: Technical Preview.** The native egress firewall is currently in technical preview. Audit mode and Layer 7 visibility are available for design partners on Linux.

> [!NOTE]
> **Performance during preview.** With the firewall enabled and TLS inspection in the path, expect roughly a **15–20% increase in workflow runtime** for typical workloads, driven by the extra TLS handshake at the egress boundary and a small fixed sidecar cold-start cost. Reducing this overhead (connection reuse, selective inspection, sidecar warmup) is an active investment, and we will publish per-release performance deltas alongside public preview.

Welcome to the early access program for the **GitHub Actions native egress firewall**. This program gives design partners hands on access to a Layer 7 egress firewall for GitHub hosted runners, with both audit and enforcement modes.

## What is the native egress firewall?

GitHub hosted runners today allow unrestricted outbound network access. Any workflow can reach any host on the internet, regardless of `GITHUB_TOKEN` permissions, secret scoping, OIDC, or SHA pinning. Those controls govern *identity*, *code*, and *what the workflow can do inside GitHub* — but nothing today governs *where the workflow can talk on the network*.

The native egress firewall closes that gap. It runs **outside** the runner VM at Layer 7, inspects DNS and HTTP/HTTPS traffic, and remains immutable even if a workflow gains root access inside the runner. It complements — and does not replace — OIDC, SHA pinning, and `GITHUB_TOKEN` permissions; together they produce a workflow that is identified, deterministic, least-privileged, and network-bounded.

The capability ships in two modes:

- **Audit mode** records every outbound DNS lookup and HTTP request without blocking anything. This is the safe entry point.
- **Enforcement mode** applies an allow list. Traffic outside the list is blocked, recorded, and surfaced in the workflow summary with the offending command and the rule that denied it.

### How HTTPS inspection works

To support URL-level allow rules, the firewall **terminates TLS at the egress boundary and re-establishes TLS to the destination**. Each workflow run gets a unique, ephemeral certificate that is destroyed when the run ends, so traffic remains private to that run.

- If your workflow trusts the operating system certificate store (the default for `curl`, `git`, `npm`, `pip`, `docker`, etc.), you will see a normal HTTPS connection — no changes required.
- If your workflow does **certificate pinning** or **mTLS**, you will need to update it to trust the per-run ephemeral certificate.

## Two delivery paths

| Adoption path | Runner type | How it is enabled | Best for |
|---|---|---|---|
| Firewall enabled label | Standard GitHub hosted runners | Set `runs-on: ubuntu-24.04-firewall` in the workflow | Individual repositories, open source projects, fast adoption with no admin setup |
| Firewall enabled image | Larger runners | Select the GitHub maintained `Ubuntu 24.04 with Egress Firewall` image when creating the larger runner, or build a custom image based on it | Enterprises that already use larger runners, custom tooling, private networking, or runner groups |

Both paths produce identical Layer 7 enforcement, identical telemetry, and identical rule semantics. Custom larger runner images built on the firewall base image cannot disable or bypass the firewall, because enforcement lives outside the VM.

## Phased rollout

| Phase | Capability | Scope |
|---|---|---|
| Technical preview | Audit mode, full Layer 7 visibility, correlation to workflow, job, step, and command | Linux, opt in via runner label or larger runner image |
| Public preview | Enforcement mode with allow list rules, YAML defined in the repository | Linux, repository level only |
| GA | Organization and enterprise rule definition via **Actions rulesets** (target by name, pattern, or custom repository property), **managed developer-intent rules**, log streaming via the **Actions data stream** | Linux, with Windows and macOS to follow |

### How rules compose

At GA, rules can be defined at the **enterprise**, **organization**, and **repository** levels. They compose into a single effective allow list at runtime:

- Organization and enterprise rules are **authoritative**. A repository cannot grant itself access the organization has not allowed.
- A repository **can further restrict** its own egress beyond what the organization allows.
- The effective policy and its source are recorded with every run for audit.

### Where logs go

- **Preview:** firewall events are surfaced in the workflow run summary and as a workflow run artifact.
- **GA:** events stream to the **Actions data stream** with workflow, job, step, and command attribution, ready for ingestion into existing SIEM and detection pipelines.

## Example rule file

> [!NOTE]
> **We want your feedback on this schema.** The rule formats below are illustrative and will evolve based on early access input. If a rule kind is missing, a field is awkward to express, or you need a higher-level managed rule, please open a [feature request](../../issues/new?template=feature-request.yml).

### Raw rules (preview)

```yaml
version: 0.0.1
mode: allow-list
rules:
  - kind: dns-rule
    description: any subdomain of github.com
    domain: github.com
    allow-any-subdomain: true

  - kind: http-rule
    description: pull source and read APIs in my org
    domains:
      - api.github.com
      - github.com
    scheme: [https]
    url-path-base: /my-org/
    methods: [GET]

  - kind: http-rule
    description: npm registry, read only
    domains:
      - registry.npmjs.org
    scheme: [https]
    methods: [GET]
```

### Managed developer-intent rules (GA)

Most customers should not need to write low-level DNS/HTTP rules. Managed rules express intent in terms developers and security teams already share, and expand to the underlying primitives automatically:

```yaml
version: 0.0.1
mode: allow-list
rules:
  - kind: github-access
    description: GitHub API and git access, only inside my org
    endpoints: [api, graphql, git-pull, git-push]
    orgs: [my-org]
    enforce-actions-identity: true

  - kind: ghcr
    description: pull from my org's container registry
    methods: [pull]
    orgs: [my-org]

  - kind: http-rule
    description: npm registry, read only
    domains:
      - registry.npmjs.org
    scheme: [https]
    methods: [GET]
```

> **Roadmap (aspirational):** a one-line managed rule to scope a workflow's GitHub access to its **own organization** — or even its **own repository** — via `enforce-actions-identity: true`. This is the primary motivation for the managed-rule layer at GA.

### Centralized policy via Actions rulesets (GA)

Administrators reference firewall rule files from an Actions ruleset and target repositories by name, pattern, or custom property — the same model used today for branch protection and push rules.

```yaml
# Ruleset configuration, illustrative
name: production-egress
target:
  repositories:
    properties:
      tier: gold
      data_classification: restricted
rules:
  - type: actions_firewall
    enforcement: active
    rule_files:
      - repo: my-org/security-policies
        path: firewall/base.yml
      - repo: my-org/security-policies
        path: firewall/release.yml
```

## How this relates to the package firewall

The egress firewall and the package firewall are **complementary** controls on opposite sides of the network connection:

| | What it controls | Where it runs | Decision based on |
|---|---|---|---|
| **Package firewall** | *Ingress* into your code: which package versions are allowed to be installed | Registry side | Package metadata, vulnerabilities, malware signals, version pinning |
| **Native egress firewall** | *Egress* from your runner: which hosts and URLs the workflow may contact | Runner side | Domain, URL, IP, HTTP method as defined in your ruleset |

Use both: the package firewall for *"what code is allowed in"*, the egress firewall for *"where that code is allowed to talk."*

## Getting started in early access

1. Open an [Onboarding request](../../issues/new?template=onboarding.yml) and provide your repository, organization, or enterprise details.
2. Once onboarded, select a firewall enabled runner. For standard runners, set `runs-on: ubuntu-24.04-firewall`. For larger runners, ask your administrator to create a Linux larger runner using the `Ubuntu 24.04 with Egress Firewall` image (or a custom image based on it).
3. Run your workflows in **audit mode**, review the recorded traffic against your expectations, then add a rule file at `.github/firewall.yml` to enforce.

## Reporting feedback

- [Onboarding request](../../issues/new?template=onboarding.yml) — request access for a repository, organization, or enterprise.
- [Issue report](../../issues/new?template=issue-report.yml) — report a bug, blocked request, false positive, or unexpected behavior.
- [Feature request](../../issues/new?template=feature-request.yml) — request a new rule kind, a managed rule, or a platform expansion.

## Confidentiality

This is a private early access repository. Do not share screenshots, rule files, runner configurations, or workflow output outside this program without explicit approval. Treat everything in this repository as confidential to the early access program.
