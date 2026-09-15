# Actions Native Egress Firewall: Early Access
> [!IMPORTANT]
> **Status: Technical Preview.** The native egress firewall supports log mode and file-based enforcement in technical preview. Linux is the only supported platform at this time.

Welcome to the early access program for the **GitHub Actions native egress firewall**. This program gives design partners hands on access to log and file-based enforcement for GitHub-hosted runners[...]

## Getting started in early access

The native egress firewall builds on GitHub Actions' existing runner image architecture, but operates within the context of a nested-VM. Separating the runner from the firewall. We provide both a s[...]

Runner access in early access:

- **Standard GitHub-hosted runners** are available by default. Set `runs-on: ubuntu-24.04-firewall` in your workflow file.
- **Larger runners** are available by request during technical preview. Open a [larger runner access request](../../issues/new?template=larger-runner-access-request.yml) and include:
  - whether access is for an enterprise or organization
  - the exact enterprise or organization name
  - a point of contact

After approval, ask your administrator to create a Linux larger runner using the GitHub-maintained **`Ubuntu 24.04 with Firewall`** image.

> [!NOTE]
> **Performance during preview.** With the firewall enabled, expect roughly a **15–20% increase in workflow runtime** for typical workloads, driven by the virtual machine monitoring your network[...]


## What is the native egress firewall?

GitHub hosted runners today allow unrestricted outbound network access. Any workflow can reach any host on the internet, regardless of `GITHUB_TOKEN` permissions, secret scoping, OIDC, or SHA pinn[...]

The native egress firewall closes that gap. It runs **outside** the runner VM, inspects DNS and HTTP/HTTPS traffic, and remains immutable even if a workflow gains root access inside the runner. It[...]

The capability ships in two modes:

- **Log mode** records every outbound DNS lookup and HTTP request without blocking anything. This is the safe entry point.
- **`enforce` mode** applies an allow list. Traffic outside the list is blocked by default, recorded, and surfaced in the workflow summary with the offending command and the rule that denied it.

> [!WARNING]
> **Log mode can still affect traffic.** Because the firewall terminates and re-establishes TLS at the egress boundary (see [How HTTPS inspection works](#how-https-inspection-works)), the proxy si[...]
>
> Treat "log mode records traffic without blocking anything" as a goal, not a guarantee: log mode *can* break a workflow. This most often affects Node-based actions calling cloud endpoints, such a[...]

### How HTTPS inspection works

To support URL-level allow rules, the firewall **terminates TLS at the egress boundary and re-establishes TLS to the destination**. Each workflow run gets a unique, ephemeral certificate that is d[...]

- If your workflow trusts the operating system certificate store (the default for `curl`, `git`, `npm`, `pip`, `docker`, etc.), you will see a normal HTTPS connection. No changes are required.
- If your workflow does **certificate pinning** or **mTLS**, you will need to update it to trust the per-run ephemeral certificate.

## File-based enforcement

File-based enforcement defines egress policy in a configuration file committed to your repository. The firewall evaluates that policy at the egress boundary. The file declares a mode and an allow [...]

Keeping policy alongside a workflow makes it reviewable in pull requests, version controlled, diffable, and portable with the repository. Use file-based rules when those policy-as-code properties [...]

> [!IMPORTANT]
> **Technical preview.** File-based enforcement is in technical preview. The experience, rule format and schema (including `no-default-urls`), file discovery, and APIs may change based on customer[...]

### Where the policy file goes

Follow these steps to add the policy file:

1. Create (or open) the `.github` directory at the root of the repository that contains your workflow.
2. Add a file named `egress-firewall.yaml` in that directory — the full path must be `.github/egress-firewall.yaml`.
   > [!NOTE]
   > The file name is case sensitive. Use `egress-firewall.yaml` exactly as shown; `Egress-Firewall.yaml` or any other casing will not be discovered.
3. Define your policy in that file (see the configuration example below).
4. Commit the file to the branch the workflow will run from.

> [!IMPORTANT]
> Do **not** place `egress-firewall.yaml` in `.github/workflows`. If the file is missing, misnamed, or in the wrong directory, it will not be discovered and no policy will be enforced.

```text
your-repo/
├── .github/
│   ├── egress-firewall.yaml   # ✅ policy file goes here
│   └── workflows/
│       └── ci.yml             # ❌ do NOT put egress-firewall.yaml here
```

> [!NOTE]
> The policy file is read from the same ref (branch, tag, or SHA) that the workflow runs from—the same ref that Actions uses to resolve the workflow file. For a `workflow_dispatch` run, this is [...]

### Configuration example

The policy file is committed to the repository with the workflow it protects. For this technical preview, use the path and filename documented above. The path, filename, and discovery mechanism ma[...]

By default, policies allow egress to:

- `github.com`
- `*blob.core.windows.net` (*)
- `codeload.github.com`
- `*actions.githubusercontent.com` (*)

The `(*)` entries are GitHub-managed defaults, not user-configurable wildcard patterns. The firewall expands them to matching URLs returned by the GitHub `/meta` endpoint, so the allowed set canno[...]

The following example adds `api.github.com` and `release-assets.githubusercontent.com`; the other GitHub endpoints shown above are already covered by the default allow list.

```yaml
mode: enforce
allow:
  - api.github.com
  - release-assets.githubusercontent.com
```

- `mode` selects behavior: `enforce` denies hosts not matched by `allow`; `log` records traffic without intentionally denying it.
- `allow` adds hosts to the default allow list.

### Building your allow list

For the initial run, set `mode: log`. Use the resulting list of outbound URLs and hosts that were requested to construct your `allow` list before switching to `mode: enforce`. Customers using Git[...]

The [GitHub Enterprise Cloud meta endpoint](https://docs.github.com/en/enterprise-cloud@latest/rest/meta/meta?apiVersion=2026-03-10#get-github-enterprise-cloud-meta-information) can help you unde[...]

Set `no-default-urls: true` to disable the default allow list. You must then explicitly list every endpoint your workflow needs, including GitHub endpoints such as `github.com` and `codeload.gith[...]

```yaml
mode: enforce
no-default-urls: true
allow:
  - github.com
  - api.github.com
  - codeload.github.com
```

The following workflow uses the firewall runner with the additive policy example. The first request is allowed; the PyPI request is a representative CI dependency lookup to a host not in the allo[...]

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

In `enforce` mode, a request to a denied host fails from inside the workflow as a failed connection or request error in the affected step. The workflow run summary identifies the denied traffic, [...]

Preview firewall events are surfaced in the workflow run summary and as a workflow run artifact. Events include the binary name, without command-line flags or environment variables, and the URL w[...]

## Two delivery paths

| Adoption path | Runner type | How it is enabled | Best for |
|---|---|---|---|
| Firewall enabled label | Standard GitHub hosted runners | Set `runs-on: ubuntu-24.04-firewall` in the workflow | Individual repositories, open source projects, fast adoption with no admin setup[...]
| Firewall enabled image | Larger runners | Select the GitHub maintained `Ubuntu 24.04 with Firewall` image when creating the larger runner | Enterprises that already use larger runners, custom t[...]

Both paths produce identical Layer 7 enforcement, identical telemetry, and identical rule semantics. Custom larger runner images built on the firewall base image cannot disable or bypass the fire[...]

## Phased rollout

| Phase | Capability | Scope |
|---|---|---|
| Technical preview | Log mode and file-based enforcement with allow list rules. Deny all by default in `enforce` mode | Linux, opt in via runner label or larger runner image |
| Public preview | Expanded policy and administration experiences, informed by preview feedback | Scope to be determined |
| GA | Further capability expansion, informed by customer feedback | Scope to be determined |

### Where logs go

- **Preview:** firewall events are surfaced in the workflow run summary and as a workflow run artifact. We will log the binary name (without command line flags or environment variables) as well a[...]
- **GA:** events stream to the **Actions data stream** with workflow, job, step, and command attribution, ready for ingestion into existing SIEM and detection pipelines.

## Feedback Requested

Technical preview participants: please tell us about your experience with:

- File-based rule authoring, including how policy changes fit into pull request review.
- The rule format and schema.
- The shape and usefulness of returned enforcement data.
- Logging, observability, and troubleshooting.
- Missing capabilities, edge cases, and scale requirements.

- [Issue report](../../issues/new?template=issue-report.yml): report a bug, blocked request, false positive, or unexpected behavior.
- [Feature request](../../issues/new?template=feature-request.yml): request a new rule kind, a managed rule, or a platform expansion.
- [Larger runner access request](../../issues/new?template=larger-runner-access-request.yml): request early access to larger runners using the `Ubuntu 24.04 with Firewall` image.

## Future Direction and Scale Considerations

We recognize that many customers need to manage and enforce egress policies consistently across large organizations and enterprises. Managing policy across hundreds or thousands of repositories r[...]

Organization-level and enterprise-level management, governance, and policy distribution are not the focus of this technical preview. The immediate goal is to provide meaningful enforcement and va[...]
