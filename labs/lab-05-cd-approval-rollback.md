# Lab 5: Build a CD Pipeline with an Approval Gate and Rollback

**Module:** 5 (CD Pipelines and Deployment Strategies)
**Time:** 45-60 minutes
**Format:** Hands-on (GitHub Actions + GitHub Environments + Azure OIDC)

---

## Why this lab

This is where the pipeline stops being about *building* and starts being about *shipping*. Day 1 ended with known-good, SHA-tagged ShipIt images sitting in Azure Container Registry. That is potential energy, not delivered software — a correct artifact nobody can use yet.

Continuous delivery is not a tool you install; it is **a set of decisions about where a human belongs in the release**. In this lab you make those decisions concrete. The same image the CI build produced is promoted to staging automatically, held before production until a human deliberately clicks "go," and rolled back on its own when a deploy fails a health check. You will authenticate to Azure with OIDC federation, so there is no long-lived cloud secret in the repository. By the end you will have watched the pipeline both *pause for you* and *heal itself* — the two halves of delivery.

Everything you build here is the direct fix for the Module 1 anti-pattern: "deploy by hand on a Friday." The whole point is that a repeatable, gated, self-correcting release replaces the heroics.

> **From the slides:** The promotion path is "build once → dev → staging → gate → prod." The single most important property is that **the same SHA-tagged artifact moves through environments unchanged — we never rebuild per environment.** The image you approve for production is byte-for-byte the image you tested in staging.

## Concepts this lab makes real

| Slide concept | Where you see it in this lab |
|---|---|
| **Continuous delivery vs continuous deployment** | Staging deploys on its own (deployment); production waits for one deliberate human click (delivery). |
| **Promotion / "build once, promote the same image"** | Every step deploys the *same* `$ACR.azurecr.io/shipit:$SHA` — never a rebuild. |
| **GitHub Environment** | `staging` and `production` — each "a named set of rules and values." |
| **Required reviewers / approval gate** | A setting on the `production` environment that *pauses the run* — not code in the workflow. |
| **OIDC federation** | Short-lived token from GitHub, trusted by Azure; the subject claim is the security boundary. |
| **Rolling update / blue-green / canary** | Container Apps shifts traffic to the new revision; rollback is redeploying the previous SHA. |
| **Health checks — liveness vs readiness** | You poll `/readyz` ("is it ready to serve real traffic") to decide pass/fail. |
| **Automated rollback to known-good SHA** | A failed `/readyz` poll — not a human noticing later — triggers redeploy of the exact previous image. |
| **Traceability** | The stretch goal closes the commit → PR → work item → deploy loop with `Fixes #142`. |

## What you will build

- OIDC federation from GitHub Actions to Azure, scoped per environment (no stored secret).
- Two GitHub Environments, `staging` and `production`, with a required reviewer on production.
- A `cd.yml` workflow that promotes the SHA-tagged image to staging automatically, then to production after approval.
- A post-deploy health check against `/readyz` that triggers an automatic rollback to the previous known-good SHA.

## Success looks like

Production does not deploy without an approval, a failed health check rolls back to the previous image automatically, and no long-lived Azure secret exists anywhere in the repo.

## Prerequisites

- You finished Lab 4: ShipIt builds, tests, scans, and pushes a SHA-tagged image to ACR.
- Your own resource group with two Container Apps targets, `shipit-staging` and `shipit-production` (each with external ingress on the app's port), and your own Entra app registration for the pipeline — all created by `./scripts/lab0-provision.sh` when you ran it yourself in Lab 0.
- Commands below run in VS Code's integrated terminal, set to Git Bash (see Lab 0 if you need to switch your terminal profile).

Set these once in your shell for the commands below (the same values from your Lab 0 script run):

```bash
export RG=rg-shipit
export ACR=<your-registry-name>           # your registry name from Lab 0, without .azurecr.io
export APP_ID=<pipeline-entra-app-client-id>   # AZURE_CLIENT_ID from Lab 0
GH_USER=$(gh api user --jq .login)         # derived automatically, same as the Lab 0 script
```

---

## Step 1: Configure OIDC federation (no stored secret)

Create one federated credential per environment, so a token minted for `staging` can never deploy to `production`. The `subject` is the security boundary, and it must match the **exact `sub` claim GitHub puts in the OIDC token**.

> **Why:** A secret you never store can never leak. Instead of pasting an Azure client secret into GitHub, you tell Azure to *trust a specific, short-lived token* that GitHub mints at runtime. The token lives for minutes and is scoped to exactly one environment. This is the "no long-lived cloud secret in the repo" promise made real.

> **From the slides:** Trace the **OIDC Federation** handshake — request → sign → trust via the federated credential → short-lived token. The **subject claim is the security boundary**: it is the one field Azure checks to decide whether this run is allowed in. Get it wrong and you see `AADSTS700213` (the subject-mismatch diagram).

> You already created these — the Lab 0 script (`./scripts/lab0-provision.sh`) creates all three federated credentials (`ref:refs/heads/main` for CI, plus `environment:staging` and `environment:production` for this lab) the moment your fork exists. Confirm they're there with `az ad app federated-credential list --id "$APP_ID"`, then skip straight to Step 2. The commands below show exactly what that script ran on your behalf, so you understand the mechanism even though you don't need to type it again.

> **Important — the subject is not `repo:<you>/shipit:...`.** GitHub now embeds immutable numeric IDs in the OIDC subject, e.g. `repo:<you>@<ownerId>/<repo>@<repoId>:environment:staging`. A credential using the old `repo:<you>/shipit:...` form will never match and `azure/login` fails with `AADSTS700213`. Read your repo's real prefix and build the subject from it:

```bash
SUB_PREFIX=$(gh api "repos/$GH_USER/shipit/actions/oidc/customization/sub" --jq .sub_claim_prefix)
echo "$SUB_PREFIX"   # e.g. repo:jsmith@3454137/shipit@1305062596

for ENV in staging production; do
  az ad app federated-credential create --id "$APP_ID" --parameters "{
    \"name\": \"shipit-$ENV\",
    \"issuer\": \"https://token.actions.githubusercontent.com\",
    \"subject\": \"$SUB_PREFIX:environment:$ENV\",
    \"audiences\": [\"api://AzureADTokenExchange\"]
  }"
done
```

> **Why derive the prefix instead of typing it?** GitHub's move to immutable numeric IDs (`@<ownerId>`, `@<repoId>`) means the subject is no longer a string you can guess from your username. Renaming the repo or the account no longer breaks the credential — but it also means a hardcoded `repo:you/shipit:...` subject silently never matches. Asking GitHub for `sub_claim_prefix` guarantees the credential matches the token GitHub will actually send.

The pipeline app also needs permission to update the Container Apps — the Lab 0 script already granted this (`Contributor` on `$RG`) along with each Container App's own registry pull auth. Confirm it's there:

```bash
SUB=$(az account show --query id -o tsv)
az role assignment list --assignee "$APP_ID" --scope "/subscriptions/$SUB/resourceGroups/$RG" -o table
```

> **What just happened:** Azure now trusts a short-lived token from your repo, but only for the exact environment named in the subject. There is no client secret to store or leak.

## Step 2: Create the GitHub Environments

On GitHub.com, go to your repo's **Settings → Environments**, click **New environment**, type `staging`, and click **Configure environment** (there's nothing to configure for staging, so that's it). Click **Environments** in the sidebar again, **New environment**, type `production`, and click **Configure environment**.

> **Why:** An environment is "a named set of rules and values." Creating one is how you attach policy — who must approve, which branch may deploy — to a deploy target *without writing any of it into the workflow*. The same `cd.yml` behaves differently against `staging` versus `production` purely because of what these environments say.

- On the **production** configuration page, check **Required reviewers**, add yourself in the box that appears, then click **Save protection rules**. This is the human gate. (Required reviewers work on **public** repositories for personal GitHub accounts, which is why your ShipIt repo is public.)
- You do **not** need to add any variables to the environments. `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`, `RG`, `ACR`, and `AKS` are all **repository** variables (set in Lab 0), so every job — including the no-environment CI job — can read them.
- Optionally, on the same **production** page, set **Deployment branches and tags** to **Selected branches and tags** and add `main`, so only `main` can deploy to it.

> **From the slides:** This is the human-belongs-here decision, made once, in a setting. The required reviewer is exactly the "a human clicks go for production" line that separates continuous *delivery* from continuous *deployment*.

> **What just happened:** the approval gate is a property of the `production` environment, not code in your workflow. A job only has to name the environment to inherit its rules.

## Step 3: Deploy to staging automatically

Create a branch for this lab's changes (VS Code status bar → branch name → **Create new branch...** → type `add-cd`). Then, in the Explorer, hover over `.github/workflows`, click the **New File** icon, and type `cd.yml`. It runs after CI succeeds on `main`, then promotes the image the CI build produced.

> **Why:** This is the friction-free half of delivery. There is no gate on staging on purpose — every green build on `main` should land in a real environment automatically, with zero clicks, so staging always reflects the latest tested code. Notice the job deploys `$ACR.azurecr.io/shipit:$SHA` using the SHA CI already built. It does not rebuild — it *promotes* the exact artifact, the first move on "The Promotion Path."

```yaml
name: CD
on:
  workflow_run:
    workflows: ["CI"]        # your CI workflow (built in Module 2, pushes SHA-tagged images since Module 3)
    types: [completed]
    branches: [main]

permissions:
  id-token: write            # required to request an OIDC token
  contents: read

jobs:
  deploy-staging:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    environment: staging
    env:
      SHA: ${{ github.event.workflow_run.head_sha }}
    steps:
      - uses: azure/login@v3
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - name: Deploy image to staging
        run: |
          az containerapp update -n shipit-staging -g "$RG" \
            --image "$ACR.azurecr.io/shipit:$SHA"
        env:
          RG: ${{ vars.RG }}
          ACR: ${{ vars.ACR }}
```

> **From the slides:** `environment: staging` and `id-token: write` together are the OIDC handshake in action — the job requests a token scoped to `staging`, and Azure trusts it because it matches the `staging` federated credential from Step 1. The token for `staging` cannot touch `production`.

## Step 4: Gate production behind an approval

Back in `cd.yml` (still open from Step 3), add a production job below `deploy-staging` that needs it and names the `production` environment. Capture the currently-live image first, so you can roll back to it.

> **Why:** This job *is* continuous delivery: fully automated right up to one deliberate human pause. Because you record the live image before touching anything, rollback becomes **nameable** — you are not guessing what "the previous version" was, you have its exact SHA-tagged reference saved. An untracked previous state is a rollback you can't actually perform.

```yaml
  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production        # run pauses here for the required reviewer
    env:
      SHA: ${{ github.event.workflow_run.head_sha }}
      RG: ${{ vars.RG }}
      ACR: ${{ vars.ACR }}
    steps:
      - uses: azure/login@v3
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - name: Record current image (for rollback)
        id: current
        run: |
          PREV=$(az containerapp show -n shipit-production -g "$RG" \
            --query "properties.template.containers[0].image" -o tsv)
          echo "prev=$PREV" >> "$GITHUB_OUTPUT"
      - name: Deploy new image
        run: |
          az containerapp update -n shipit-production -g "$RG" \
            --image "$ACR.azurecr.io/shipit:$SHA"
```

> **From the slides:** The pause at `environment: production` is the gate in "build once → dev → staging → **gate** → prod." Deploying `shipit:$SHA` — the same SHA that ran in staging — is "build once, promote the same image." No rebuild happens between staging and prod, so there is no chance the two environments run different bytes.

> **What just happened:** because `production` has a required reviewer, the run stops at this job and waits for a click. The previous image is saved as `steps.current.outputs.prev` before anything changes.

## Step 5: Add a health check and automatic rollback

Still in `cd.yml`, add two more steps to the end of the `deploy-production` job (after "Deploy new image"). Let the new revision's traffic settle, then require `/readyz` to be healthy for several **consecutive** checks; if it never is, redeploy the previous image and fail the run.

> **Why:** Deploying and hoping is not a strategy. A deploy is not "done" when the command returns — it is done when the new version is actually serving healthy traffic. The `/readyz` endpoint answers the question that matters: "is it ready to serve real traffic?" (as opposed to liveness `/healthz`, which only asks "is the process alive"). Making a failed poll *the trigger* for rollback means the pipeline catches a bad deploy in seconds, rather than a human noticing angry users an hour later.

```yaml
      - name: Health check
        id: health
        continue-on-error: true
        run: |
          FQDN=$(az containerapp show -n shipit-production -g "$RG" \
            --query "properties.configuration.ingress.fqdn" -o tsv)
          # Container Apps sets the new revision's traffic weight to 100 in config
          # almost immediately, but the *actual* data-plane traffic shift lags by up to
          # a minute or two. If we polled once and exited on the first 200, we could hit
          # the OLD (healthy) revision and wave a broken new deploy straight through.
          # So: let traffic settle, then require /readyz to be healthy for several
          # CONSECUTIVE checks. A bad revision returns 503 once it starts serving, which
          # resets the streak and fails the job -> triggering rollback.
          sleep 30
          ok=0
          for i in $(seq 1 18); do
            if curl -fsS "https://$FQDN/readyz" >/dev/null 2>&1; then
              ok=$((ok+1))
            else
              ok=0
            fi
            if [ "$ok" -ge 5 ]; then echo "sustained ready"; exit 0; fi
            sleep 5
          done
          echo "never became reliably ready"; exit 1
      - name: Roll back on failure
        if: steps.health.outcome == 'failure'
        run: |
          echo "::error::Deploy failed readiness. Rolling back to ${{ steps.current.outputs.prev }}"
          az containerapp update -n shipit-production -g "$RG" \
            --image "${{ steps.current.outputs.prev }}"
          exit 1
```

> **Why "sustained" readiness, not a single 200:** Container Apps performs a rolling-style handoff — the old revision keeps serving until the new one takes over, and the config traffic weight flips to the new revision *before* real traffic actually moves. If you probed `/readyz` once and trusted the first `200`, you would hit the *old, healthy* revision and wave a broken new deploy straight through (a subtle race that is easy to get wrong). Settling first and then requiring several consecutive healthy responses makes the check honest: once the bad revision is truly serving, its `503`s break the streak and the rollback fires.

> **From the slides:** This is the **Automated Rollback on Failure** diagram, executed. A failed check — not a human — is the trigger, and rollback is simply redeploying the *exact previous SHA* you saved in Step 4. This is the "flip back" idea from blue/green, expressed at the image level: the known-good version is one command away. (Canary — shifting a small percentage of traffic first, then ramping — is the next step beyond this course.)

> **What just happened:** the health check is the trigger, not a human noticing. It settles, then insists on *sustained* readiness before declaring success — otherwise it could test the old revision and wave a broken deploy through. On failure the job redeploys the exact previous image by tag, then exits non-zero so the run is clearly marked failed.

Finish editing `cd.yml` and save it, then in the **Source Control** panel stage it (**+**), commit ("Add CD pipeline"), and **Publish Branch**. Open a PR into `main` on GitHub.com, wait for the `build-and-test` and `CodeQL` checks to go green, then click **Merge pull request**.

## Step 6: Prove it, both ways

> **Why:** An untested rollback is a fire escape welded shut. It is not enough to *have* a rollback path — you have to watch it fire and restore health, or you don't actually know it works. Proving both the happy path (approve, deploy, healthy) and the failure path (bad deploy, auto-rollback, healthy again) is what turns "we have rollback" into "we have rollback that works."

1. That merge in Step 5 already triggered this once. Click the **Actions** tab at the top of your repo. CI runs first; CD only starts once CI finishes on `main` (it's triggered by CI completing, not by the push itself), so give it a few seconds after CI goes green, then a **CD** run appears in the list. Click into it — `deploy-staging` runs and finishes on its own, no click required.
2. On that same CD run page, `deploy-production` shows a yellow **Waiting** status. Click the **Review deployments** button (top-right of the job list). In the panel that opens, check the box next to **production**, optionally add a comment, then click **Approve and deploy**. Watch `deploy-production` run to completion.
3. Now force a bad deploy. Create a new branch named `test-bad-deploy`. Open `src/ShipIt/Program.cs` and find this line near the top:
   ```csharp
   static bool IsReady() =>
       !string.Equals(Environment.GetEnvironmentVariable("SHIPIT_READY"), "false", StringComparison.OrdinalIgnoreCase);
   ```
   Replace it with:
   ```csharp
   static bool IsReady() => false;
   ```
   This bakes "always NOT READY" directly into the image, so rolling back to the previous image actually restores health. (Don't instead set `SHIPIT_READY=false` as a Container App env var — env vars persist across an image rollback, so the app would stay unready even after the rollback "succeeds.")

   Now open `tests/ShipIt.Tests/StatusAndHealthTests.cs` and find the `Readyz_is_ready_by_default` test:
   ```csharp
   [Fact]
   public async Task Readyz_is_ready_by_default()
   {
       var client = _factory.CreateClient();
       var response = await client.GetAsync("/readyz");
       Assert.Equal(HttpStatusCode.OK, response.StatusCode);
   }
   ```
   Change the assertion to match the new behavior, since `/readyz` will now always return 503:
   ```csharp
   Assert.Equal(HttpStatusCode.ServiceUnavailable, response.StatusCode);
   ```
   Without this change CI's test job fails and never builds the image, so there's nothing bad to deploy. Stage both files, commit ("Force /readyz to always fail, to test rollback"), publish the branch, open a PR into `main`, wait for checks, and merge.
4. Back in the **Actions** tab, open the new CD run and approve `deploy-production` the same way as step 2. Watch the **Roll back on failure** step fire and restore the previous image, then confirm `/readyz` is healthy again.
5. **Revert the bad `/readyz` code now, before moving on.** Item 4's rollback fixed *production* (Container Apps is back on the last good image), but `main`'s actual source code still has the permanently-broken `/readyz` handler — the pipeline will keep building and trying to promote that broken code on every future merge. Create a new branch named `revert-bad-deploy`. In `src/ShipIt/Program.cs`, put `IsReady()` back to:
   ```csharp
   static bool IsReady() =>
       !string.Equals(Environment.GetEnvironmentVariable("SHIPIT_READY"), "false", StringComparison.OrdinalIgnoreCase);
   ```
   In `tests/ShipIt.Tests/StatusAndHealthTests.cs`, put the `Readyz_is_ready_by_default` assertion back to `Assert.Equal(HttpStatusCode.OK, response.StatusCode);`. Commit ("Revert forced /readyz failure"), push, open a PR, wait for checks, and merge. Confirm the resulting deploy is healthy without a rollback firing this time. Module 6 builds on top of whatever `main` looks like right now, so leaving it broken here breaks that lab too.

> **From the slides:** Step 3 bakes the failure *into the image* on purpose. That is the whole reason "build once, promote the same image" makes rollback trustworthy — because the fault travels with the image, redeploying the previous SHA genuinely removes the fault. A config-only failure (like an env var) would survive the rollback, which is exactly why the note steers you away from it.

## Success criteria

- Production never deploys without your approval.
- A failed readiness check automatically redeploys the previous known-good SHA.
- `git grep -i AZURE_CLIENT_SECRET` (and any password) returns nothing; auth is OIDC only.
- You can point at the run log and name which SHA was rolled back to, and why.

## Troubleshooting

- **`AADSTS700213` / no matching federated credential:** the presented OIDC `subject` does not match any federated credential. Confirm the credential's subject uses your repo's real prefix from `gh api "repos/$GH_USER/shipit/actions/oidc/customization/sub" --jq .sub_claim_prefix` (it embeds immutable numeric IDs) and that `environment:` in the job equals the environment in the subject string, exactly.
- **`Error: id-token: write` / login can't get a token:** you left out the workflow-level `permissions: id-token: write`.
- **Health check always fails:** confirm the app's ingress is external and the port matches ShipIt's listening port; test `curl https://$FQDN/readyz` by hand.
- **CD never triggers:** the `workflow_run` name must match your CI workflow's `name:` exactly, and CI must have run on `main`.

## Stretch goals

- Add a **wait timer** to production in addition to the reviewer.
- Post the deployed version and SHA to a Slack or Teams channel on success and on rollback.
- Reference a work item (for example `Fixes #142`) in the PR and surface it in the deploy summary, closing the traceability loop from the slides.

> **From the slides:** `Fixes #142` is the last link in the traceability chain — commit → PR → work item → deploy. When someone six months from now asks "why is this SHA in production?", that reference lets them walk backward from the running image to the exact decision that put it there.

## What this sets up

Your pipeline now promotes one image through environments with a real gate and a real safety net, but the deploy target is a single managed container. In **Module 6** you replace it with a genuine Kubernetes deployment: ShipIt runs on AKS via a Helm chart, gated by the same approval and using the same SHA-tagged image, and the `/healthz` and `/readyz` checks become native Kubernetes probes.

> **From the slides:** The manual traffic-weight polling and `curl` loop you wrote here are exactly what Kubernetes gives you for free — the single Container App is replaced by Kubernetes self-healing, and your `/healthz` and `/readyz` endpoints become native liveness and readiness probes the platform runs on its own.
