---
layout: post
title: "Turning a Vulnerability Scanner into a Real Security Gate"
date: 2026-10-15
tags: [devsecops, snyk, trivy, github-actions, sca]
---

## A dashboard nobody has to open

A few weeks ago I connected Snyk to my GitHub account and pointed it at two intentionally vulnerable apps, OWASP Juice Shop and WebGoat, just to get hands-on with the tooling. It scanned both and gave me a dashboard full of issues. Useful, but it bothered me. A dashboard that nobody has to look at isn't security. It's a to-do list someone can ignore forever.

So I rebuilt it as something that can't be ignored: a check inside the CI/CD pipeline that fails the build and blocks the merge when it finds something serious.

That's the difference between monitoring and gating. Monitoring tells you about a problem after the fact. A gate stops the problem from moving forward at all.

## What I built

With Juice Shop as the target repo, I added a GitHub Actions workflow with two jobs that run in parallel on every push and pull request:

- **Snyk** scans the npm dependencies across the whole project and fails the build on any High or Critical finding.
- **Trivy** scans the built Docker image for OS-level vulnerabilities in the base layers, with the same severity threshold.

Snyk authenticates with an API token from the Snyk dashboard. I stored it as an encrypted GitHub Actions secret and referenced it in the workflow, so it never appears in the repo or in the logs.

Once both jobs were confirmed working, I added a branch protection rule that requires both checks to pass before anything merges into the main branch. This part matters more than it sounds. A workflow that turns red is not a gate by itself. Nothing stops someone from merging anyway unless GitHub is told to enforce it. I tested that directly later on, in the section about the pull request.

## What went wrong along the way

None of it worked on the first try, which made it a better exercise.

**The pipeline died before the scan even started.** Juice Shop's install step triggers an Angular frontend build, and Angular's CLI needs a newer Node version than the one I'd pinned. I bumped Node and skipped the build step entirely, since a dependency scan only needs the manifest files, not a compiled app.

**The scan tried to check a build output folder.** Once I widened it to every manifest in the repo, it hit a generated folder with no installed dependencies. Easy fix once I understood what it was trying to do.

**My edit never triggered anything.** I changed a file in GitHub's web editor and it landed on a side branch, not the main branch. The pipeline never ran. No error, no red X, just silence. A missing result and a failing result look very different, and they need different troubleshooting instincts.

## What the scan found

Once everything was wired correctly, Snyk reported 49 issues across 79 vulnerable paths. Three stood out.

**JWT handling.** Several JWT dependencies are old enough to be vulnerable to the classic `alg: none` attack. If the library isn't told which algorithms to trust, a token that declares itself unsigned gets accepted as valid. That's a full authentication bypass sitting in a dependency tree.

**Multer.** The file upload library had a critical uncaught exception bug. One malformed upload request can crash the whole process.

**A decompression library.** It had a directory traversal flaw in archive extraction, and no patch exists. The fix isn't a version bump. It's a decision about whether to keep using an unmaintained dependency.

That last one interested me most. Most security advice assumes a patch is waiting for you. Sometimes there isn't one, and working out what to do next is its own skill.

## Proving the gate blocks a merge

A failing build on `master` is convincing, but it doesn't prove the gate stops a real pull request. So I opened one, a throwaway one-line README edit, to watch it happen.

There was one more wrinkle. GitHub Actions was disabled on the repo by default. GitHub does this on forks that already contain workflow files, so it wasn't something I misconfigured. One settings change and one more commit later, the checks ran.

Both `snyk-scan` and `trivy-scan` failed on the PR. Instead of a normal merge button, GitHub showed this:

> Merge without waiting for requirements to be met (bypass rules)

That's the gate. A regular contributor wouldn't even get that option. They'd see merging disabled. As the repo owner I can override it, but now overriding is a visible, deliberate act and not the default.

![Pull request with failed Snyk and Trivy checks and the bypass-rules merge option]({{ '/assets/images/juice-shop-bypass-rules.png' | relative_url }})

## Takeaway

- **A gate is the scanner plus enforcement.** The workflow that fails is half of it. The branch protection rule is the other half.
- **Keep tokens in secrets.** The Snyk token lives in an encrypted GitHub Actions secret, never in the repo or the logs.
- **No result is not a pass.** When the pipeline stays silent, check where your change actually landed before you blame the scanner.
- **A finding isn't always fixable.** Some dependencies have no patch, so the real work is deciding whether to replace them.
- **A scanner reports what's installed, not what's exploitable.** Next I want to check whether the JWT finding is reachable from Juice Shop's live login code or just sits unused in the tree. After that I'm moving on to SAST, to see what static analysis catches in the application's own code.
