# Lab 2: Build a CI Pipeline for ShipIt

**Module:** 2 (CI Pipelines with GitHub Actions)
**Time:** 45-60 minutes
**Format:** Hands-on (GitHub repository + GitHub Actions)

---

## Why this lab

In Lab 1 you found that Contoso Parcel builds ShipIt by hand on a developer's laptop and lets one person deploy on Fridays. Continuous integration is **the first fix** for both problems. Instead of the build living on "fifteen laptops" — each with its own SDK version, its own leftover files, its own "works on my machine" — you move it onto **one authoritative build on a clean machine** that runs on every single change.

That timing is the whole point. When the build and tests run on every push and pull request, a problem surfaces **while it is still small and cheap** — minutes after it was introduced, next to the change that caused it, instead of on a Friday afternoon when someone finally tries to ship. A broken change now stops at the door instead of reaching `main`.

This lab takes the first concrete step against those targets: you move the build and the tests off the laptop and into an automated workflow that runs on every change, then you protect `main` so a broken change cannot merge. By the end, a red X on a pull request means a real, reproducible failure, and nobody can route around it — the concrete end of "one person owns every deploy."

Keep one idea in mind as you go: **CI is a habit before it is a tool.** The YAML is easy. The discipline of letting the machine be the source of truth — and of building the gate that makes its verdict binding — is the actual skill.

## Concepts this lab makes real

> This lab turns the Module 2 slides into muscle memory. Watch for these as you build:
>
> - **The five words** — **workflow** (the YAML file that runs one or more jobs), **job** (a unit of work that gets its own fresh runner), **step** (one action within a job), **runner** (the clean machine that runs the job), and **trigger** (the `on:` block that decides when to run). Steps 1–2.
> - **Anatomy of a Workflow** (event → workflow → jobs → runner → steps) — Step 2 is this diagram, made real.
> - **Artifacts and job isolation** — files saved so output "outlives the runner"; the bridge from CI to deploy. Step 3.
> - **Caching** — the cache key as "a fingerprint of your dependencies." Step 4.
> - **Branch protection and the Pull Request Quality Gate** — "what gives CI teeth," and the funnel every change must pass through. Steps 5–6.

## What you will build

- A GitHub Actions workflow (`.github/workflows/ci.yml`) that checks out ShipIt, builds it, and runs the tests on every push and pull request.
- Published artifacts (the built app and the test results) so the run's output outlives the runner.
- Dependency caching so the second run is noticeably faster than the first.
- Branch protection on `main` that requires the CI check and a pull request.

## Success looks like

A pull request cannot merge to `main` unless the build and tests pass, artifacts appear on the run, and the cached run is visibly faster than the cold run.

## Prerequisites

- You finished Lab 0 and have the ShipIt starter repository on GitHub, cloned locally.
- Local tools: Git, the .NET 10 SDK. Verify with `dotnet --version` (should report 10.x).
- Commands below run in VS Code's integrated terminal, set to Git Bash (see Lab 0 if you need to switch your terminal profile).

---

## Step 1: Confirm the app builds locally

Before automating anything, make sure the build is green on your machine.

```bash
dotnet restore
dotnet build --configuration Release
dotnet test --configuration Release
```

You should see the solution build and the ShipIt.Tests suite pass. If this fails locally, fix it before moving on. CI cannot make a broken build pass; it just tells the truth faster.

> **Why:** These are the exact three commands your workflow will run. Proving them green locally first means that when you automate them, any failure you see is a failure of the *pipeline* (a bad path, a missing SDK step), not a failure of the *code*. Separating those two questions saves you a lot of confused debugging later.
>
> **From the slides:** CI does not have magic. It runs the same build you run — just on a clean, shared machine, automatically, on every change. "CI cannot make a broken build pass; it just tells the truth faster."

## Step 2: Create the workflow file

### Cleanup existing folders and files

* **Delete the exiting `.github` folder.** 
* **Delete the existing `Dockerfile`.**
* **Delete the existing `docker-compose.yml`.**


First, create a branch for this change: click the branch name in the blue status bar at the bottom-left of VS Code (it says `main`), choose **Create new branch...**, type `add-ci`, and press **Enter**.

Now create the workflow file. Click the **Explorer** icon at the top of the left Activity Bar, hover over the `shipit` root folder, and click the **New File** icon. Type the full path `.github/workflows/ci.yml` (with the slashes) and press **Enter** — VS Code creates the `.github` and `workflows` folders for you automatically. Paste this into the empty file that opens, then save (`Ctrl+S`):

```yaml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Check out the code
        uses: actions/checkout@v5

      - name: Set up .NET
        uses: actions/setup-dotnet@v5
        with:
          dotnet-version: '10.0.x'

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --configuration Release --no-restore

      - name: Test
        run: dotnet test --configuration Release --no-build --logger "trx;LogFileName=test-results.trx"
```

Commit and push it: click the **Source Control** icon in the Activity Bar, hover over **Changes** and click the **+** to stage `ci.yml`, type a commit message ("Add CI workflow") in the box at the top, click the checkmark (**Commit**), then click **Publish Branch**.

Open a pull request on GitHub.com: go to your repo, click the **Compare & pull request** button that appears for your just-pushed branch (or the **Pull requests** tab → **New pull request**).

**Check the base repository carefully before creating it.** Because your repo is a fork, GitHub defaults the base to the *original* `jruels/shipit` template, not your own fork — you'll see two dropdowns at the top of the page reading something like `base repository: jruels/shipit` ↔ `head repository: <you>/shipit`. Click the **base repository** dropdown and change it to **your own** `<you>/shipit`, so the comparison reads `base: <you>/shipit:main` ← `compare: <you>/shipit:add-ci` — both sides your own fork, not the upstream. If you skip this, you'll open a PR against the original template repo instead, where you have no merge rights, and you'll hit a review requirement you can't satisfy.

Once both sides point at your own repo, confirm the base branch is `main` and compare branch is `add-ci`, then click **Create pull request**. Watch the **Checks** tab on the PR: the `build-and-test` job should start, run each step in order, and finish green.

> **What you are seeing:** the `on:` block is your trigger, `build-and-test` is the job, `runs-on` is the runner, and each `- name:` is a step. This is the anatomy diagram from the slides, made real.

> **Why:** Map this file onto the **five words**. `on:` is the **trigger** — it answers *when*. `build-and-test` is a **job** — a named unit of work. `runs-on: ubuntu-latest` is the **runner** — the clean machine GitHub spins up for this job. Each `- name:` is a **step**, and steps in one job share a filesystem and run top to bottom: that is why Restore, Build, and Test can hand results to one another. The whole file is the **workflow**. That is the "Anatomy of a Workflow" diagram — event → workflow → jobs → runner → steps — running for real.
>
> **From the slides:** `pull_request` is **the heart of CI**. Push triggers protect the branch after the fact; `pull_request` runs the checks *before* the change is allowed in, so a broken change is caught at the door. The runner is "a clean machine that forgets everything the moment the job ends" — which is exactly why it is trustworthy: nothing lingers from a previous run to make a bad build look good.

## Step 3: Publish artifacts

Add two steps to the end of the job so the run keeps its output. Remember: two jobs do not share a filesystem, and the runner is deleted when the job ends, so anything you want to keep you upload on purpose.

```yaml
      - name: Publish the app
        run: dotnet publish src/ShipIt --configuration Release --no-build -o ./publish

      - name: Upload the published app
        uses: actions/upload-artifact@v4
        with:
          name: shipit-app
          path: ./publish

      - name: Upload the test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: "**/test-results.trx"
```

Save, then stage/commit/push the change from VS Code's **Source Control** panel (same **+**, commit message, **Commit**, **Sync Changes** flow as Step 2). On the completed run, scroll to the **Artifacts** section and confirm `shipit-app` and `test-results` are downloadable.

> **Why `if: always()`?** Without it, the test-results upload is skipped when the tests fail, which is exactly when you most want to see them. `always()` runs the step even after a failure.

> **Why:** The runner is torn down the moment the job ends, so any file that exists only on that machine is gone with it. An **artifact** is how output "outlives the runner" — you save it *on purpose*. That is not a formality: it is the **CI-to-deploy bridge**. Because "two jobs do not share files," the published app you upload here is exactly what a later deploy job (Lab 3) will download and ship. You are practicing the deliberate hand-off that job isolation forces on you.
>
> **From the slides:** `if: always()` is an **expression** making "a small decision" — run this step even after a failure. (In an `if:`, GitHub already evaluates the expression, so you write it bare — `if: always()`, not `if: ${{ always() }}`.) It exists so you "see results exactly when tests fail," which is the one moment the results matter most.

## Step 4: Add dependency caching

Add this step **before** the Restore step. It caches the NuGet package folder so restore is fast on later runs.

```yaml
      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: nuget-${{ runner.os }}-${{ hashFiles('**/*.csproj') }}
          restore-keys: |
            nuget-${{ runner.os }}-
```

Commit and push once (via Source Control) to populate the cache (a cache miss, so it is slow). Make a trivial change (for example a comment) and push again, then compare the two runs: the Restore step should drop from a minute or two to a few seconds, and the log will say `Cache restored from key: ...`.

> **The key is the whole game.** `hashFiles('**/*.csproj')` changes only when a project file changes, so you get a fresh restore when a dependency in the project file changes and a fast hit every other time. A caveat worth knowing: hashing the `.csproj` misses dependency changes that do not edit the project file (floating versions, or Central Package Management in `Directory.Packages.props`). For a rock-solid key, commit a `packages.lock.json` and hash that instead. `actions/setup-dotnet@v5` also has a built-in `cache: true` option that does this for you.

> **Why:** A fresh runner starts empty every time, so restore — downloading every dependency from scratch — is "usually the slowest step" in a .NET pipeline. Speed is not a vanity metric: "a fast pipeline is a safety feature," because a pipeline under ten minutes "keeps developers in flow" and a slow one tempts people to skip or route around it. Caching buys most of that speed back.
>
> **From the slides:** The cache key is "a fingerprint of your dependencies." Get the resolution right: too **broad** a key (say, one shared across every branch regardless of dependencies) goes stale and serves the wrong packages; too **narrow** a key (one that changes on every commit) never hits and you pay full price every run. `hashFiles('**/*.csproj')` aims for the middle — it changes exactly when your dependencies could have.

## Step 5: Protect `main`

A green check does nothing until a red one can stop a merge. On GitHub.com, go to your repo, click the **Settings** tab, then **Branches** in the left sidebar. Under **Branch protection rules**, click **Add branch protection rule** (or **Add rule**). In the **Branch name pattern** field, type `main`, then check:

- **Require a pull request before merging.** Checking this reveals a sub-option, **Require approvals**, switched on with a default of **1** — **uncheck it, or set the number to 0.** You're working solo right now, and GitHub will not let you approve your own pull request, so leaving this on locks you out of merging anything.
- **Require status checks to pass before merging**, and search for/select the `build-and-test` check (it only appears in this list after the workflow has run at least once — see Troubleshooting below if you don't see it).
- **Require branches to be up to date before merging.**
- **Do not allow bypassing the above settings** (near the bottom of the page). **This one matters more than it looks.** You are the owner of your own fork, and GitHub treats repo owners as admins — without this box checked, *you specifically* can still push straight to `main` and skip every rule above, no error, just a quiet "bypassed rule violations" note in the push output. This box is what actually makes the rule apply to you too, not just to hypothetical other collaborators.

Scroll down and click **Create** (or **Save changes**).

> **This is the fix for the Friday-hero anti-pattern.** Nobody, including you, can now push straight to `main`, and nothing merges with a red check — but only because you checked **"Do not allow bypassing the above settings."** Skip that box and the rule is real for everyone except you, which quietly defeats the whole lesson.

> **Why:** This is the step where CI "changes from a suggestion into a rule." Up to now the green check has been advisory — helpful, but skippable. **Branch protection is what gives CI teeth.** As the slides put it, "a green check that cannot block a bad merge is just decoration." Requiring the status check makes the machine's verdict binding; requiring "up to date before merging" makes sure a change is tested against the latest `main`, not a stale copy that passed in isolation.
>
> **From the slides:** A **pull request quality gate** means two conditions before code lands: "automated checks pass AND a human reviewer approves." You are wiring the automated half now; the reviewer half is a checkbox on the same page. Together they end the Friday-hero pattern where "one person owns every deploy" — the gate, not a person, now owns the merge.

## Step 6: Prove the gate works

Open a pull request that deliberately breaks a test, so you can watch the gate hold the line.

1. Create a new branch (VS Code status bar → **Create new branch...**), then open `tests/ShipIt.Tests/ShipmentApiTests.cs` in the Explorer and change an assertion so it fails (for example, expect the wrong shipment count).
2. Stage, commit, and push (Source Control panel), then open a PR into `main`.
3. Watch the `build-and-test` check go red. The **Merge** button is now blocked.
4. Fix the test, commit and push again, watch the check go green, and confirm the PR can merge.

Take a screenshot of the blocked merge. That red gate is the whole point of the module.

> **Why:** Everything before this was setup; this step is the payoff. You are watching the **Pull Request Quality Gate** work as a funnel — the choke point every change must pass through before it reaches `main`. "A red check blocks the merge, full stop." Seeing the Merge button actually disabled is what convinces a team that the gate is real and not decoration.
>
> **From the slides:** This is the "Pull Request Quality Gate" diagram made real. It grows in Module 4 (more checks, required reviewers, environments) but the shape is fixed here: a broken change stops at the door instead of reaching `main`.

---

## Success criteria

You are done when all of these are true:

- Opening or updating a pull request into `main` (and pushing to `main` itself) triggers the `build-and-test` job automatically.
- The run publishes `shipit-app` and `test-results` artifacts.
- A second run with no dependency change is visibly faster (cache hit in the log).
- Branch protection blocks a merge to `main` when the check is red, and allows it when green.

## Troubleshooting

- **Workflow does not run:** the file must be under `.github/workflows/` and be valid YAML. Check the **Actions** tab for a parse error. YAML is whitespace-sensitive; use spaces, not tabs.
- **`dotnet: command not found`:** the `setup-dotnet` step must come before any `dotnet` command.
- **`--no-build` errors on publish or test:** a previous step must have built the same `--configuration`. All build, test, and publish steps here use `Release`.
- **Cache never hits:** confirm the `key` is identical between runs and that `*.csproj` files did not change.
- **Cannot select the status check in branch protection:** the check name only appears after the workflow has run at least once. Run it, then add the rule.
- **"Review required" / you can't approve your own pull request:** first check the PR itself — GitHub defaults the base repository to the original `jruels/shipit` template when you open a PR from a fork, not your own fork (see the note in Step 2 above). If the PR's base says `jruels/shipit` instead of your own `<you>/shipit`, that's the problem: close this PR (you can't edit its base repository after the fact) and open a new one with **both** sides pointed at your own fork. If the base is correctly your own repo and you still see this, then it's your branch protection rule: go to **Settings → Branches**, edit the rule for `main`, uncheck **Require approvals** (or set it to 0), and save.
- **A direct push to `main` seemed to work even though you didn't go through a PR:** check whether you left **"Do not allow bypassing the above settings"** unchecked in Step 5. Without it, the rule doesn't apply to you as the repo owner — the push succeeds with a "bypassed rule violations" note instead of being rejected. Go back to **Settings → Branches**, edit the rule, and check that box.

## Stretch goals

- Split the job into two jobs, `build` and `test`, and pass the build output from `build` to `test` with `upload-artifact` / `download-artifact`. Notice how much you have to do by hand because jobs do not share a filesystem.

  > **Why:** This is **job isolation** you can feel. Each job gets "its own fresh runner," so the second job starts with nothing the first produced. When a small value is all you need, a **job output** is lighter than a full artifact; when you need files, you upload and download them. Either way, you learn why data between jobs must be passed *deliberately* — nothing crosses the boundary for free.

- Add a `workflow_dispatch` trigger so you can run CI manually from the Actions tab.

  > **Why:** `workflow_dispatch` is another **trigger** — a manual "run now" button — alongside `push`, `pull_request`, and `schedule`. Different triggers answer different needs for *when* a workflow should run.

- Add a CI status badge to the repository `README.md`.

- Add a `paths-ignore` filter so a change to `README.md` alone does not run the full build.

  > **Why:** This tunes the **trigger** so you "don't rebuild the world for a README typo." A fast, relevant pipeline is a safety feature; skipping work that cannot affect the build keeps runs under the ten-minute mark that keeps developers in flow.

---

## What this sets up

You now have a `ci.yml` that verifies every change and a protected `main` that enforces it. In **Lab 3** you extend this same workflow to package ShipIt as a container image and push it to Azure Container Registry, so CI ends with a real, deployable artifact instead of loose files. The caching and artifact habits you built here carry straight into that work.

---

## Appendix: the complete `ci.yml`

```yaml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Check out the code
        uses: actions/checkout@v5

      - name: Set up .NET
        uses: actions/setup-dotnet@v5
        with:
          dotnet-version: '10.0.x'

      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: nuget-${{ runner.os }}-${{ hashFiles('**/*.csproj') }}
          restore-keys: |
            nuget-${{ runner.os }}-

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --configuration Release --no-restore

      - name: Test
        run: dotnet test --configuration Release --no-build --logger "trx;LogFileName=test-results.trx"

      - name: Publish the app
        run: dotnet publish src/ShipIt --configuration Release --no-build -o ./publish

      - name: Upload the published app
        uses: actions/upload-artifact@v4
        with:
          name: shipit-app
          path: ./publish

      - name: Upload the test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: "**/test-results.trx"
```
