# Lab 7: Provision Infrastructure with Bicep and Add a Monitoring Alert

**Module:** 7 (Infrastructure as Code and Monitoring)
**Time:** ~60 minutes
**Format:** Hands-on (Bicep + GitHub Actions + Application Insights)

---

## Why this lab

Through Labs 1 through 6 you built a real delivery pipeline: a commit triggers CI, an image lands in ACR, a gated promotion ships it to AKS with Helm. But two honest gaps remain, and this lab exists to close both.

**The first gap is how your infrastructure got there.** Every Azure resource you have used so far was created by the Lab 0 script running individual, imperative `az` CLI commands — one resource at a time, with no file describing the end state you want. That is still a version of *click-ops*, just scripted instead of clicked: nobody can review a shell command the way they review a diff, nobody can be certain re-running it changes only what should change, and when someone quietly changes a setting six weeks from now, nothing records it — that silent divergence is *configuration drift*. Infrastructure as Code fixes this by treating cloud resources like software: repeatable, reviewable, and version-controlled.

**The second gap is that the app ships with no one watching.** You have been deploying blind. Monitoring fixes that — and it does something bigger. It *closes the loop* this course opened with. In Lab 1 you set the four DORA metrics as a scoreboard you could only estimate. Monitoring is what finally lets you *measure* them against real numbers. This is where Day 1 and Day 2 meet.

In this lab you will provision a Log Analytics workspace and Application Insights with Bicep, deployed through the same OIDC-gated pipeline you already trust; instrument ShipIt so its live traffic becomes telemetry; build a small dashboard; and add an alert that fires on a bad deploy and clears on rollback. By the end you can point at real data and say exactly how each of the four DORA metrics is measured.

> **From the slides:** Keep the diagram **"The Full Pipeline, End to End"** in mind — commit → CI → ACR → gated promotion → Helm/AKS → Bicep infra → monitoring, with a dashed line looping back to DORA. This lab builds the last two boxes and draws that dashed line.

## Concepts this lab makes real

This lab is where the abstract ideas from the Module 7 slides become things you can run, break, and measure:

- **Infrastructure as Code vs. click-ops** — you replace hand-built resources with one reviewable file.
- **Declarative and desired state** — you declare the end result you want, not the click-by-click steps to get there (the same idea behind Kubernetes manifests).
- **Idempotence** — running the deploy again changes nothing if nothing has changed.
- **Configuration drift** — and why a re-apply is how you reconcile it.
- **Bicep and ARM** — Bicep is readable ARM; it transpiles to ARM JSON, and there is no state file to manage because Azure itself holds the current state.
- **Parameters, variables, and outputs** — arguments in, local computations, return values out; the output here is the *glue* to monitoring.
- **Observability: metrics, logs, and traces** — the count, the record, and the request's journey — flowing through Azure Monitor, Application Insights, and a Log Analytics workspace.
- **P95, not the average** — because the average hides the slow tail that users actually feel.
- **Alerts, action groups, and alert fatigue** — alerting on symptoms users feel, without training people to ignore alerts.
- **The four DORA metrics + MTTR** — and *closing the loop*: measure, change, measure again.

## What you will build

- A Bicep template under `infra/` for a Log Analytics workspace and Application Insights, with the connection string as an output.
- A pipeline job that deploys the Bicep with OIDC and captures the output.
- ShipIt instrumented with Application Insights, the connection string injected as a Secret.
- A dashboard and a metric alert with an action group.

## Success looks like

The Bicep deploy is idempotent (re-running changes nothing), telemetry flows from the AKS-hosted app, an alert fires on a bad deploy and clears on rollback, and you can state how each DORA metric is measured from this data.

## Prerequisites

- You finished Lab 6: ShipIt runs on AKS via Helm from the pipeline.
- The pipeline's OIDC login to Azure works (Lab 5), with `Contributor` on the resource group.
- Commands below run in VS Code's integrated terminal, set to Git Bash (see Lab 0 if you need to switch your terminal profile).

```bash
export RG=rg-shipit
```

---

## Step 1: Write the Bicep

Create a branch for this lab's changes (VS Code status bar → branch name → **Create new branch...** → type `add-monitoring`). Then, in the Explorer, hover over the `shipit` root folder, click the **New File** icon, and type `infra/main.bicep` (with the slash) — VS Code creates the `infra` folder for you. Paste in:

```bicep
param location string = resourceGroup().location
param env string

resource logs 'Microsoft.OperationalInsights/workspaces@2023-09-01' = {
  name: 'shipit-logs-${env}'
  location: location
  properties: {
    retentionInDays: 30
    sku: { name: 'PerGB2018' }
  }
}

resource appi 'Microsoft.Insights/components@2020-02-02' = {
  name: 'shipit-appi-${env}'
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
    WorkspaceResourceId: logs.id      // workspace-based App Insights
  }
}

output connectionString string = appi.properties.ConnectionString
```

> **Why:** Notice what this file does *not* say. It never lists the steps to create a workspace and then wire Application Insights to it. It simply *declares the end result you want* — two resources that exist and are connected — and lets Azure figure out how to get there. That is declarative, desired-state thinking, the same mental model as a Kubernetes manifest. The `WorkspaceResourceId: logs.id` line is where App Insights is told to send its data *into* the Log Analytics workspace; that workspace is the store your telemetry will live in. And because there is no state file to babysit — Azure itself holds the current state — this one file is the whole source of truth.

> **From the slides:** `param` values are the arguments, and the `output` is the return value. That `connectionString` output is not just tidy — it is the **glue to monitoring**. Step 3 hands it straight to the app. One readable file, reviewable in a pull request like any other code — the cure for click-ops.

> **What just happened:** one readable file declares both resources and hands back the connection string ShipIt needs. It is reviewable in a pull request like any other code.

## Step 2: Deploy the Bicep from the pipeline

Open `.github/workflows/cd.yml` in the Explorer. Add a new top-level `infra` job — at the same indent level as `deploy-staging` and `deploy-production`, not nested inside either one (same OIDC login as Module 5). Capture the connection string output for the next step.

```yaml
  infra:
    runs-on: ubuntu-latest
    environment: production
    outputs:
      appiConn: ${{ steps.deploy.outputs.appiConn }}
    steps:
      - uses: actions/checkout@v5
      - uses: azure/login@v3
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - id: deploy
        run: |
          CONN=$(az deployment group create -g "$RG" \
            --template-file infra/main.bicep \
            --parameters env=prod \
            --query "properties.outputs.connectionString.value" -o tsv)
          echo "appiConn=$CONN" >> "$GITHUB_OUTPUT"
        env:
          RG: ${{ vars.RG }}
```

Now ship it: save both files, then in **Source Control** stage `infra/main.bicep` and `cd.yml` (**+**), commit ("Add Bicep infra job"), and **Publish Branch**. Open a PR into `main` on GitHub.com, wait for checks to go green, and **Merge pull request** — that's the first run. For the second run, go to the **Actions** tab, open that same CD run, and click **Re-run all jobs**. The second run reports no changes: that is idempotence.

> **Why:** Your infrastructure now goes through the *exact same* OIDC-gated pipeline as your application code — same login, same `production` environment gate, same audit trail. There is no separate, privileged, hand-driven path for infra; it is reviewed and promoted like everything else. The two-run instruction is not busywork: the first run creates the resources, and the second run proves *idempotence* — because Bicep declares desired state, re-applying an unchanged template changes nothing. That is exactly the property that makes IaC safe to run on every merge.

> **From the slides:** This job is the "Bicep infra" box in **"The Full Pipeline, End to End."** With it in place, the pipeline provisions its own foundation instead of depending on resources someone clicked into existence.

## Step 3: Instrument ShipIt

In your VS Code Git Bash terminal, from the repo root:

```bash
dotnet add src/ShipIt package Microsoft.ApplicationInsights.AspNetCore --version 2.22.0
```

**Pin the version.** The current default (3.x) pulls in a newer OpenTelemetry-based exporter chain (`Azure.Monitor.OpenTelemetry.Exporter` → `System.ClientModel`) that has failed to restore inside the Module 3 Dockerfile's `dotnet publish` step on some CI runners (`NETSDK1064: Package ... was not found`) even when it restores fine locally. `2.22.0` is the last release before that rewrite and avoids the issue.

Open `src/ShipIt/Program.cs` in the Explorer and add this right after the `var builder = WebApplication.CreateBuilder(args);` line, before `builder.Services.AddSingleton<ShipmentStore>();`:

```csharp
if (!string.IsNullOrEmpty(builder.Configuration["APPLICATIONINSIGHTS_CONNECTION_STRING"]))
{
    builder.Services.AddApplicationInsightsTelemetry();
}
```

**Guard it, don't call it unconditionally.** `AddApplicationInsightsTelemetry()` throws `InvalidOperationException: A connection string was not found` at host startup if no connection string is configured — and the test project's `WebApplicationFactory<Program>` boots this same `Program.cs` with no `APPLICATIONINSIGHTS_CONNECTION_STRING` set, since CI's `dotnet test` step runs before this Secret exists anywhere. Without the guard, all of Lab 2's tests fail the instant this line is added, `build-and-test` goes red, and the PR can never merge under branch protection. The guard makes the line a no-op in test/CI and real in the deployed pod, where the Secret you create below actually sets that variable.

The SDK reads `APPLICATIONINSIGHTS_CONNECTION_STRING` from configuration. Inject it into the AKS deployment as a Secret, sourced from the Bicep output (do not bake it into the image). The pipeline captured this string as a job output in Step 2; to run the `kubectl` command by hand, first fetch it into your shell from the deployed App Insights resource:

```bash
export RG=rg-shipit   # the fixed resource-group name from Lab 0
CONN=$(az monitor app-insights component show \
  -g "$RG" -a shipit-appi-prod --query connectionString -o tsv)

kubectl create secret generic shipit-appi \
  --from-literal=APPLICATIONINSIGHTS_CONNECTION_STRING="$CONN"
```

Reference the Secret in the Helm chart's deployment: open `charts/shipit/templates/deployment.yaml` (from Lab 6) and add a line to the `envFrom` list you already have there:

```yaml
          envFrom:
            - configMapRef: { name: shipit-config }
            - secretRef:    { name: shipit-appi }
```

Save both files, then commit and push: in **Source Control**, stage `Program.cs`, `ShipIt.csproj` (changed by the `dotnet add package` command), and `deployment.yaml` (**+**), commit ("Instrument ShipIt with App Insights"), and **Publish Branch**. Open a PR into `main`, wait for checks to go green, and **Merge pull request** — this redeploys ShipIt with telemetry wired in.

> **Why:** This is the moment observability becomes real — one SDK line turns every request the app handles into metrics, logs, and traces. But look at *where the connection string comes from*: it is the Bicep `output` from Step 2, flowing through the pipeline into a Kubernetes Secret. Infrastructure and monitoring are now wired together by the same value, not by copy-paste. And it lives in a Secret sourced from config, never baked into the image — that is the Module 5 rule holding the line: secrets come from configuration at deploy time, so the same image is safe to promote across environments.

> **From the slides:** Outputs are the glue to monitoring — here you see the glue actually hold. The connection string is the return value of one system (Bicep) becoming the argument to another (the app).

> **What just happened:** one line of code plus a connection string. The string came from the Bicep output, so infrastructure and monitoring are wired together, and it lives in a Secret, never in the image.

## Step 4: Generate traffic and confirm telemetry

```bash
FQDN=$(kubectl get service shipit -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
for i in $(seq 1 200); do curl -s "http://$FQDN/" >/dev/null; done
```

In the portal, open the Application Insights resource. Within a couple of minutes you should see request count, response times, and any failures for ShipIt.

> **Why:** Until now the telemetry plumbing was theoretical. This loop drives real traffic so you can watch observability made real: request *count* (a metric), the per-request *record* (logs), and each request's *journey* through the app (traces) — the three pillars, arriving from a live, AKS-hosted deployment rather than from your laptop. If nothing shows up, the wiring from Step 3 is where to look before anything else.

## Step 5: Build a small dashboard *(portal UI)*

This step is done entirely in the Azure portal — there is no CLI for building a dashboard.

1. Go to [portal.azure.com](https://portal.azure.com), search for your `shipit-appi-prod` Application Insights resource, and open it.
2. In its left sidebar, click **Metrics**. For each of the three charts below: pick the metric from the dropdown, set the aggregation, then click **Pin to dashboard** (top-right of the chart) → **Create new** (name it something like `ShipIt Prod`) or pin to an existing dashboard:
   - Request rate (metric: **Server requests**, aggregation: **Count**)
   - Failure rate (metric: **Failed requests**, aggregation: **Count**)
   - P95 server response time (metric: **Server response time**, aggregation: **95th percentile**, not Average)
3. Go to **Dashboard** in the top Azure portal menu to see your pinned charts together.

Keep it to what you would actually watch during a deploy. Add a release annotation (in Application Insights, left sidebar → **Usage → Events**, or the **Release Annotations** setting under **Configure**) so deploys show as markers on the charts.

> **Why:** The instruction to chart **P95, not the average**, is the whole point. The average is comforting and misleading — it hides the slow tail that your unhappiest users actually feel. P95 says "95% of requests were at least this fast," which surfaces the pain the average smooths over. The deploy markers matter just as much: a chart without them shows you *that* something changed; a marker lets you answer the real question — "did *that release* move this?"

> **From the slides:** This dashboard is the raw material for the diagram **"Correlating a Deploy with a Performance Change"** — the spike after Deploy v2 and the recovery after rollback. You are building the instrument that makes that story readable. Keep it lean: every extra chart you would not actually watch during a deploy is a step toward noise.

## Step 6: Add an alert with an action group

```bash
# the app-insights commands below live in an extension; add it once (az will
# otherwise prompt to auto-install it on first use):
az extension add --name application-insights --upgrade

# who gets notified (receiver format is: --action email <NAME> <EMAIL_ADDRESS>)
az monitor action-group create -g "$RG" -n shipit-oncall \
  --short-name shipit \
  --action email you you@example.com

# alert when the server-error rate crosses a threshold
APPI=$(az monitor app-insights component show -g "$RG" -a shipit-appi-prod --query id -o tsv)
az monitor metrics alert create -g "$RG" -n shipit-failures \
  --scopes "$APPI" \
  --condition "count requests/failed > 10" \
  --window-size 5m --evaluation-frequency 1m \
  --action shipit-oncall \
  --description "ShipIt server failures elevated"
```

> **Why:** An *action group* is the who-and-how of notification — it separates *what is wrong* (the alert rule) from *who to tell and how* (email, SMS, webhook), so you can reuse the same recipients across many alerts. Notice what this alert fires on: failed requests, a symptom your users actually feel — not CPU percentage or some internal counter nobody experiences. That distinction is your defense against *alert fatigue*: every noisy, low-signal alert trains people to ignore the next one, so by the time a real incident fires, the alert has already been mentally muted. Alert on symptoms, keep the threshold meaningful, and each page stays worth reading.

## Step 7: Break it, watch the alert, roll back, recover

1. Deploy a deliberately failing version (for example set `SHIPIT_READY=false` or point at a broken build) and drive traffic.
2. Watch the failure metric climb and the alert fire to your action group.
3. Roll back using the Lab 5 logic (`helm rollback shipit`, or the pipeline's automatic rollback).
4. Watch the failure rate return to baseline and the alert resolve.
5. Map what you saw: the spike is **change failure rate**; the time from alert to recovery is **time to restore (MTTR)**.

> **Why:** This is the payoff — two DORA metrics stop being definitions and become things you watched happen. The bad deploy that spiked the failure rate *is* change failure rate. The stretch from "alert fired" to "baseline restored" *is* MTTR. You did not read those numbers in a report; you produced them, on purpose, and recovered from them.

> **From the slides:** What you just watched is the diagram **"Correlating a Deploy with a Performance Change"**, playing out live: the spike after the bad deploy, the recovery after rollback. The dashboard markers from Step 5 are what let you tie the spike to a specific release and the recovery to a specific rollback — the correlation is the evidence.

## Success criteria

- Re-running the Bicep deploy reports no changes (idempotence).
- Telemetry flows from the AKS-hosted ShipIt into Application Insights.
- The alert fires on the bad deploy and clears on rollback.
- You can state, for each DORA metric, where the number comes from: deployment frequency (pipeline runs), lead time (commit to deploy), change failure rate (failed deploys and post-release alerts), time to restore (alert to recovery).

## Troubleshooting

- **No telemetry:** confirm the Secret is mounted (`kubectl describe pod` shows the env var) and the connection string is the full string, not just the instrumentation key.
- **Bicep deploy fails on the workspace name:** names must be unique within the resource group; check the `env` parameter.
- **Alert never fires:** your threshold or window may be too high or too short; lower the threshold for the demo and drive more traffic.
- **Second deploy shows changes you did not make:** something outside Bicep edited the resource (drift); that is exactly what IaC is meant to prevent, so re-apply to reconcile. (Application Insights specifically will always show a small `Flow_Type`/`Request_Source` diff on `what-if` — Azure sets those server-side and the template doesn't declare them. Harmless; a real re-apply changes nothing functionally.)
- **CI fails at `dotnet test` with `InvalidOperationException: A connection string was not found` right after Step 3:** you added `AddApplicationInsightsTelemetry()` without the guard shown above. Add the `if (!string.IsNullOrEmpty(...))` check.
- **Pipeline's `dotnet publish` (inside the Docker build) fails with `NETSDK1064: Package ... was not found`, but the restore step just before it succeeded:** you're on `Microsoft.ApplicationInsights.AspNetCore` 3.x instead of the pinned `2.22.0`, or the Module 3 Dockerfile still has `--no-restore` on the publish step. Fix whichever one changed back.
- **Alert fires and stays "New," never resolves, even after you stop your own test traffic:** if your Container App's IP is public, background internet scanners will keep generating some 404s on their own — that's expected noise on any public endpoint, not a bug in the alert.

## Stretch goals

- Refactor the Bicep into modules (a `monitoring` module called from `main.bicep`).
- Add a `what-if` step to the pipeline that prints the planned changes before deploying.
- Add an availability test that pings `/healthz` from outside Azure.
- Overlay your Lab 1 DORA targets against the measured numbers and note the gap.

> **Why:** These are not filler — they map directly to two more slide concepts. *Modules* are to Bicep what functions are to code: the cure for copy-paste, so the day you provision a second environment you reuse the `monitoring` module instead of duplicating it. And `what-if` is your plan step — it shows *exactly* what a deploy would change before it changes anything, so you never accidentally delete a database on a Friday afternoon. The last stretch goal literally overlays the Lab 1 scoreboard against reality: the gap you measure there is the loop closing.

## What this closes

This is the last lab. You started the course with a scoreboard, the four DORA metrics, that you could only estimate. You now collect the telemetry that measures every one of them, on an app that flows from a single commit to a monitored production deployment on its own. That is the DevOps loop, closed: measure, improve one thing, measure again.

> **From the slides:** This is the dashed line in **"The Full Pipeline, End to End"** — the arrow that loops from monitoring back to the DORA scoreboard. You opened the course estimating those four numbers; you close it measuring them. That round trip — measure, change, measure again — is the whole discipline in one motion.
