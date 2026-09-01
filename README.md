# Actions Native Egress Firewall — Early Access
> [!IMPORTANT]
> **Status: Technical Preview.** The native egress firewall supports audit mode and file-based enforcement in technical preview. Linux is the only supported platform at this time.

Welcome to the early access program for the **GitHub Actions native egress firewall**. This program gives design partners hands on access to audit and file-based enforcement for GitHub-hosted runners.

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


## What is the native egress firewall?

GitHub hosted runners today allow unrestricted outbound network access. Any workflow can reach any host on the internet, regardless of `GITHUB_TOKEN` permissions, secret scoping, OIDC, or SHA pinning. Those controls govern *identity*, *code*, and *what the workflow can do inside GitHub* — but nothing today governs *where the workflow can talk on the network*.

The native egress firewall closes that gap. It runs **outside** the runner VM, inspects DNS and HTTP/HTTPS traffic, and remains immutable even if a workflow gains root access inside the runner. It complements — and does not replace — OIDC, SHA pinning, and `GITHUB_TOKEN` permissions; together they produce a workflow that is identified, deterministic, least-privileged, and network-bounded.

The capability ships in two modes:

- **Audit mode** records every outbound DNS lookup and HTTP request without blocking anything. This is the safe entry point.
- **Enforcement mode** applies an allow list. Traffic outside the list is blocked by default, recorded, and surfaced in the workflow summary with the offending command and the rule that denied it.

> [!WARNING]
> **Audit mode can still affect traffic.** Because the firewall terminates and re-establishes TLS at the egress boundary (see [How HTTPS inspection works](#how-https-inspection-works)), the proxy sits inline with your requests even in audit mode. It can occasionally reject a request the destination never saw, for example returning a proxy-generated `4xx` (such as an empty `HTTP 400`) on certain HTTPS `POST` requests.
>
> Treat "audit mode records traffic without blocking anything" as a goal, not a guarantee: audit mode *can* break a workflow. This most often affects Node-based actions calling cloud endpoints, such as [`aws-actions/configure-aws-credentials`](https://github.com/aws-actions/configure-aws-credentials) against AWS STS, where the same request succeeds from `curl` or `botocore` but fails from Node. These responses are not currently attributed to the firewall in the job log. Tracked in [#14](https://github.com/github-early-access/actions-native-egress-firewall/issues/14).

### How HTTPS inspection works

To support URL-level allow rules, the firewall **terminates TLS at the egress boundary and re-establishes TLS to the destination**. Each workflow run gets a unique, ephemeral certificate that is destroyed when the run ends, so traffic remains private to that run.

- If your workflow trusts the operating system certificate store (the default for `curl`, `git`, `npm`, `pip`, `docker`, etc.), you will see a normal HTTPS connection — no changes required.
- If your workflow does **certificate pinning** or **mTLS**, you will need to update it to trust the per-run ephemeral certificate.

## File-based enforcement

File-based enforcement defines egress policy in a configuration file committed to your repository. The firewall evaluates that policy at the egress boundary. The file declares a mode and an allow list; in enforcement mode, traffic to hosts outside the allow list is denied by default, recorded, and surfaced back to you.

Keeping policy alongside a workflow makes it reviewable in pull requests, version controlled, diffable, and portable with the repository. Use file-based rules when those policy-as-code properties fit your workflow. They can be used instead of, or alongside, other available rule formats.

> [!IMPORTANT]
> **Technical preview.** File-based enforcement is in technical preview. The experience, rule format and schema (including `no-default-urls`), file discovery, and APIs may change based on customer feedback.

### Configuration example

The policy file is committed to the repository with the workflow it protects. The preview file path, filename, and discovery mechanism are subject to change; use the onboarding guidance supplied to preview participants to configure the firewall to discover your policy file.

By default, policies allow egress to:

- `github.com`
- `*blob.core.windows.net` (*)
- `codeload.github.com`
- `*actions.githubusercontent.com` (*)

The `(*)` entries are GitHub-managed defaults, not user-configurable wildcard patterns. The firewall expands them to matching URLs returned by the GitHub `/meta` endpoint, so the allowed set cannot escape GitHub-controlled endpoints.

The following example adds `api.github.com` and `release-assets.githubusercontent.com`; the other GitHub endpoints shown above are already covered by the default allow list.

```yaml
mode: enforce
allow:
  - api.github.com
  - release-assets.githubusercontent.com
```

- `mode` selects behavior: `enforce` denies hosts not matched by `allow`; `audit` records traffic without intentionally denying it.
- `allow` adds hosts to the default allow list.

Set `no-default-urls: true` to disable the default allow list. You must then explicitly list every endpoint your workflow needs, including GitHub endpoints such as `github.com` and `codeload.github.com`.

```yaml
mode: enforce
no-default-urls: true
allow:
  - github.com
  - api.github.com
  - codeload.github.com
```

The following workflow uses the firewall runner with the additive policy example. The first request is allowed; the PyPI request is a representative CI dependency lookup to a host not in the allow list, so it fails in enforcement mode.

```yaml
name: Test firewall policy

on:
  workflow_dispatch:

jobs:
  egress:
    runs-on: ubuntu-24.04-firewall
    steps:
      - name: Allowed GitHub API request
        run: curl --fail --silent --show-error https://api.github.com
      - name: Blocked PyPI dependency lookup
        run: curl --fail --silent --show-error https://pypi.org/simple/requests/
```

## Enforcement results and reporting

In enforcement mode, a request to a denied host fails from inside the workflow as a failed connection or request error in the affected step. The workflow run summary identifies the denied traffic, including the offending command or binary and the rule that denied it.

Preview firewall events are surfaced in the workflow run summary and as a workflow run artifact. Events include the binary name, without command-line flags or environment variables, and the URL without query arguments or URL fragments. Avoid placing secrets in command names or URL paths.

The following is an **illustrative** summary entry; field names and record shape are subject to change during preview:

```text
Denied egress: binary=curl host=pypi.org rule=allow-list
```

An **illustrative** artifact record might be:

```yaml
outcome: denied
binary: curl
host: pypi.org
rule: allow-list
```

## Two delivery paths

| Adoption path | Runner type | How it is enabled | Best for |
|---|---|---|---|
| Firewall enabled label | Standard GitHub hosted runners | Set `runs-on: ubuntu-24.04-firewall` in the workflow | Individual repositories, open source projects, fast adoption with no admin setup |
| Firewall enabled image | Larger runners | Select the GitHub maintained `Ubuntu 24.04 with Firewall` image when creating the larger runner | Enterprises that already use larger runners, custom tooling, private networking, or runner groups |

Both paths produce identical Layer 7 enforcement, identical telemetry, and identical rule semantics. Custom larger runner images built on the firewall base image cannot disable or bypass the firewall, because enforcement lives outside the VM.

## Phased rollout

| Phase | Capability | Scope |
|---|---|---|
| Technical preview | Audit mode and file-based enforcement with allow list rules. Deny all by default in enforcement mode | Linux, opt in via runner label or larger runner image |
| Public preview | Expanded policy and administration experiences, informed by preview feedback | Scope to be determined |
| GA | Further capability expansion, informed by customer feedback | Scope to be determined |

### Where logs go

- **Preview:** firewall events are surfaced in the workflow run summary and as a workflow run artifact. We will log the binary name (without command line flags or environment variables) as well as the URL (without query arguments or URL fragment). Be careful that you don't accidentally log secrets in your command name or URL path!
- **GA:** events stream to the **Actions data stream** with workflow, job, step, and command attribution, ready for ingestion into existing SIEM and detection pipelines.

## Feedback Requested

Technical preview participants: please tell us about your experience with:

- File-based rule authoring, including how policy changes fit into pull request review.
- The rule format and schema.
- The shape and usefulness of returned enforcement data.
- Logging, observability, and troubleshooting.
- Missing capabilities, edge cases, and scale requirements.

- [Issue report](../../issues/new?template=issue-report.yml) — report a bug, blocked request, false positive, or unexpected behavior.
- [Feature request](../../issues/new?template=feature-request.yml) — request a new rule kind, a managed rule, or a platform expansion.
- [Larger runner access request](../../issues/new?template=larger-runner-access-request.yml) — request early access to larger runners using the `Ubuntu 24.04 with Firewall` image.

## Future Direction and Scale Considerations

We recognize that many customers need to manage and enforce egress policies consistently across large organizations and enterprises. Managing policy across hundreds or thousands of repositories remains an important requirement for many customers.

Organization-level and enterprise-level management, governance, and policy distribution are not the focus of this technical preview. The immediate goal is to provide meaningful enforcement and validation so customers can begin protecting workflows and provide feedback on the policy model. Feedback gathered during preview will inform future investments in large-scale policy management and administration experiences.
