# Actions Native Egress Firewall — Early Access
> [!IMPORTANT]
> **Status: Technical Preview.** The native egress firewall is currently in technical preview. Audit mode and Layer 7 visibility are available for design partners on Linux. 

Welcome to the early access program for the **GitHub Actions native egress firewall**. This program gives design partners hands on access to a Layer 7 egress firewall for GitHub hosted runners, with both audit and enforcement modes, ahead of public preview.

## What is the native egress firewall?

GitHub hosted runners today allow unrestricted outbound network access. Any workflow can reach any host on the internet, regardless of `GITHUB_TOKEN` permissions, secret scoping, OIDC, or SHA pinning. This is the largest unaddressed gap in the Actions security model and the precondition behind a growing class of supply chain incidents, including secret exfiltration, dependency confusion, and release pipeline compromise via cache poisoning.

The native egress firewall closes that gap. It runs **outside** the runner VM at Layer 7, inspects DNS and HTTP or HTTPS traffic, and remains immutable even if a workflow gains root access inside the runner. Customers define explicit allow lists for domains, IP ranges, HTTP methods, and TLS requirements, and every outbound request is correlated to the originating workflow, job, step, and command.

The capability ships in two modes:

- **Audit mode** records every outbound DNS lookup and HTTP request without blocking anything. This is the safe entry point.
- **Enforcement mode** applies an allow list. Traffic outside the list is blocked, recorded, and surfaced in the workflow summary with the offending command and the rule that denied it.

## Two delivery paths

| Adoption path | Runner type | How it is enabled | Best for |
|---|---|---|---|
| Firewall enabled label | Standard GitHub hosted runners | Set `runs-on: ubuntu-24.04-firewall` in the workflow | Individual repositories, open source projects, fast adoption with no admin setup |
| Firewall enabled image | Larger runners | Select the GitHub maintained `Ubuntu 24.04 with Egress Firewall` image when creating the larger runner, or build a custom image based on it | Enterprises that already use larger runners, custom tooling, private networking, or runner groups |

Both paths produce identical Layer 7 enforcement, identical telemetry, and identical rule semantics. Custom larger runner images built on the firewall base image cannot disable or bypass the firewall, since enforcement lives outside the VM.

## Phased rollout

| Phase | Capability | Scope |
|---|---|---|
| Technical preview | Audit mode, full Layer 7 visibility, correlation to workflow, job, step, and command | Linux, opt in via runner label or larger runner image |
| Public preview | Enforcement mode with allow list rules, YAML defined in the repository | Linux, repository level only |
| GA | Organization and enterprise rule definition via Actions rulesets, managed developer intent rules, log streaming via the Actions data stream | Linux, with Windows and macOS to follow |

## Example rule file

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
    domain:
      domain: github.com
      allow-any-subdomain: true
    scheme: [https]
    url-path-base: /my-org/
    methods: [GET]

  - kind: http-rule
    description: npm registry, read only
    domain:
      domain: registry.npmjs.org
    scheme: [https]
    methods: [GET]
```

## Getting started in early access

1. Open an [Onboarding request](../../issues/new?template=onboarding.yml) and provide your repository, organization, or enterprise details.
2. Once onboarded, select a firewall enabled runner. For standard runners, set `runs-on: ubuntu-24.04-firewall`. For larger runners, ask your administrator to create a Linux larger runner using the `Ubuntu 24.04 with Egress Firewall` image.
3. Run your workflows in audit mode, review the recorded traffic, then add a rule file at `.github/firewall.yml` to enforce.

## Reporting feedback

- [Onboarding request](../../issues/new?template=onboarding.yml) — request access for a repository, organization, or enterprise.
- [Issue report](../../issues/new?template=issue-report.yml) — report a bug, blocked request, false positive, or unexpected behavior.
- [Feature request](../../issues/new?template=feature-request.yml) — request a new rule kind, a managed rule, or a platform expansion.

## Confidentiality

This is a private early access repository. Do not share screenshots, rule files, runner configurations, or workflow output outside this program without explicit approval. Treat everything in this repository as GitHub confidential.# actions-native-egress-firewall
