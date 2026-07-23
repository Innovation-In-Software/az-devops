# Lab 6: Deploy ShipIt to AKS with Helm

**Module:** 6 (Deploying to Kubernetes and OpenShift)
**Time:** ~60 minutes
**Format:** Hands-on (Helm + AKS + GitHub Actions)

---

## Why this lab

This is where "deploy" stops being a word on a slide and becomes real. Back in Module 5, "deploy to staging" was a single box in the pipeline diagram, and behind that box was one container running on one host. That is a single point of failure: if the host dies, the process dies, and someone gets paged to restart it by hand.

Kubernetes changes the contract. Instead of *starting* a container and *watching* it, you **declare the desired state** — "run three healthy copies of ShipIt, route traffic only to the ones that are ready" — and the cluster's controllers continuously **reconcile reality to that spec**. A pod that dies is restarted or rescheduled automatically, with no human in the loop. That is what "K8s keeps the right number of healthy copies running" actually means in practice.

Helm is the second half of the story. Without it, each environment needs its own hand-maintained YAML, and those copies **drift apart** over time. Helm packages your Deployment, Service, and ConfigMap into **one versioned chart** and lets you swap **values** per environment — so you build once and configure per environment, instead of copy-pasting manifests.

In this lab you package ShipIt as a Helm chart, deploy it to Azure Kubernetes Service (AKS) by hand to see reconciliation work, then wire that deploy into the CD pipeline behind the same Module 5 approval gate. You will change configuration with no rebuild, deliberately break a pod, and diagnose it with `kubectl` — the exact skill you will use on your first day running anything on Kubernetes.

## Concepts this lab makes real

| Slide concept | Where you see it in this lab |
| --- | --- |
| **Declarative desired state / reconciliation** ("you say what you want, it figures out how") | Every `helm upgrade` submits a spec; the cluster reconciles toward it (Steps 3–7). |
| **Self-healing** ("a pod that dies is restarted or rescheduled") | The Deployment keeps the replica count whole when you break or scale pods (Steps 6–7). |
| **Pod / Deployment / Service / ConfigMap** | You template each one in Step 2; they trace the "Ingress → service → deployment → pods" request path. |
| **ClusterIP vs LoadBalancer** ("the moving-target problem") | The Service in Step 2 gives pods one stable address that load-balances across live replicas. |
| **Liveness vs readiness probes** | `/healthz`→liveness, `/readyz`→readiness in Step 2; readiness gates the rolling update in Step 7. |
| **ConfigMap: build-once-configure-per-environment** | ShipIt reads region and banner from env in Steps 2 and 5 — config changes with no rebuild. |
| **Helm chart / values / install / upgrade / rollback** | The chart is scaffolded in Step 1; `upgrade`/`rollback` are first-class in Steps 3–6. |
| **Rolling update, gated by readiness** | New pods take traffic only after `/readyz` passes (Step 7). |
| **Status decoder ring** (`ImagePullBackOff` / `CrashLoopBackOff` / `Pending`) | You cause and diagnose a failure in Step 6. |

## What you will build

- A Helm chart under `charts/shipit/`: a templated Deployment, a Service, and a ConfigMap.
- Readiness and liveness probes wired to `/readyz` and `/healthz`.
- A pipeline job that deploys the SHA-tagged image to AKS with `helm upgrade`, using OIDC.

## Success looks like

A merge to `main` deploys ShipIt to AKS through the approval gate, the status page is reachable and reflects environment config, a rolling update stays available, and you can explain why a broken pod failed.

## Prerequisites

- You finished Lab 5: a gated CD pipeline with OIDC working.
- Your own AKS cluster and ACR from Lab 0 — the Lab 0 script attached your registry to your cluster at creation time (`--attach-acr`), so `AcrPull` is already granted; there's no separate attach step to run here.
- `kubectl` and `helm` installed locally.
- Commands below run in VS Code's integrated terminal, set to Git Bash (see Lab 0 if you need to switch your terminal profile).

First, set your values — `RG` and `AKS` are fixed names from Lab 0, and `ACR` is the registry name your Lab 0 script printed:

```bash
export RG=rg-shipit
export ACR=<your-registry-name>   # from your Lab 0 script run, e.g. shipitjrsacr
export AKS=shipit-aks
```

> **Lost the `ACR` value from Lab 0?** `RG` (`rg-shipit`) and `AKS` (`shipit-aks`) are fixed names, so there's nothing to look up for those — but your registry name is unique to you. Recover it one of these ways:
> - **Fastest:** on GitHub.com, go to your repo's **Settings → Secrets and variables → Actions → Variables** tab. The Lab 0 script already set `ACR` there as a repository variable (along with `RG`, `AKS`, and the three `AZURE_*` values) — read the value directly off that page.
> - **Azure CLI:** `az acr list -g rg-shipit --query "[].name" -o tsv` prints your registry's name (there's only one in your resource group).
> - **Azure portal:** go to [portal.azure.com](https://portal.azure.com), open your `rg-shipit` resource group, and find the resource with type **Container registry** in the list — its name is shown right there, and again at the top of its **Overview** page if you click into it.

Then point `kubectl` at your cluster (this pulls the cluster's credentials into your kubeconfig):

```bash
az aks get-credentials -g "$RG" -n "$AKS"
```

---

## Step 1: Scaffold the Helm chart

> **Why:** A Helm chart is the single versioned package that replaces the three hand-maintained YAML files (Deployment, Service, ConfigMap) that otherwise drift apart across environments. `helm create` gives you a working skeleton fast; you then strip out the pieces ShipIt does not need so `helm upgrade` renders cleanly. Everything from here forward changes *this one artifact*, not three files in three places.
>
> **From the slides:** This is the "What Helm Gives You" diagram — a chart plus values. You are building the "chart" half now; the "values" half comes in the next steps.

```bash
mkdir -p charts                 # helm create needs the parent directory to exist
helm create charts/shipit
# remove the scaffolded templates we do not use, so helm upgrade renders cleanly.
# service.yaml is removed here too; we add a simple one of our own in Step 2 (the
# scaffolded service.yaml references values we are about to delete and would fail to render).
# httproute.yaml is a Gateway API template some Helm versions scaffold by default;
# it references values we are about to delete too and will crash `helm template`/
# `helm upgrade` with a nil-pointer error if you leave it in.
rm charts/shipit/templates/serviceaccount.yaml \
   charts/shipit/templates/ingress.yaml \
   charts/shipit/templates/hpa.yaml \
   charts/shipit/templates/service.yaml \
   charts/shipit/templates/httproute.yaml \
   charts/shipit/templates/NOTES.txt
rm -rf charts/shipit/templates/tests
```

In the Explorer, open `charts/shipit/values.yaml` (already created by `helm create`), select all (`Ctrl+A`), and replace it with just what ShipIt needs:

```yaml
image:
  repo: <your-acr>.azurecr.io/shipit   # your registry from Lab 0
  tag: latest                          # the pipeline overrides this with the commit SHA
replicas: 3
config:
  region: eastus
  bannerColor: green
```

Notice what lives in `values.yaml`: the image tag, the replica count, and the per-environment config. These are the knobs you turn without editing a template. The `tag: latest` here is only a default — the pipeline (and your manual deploys) will override it with the exact commit SHA, so every deploy is traceable to a build.

## Step 2: Template the Deployment, Service, and ConfigMap

> **Why:** Two ideas become concrete here. First, the image tag is now a **value** (`{{ .Values.image.tag }}`), not a hard-coded string — the same template deploys any build to any environment. Second, the **probes** are how you tell Kubernetes the difference between "the process is alive" and "the process is ready to serve traffic." That distinction is not academic: the readiness probe is exactly what gates the rolling update you run in Step 7. The ConfigMap is build-once-configure-per-environment made real — ShipIt reads `region` and `bannerColor` from the environment (the Module 3 idea), so the same image behaves differently per environment with zero rebuild.
>
> **From the slides:** This step builds the middle and end of the "Ingress → service → deployment → pods" request path. The probes map straight to the slide rule: "Liveness: is the process alive? if it fails, restart. Readiness: is it ready to serve? if it fails, hold traffic back." ShipIt maps `/healthz`→liveness and `/readyz`→readiness.

Open `charts/shipit/templates/deployment.yaml` (already scaffolded by `helm create`), select all, and replace it with (probes included from the start):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shipit
spec:
  replicas: {{ .Values.replicas }}
  selector: { matchLabels: { app: shipit } }
  template:
    metadata:
      labels: { app: shipit }
      annotations:
        # Roll the pods whenever the ConfigMap changes. Without this, editing config
        # (e.g. bannerColor) updates the ConfigMap but the running pods keep their old
        # env values until something else restarts them, so the change appears to do nothing.
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      containers:
        - name: shipit
          image: "{{ .Values.image.repo }}:{{ .Values.image.tag }}"
          ports: [ { containerPort: 8080 } ]
          envFrom:
            - configMapRef: { name: shipit-config }
          readinessProbe:
            httpGet: { path: /readyz, port: 8080 }
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 10
            periodSeconds: 10
```

The `Deployment` is the object that "declares the desired replica count and update strategy." You never create pods directly — you tell the Deployment how many you want, and its controller reconciles the world to match. The `checksum/config` annotation is a deliberate trick: it forces a rollout whenever the ConfigMap changes, which is what makes Step 5's no-rebuild config change actually reach the running pods.

This one doesn't exist yet — in the Explorer, hover over `charts/shipit/templates`, click the **New File** icon, and create `configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: shipit-config
data:
  SHIPIT_REGION: "{{ .Values.config.region }}"
  SHIPIT_BANNER_COLOR: "{{ .Values.config.bannerColor }}"
```

A ConfigMap holds **non-sensitive** config as environment variables. (For credentials you would reach for a Secret instead — remember the slide caveat that a Secret is only base64-encoded, not encrypted at rest by default.) Region and banner color are safe to keep in plain sight, which is exactly why they belong here.

You deleted the scaffolded `service.yaml` in Step 1, so add a simple one of your own: in the Explorer, hover over `charts/shipit/templates`, click the **New File** icon, and create `service.yaml`. We use `type: LoadBalancer` so the status page is reachable from your laptop in Steps 3 and 5 (a `LoadBalancer` gives the Service an external IP; if your Lab 0 set up an ingress, you may use `ClusterIP` + that ingress instead):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: shipit
spec:
  type: LoadBalancer        # external IP for the lab; use ClusterIP + your Lab 0 ingress instead if you prefer
  selector: { app: shipit }
  ports:
    - { port: 80, targetPort: 8080, protocol: TCP, name: http }
```

The Service solves "the moving-target problem": pods come and go, and their IPs change every time. The Service is a **stable address that load-balances across whichever pods are currently live and ready**. `ClusterIP` gives you that stable address *inside* the cluster; `LoadBalancer` extends it to an external IP so you can hit the status page from your laptop. The `selector: { app: shipit }` is the glue — it is how the Service finds the pods the Deployment created.

> **What just happened:** the image tag and config are now values, not hard-coded. The probes tell Kubernetes when a pod is alive versus ready to serve.

## Step 3: Deploy it by hand first

> **Why:** Before you trust the pipeline to do this, you watch reconciliation happen with your own eyes. `helm upgrade --install` submits your desired state; the cluster figures out how to get there. The `--wait` flag is the important part: it **blocks until the pods are actually Ready**, so a bad deploy fails *here*, loudly, instead of limping along half-broken. That is readiness-as-a-gate, and it is the exact behavior the pipeline will lean on in Step 4.
>
> **From the slides:** This is the "Ingress → service → deployment → pods" path lighting up end to end for the first time — a request reaches the Service, which load-balances to a pod the Deployment is keeping alive.

```bash
az acr login -n "$ACR"
SHA=$(git rev-parse HEAD)   # full commit SHA, matching the tag CI pushed
helm upgrade --install shipit charts/shipit \
  --set image.tag="$SHA" --wait
kubectl get pods            # watch them reach Running/Ready
```

Watch the `READY` column: a pod can be `Running` (the process started) yet not `Ready` (the readiness probe has not passed). Traffic only flows once it is Ready — that is the readiness probe doing its job.

Reach the status page (get the external address with `kubectl get service shipit`) and confirm the region and banner color match your values.

> **What just happened:** `--wait` makes Helm block until the pods are actually Ready, so a bad deploy fails here instead of limping. This is the same behavior the pipeline will rely on.

## Step 4: Wire the deploy into the pipeline

> **Why:** Nothing new is being invented here — that is the point. The pipeline uses the **same OIDC login** from Lab 5, deploys the **same SHA-tagged image** it just built, and stays behind the **same Module 5 approval gate**. The only thing that changed is the target: a real, self-healing cluster instead of a single container. This is the moment the loop from commit to running pods closes without a human touching `kubectl`.
>
> **From the slides:** This realizes the "build-once-configure-per-environment" story at pipeline scale — one image, promoted through the gate, reconciled onto the cluster by the same tool you just ran by hand.

Create a new branch named `add-helm-deploy` (VS Code status bar → **Create new branch...**), then open `.github/workflows/cd.yml` in the Explorer. In the `deploy-production` job, **replace the entire job body** — every step below `environment: production` — with the steps below. Keep the `environment: production` gate as the job's own line; everything under it is new.

**Why the whole body, not just "Deploy new image":** Lab 5's `deploy-production` job has four moving parts built for Container Apps: a "Record current image" step, the deploy itself, a hand-written `curl` health-check loop, and a rollback step — and the last three all call `az containerapp show`/`az containerapp update`, which have nothing to do with the AKS cluster you're deploying to now. None of that logic transfers. AKS gets a cleaner design instead, because Helm already does most of the work:

- **No "record current image" step at all.** Helm keeps a release history on its own — `helm rollback shipit` with no revision number automatically goes to the previous release. There's nothing to manually remember.
- **No separate health-check loop.** `helm upgrade --install ... --wait` already blocks until the Deployment's pods pass their readiness probe (the one you wrote in Step 2) and **fails the step** if they never do. A hand-rolled `curl` loop would just be duplicating what `--wait` already does, worse.
- **A real rollback, triggered by that one failure.** If the deploy step fails, `helm rollback shipit` puts the previous release back — genuinely, not just a red X with nothing behind it.

```yaml
      - uses: actions/checkout@v5
      - uses: azure/login@v3
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - uses: azure/aks-set-context@v5
        with:
          resource-group: ${{ vars.RG }}
          cluster-name: ${{ vars.AKS }}
      - name: Deploy with Helm
        id: deploy
        continue-on-error: true
        run: |
          helm upgrade --install shipit charts/shipit \
            --set image.tag=${{ github.sha }} --wait
      - name: Roll back on failure
        if: steps.deploy.outcome == 'failure'
        run: |
          echo "::error::Deploy failed readiness. Rolling back to the previous Helm release."
          helm rollback shipit
          exit 1
```

`continue-on-error: true` on the deploy step is what lets the rollback step run *after* a failed deploy instead of the job stopping immediately — this is the same pattern Lab 5 used for its health-check step, just pointed at Helm instead of a `curl` loop. The final `exit 1` matters: without it, the job would report success even though the deploy failed and had to roll back, which would hide the incident instead of surfacing it.

> **What just happened:** the same SHA-tagged image, the same approval gate, now lands on a real cluster — and a bad deploy now genuinely rolls itself back on AKS, the same promise Lab 5 made for Container Apps, kept with Helm's own tools instead of borrowed Container Apps commands.

Now commit everything from Steps 1-4 — the whole `charts/` folder and the updated `cd.yml` — so the pipeline can actually use them (CI checks out a fresh copy of the repo; your local manual deploy in Step 3 doesn't count). In the **Source Control** panel, stage all the changed/new files (**+** next to **Changes**), commit ("Add Helm chart and AKS deploy"), and **Publish Branch**. Open a PR into `main` on GitHub.com, wait for the checks to go green, and **Merge pull request**.

## Step 5: Change config with no rebuild

> **Why:** This is the payoff of separating config from image. You are shipping a visible change to production behavior — a different banner color — with **no rebuild, no re-scan, and no drift**. The image stays byte-for-byte identical; only the ConfigMap changes, and the `checksum/config` annotation from Step 2 rolls the pods so they pick up the new value. In a copy-paste-YAML world this is exactly the kind of change that silently diverges between environments; here it is one flag.
>
> **From the slides:** This is "build-once-configure-per-environment" in its purest form — the same artifact, reconfigured, with the image tag unchanged.

Change the banner to yellow without touching the image:

```bash
helm upgrade --install shipit charts/shipit \
  --set image.tag="$SHA" --set config.bannerColor=yellow --wait
```

Refresh the status page: the banner is yellow, and `kubectl get pods` shows the same image. No rebuild, no re-scan.

## Step 6: Break a pod on purpose and diagnose it

> **Why:** On Kubernetes you diagnose from *status first*, and the status name usually tells you the category of problem before you read a single log line. This step hands you the decoder ring by breaking a pod deliberately: pointing at a nonexistent tag produces `ImagePullBackOff` — a pull or auth issue, which on this cluster traces back to the Lab 0 `AcrPull` identity. (`CrashLoopBackOff` would mean the app is dying on start; `Pending` would mean there is nowhere to schedule it.) Then you confirm the category with `describe` (events) and `logs --previous` (the last run's output, for crashes). Finally, `helm rollback` restores the last good release in one command — because rollback is a **first-class Helm operation**, not a manual scramble.
>
> **From the slides:** This is the "status decoder ring" slide made real, backed by the self-healing model — the Deployment keeps trying to reconcile toward your (broken) spec, which is why a bad tag keeps retrying instead of silently disappearing.

Point the chart at an image tag that does not exist, then investigate:

```bash
helm upgrade shipit charts/shipit --set image.tag=does-not-exist
kubectl get pods                       # look for ImagePullBackOff
kubectl describe pod <pod-name>        # events explain the pull failure
kubectl logs <pod-name> --previous     # for a crash, the last run's output
```

Notice that your *old* pods are still serving — the Deployment does not tear down healthy replicas to make room for ones that cannot even pull their image. That is self-healing protecting you from your own bad deploy.

Then fix it by rolling back to the last good release:

```bash
helm rollback shipit
```

> **What just happened:** you read the pod status first (`ImagePullBackOff` = pull/auth, `CrashLoopBackOff` = app dying, `Pending` = nowhere to schedule), then confirmed it with `describe` and `logs`. `helm rollback` restored the previous working release in one command.

## Step 7: Scale and watch a rolling update

> **Why:** Two of the headline promises come together here. Scaling to five replicas shows **self-healing / declarative state** — you change one number, and the controller creates the missing pods to match. Then a real rolling update shows the safety mechanism: new pods only start taking traffic **after `/readyz` passes**, and old pods are retired as new ones become Ready (`maxSurge`/`maxUnavailable` govern the pace). A broken new version never displaces a working one, because readiness gates the whole rollout.
>
> **From the slides:** This is the "rolling update, gated by readiness" concept end to end, and it is why the readiness probe you wrote in Step 2 mattered all along.

```bash
kubectl scale deployment/shipit --replicas=5
kubectl get pods -w          # watch new pods gate on readiness before serving
```

Make a small code change on a new branch named `test-rolling-update`: in `src/ShipIt/Program.cs`, find `static string AppVersion() => Environment.GetEnvironmentVariable("SHIPIT_VERSION") ?? "0.1.0-dev";` and change the fallback string to `"0.1.1-dev"`. This is visible on the status page (`Version:`), so you can confirm which pods are old versus new while the rollout is in progress. Stage, commit ("Bump version to test rolling update"), push, open a PR into `main`, wait for checks, and merge — then approve production and watch the rolling update proceed: new pods only take traffic once `/readyz` passes.

## Success criteria

- A merge to `main` deploys ShipIt to AKS through the approval gate.
- The status page is reachable and reflects the ConfigMap values.
- A rolling update stays available (no full outage), gated by readiness.
- You can state why a broken pod failed, from its status plus `describe`/`logs`.

## Troubleshooting

- **`ImagePullBackOff`:** the cluster cannot pull the image. Confirm the AKS cluster has `AcrPull` on the registry (Lab 0) and the tag exists in ACR.
- **Pod `Running` but never `Ready`:** the readiness probe is failing. Check the path (`/readyz`) and port (8080), and `kubectl describe pod` for the probe error.
- **`helm upgrade` hangs then fails with `--wait`:** the rollout never became healthy; that is the safety net working. Inspect pods before assuming a Helm bug.
- **Service has no external address:** a `LoadBalancer` can take a minute to provision, or use the ingress from Lab 0.
- **`rm` in Step 1 prints "No such file or directory" for `httproute.yaml`:** harmless — your Helm version didn't scaffold that file, so there was nothing to delete. The other files were still removed.
- **`helm template`/`helm upgrade` fails with a nil-pointer error mentioning `.Values.httpRoute` (or similar):** you're on a Helm version that scaffolds `httproute.yaml` and it wasn't deleted in Step 1. Delete `charts/shipit/templates/httproute.yaml` and retry.

## Stretch goals

- Deploy the same chart to the **OpenShift Developer Sandbox** and expose it with a **route** instead of an ingress. A route does the edge job an ingress does — HTTP routing and TLS termination — and because ShipIt's image runs as a non-root user (the Module 3 hardening), it clears OpenShift's Security Context Constraints (SCCs) unchanged, no edits required. This is the "AKS vs OpenShift at a Glance" slide in practice: projects instead of namespaces, routes instead of ingress, `oc` instead of `kubectl`, same chart.
- Add a **HorizontalPodAutoscaler** so ShipIt scales on CPU.
- Add a second environment's values file (`values-staging.yaml`) and deploy both — the clearest demonstration that one chart plus swapped values covers every environment.

## What this sets up

ShipIt now runs on a real cluster from the pipeline, but you created that cluster and its registry by running individual `az` CLI commands in Lab 0 — imperative and unreviewed, even though it was scripted rather than clicked. In **Module 7** you define that infrastructure as code with Bicep, deploy it through the same gated pipeline, then instrument ShipIt with Application Insights and build the dashboard and alert that finally let you measure the DORA metrics you set as targets in Lab 1.
