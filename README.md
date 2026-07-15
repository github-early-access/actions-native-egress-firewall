# Actions Native Egress Firewall — Early Access
> [!IMPORTANT]
> **Status: Technical Preview.** The native egress firewall is currently in technical preview for audit mode without rule enforcement. Linux is the only supported platform at this time.

Welcome to the early access program for the **GitHub Actions native egress firewall**. This program gives design partners hands on access to the firewall for GitHub-hosted runners, with both audit and enforcement (later date).

## Getting started in early access

The native egress firewall builds on GitHub Actions' existing runner image architecture, but operates within the context of a nested-VM. Separating the runner from the firewall. We provide both a standard firewall enabled runner image and a larger firewall enabled runner image, giving customers the same image options they use today with native egress controls included. For now, logs are available as an artifact at the conclusion of a workflow run.

Runner access in early access:

- **Standard GitHub-hosted runners** are available by default. Set `runs-on: ubuntu-24.04-firewall` in your workflow file.
- **Larger runners** are available by request during technical preview. Open a [larger runner access request](../../issues/new?template=larger-runner-access-request.yml) and include:
  - whether access is for an enterprise or organization
  - the exact enterprise or organization name
  - a point of contact

After approval, ask your administrator to create a Linux larger runner using the GitHub-maintained **`Ubuntu 24.04 with Firewall`** image.

> [!NOTE]
> **Performance during preview.** With the firewall enabled, expect roughly a **15–20% increase in workflow runtime** for typical workloads, driven by the virtual machine monitoring your network traffic. Reducing this overhead is an active investment, and we will publish per-release performance deltas alongside public preview.


## Reporting feedback

- [Issue report](../../issues/new?template=issue-report.yml) — report a bug, blocked request, false positive, or unexpected behavior.
- [Feature request](../../issues/new?template=feature-request.yml) — request a new rule kind, a managed rule, or a platform expansion.
- [Larger runner access request](../../issues/new?template=larger-runner-access-request.yml) — request early access to larger runners using the `Ubuntu 24.04 with Firewall` image.

## What is the native egress firewall?

GitHub hosted runners today allow unrestricted outbound network access. Any workflow can reach any host on the internet, regardless of `GITHUB_TOKEN` permissions, secret scoping, OIDC, or SHA pinning. Those controls govern *identity*, *code*, and *what the workflow can do inside GitHub* — but nothing today governs *where the workflow can talk on the network*.

The native egress firewall closes that gap. It runs **outside** the runner VM, inspects DNS and HTTP/HTTPS traffic, and remains immutable even if a workflow gains root access inside the runner. It complements — and does not replace — OIDC, SHA pinning, and `GITHUB_TOKEN` permissions; together they produce a workflow that is identified, deterministic, least-privileged, and network-bounded.

The capability ships in two modes:

- **Audit mode** records every outbound DNS lookup and HTTP request without blocking anything. This is the safe entry point.
- **Enforcement mode (Future)** applies an allow list. Traffic outside the list is blocked, recorded, and surfaced in the workflow summary with the offending command and the rule that denied it.

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
| Public preview | Enforcement mode with allow list rules. Deny all by default | Linux, repository level only |
| GA | Organization and enterprise rule definition via **Actions rulesets** (target by name, pattern, or custom repository property), **managed developer-intent rules**, log streaming via the **Actions data stream** | Linux, with Windows and macOS to follow |

### Where logs go

- **Preview:** firewall events are surfaced in the workflow run summary and as a workflow run artifact. We will log the binary name (without command line flags or environment variables) as well as the URL (without query arguments or URL fragment). Be careful that you don't accidentally log secrets in your command name or URL path!
- **GA:** events stream to the **Actions data stream** with workflow, job, step, and command attribution, ready for ingestion into existing SIEM and detection pipelines.
