# Lab 3: Containerize ShipIt and Push to ACR

**Module:** 3 (Containerization with Docker)
**Time:** 45-60 minutes
**Format:** Hands-on (Docker + GitHub Actions + Azure Container Registry)

---

## Why this lab

Lab 2 gave us a build we trust, but what came out is still just loose files on a disk. A green pipeline that leaves behind a folder of DLLs has proven your code compiles and passes tests, but it has not produced anything you can *run* the same way twice. Production does not run loose files; production runs containers.

A container is the code plus the exact environment it needs, bundled as one unit. That is what finally kills "works on my machine" and the slow environment drift that creeps in between a developer's laptop, the CI runner, and the server. When the runtime, the base OS, and the dependencies travel *inside* the artifact, there is no gap left for drift to hide in.

In this lab you package ShipIt into a container image, run it locally to prove it works, then extend your pipeline so every good build pushes a SHA-tagged image to Azure Container Registry (ACR). The promise: **every good build produces one container image in ACR, tagged with the exact commit that made it.** By the end, CI does not stop at "tests passed" — it stops at "here is the exact, deployable image for this commit." That SHA-tagged image is the single artifact the rest of the course moves around: Module 5 promotes it through environments, Module 6 runs it on Kubernetes.

## Concepts this lab makes real

> This lab turns the Module 3 slides into muscle memory. Watch for these ideas as you go:
>
> - **Image / container / registry** — the image is the class, a container is an object you run from it, and the registry (ACR) is the package feed you pull from. You will build one class and run objects from it.
> - **Image layers & build cache** — an image is a stack of read-only layers, one per Dockerfile instruction; a running container adds a thin writable layer on top. Docker reuses a layer if nothing above it changed, so the golden rule is *copy what changes least, first*.
> - **The multi-stage pattern** — compile in the big SDK image, ship only the result in a small runtime image. The `COPY --from=build` line is the hinge.
> - **Non-root by UID** — `USER $APP_UID` runs the app as the user baked into the .NET images since .NET 8, identified by UID, which Kubernetes will insist on in Module 6.
> - **SHA tags & immutability** — tag every build with the git commit SHA to get an immutable pointer to exactly one build. If you deploy `latest`, "roll back" has no meaning.
> - **Push vs pull identities** — CI needs `AcrPush`; the cluster later needs `AcrPull`. Two jobs, two roles.
> - **OIDC federation** — CI logs in to Azure with no stored password, using an Entra ID federated credential instead of a secret.
> - **Scanning as a habit** — a CVE is the public catalog of known security flaws; scanning is a habit, not a one-time gate.

## What you will build

- A multi-stage `Dockerfile` that produces a small, non-root ShipIt image.
- A `docker-compose.yml` for the local development loop.
- An extension to `ci.yml` that logs in to Azure with OIDC and pushes a SHA-tagged image to ACR.
- An image vulnerability scan in the pipeline.

## Success looks like

A push to `main` results in a `shipit:<sha>` image in your ACR, the image runs locally and serves the status page (honoring `SHIPIT_REGION`), and the scan step runs and reports findings.

## Prerequisites

- You finished Lab 2: `ci.yml` builds and tests ShipIt, and `main` is protected.
- Local tools: Docker Desktop (running), the Azure CLI (`az`), Git, .NET 10 SDK.
- An ACR instance and an Entra ID app with a federated credential for your repo, plus the `RG`, `ACR`, `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, and `AZURE_SUBSCRIPTION_ID` repository variables — all created by you in Lab 0's `./scripts/lab0-provision.sh` run and set as repository variables there.
- Commands below run in VS Code's integrated terminal, set to Git Bash (see Lab 0 if you need to switch your terminal profile).
- Behind? Ask your instructor for the checkpoint branch name and check it out from VS Code's Source Control panel (**...** menu → **Branch → Checkout to...**).

---

## Step 1: Write a multi-stage Dockerfile

Create a branch for this lab's changes (VS Code status bar → branch name → **Create new branch...** → type `add-docker`). Then, in the Explorer panel, hover over the `shipit` root folder, click the **New File** icon, and type `Dockerfile` (no extension). The two stages matter: the SDK image compiles, and only the published output lands in the small runtime image. Paste this in and save (`Ctrl+S`):

```dockerfile
# build stage: compile and publish using the SDK
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# copy the project file and restore first, so this layer stays cached
COPY src/ShipIt/ShipIt.csproj src/ShipIt/
RUN dotnet restore src/ShipIt/ShipIt.csproj

# now copy the rest of the source and publish
COPY . .
RUN dotnet publish src/ShipIt -c Release -o /app --no-restore

# runtime stage: small image, only the published app
FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY --from=build /app .
USER $APP_UID
ENTRYPOINT ["dotnet", "ShipIt.dll"]
```

> **From the slides:** This is *The Multi-Stage Pattern* diagram in code. The first `FROM ... AS build` uses `sdk:10.0` — everything needed to compile — while the second `FROM ... aspnet:10.0` is the far smaller runtime-only image. The `COPY --from=build /app .` line is the hinge of the whole pattern: you *compile in the big image and ship only the result in the small one*. The compiler, the NuGet caches, and the intermediate build junk never make it into what you ship.

> **Why:** Each instruction in this file is one read-only layer in the image (*a stack of read-only layers, one per Dockerfile instruction*). Docker reuses a cached layer whenever nothing above it changed — so layer *order* is a performance decision, not a cosmetic one. Copying the `.csproj` and running `dotnet restore` *before* copying the rest of the source means a one-line code change never invalidates the restore layer. The rule is *copy what changes least, first*: your dependency list changes rarely, your source changes constantly, so restore sits below the source copy and stays cached across most builds.

> **Why:** `USER $APP_UID` runs the app as the non-root user *baked into the .NET images since .NET 8*, identified *by UID* rather than by name. Running as non-root shrinks the blast radius if the app is ever compromised, and Kubernetes will *insist on* a numeric UID in Module 6 (a `runAsNonRoot` policy can only verify a UID, not a username) — so setting it here means you never have to retrofit it later.

## Step 2: Build and run the image locally

```bash
docker build -t shipit:dev .
docker run --rm -p 8080:8080 -e SHIPIT_REGION="local-dev" shipit:dev
```

Open `http://localhost:8080/` and confirm the status page loads and shows the region `local-dev`. Then hit `http://localhost:8080/readyz` and confirm it returns healthy.

> **Why:** This is the *From Source to a Running Container* diagram happening on your machine — the `build` turns the class (image) into something concrete, and `run` starts an object (container) from it, adding the thin writable layer on top of the read-only stack. The artifact you build here is *byte-for-byte the artifact that runs* — locally now, in the cluster later. That identity is the whole point of containers: no rebuild between environments means no drift between environments.

> **From the slides:** Notice you changed the region with `-e SHIPIT_REGION="local-dev"` — a *config value passed at run time, not a rebuild*. Same image, different behavior. That is configuration-by-environment, and it is exactly the seam Kubernetes ConfigMaps will plug into in Module 6.

Once you've confirmed both checks, stop the container: go back to the terminal where it's running and press `Ctrl+C`. (It was started without `-d`, so it's running in the foreground and holding port 8080 — the `--rm` flag means it's removed automatically as soon as it stops.) You'll need port 8080 free for Step 3.

## Step 3: Add Docker Compose for the local loop

In the Explorer panel, hover over the `shipit` root folder, click the **New File** icon, and type `docker-compose.yml`, so a teammate can run ShipIt with one command:

```yaml
services:
  shipit:
    build: .
    ports:
      - "8080:8080"
    environment:
      SHIPIT_REGION: "local-dev"
      SHIPIT_BANNER_COLOR: "green"
```

Run it and confirm the banner is green:

```bash
docker compose up --build
```

Change `SHIPIT_BANNER_COLOR` to `yellow`, re-run, and watch the status page change with no code rebuild. That config-by-environment habit pays off in Module 6.

> **From the slides:** Compose is *for one developer on one machine* — a fast local loop, not *a tiny Kubernetes*. Do not mistake it for a production orchestrator; it has no scheduling, no self-healing, no multi-node story. What it *is* good for is this: the `environment:` block you just edited is a *quiet preview of ConfigMaps* in Module 6. The same "declare config outside the image, inject it at run time" idea reappears there, at cluster scale.

## Step 4: Push a SHA-tagged image from the pipeline

Open `.github/workflows/ci.yml` in the Explorer. First grant the workflow permission to request an OIDC token — add this block right after the `on:` block, before `jobs:`, so it applies to the whole workflow:

```yaml
permissions:
  id-token: write
  contents: read
```

Now add these two steps to the end of the `build-and-test` job (after the artifact-upload steps from Lab 2). Note the `if:` on these steps — they run **only when the workflow was triggered by a push to `main`** (i.e. after a merge), not on pull requests. That keeps you from pushing an image for every PR, and it means the OIDC token's subject is `...:ref:refs/heads/main`, which is the federated credential the Lab 0 setup created for CI:

```yaml
      - name: Log in to Azure (OIDC, no stored password)
        if: github.event_name == 'push'
        uses: azure/login@v3
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

      - name: Build and push the image
        if: github.event_name == 'push'
        run: |
          az acr login --name ${{ vars.ACR }}
          IMAGE=${{ vars.ACR }}.azurecr.io/shipit:${{ github.sha }}
          docker build -t "$IMAGE" .
          docker push "$IMAGE"
```

The workflow reads your registry name from the `ACR` repository variable you set in Lab 0, and `AZURE_CLIENT_ID` / `AZURE_TENANT_ID` / `AZURE_SUBSCRIPTION_ID` are also **repository** variables (the CI job has no `environment:`, so it can only read repository-level variables). Commit and push to a branch (VS Code Source Control), open a PR (CI builds and tests, no image push), merge, and confirm the run on `main` logs the build and push.

> **From the slides:** ACR is a *managed private registry that plugs into the rest of Azure* — the *package feed* from the image/container/registry trio. Because it is native to Azure, AKS will later pull from it *without wiring up credentials by hand*.

> **Why:** This step uses the **push** identity. Remember the split from the slides: *CI needs `AcrPush`* to publish an image; the *cluster later needs `AcrPull`* to fetch one. Different jobs, different roles — least privilege means CI can never accidentally deploy and the cluster can never accidentally publish. If you ever see `ImagePullBackOff` in Module 6, that is the *pull* side of this same identity story failing.

> **Why:** The login uses OIDC — *no stored password*. `id-token: write` is what lets this workflow *request an OIDC token as its Entra identity*; Azure trusts that token because of the federated credential Lab 0 registered against your exact repo and branch. There is no client secret sitting in GitHub to leak or rotate. That is also why the `if: github.event_name == 'push'` gate matters beyond cost: it pins the token's subject to `refs/heads/main`, which is the one subject the federated credential is configured to trust.

## Step 5: Confirm the image is in ACR

This is a local `az` command, so set `ACR` in your shell first (it is a GitHub *repository* variable in the pipeline, but your terminal does not know it):

```bash
export ACR=<your-registry-name>   # the ACR name from your Lab 0 script run, e.g. shipitjrsacr
az acr repository show-tags --name "$ACR" --repository shipit --output table
```

You should see a tag matching your merge commit's SHA. That tag is an immutable pointer to exactly this build.

> **From the slides:** *Tag every build with the git commit SHA* and you get an *immutable pointer* — a name that means one, and only one, build forever. This is why `latest` is a trap: *if you deploy `latest`, "roll back" has no meaning*, because `latest` points at a moving target. A SHA tag *ties the image to the exact commit* that produced it, so you can always answer "what code is running?" and "what code do I go back to?" That traceability is precisely *what makes rollback possible in Module 5*.

## Step 6: Scan the image in the pipeline

In `ci.yml`, add a scan step right after the "Build and push the image" step you just added, using the Trivy action:

```yaml
      - name: Scan the image
        if: github.event_name == 'push'
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ vars.ACR }}.azurecr.io/shipit:${{ github.sha }}
          format: table
          severity: HIGH,CRITICAL
          exit-code: "0"
```

Read the findings in the run log. Leave `exit-code: "0"` for now so the scan reports without failing the build; the stretch goal makes it blocking.

> **From the slides:** *Trivy* (and *Microsoft Defender for ACR* on the registry side) checks your image against the CVE catalog — the *public catalog of known security flaws*. Scanning in the pipeline is "shift-left": you catch problems at build time, close to the developer who can fix them, instead of in production. And it is *a habit, not a one-time gate* — *a base image you pulled last month may have a known CVE today* even though you never changed a line of code, because the vulnerability was discovered after you built. That is why the scan lives in CI and runs on every build, not once at the start.

> **Pin the scanner too.** In real use, pin this action to a release tag or commit SHA instead of `@master`, the same way you pin every other action (Module 4 covers keeping those pins patched with Dependabot).

---

## Success criteria

- `docker build` produces an image that runs locally and serves the status page with the region from `SHIPIT_REGION`.
- A merge to `main` pushes `shipit:<sha>` to ACR, and you can list the tag with `az acr repository show-tags`.
- The scan step runs and reports HIGH/CRITICAL findings (or none).

## Troubleshooting

> **Why:** *A push failure and a pull failure have different fixes.* This step (CI) is all about **push**, so authorization errors here point at the `AcrPush` role and the federated credential — not at the cluster's `AcrPull` role you will meet in Module 6. Read the error for *which* identity and *which* direction before you start changing things.

- **`az acr login` fails with authorization error:** the workflow identity needs the `AcrPush` role on the registry, and the federated credential's subject must match this repo and branch.
- **`docker push` denied:** you ran `az acr login` for a different registry name than the image tag. They must match.
- **`USER $APP_UID` unknown:** you are on a base image older than .NET 8. Use the `mcr.microsoft.com/dotnet/aspnet:10.0` tag.
- **Slow builds every time:** you copied all source before restoring. Put the `.csproj` copy and `dotnet restore` before `COPY . .`. (This is the layer-cache rule from Step 1 — a change above the restore layer invalidates it.)
- **Container exits immediately:** check the `ENTRYPOINT` DLL name matches your published assembly (`ShipIt.dll`).
- **`docker compose up` fails with a port-in-use / address already allocated error:** the Step 2 container is still running and holding port 8080. Go to that terminal and press `Ctrl+C` to stop it (or run `docker ps` to find it and `docker stop <container-id>`), then retry.

## Stretch goals

- Add a `latest` tag alongside the SHA on `main` (two `-t` flags on `docker build`, push both). Keep the SHA as the source of truth — `latest` is a convenience pointer, never the thing you deploy.
- Reorder the Dockerfile to copy source before restoring, rebuild twice, and measure how much slower it gets. Then put it back. This is the layer-cache rule made visible: you are watching the restore layer get invalidated on every code change.
- Make the scan blocking: set `exit-code: "1"` so a HIGH or CRITICAL finding fails the build. This turns scanning from a report into a gate.
- Swap the runtime base to a chiseled image (`mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled`) and compare the image size with `docker images`. A smaller base means fewer packages, which means a smaller CVE surface for Step 6 to scan.

---

## What this sets up

CI now ends at a SHA-tagged image in your registry: a real, deployable artifact instead of loose files. That single image — tagged with the exact commit that made it — is the unit the rest of the course carries forward. In **Lab 4** you add security gates (Dependabot, CodeQL, secret scanning) so nothing dangerous is ever packaged into that image. On **Day 2**, Module 5's pipeline promotes exactly these SHA-tagged images through environments (which is why the SHA tag mattered), and Module 6 runs them on Kubernetes using the `AcrPull` half of the identity split you set up here.

---

## Appendix: the complete `Dockerfile`

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY src/ShipIt/ShipIt.csproj src/ShipIt/
RUN dotnet restore src/ShipIt/ShipIt.csproj
COPY . .
RUN dotnet publish src/ShipIt -c Release -o /app --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY --from=build /app .
USER $APP_UID
ENTRYPOINT ["dotnet", "ShipIt.dll"]
```
