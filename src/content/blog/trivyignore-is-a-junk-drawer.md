---
title: "Your .trivyignore Is a Junk Drawer"
description: "Suppressing CVEs is fine. Suppressing them without a reason, an owner, or an expiry date is just amnesia. Here's how to treat Trivy suppressions like code."
pubDate: "Jun 03 2026"
heroImage: "../../assets/trivy-cover.svg"
tags: ["devops", "security", "trivy", "ci-cd", "supply-chain"]
---

Your container scanner finds 13 CVEs. You suppress 2 of them. Months later, someone asks or requests additional information about the suppression.

Would you remember why? I certainly won't.

This is one of the biggest issues I had working in companies that had security certifications like SOC2 or ISO27001. Security pipelines are probably the worst nightmare for developers but it has nothing to do with the tool. Trivy, Grype, whatever you use, they all find the vulnerabilities just fine. The problem is what teams do with the findings they decide not to fix. Those decisions can go under the radar, usually done in a meeting where there is no audit trail for what drove that.

Here's how to address that, using [Trivy](https://trivy.dev/) as the tool of choice. I will do a practical example that you can run locally too.

## A deliberately vulnerable image

I'll start with an older image and some outdated pip packages:

```dockerfile
# Dockerfile
FROM python:3.9.16-alpine3.16
RUN pip install --no-cache-dir requests==2.19.1 pyyaml==5.1 Jinja2==2.10
```

```bash
docker build -t vuln-demo:latest .
trivy image --severity HIGH,CRITICAL vuln-demo:latest
```

Running the scan produces a lot of output. Trimmed to the relevant parts:

```
vuln-demo:latest (alpine 3.16.5)
Total: 7 (HIGH: 7, CRITICAL: 0)

Python (python-pkg)
Total: 13 (HIGH: 11, CRITICAL: 2)

┌───────────────────┬────────────────┬──────────┬───────────────────┬───────────────┐
│      Library      │ Vulnerability  │ Severity │ Installed Version │ Fixed Version │
├───────────────────┼────────────────┼──────────┼───────────────────┼───────────────┤
│ Jinja2 (METADATA) │ CVE-2019-10906 │ HIGH     │ 2.10              │ 2.10.1        │
│ PyYAML (METADATA) │ CVE-2019-20477 │ CRITICAL │ 5.1               │ 5.2           │
│ PyYAML (METADATA) │ CVE-2020-1747  │ CRITICAL │ 5.1               │ 5.3.1         │
│ requests          │ CVE-2018-18074 │ HIGH     │ 2.19.1            │ 2.20.0        │
│ ...               │ ...            │ ...      │ ...               │ ...           │
└───────────────────┴────────────────┴──────────┴───────────────────┴───────────────┘
```

Some of these you'll fix. Some you'll legitimately decide to accept: a vulnerable function you never call, or a transitive dependency with no fix available yet. That accepted risk is the part that needs governance.

## The junk drawer

The cleanest way to silence the warning is by creating a `.trivyignore` file. It accepts CVE ID and you can add comments with a `#`.

```
#List of CVEs
CVE-2019-10906
CVE-2018-18074
```

"But I'll add comments!" - although this is better than just a plain list of CVEs, Trivy can't "understand" them. The `#` lines are valid syntax, but they never reach the scan report and nothing prevents them from going stale. And going stale is why I'm writing this article in the first place. Here's what Trivy actually shows for that file:

```bash
trivy image --severity HIGH,CRITICAL --show-suppressed vuln-demo:latest
```

```
Suppressed Vulnerabilities (Total: 2)

┌──────────┬────────────────┬──────────┬─────────┬───────────┬──────────────┐
│ Library  │ Vulnerability  │ Severity │ Status  │ Statement │    Source    │
├──────────┼────────────────┼──────────┼─────────┼───────────┼──────────────┤
│ Jinja2   │ CVE-2019-10906 │ HIGH     │ ignored │ N/A       │ .trivyignore │
│ requests │ CVE-2018-18074 │ HIGH     │ ignored │ N/A       │ .trivyignore │
└──────────┴────────────────┴──────────┴─────────┴───────────┴──────────────┘
```

`Statement: N/A`. Two security decisions with no record of why they were made. You can say "Use `git blame` and see who added that CVE to list of ignored ones and track the comment if any", but comment next to the suppression is not a mandatory thing and that person is maybe not in the company any longer. We want to attach the properties to it, properties like expiration, reasoning etc. The tool can't surface it, enforce it, or expire it by reading the comments.

## Suppressions as structured data

The fix is `.trivyignore.yaml`. Same suppressions, but now the reasoning is structured so the tool understands:

```yaml
vulnerabilities:
  - id: CVE-2018-18074
    statement: "We never follow HTTPS->HTTP redirects; tracked in SEC-412"
    expired_at: 2026-09-01
  - id: CVE-2019-10906
    statement: "Jinja2 not used to render untrusted templates"
    expired_at: 2026-07-15
```

One catch worth knowing: the YAML format is still experimental, so it doesn't auto-load. You point at it explicitly:

```bash
trivy image --ignorefile .trivyignore.yaml --show-suppressed --severity HIGH,CRITICAL vuln-demo:latest
```

Now the suppressed table shows the reason in the `Statement` column instead of `N/A`. That is the practical difference between the two formats.

Commit that YAML file to the repo and three things change:

- **Every suppression goes through a PR.** Adding a CVE to the ignore list becomes a reviewed decision.
- **`git blame` tells you who ignored what, and when.** Accountability without extra tooling.
- **`expired_at` limits how long the ignore stays active.** Maybe you are not sure if this is the issue, or there is no available fix for it at the moment. Once the date passes, the CVE returns to your scan results and you review it again with fresh context.

## Permanent suppression is sometimes correct

Not every suppression needs an expiry date. A CVE in a code path you genuinely never execute, or a dependency you've vendored and patched yourself, does not need to keep appearing in scan results. Leave `expired_at` off and it stays suppressed indefinitely.

The problem was never permanence. It is unaccountable permanence: no reason, no owner, no review. A permanent ignore with a `statement` and a `git blame` trail is a defensible engineering decision. A bare line in `.trivyignore` is just something the team agreed to stop looking at.

So: `expired_at` for risks you're accepting temporarily (waiting on an upstream fix, a planned upgrade). A `statement` with no expiry for risks you've accepted permanently and on purpose.

## Move the action results into your PR

The real leverage is bringing this into the pull request, where the whole team can see it.

Wire the scan into your CI and have it post results as a PR comment. Suppressions are no longer something someone might check. They are visible in the comment section of your PR.

> 🛡️ **Trivy scan** — 3 CRITICAL blocking · 2 suppressed
> `CVE-2018-18074` ignored — *"We never follow HTTPS→HTTP redirects (SEC-412)"* · expires 2026-09-01

A reviewer can challenge a suppression before it ships. "Why are we ignoring that?" becomes a comment thread, not a discovery six months later in an incident retro.

The findings that actually matter should still block the merge. Team can decide how many vulnerabilities they accept to have active and still allow merge. As an example, you can say that having more than 0 CRITICAL vulnerabilities will automatically block the merge. This is how you should approach your pipeline logic. I'll talk more about pipeline setup in the coming posts.

Responsible suppression is not about turning the build green. It is about documenting the risks you have consciously accepted, while the genuinely dangerous findings still block the merge.

## A closing irony

Worth keeping in mind: your security tooling is itself an attack surface. In early 2026, Trivy's own GitHub Actions were compromised in a supply-chain attack that abused a `pull_request_target` workflow to steal CI secrets and hijack release tags.

The lesson is not "don't use Trivy". The pipeline doing your scanning deserves the same scrutiny as the code it scans: least-privilege tokens, pinned action SHAs, and careful use of `pull_request_target`. That is a topic for another post.

---

Scanning images is easy. It's the flow that matters, and ownership of your vulnerabilities. Suppress-and-forget should never happen.
