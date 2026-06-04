---
title: "Your .trivyignore Is a Junk Drawer"
description: "Suppressing CVEs is fine. Suppressing them without a reason, an owner, or an expiry date is just amnesia. Here's how to treat Trivy suppressions like code."
pubDate: "Jun 03 2026"
heroImage: "../../assets/trivy-cover.svg"
tags: ["devops", "security", "trivy", "ci-cd", "supply-chain"]
---

Your container scanner finds 13 CVEs. You suppress 2 of them. Months later, someone opens the file and finds two bare CVE IDs with no explanation.

Would you remember why?

This is the most common failure mode I see in pipeline security, and it has nothing to do with the scanner. Trivy, Grype, whatever you use, they all find the vulnerabilities just fine. The problem is what teams do with the findings they decide *not* to fix. Those decisions quietly rot, and by audit season nobody can defend a single one of them.

Here's how to address that, using [Trivy](https://trivy.dev/) as the example. Every step below can be reproduced locally in about five minutes.

## A deliberately vulnerable image

Start with an end-of-life Alpine base and some outdated pip packages:

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

The fastest way to silence a finding is `.trivyignore`, a flat list of IDs that Trivy auto-loads from the working directory:

```
CVE-2019-10906
CVE-2018-18074
```

"But I'll add comments!" Trivy ignores them. The `#` lines are valid syntax, but they never reach the scan report and nothing prevents them from going stale. Here's what Trivy actually shows for that file:

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

`Statement: N/A`. Two security decisions with no record of who made them or why. Any context lives in a comment next to the suppression, not attached to it. The tool can't surface it, enforce it, or expire it.

## Suppressions as structured data

The fix is `.trivyignore.yaml`. Same suppressions, but now the reasoning is data the tool understands:

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

- **Every suppression goes through a PR.** Adding a CVE to the ignore list becomes a reviewed decision, not something quietly pushed on a Friday afternoon.
- **`git blame` tells you who ignored what, and when.** Accountability without extra tooling.
- **`expired_at` limits how long the ignore stays active.** Once the date passes, the CVE returns to your scan results and you review it again with fresh context.

You can verify the expiry behavior directly. Set a date in the past and re-scan:

```bash
# macOS uses BSD sed, hence the empty ''
sed -i '' 's/expired_at: 2026-07-15/expired_at: 2024-01-01/' .trivyignore.yaml
trivy image --ignorefile .trivyignore.yaml --show-suppressed --severity HIGH,CRITICAL vuln-demo:latest
```

## Permanent suppression is sometimes correct

Not every suppression needs an expiry date. A CVE in a code path you genuinely never execute, or a dependency you've vendored and patched yourself, does not need to keep appearing in scan results. Leave `expired_at` off and it stays suppressed indefinitely.

The problem was never permanence. It is *unaccountable* permanence: no reason, no owner, no review. A permanent ignore with a `statement` and a `git blame` trail is a defensible engineering decision. A bare line in `.trivyignore` is just something the team agreed to stop looking at.

So: `expired_at` for risks you're accepting *temporarily* (waiting on an upstream fix, a planned upgrade). A `statement` with no expiry for risks you've accepted *permanently* and on purpose. Both are honest. The naked CVE ID is the only dishonest option.

## Move the action results into your PR

The real leverage is bringing this into the pull request, where the whole team can see it.

Wire the scan into your CI and have it post results as a PR comment. Suppressions are no longer something someone might check. They are visible in the review, next to the diff:

> 🛡️ **Trivy scan** — 3 CRITICAL blocking · 2 suppressed
> `CVE-2018-18074` ignored — *"We never follow HTTPS→HTTP redirects (SEC-412)"* · expires 2026-09-01

A reviewer can challenge a suppression *before* it ships. "Why are we ignoring that?" becomes a comment thread, not a discovery six months later in an incident retro.

The findings that actually matter still block the merge. In this demo the PyYAML CRITICALs are left untouched, so the gate fails as expected:

```bash
trivy image --ignorefile .trivyignore.yaml --exit-code 1 --severity HIGH,CRITICAL vuln-demo:latest
echo "exit code: $?"   # 1 — CRITICALs still live, merge blocked
```

Responsible suppression is not about turning the build green. It is about documenting the risks you have consciously accepted, while the genuinely dangerous findings still block the merge.

## A closing irony

Worth keeping in mind: your security tooling is itself an attack surface. In early 2026, Trivy's own GitHub Actions were compromised in a supply-chain attack that abused a `pull_request_target` workflow to steal CI secrets and hijack release tags.

The lesson is not "don't use Trivy". The pipeline doing your scanning deserves the same scrutiny as the code it scans: least-privilege tokens, pinned action SHAs, and careful use of `pull_request_target`. That is a topic for another post.

---

Scanning images is easy. Governing what you choose to ignore is the actual security work.
