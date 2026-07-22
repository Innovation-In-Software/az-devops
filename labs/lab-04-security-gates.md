# Lab 4: Add Security Gates to the ShipIt Pipeline

**Module:** 4 (Security in the Pipeline)
**Time:** 45-60 minutes
**Format:** Hands-on (GitHub Advanced Security + GitHub Actions)

---

## Why this lab

Your pipeline now builds, tests, and containerizes ShipIt on every change. That is exactly the problem this lab solves. As the slides put it: **"Speed without guardrails just ships bugs faster."** The moment the pipeline containerizes every commit automatically, it will also ship a vulnerable dependency, an unsafe code pattern, or a leaked key just as automatically — all the way to the registry, no human in the loop.

The economic case is the whole reason to act now. A defect that reaches production costs **10 to 100 times more** to fix than one caught early (the "10–100x cost multiplier" diagram from the slides). Security work is not a separate phase you bolt on at the end — it is **shift-left**: "move the security check as close to the keystroke as possible." This lab lives at the cheap end of that curve. Every scan you wire up here catches a class of problem while it is still a pull request comment instead of an incident report.

The plan is **"three risks, three tools, one gate"**: other people's code (Dependabot), your code (CodeQL), and secrets (secret scanning), all converging on the pull request as the single choke point before `main`. You will enable the three scanners, wire them into branch protection, and then watch a planted problem get blocked at the PR. By the end, nothing dangerous can merge to `main` — and this is the finished Day 1 gate.

> **From the slides:** No single tool covers all three risks. "Dependabot watches other people's code. CodeQL watches yours." Secret scanning covers the third. That is why the answer is three tools converging on one gate, not one magic scanner.

## Concepts this lab makes real

This lab is where the Module 4 slide concepts stop being definitions and become something you can see block a merge:

- **Shift-left** — catching defects near the keystroke, at the cheap end of the 10–100x cost curve, instead of in production.
- **SAST (static application security testing)** — analyzing source code *without executing it*; CodeQL is the SAST engine here.
- **CodeQL** — "treats your code as data you can query for dangerous patterns," following data flow to find issues like command injection.
- **Dependabot** — watches your dependencies and "opens automated PRs to bump the vulnerable package"; understands the difference between **security updates** (fix a known CVE) and **version updates** (routine freshness).
- **CVE** — a known, cataloged vulnerability; unpatched dependencies with CVEs are "the most common way applications get compromised."
- **Secret scanning + push protection** — detects known token formats and "blocks the commit at push time, before it lands."
- **Required status checks** — a blocking finding stops the merge "just like a failing test."
- **The quality gate** — the pull request as "the single choke point before main."
- **Triage and security policy** — deciding "what is real, what is urgent, what is a false positive" under a written rule like "Block on high and critical, warn on the rest."

## What you will build

- A `.github/dependabot.yml` that watches NuGet packages and GitHub Actions.
- CodeQL static analysis running on pull requests.
- Secret scanning with push protection turned on.
- The security checks added as required status checks on `main`.

## Success looks like

A pull request with a blocking security finding cannot merge, push protection stops a committed secret before it lands, and the Security tab shows findings with your triage decisions recorded.

## Prerequisites

- You finished Lab 3: ShipIt containerizes and pushes to ACR.
- Your ShipIt repository is **public** (from Lab 0). On a personal GitHub account, Dependabot, CodeQL code scanning, and secret scanning are **free on public repositories**; a private personal repo would require GitHub Advanced Security.
- Commands below run in VS Code's integrated terminal, set to Git Bash (see Lab 0 if you need to switch your terminal profile).
- Behind? Ask your instructor for the checkpoint branch name and check it out from VS Code's Source Control panel (**...** menu → **Branch → Checkout to...**).

---

## Step 1: Turn on the security features

In the repository, go to **Settings → Code security** (or **Advanced Security**) and enable:

- **Dependabot alerts** and **Dependabot security updates.**
- **CodeQL analysis** (choose **Default setup** for the quickest path; it configures the languages for you and, for C#, scans without a separate build step).
- **Secret scanning** and **Push protection.**

> **Why:** You are enabling all three at once on purpose. Each tool covers a different attack surface, and a gap in any one is a way in. Turning them on together is the "three risks, three tools" idea made concrete before you wire anything to the gate.
>
> **From the slides:** "No single tool covers all three risks." Dependencies, your own code, and secrets are three separate failure modes — this screen is where you close all three.

> **What just happened:** alerts for vulnerable dependencies and leaked secrets now work immediately. The next steps make the scanning explicit and enforceable.

## Step 2: Add the Dependabot config

Create a branch for this (VS Code status bar → branch name → **Create new branch...** → type `add-dependabot`). Then, in the Explorer, hover over the `.github` folder, click the **New File** icon, and type `dependabot.yml` (the file goes inside `.github/`, alongside `workflows/`) to schedule version updates and keep your Actions patched:

```yaml
version: 2
updates:
  - package-ecosystem: "nuget"
    directory: "/"
    schedule:
      interval: "weekly"
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

Stage, commit, and **Publish Branch** (Source Control panel), then open a PR into `main` on GitHub.com and merge it — Dependabot only reads this file from your **default branch**, so it has no effect until it's merged. Within a few minutes after merging, Dependabot will open pull requests for any out-of-date or vulnerable packages and for your pinned actions. Open one and read what it proposes.

> **Why:** This is the "other people's code" risk. Most of what ships in ShipIt is dependencies you did not write, and an unpatched CVE in one of them is — per the slides — "the most common way applications get compromised." Dependabot watches for those and opens a PR to bump the package, so the fix arrives as reviewable code instead of a scramble after disclosure. Note the two ecosystems: `nuget` covers your app packages, and `github-actions` keeps the SHA-pinned actions you locked down in Module 2 from silently going stale. Pinning stops a supply-chain surprise; this keeps the pins current.
>
> **From the slides:** Dependabot distinguishes **security updates** (patch a known vulnerability — urgent) from **version updates** (routine freshness — nice to have). This is the Dependabot corner of the "Three Risks, Three Tools" diagram: "Dependabot watches other people's code."

## Step 3: Confirm CodeQL runs on pull requests

**Default setup is the recommended path for this course.** If you chose **Default setup** in Step 1, CodeQL already runs — open any pull request and confirm a **CodeQL** check appears in the Checks list. Default setup uses GitHub's tuned query pack and buildless C# analysis, which gives the broadest, most reliable coverage with zero YAML to maintain. Prefer it unless you have a specific reason not to.

If you prefer the workflow form (more control): in the Explorer, hover over `.github/workflows`, click the **New File** icon, and type `codeql.yml`. Two details matter: analyze `main` **on push** so the Security tab has a baseline to diff pull requests against, and request the **`security-extended`** query suite so injection-class queries are included (the smaller default suite omits several):

```yaml
name: CodeQL
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: "0 6 * * 1"
jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      contents: read
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-dotnet@v5
        with:
          dotnet-version: '10.0.x'
      - uses: github/codeql-action/init@v3
        with:
          languages: csharp
          queries: security-extended
      - run: dotnet build -c Release
      - uses: github/codeql-action/analyze@v3
```

> **Why the `setup-dotnet` step?** This workflow compiles the app (`dotnet build`) so CodeQL can analyze it, and ShipIt pins the .NET 10 SDK in `global.json`. Without `setup-dotnet`, the runner may not have that exact SDK and the build — and therefore the analysis — fails. (Default setup avoids this entirely with buildless C# analysis, which is one more reason it is the recommended path.)

> **Why:** This is the "your code" risk. CodeQL is **SAST** — it analyzes your source *without running it*, treating the code as data it can query for dangerous patterns and following how untrusted input flows through the program. That data-flow tracing is how it will catch the planted command-injection bug in Step 6 before that code ever executes. It runs on the pull request so the finding lands next to the diff that caused it — shift-left in action.
>
> **From the slides:** This is the CodeQL corner of "Three Risks, Three Tools": "CodeQL watches yours." Together with Step 2, you now have two of the three scanners aimed at the same gate.

> **Why the build step?** In this manual workflow form, C# is compiled, so CodeQL needs a build between `init` and `analyze`. GitHub's Default setup avoids this by scanning C# without a build (buildless analysis), which is why it needs no workflow file.

## Step 4: Make the security checks required

Go to **Settings → Branches**, find the branch protection rule for `main` you created in Lab 2, and click **Edit**. Under **Require status checks to pass before merging**, search for and add the **CodeQL** check (your **build-and-test** check from Lab 2 should already be listed there). Scroll down and click **Save changes**.

Now a red security check blocks the merge exactly like a failing test.

> **Why:** This is the step that turns scanning into a *gate*. Up to now the scanners only produce findings; a finding no one is forced to act on is just a notification. Making CodeQL a required status check means a blocking result holds the merge open the same way a failing unit test does — the tooling can finally stop bad code, not just report it. This reuses the exact branch-protection mechanism you built in Module 2; you are adding a security check to a lane that already exists.
>
> **From the slides:** "A blocking finding stops the merge, just like a failing test." This is the pull request acting as the quality gate — "the single choke point before main" — and it is the next layer in **"The Complete Day 1 Gate"** (build + tests from M2, PR/branch protection from M2, container from M3, and now the M4 scans).

## Step 5: Prove push protection works

Try to commit a real-but-throwaway credential and watch push protection stop you before it lands. **This must be an actual GitHub token, not a made-up string** — GitHub validates the token's built-in checksum, so a hand-typed or random `ghp_...` value is *not* detected and will push right through.

1. Create a throwaway token: on GitHub.com, click your profile picture (top-right) → **Settings**. This opens your personal account settings (not the repo's) — scroll to the bottom of the left sidebar and click **Developer settings**, then **Personal access tokens → Tokens (classic) → Generate new token**. Give it **no scopes** and a short expiry. Copy the `ghp_...` value.
2. In VS Code, create a file `leak.txt` at the repo root containing that value, then stage, commit, and **Sync Changes** from the Source Control panel, same as always.

Push protection rejects the push — VS Code shows this as a notification and in the **Output** panel's **Git** channel, which names the detected secret type (GitHub Personal Access Token). This happens on GitHub's server the instant it receives the push, so it fires the same way no matter which Git client sent it.

3. Remove `leak.txt`, commit and push that removal too, then **delete/revoke the token** in the same settings page — that is the real lesson: a leaked key must be revoked and rotated, not just deleted from the commit, because it may already have been copied.

> **Why:** This is the third risk — secrets — and the earliest possible catch on the shift-left curve. Push protection detects known token formats and "blocks the commit at push time, before it lands," so the secret never enters history where it could be cloned, cached, or scraped. Notice the ordering in step 3: you revoke the token *and* remove the file. Deleting the commit does not un-leak it — **rotate, don't just delete** — because the moment a real secret hits a push it must be assumed compromised.
>
> **From the slides:** Secret scanning "detects known token formats"; push protection "blocks the commit at push time, before it lands." This is the secrets corner of "Three Risks, Three Tools," and the whole point is that it fires *before* the gate rather than after — the cheapest catch of all.

> **Why a real token?** Push protection matches recognized secret **types**, and for GitHub tokens that includes a checksum built into the value. A fake or randomly generated `ghp_` string fails the checksum and is silently ignored — so the demo only works with a genuine token you generate and immediately revoke. (A lone AWS access key ID also won't trigger it — GitHub needs the id/secret pair — and the docs example `AKIAIOSFODNN7EXAMPLE` is allowlisted.)

## Step 6: Watch the gate block a real finding

You have already seen the gate itself hold the line twice: a failing unit test blocked a merge in Lab 2, and push protection (Step 5) stopped a secret before it landed. The gate mechanism is proven — a required red check blocks the **Merge** button, full stop. This step focuses that gate on your *own code* via CodeQL.

**Use Default setup (Step 1) for this exercise.** It gives CodeQL the best chance of catching a planted issue, because it uses GitHub's tuned query pack and buildless C# analysis. If you built the workflow form instead, make sure it uses `queries: security-extended` and analyzes `main` on push (Step 3), or CodeQL may not surface the finding.

Introduce a flagged issue and watch it block the PR. The starter ships an example under `examples/insecure/` (see its `README.md`):

1. Create a new branch (VS Code status bar → **Create new branch...**). In the Explorer, open `examples/insecure/CommandInjectionExample.cs.txt` and copy the **vulnerable** endpoint (the first one in the file). Open `src/ShipIt/Program.cs`, paste it in next to the other `app.MapGet(...)` calls, and save. It reads user input and passes it into a shell command. **Also add `using System.Diagnostics;` at the top of `Program.cs`** (the endpoint uses `Process`), or the build will not compile.
2. Stage, commit, and push (Source Control panel), then open a PR into `main`.
3. Give CodeQL a couple of minutes. If it flags the issue, the **CodeQL** check goes red and — because it is a required check (Step 4) — the **Merge** button is blocked.
4. Remediate with the **fixed** version in the same example file, push again, and watch the check clear.

> **Reality check on CodeQL coverage.** Whether CodeQL flags a *specific* planted line depends on its current C# models and your configuration. On this minimal-API app, some injection patterns are not traced to a source out of the box (the models are tuned more for controller-based apps), and the workflow form's *default* query suite omits several injection queries entirely — which is why Default setup with the extended suite is recommended. If your example does **not** get flagged, that is not a broken lab: it is a real, valuable lesson — **a scanner's coverage has gaps, so it is one layer of defense in depth, not a guarantee.** The gate that stops a dangerous change is proven by Steps 4–5 regardless; CodeQL is the SAST layer that widens what that gate can catch over time.

> **From the slides:** This is the finished Day 1 gate in motion — build, test, container, and the three scanners all converging on one pull-request gate. As the slides put it, **"a finding only protects you if it can stop the merge"** — and the corollary you just met is that no single scanner catches everything, which is exactly why the design is **three risks, three tools**, not one.

## Step 7: Triage one finding

Open the **Security** tab, pick one finding, and practice triage:

- Judge its severity and whether it is actually exploitable in ShipIt.
- If it is a true positive you will fix, note it and track it.
- If it is a false positive, **dismiss it with a reason** (do not just hide it).

Record your decision. This is the habit that keeps the tooling trustworthy.

> **Why:** Tooling is only half the job — someone has to decide, for each finding, "what is real, what is urgent, what is a false positive." That is triage, and it is a human skill the scanners cannot do for you. The discipline here is dismissing false positives *with a reason* rather than silently hiding them, because the alternative rots the whole system: as the slides warn, "a backlog of ignored alerts trains people to ignore the real one." An alert list you trust is one you actually read.
>
> **From the slides:** Triage is where security stops being a tool and becomes a practice. The scanners find candidates; you decide what blocks and what is noise, and you leave a record so the next person can trust the queue.

---

## Success criteria

- A pull request with a blocking (high/critical) security finding cannot merge.
- Push protection rejects a committed secret before it lands in history.
- Dependabot has opened at least one update pull request.
- The Security tab shows findings, and you have recorded a triage decision on one.

## Troubleshooting

- **No security features available:** confirm your repository is **public**. On a personal account, code scanning and secret scanning are free on public repos; a private personal repo would need GitHub Advanced Security.
- **CodeQL check never appears:** confirm Default setup is enabled, or that the `codeql.yml` workflow ran at least once. The check name only becomes selectable in branch protection after a first run.
- **Push protection did not trigger:** it blocks on recognized token formats and allowlists well-known documentation examples. Use a well-formed GitHub token (`ghp_` plus 36 random characters); avoid the canonical AWS example key, which is allowlisted and also needs its paired secret to be detected.
- **Dependabot opened no PRs:** your dependencies may already be current. Temporarily pin an older, known-vulnerable package version to see it work (then revert).

## Stretch goals

- Set a severity threshold on the Lab 3 image scan (`exit-code: "1"` on HIGH/CRITICAL) so a vulnerable base image also fails the build.
- Add a `CODEOWNERS` file so changes to security-relevant paths require a specific reviewer.
- Group Dependabot updates with a `groups:` block so you get one PR per ecosystem instead of many.

> **Why:** These extend the same principle to its natural conclusion — a written **security policy**. The image-scan threshold encodes a rule like "block on high and critical, warn on the rest," so the gate's behavior is deliberate rather than accidental. As the slides put it, "the goal isn't zero findings, it's a shared rule for what blocks." A policy everyone can point to is what keeps triage consistent when you are not the one reviewing.

---

## What this sets up

This completes Day 1. Every change to ShipIt is now built, tested, containerized, scanned, and reviewed, and a broken or unsafe change cannot merge. You have a known-good, SHA-tagged image for every commit. On **Day 2**, Module 5 promotes exactly those images through environments with approvals and automated rollback, Module 6 runs them on Kubernetes and OpenShift, and Module 7 provisions their infrastructure and puts them under monitoring, closing the loop back to the DORA targets you set in Lab 1.
