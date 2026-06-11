# Actions Native Egress Firewall — Early Access

> [!NOTE]
> **Performance during preview.** With the firewall enabled, expect roughly a **15–20% increase in workflow runtime** for typical workloads, driven by the virtual machine monitoring your network traffic. Reducing this overhead is an active investment, and we will publish per-release performance deltas alongside public preview.

Welcome to the early access program for the **GitHub Actions native egress firewall**. This program gives design partners hands on access to a Layer 7 egress firewall for GitHub hosted runners, with both audit and enforcement modes.

## Getting started in early access

> [!IMPORTANT]
> **Status: Technical Preview.** The native egress firewall is currently in technical preview for audit mode without rule enforcement. Linux is the only supported platform at this time.

1. Open an [Onboarding request](../../issues/new?template=onboarding.yml) and provide your repository, organization, or enterprise details.
2. Once onboarded, select a firewall enabled runner. For standard runners, all you have to do is set `runs-on: ubuntu-24.04-firewall`. For larger runners, ask your administrator to create a Linux larger runner using the `Ubuntu 24.04 with Firewall` image.

## Reporting feedback

- [Issue report](../../issues/new?template=issue-report.yml) — report a bug, blocked request, false positive, or unexpected behavior.
- [Feature request](../../issues/new?template=feature-request.yml) — request a new rule kind, a managed rule, or a platform expansion.

## What is the native egress firewall?

GitHub hosted runners today allow unrestricted outbound network access. Any workflow can reach any host on the internet, regardless of `GITHUB_TOKEN` permissions, secret scoping, OIDC, or SHA pinning. Those controls govern *identity*, *code*, and *what the workflow can do inside GitHub* — but nothing today governs *where the workflow can talk on the network*.

The native egress firewall closes that gap. It runs **outside** the runner VM, inspects DNS and HTTP/HTTPS traffic, and remains immutable even if a workflow gains root access inside the runner. It complements — and does not replace — OIDC, SHA pinning, and `GITHUB_TOKEN` permissions; together they produce a workflow that is identified, deterministic, least-privileged, and network-bounded.

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
| Firewall enabled image | Larger runners | Select the GitHub maintained `Ubuntu 24.04 with Firewall` image when creating the larger runner | Enterprises that already use larger runners, custom tooling, private networking, or runner groups |

Both paths produce identical Layer 7 enforcement, identical telemetry, and identical rule semantics. Custom larger runner images built on the firewall base image cannot disable or bypass the firewall, because enforcement lives outside the VM.

## Phased rollout

| Phase | Capability | Scope |
|---|---|---|
| Technical preview | Audit mode | Linux, opt in via runner label or larger runner image |
| Public preview | Enforcement mode with allow list rules, YAML defined in the repository | Linux, repository level only |
| GA | Organization and enterprise rule definition via **Actions rulesets** (target by name, pattern, or custom repository property), **managed developer-intent rules**, log streaming via the **Actions data stream** | Linux, with Windows and macOS to follow |

### How rules compose

At GA, rules can be defined at the **enterprise**, **organization**, and **repository** levels. They compose into a single effective allow list at runtime:

- Organization and enterprise rules are **authoritative**. A repository cannot grant itself access the organization has not allowed.
- A repository **can further restrict** its own egress beyond what the organization allows.
- The effective policy and its source are recorded with every run for audit.

### Where logs go

- **Preview:** firewall events are surfaced in the workflow run summary and as a workflow run artifact. We will log the binary name (without command line flags or environment variables) as well as the URL (without query arguments or URL fragment). Be careful that you don't accidentally log secrets in your command name or URL path!
- **GA:** events stream to the **Actions data stream** with workflow, job, step, and command attribution, ready for ingestion into existing SIEM and detection pipelines.

## Example rule file

> [!NOTE]
> **We want your feedback on this schema.** The rule formats below are illustrative and will evolve based on early access input. If a rule kind is missing, a field is awkward to express, or you need a higher-level managed rule, please open a [feature request](../../issues/new?template=feature-request.yml).

TBD

## Confidentiality

This is a private early access repository. Do not share screenshots, rule files, runner configurations, or workflow output outside this program without explicit approval. Treat everything in this repository as confidential to the early access program.
