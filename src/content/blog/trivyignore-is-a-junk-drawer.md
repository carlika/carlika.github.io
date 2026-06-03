---
title: "Your .trivyignore Is a Junk Drawer"
description: "Suppressing CVEs is fine. Suppressing them without a reason, an owner, or an expiry date is just amnesia. Here's how to treat Trivy suppressions like code."
pubDate: "Jun 03 2026"
heroImage: "../../assets/trivy-cover.svg"
tags: ["devops", "security", "trivy", "ci-cd", "supply-chain"]
---

Your container scanner finds 13 CVEs. You suppress 2 of them. Months later, someone opens the file and finds two bare CVE IDs with no explanation.

Be honest — would you remember why?

This is the most common failure mode I see in pipeline security, and it has nothing to do with the scanner. [Trivy](https://trivy.dev/), Grype, whatever you run — they all find the vulnerabilities just fine. The problem is what teams do with the findings they decide *not* to fix. Those decisions quietly rot, and by audit season nobody can defend a single one of them.

Here's how to stop that, using Trivy as the example. You can reproduce every step below locally in about five minutes.

## A deliberately vulnerable image

Start with something that's guaranteed to light up a scanner — an end-of-life Alpine base plus some ancient pip packages:

```dockerfile
# Dockerfile
FROM python:3.9.16-alpine3.16
RUN pip install --no-cache-dir requests==2.19.1 pyyaml==5.1 Jinja2==2.10
```

```bash
docker build -t vuln-demo:latest .
trivy image --severity HIGH,CRITICAL vuln-demo:latest
```

You get a wall of findings. Trimmed to the interesting parts:

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

Some of these you'll fix. Some you'll legitimately decide to accept — a vulnerable function you never call, a transitive dependency with no fix available yet. That decision to accept is the part that needs governance.

## The junk drawer

The fastest way to silence a finding is `.trivyignore`, a flat list of IDs that Trivy auto-loads from the working directory:

```
CVE-2019-10906
CVE-2018-18074
```

"But I'll add comments!" Sure — and Trivy throws them away. The `#` lines are real comments, but they're invisible to the tool. They never reach the scan report and nothing stops them from going stale. Here's what Trivy actually shows for that file:

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

`Statement: N/A`. Two security decisions, no record of who made them or why. The context — if it exists at all — lives in a comment *next to* the suppression, not *attached to* it. The tool can't surface it, can't enforce it, and can't expire it.

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

Now the suppressed table carries the reason in the `Statement` column instead of `N/A`. The difference between the two files is the difference between amnesia and an audit trail.

Commit that YAML file to the repo and three things change:

- **Every suppression goes through a PR.** Adding a CVE to the ignore list becomes a reviewed decision, not a quiet push at 5pm on a Friday.
- **`git blame` tells you who ignored what, and when.** Accountability for free.
- **`expired_at` makes the ignore self-destruct.** Once the date passes, the CVE returns to your scan results on its own, and you re-decide with fresh eyes instead of letting it sit untouched for two years.

You can watch the expiry behavior directly. Set a date in the past and re-scan — the CVE jumps straight back into the active findings:

```bash
# macOS uses BSD sed, hence the empty ''
sed -i '' 's/expired_at: 2026-07-15/expired_at: 2024-01-01/' .trivyignore.yaml
trivy image --ignorefile .trivyignore.yaml --show-suppressed --severity HIGH,CRITICAL vuln-demo:latest
```

## Permanent suppression is sometimes correct

It's tempting to treat every un-expiring suppression as a smell. It isn't. A CVE in a code path you genuinely never execute, or a dependency you've vendored and patched yourself, doesn't need to keep nagging you — so you leave `expired_at` off and it stays suppressed forever.

The problem was never permanence. It's *unaccountable* permanence: no reason, no owner, no review. A permanent ignore with a `statement` and a `git blame` trail is a defensible engineering decision. A bare line in `.trivyignore` is just something you've agreed to stop looking at.

So: `expired_at` for risks you're accepting *temporarily* (waiting on an upstream fix, a planned upgrade). A `statement` with no expiry for risks you've accepted *permanently* and on purpose. Both are honest. The naked CVE ID is the only dishonest option.

## Push it left, into the PR

Local scans are for you. The real leverage is moving this into the pull request, where the whole team sees it.

Wire the same scan into your CI and have it post the results as a PR comment. Now your suppressions aren't a file someone *might* read — they're sitting in the review, next to the diff:

> 🛡️ **Trivy scan** — 3 CRITICAL blocking · 2 suppressed
> `CVE-2018-18074` ignored — *"We never follow HTTPS→HTTP redirects (SEC-412)"* · expires 2026-09-01

A reviewer can challenge a suppression *before* it ships. "Why are we ignoring that?" becomes a comment thread, not a discovery six months later in an incident retro.

And the things that actually matter still block the merge. In the demo I left the PyYAML CRITICALs untouched, so the gate fails exactly as it should:

```bash
trivy image --ignorefile .trivyignore.yaml --exit-code 1 --severity HIGH,CRITICAL vuln-demo:latest
echo "exit code: $?"   # 1 — CRITICALs still live, merge blocked
```

Responsible suppression was never about turning the build green. It's about documenting the risks you've consciously accepted while the genuinely dangerous ones still block the merge.

## A closing irony

If you needed a reminder that your security tooling is itself attack surface: in early 2026, Trivy's own GitHub Actions were compromised in a supply-chain attack that abused a `pull_request_target` workflow to steal CI secrets and hijack release tags. The scanner you run to catch vulnerabilities got popped through its own pipeline.

The lesson isn't "don't use Trivy" — it's excellent. It's that the pipeline doing your scanning deserves the same scrutiny as the code it scans: least-privilege tokens, pinned action SHAs, and a very healthy suspicion of `pull_request_target`. But that's a post for another day.

---

Scanning images is easy. Governing what you choose to ignore is the actual security work.
