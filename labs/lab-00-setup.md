# Lab 0: Set Up Your Environment

**Module:** 0 (Course setup, start of Day 1)
**Time:** 30-40 minutes
**Format:** One-time setup (do this before Module 1)

---

## Why this lab

The rest of the course is hands-on, and every lab builds on the one before it. In this lab you will provision your Azure environment. A resource group, a container registry, an AKS cluster, and an Entra identity for your pipeline.

> **Why it's worth the time.** This is the foundation the whole week stands on. **ShipIt** — the small .NET shipment-tracking service from the slides — is the one app you will carry from a raw commit all the way to a monitored production deployment. Every module adds one link to that chain, and each lab starts exactly where the previous one ended.

## What you will confirm and build

- The local tools are installed at the right versions.
- A fork of the ShipIt starter repository, and you can push to it.
- An Azure resource group, container registry, and AKS cluster.
- Entra identity for the pipeline, with OIDC trust already wired to your fork.
- `kubectl` connection to AKS cluster.
- A free OpenShift Developer Sandbox.

## Prerequisites

- Your own **free personal GitHub account** (create one at github.com if you do not have one).
- An **Azure sign-in** for this class (given to you by instructor)
- A Windows lab VM, with the course tools already installed.

---

## Step 1: Verify your local tooling

1. Open **Visual Studio Code**.
2. Open the integrated terminal: **Terminal → New Terminal** (or `` Ctrl+` ``).
3. Make sure it's a **Git Bash** terminal, not PowerShell or Command Prompt: click the dropdown arrow (**⌄**) next to the **+** in the terminal panel's top-right corner and choose **Git Bash** from the list. If you don't see it there, open the Command Palette (`Ctrl+Shift+P`), run **Terminal: Select Default Profile**, choose **Git Bash**, then open a new terminal.
4. Confirm it's really Bash — the prompt should look like a Unix path (for example `jsmith@LAPTOP MINGW64 ~`), not a `PS C:\>` PowerShell prompt.

With that terminal open, confirm each tool is installed:

| Tool | Minimum version | Check |
|---|---|---|
| Git | 2.40+ | `git --version` |
| .NET SDK | 10.0 | `dotnet --version` |
| Docker Desktop | 4.x (running) | `docker run hello-world` |
| kubectl | 1.30+ | `kubectl version --client` |
| Helm | 3.14+ | `helm version` |
| Azure CLI | 2.60+ | `az version` |
| GitHub CLI (`gh`) | 2.x | `gh --version` |

Sign in to both CLIs now, from this same VS Code terminal:

```bash
az login
gh auth login
```

`az login` opens a browser — sign in with **your own class Azure account** (not a personal Microsoft account). `gh auth login` — pick **GitHub.com**, **HTTPS**, and authenticate through the browser with **your own personal GitHub account** (create one first at github.com if you don't have one yet).

## Step 2: Fork the ShipIt repository

You use your own personal GitHub account for this course

Open `https://github.com/jruels/shipit` in your browser and click **Fork** (top right of the repo page). Set the owner to **your own account**, keep the name **shipit**, and leave visibility **Public**.

**In Visual Studio Code:**

1. In VS Code, click the **Source Control** icon in the left Activity Bar (the branching icon — or press `Ctrl+Shift+G`).
2. Click **Clone Repository**.
3. Choose **Clone from GitHub**, and sign in if VS Code asks you to.
4. In the list that appears, pick your fork (`<your-username>/shipit`).
5. VS Code opens a folder picker so you can choose where to put it on disk. Create a place to keep your class repos: navigate to your **Downloads** folder, click **New Folder**, name it `repos`, then open it and click **Select as Repository Destination**. VS Code clones into `Downloads\repos\shipit`.
6. When the clone finishes, VS Code shows a notification asking if you want to open the cloned repository — click **Open**.

> **Prefer the terminal?** One command does the fork *and* the clone: `gh repo fork jruels/shipit --clone` (run it from your VS Code Git Bash terminal, in the folder where you want the project to live).

> **Why a fork, not a copy?** Forking keeps a link back to `jruels/shipit`, so if your instructor ever pushes a fix to it, GitHub can show you the difference and you can pull it in. A plain copy has no such link.

Now that the repo is on your machine, confirm your tooling one more time with the checker script that ships inside it. In your VS Code Git Bash terminal, `cd` into the cloned folder (or use **Terminal → New Terminal**, which opens already rooted there if you have the folder open) and run:

```bash
./scripts/check-prereqs.sh
```

> **What just happened:** the script reports any missing or out-of-date tool so you can fix it now, not mid-lab later in the course.

## Step 3: Provision your Azure environment

You are about to create, in **your own Azure subscription only**: a resource group, a container registry, an AKS cluster, a Container Apps environment, and an Entra identity for your pipeline. A script in the repo you just cloned does all of this in one run.

With the `shipit` folder open in VS Code (from Step 2), open a terminal rooted there: **Terminal → New Terminal** (make sure it's a **Git Bash** terminal — check the prompt as in Step 1). Run:

```bash
./scripts/lab0-provision.sh <your-initials>
```

Use two or three letters that mean something to you (for example `jrs` for Jane R. Smith). This is the **only** name you have to invent yourself — see why below.

> **Why do you pick a name here?** Azure Container Registry names must be unique across *all of Azure*, worldwide — so this is the one resource where a name collision is possible. Every other resource the script creates uses a fixed, identical name (`rg-shipit`, `shipit-aks`, and so on), which is safe because it lives in **your own dedicated subscription**. The script checks your registry name's availability for you and tries a variant automatically if your first choice is already taken by someone, somewhere, outside this class entirely.

> **What the script is doing, while you move on to Step 4:** it starts two slow operations first — registering the Azure resource providers this course needs, and creating your AKS cluster — and lets both run in the background while it creates everything else (registry, Container Apps, your Entra app registration, and the OIDC trust between Azure and your GitHub fork). By the time it prints its final summary, your AKS cluster is ready. This takes roughly 8-12 minutes total; keep this terminal running and go straight to Step 4 while you wait.

> **From the slides — the toolchain is in your hands now.** This one script is standing up the registry (ACR), the cluster (AKS), and the identity (Entra + OIDC) pieces of the toolchain diagram. You'll meet each of these again, individually, as its own lesson later in the course (Lab 3 for the registry, Lab 5 for OIDC, Lab 6 for the cluster) — this script just gets the plumbing in place first so class time goes to the ideas, not to waiting on `az` commands.

When it finishes, it prints something like this — **copy these values now**, you'll need them in the next step:

```
 RG                    = rg-shipit
 ACR                   = shipit<your-initials>acr
 AKS                   = shipit-aks
 AZURE_CLIENT_ID       = <a GUID>
 AZURE_TENANT_ID       = <a GUID>
 AZURE_SUBSCRIPTION_ID = <a GUID>
```

## Step 4: Find your subscription and tenant ID in the Azure Portal

The script already showed you `AZURE_SUBSCRIPTION_ID` and `AZURE_TENANT_ID` above, but it's worth knowing where these live in the portal — you'll want to find them yourself again later in the course without re-running a script.

1. Go to [portal.azure.com](https://portal.azure.com) and sign in with your class account.
2. Search **Subscriptions** in the top search bar. You'll see exactly one subscription — that's yours. Its **Subscription ID** is `AZURE_SUBSCRIPTION_ID`.
3. Search **Microsoft Entra ID** and open its **Overview** page. The **Tenant ID** shown there is `AZURE_TENANT_ID`.

> **Why:** Later labs (and real jobs) constantly ask "what subscription/tenant am I in?" — knowing these two portal screens means you never need anyone to hand you these values again.

## Step 5: Set your repository variables

Set the six values from Step 3 as **GitHub repository variables** so your pipelines can read them:

1. On GitHub.com, open your fork (`https://github.com/<you>/shipit`).
2. Go to **Settings → Secrets and variables → Actions → Variables tab → New repository variable**.
3. Add all six: `RG`, `ACR`, `AKS`, `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`.

> **Prefer the terminal?** `gh variable set RG --body "rg-shipit"` (repeat per value) does the same thing from your VS Code Git Bash terminal.

> **Why repository variables (not secrets, not environment variables)?** The later labs' workflows read these as `${{ vars.RG }}`, `${{ vars.AZURE_CLIENT_ID }}`, and so on. They live at the **repository** level because the Module 2 CI job has no `environment:` and still needs to read `AZURE_*` to push the image in Module 3. And none of them are secrets — with OIDC there is no client secret to hide (that is the whole point of Module 5's federation).

## Step 6: Confirm your AKS cluster is ready

Once Step 3's script has finished (its final summary printed), point `kubectl` at your cluster and confirm it's up:

```bash
az aks get-credentials -g rg-shipit -n shipit-aks
kubectl get nodes
```

You should see one node in `Ready` status. This is your own cluster, in your own subscription.

> **Why this "just works":** `az aks get-credentials` pulls a kubeconfig using *whatever Azure account you're currently signed in as*. Because your AKS cluster lives inside the same subscription you're already signed into, there's no separate cluster-access step to set up — the access you already have to your own subscription is the access you need.

## Step 7: Create your OpenShift Developer Sandbox

For the OpenShift portion of Module 6, create a free sandbox now (it takes a minute to activate):

1. Go to `developers.redhat.com/developer-sandbox` and click **Start your sandbox for free**.
2. Sign in with (or create) a free Red Hat Developer account and confirm your account.
3. Once provisioned, note your sandbox is a free, time-limited OpenShift cluster; no credit card is required.

## Step 8: Make your first commit and push

Prove you can push to your own fork — this is the muscle every later lab uses.

**In Visual Studio Code:**

1. Click the **Source Control** icon in the left Activity Bar (the branching icon, same as Step 2 — or press `Ctrl+Shift+G`).
2. Create a branch first: click the branch name in the blue status bar at the bottom-left of the window (it currently says `main`), choose **Create new branch...** from the list that appears, type `setup-check`, and press **Enter**.
3. Click the **Explorer** icon at the top of the left Activity Bar (the top icon, looks like two overlapping pages) to see your file tree. Hover over the `shipit` root folder, click the **New File** icon that appears next to it, type `NOTES.md`, and press **Enter**. In the editor that opens, type a line like `setup ok` and save (`Ctrl+S`).
4. Click the **Source Control** icon again to go back to that panel. Hover over **Changes** and click the **+** next to `NOTES.md` to stage it.
5. Type a commit message ("Lab 0: confirm I can push") in the message box at the top and click the checkmark (**Commit**).
6. Click **Sync Changes** (or **Publish Branch** if this is the first push of a new branch) — this pushes your commit to GitHub.

Open your repo on GitHub.com and confirm your `setup-check` branch and commit are there. (There is no CI pipeline yet; you build that in Module 2.)

> **What just happened:** everything you just did with clicks — stage, commit, push — is exactly what `git add`, `git commit`, and `git push` do on the command line. VS Code's Source Control panel is the same Git underneath; you'll use this same panel for every commit for the rest of the course, so it's worth getting comfortable with it now.

## Success criteria

- `./scripts/check-prereqs.sh` reports all tools present and current (run it from the repo root after cloning).
- You forked the ShipIt repo under your own account and pushed a branch to GitHub.
- `./scripts/lab0-provision.sh <initials>` completed without error and printed your six values.
- All six repository variables are set on your fork.
- `kubectl get nodes` shows your cluster's node as `Ready`.
- Your OpenShift Developer Sandbox is active.

## Troubleshooting

- **`az login` opens the wrong tenant or account:** make sure you signed in with your class Azure account, not a personal Microsoft account (`az account show` to check).
- **The provisioning script fails partway through:** most steps are safe to re-run, **except** `az ad app create` — re-running the whole script creates a *second* Entra app registration. If you need to re-run after a partial failure, delete the app it already created first (`az ad app delete --id <the appId it printed or that you find under Microsoft Entra ID → App registrations>`), then re-run the script from the top.
- **ACR name taken:** the script already retries a few variants of your initials automatically; if it still fails, pick different initials and re-run.
- **`kubectl get nodes` is unauthorized or empty:** re-run `az aks get-credentials -g rg-shipit -n shipit-aks --overwrite-existing`, and confirm the script's final "AKS cluster is ready" line actually printed (it may still be creating).
- **`docker run hello-world` fails:** make sure Docker Desktop is actually running, not just installed.
- **Cannot push to the repo:** confirm you are signed in to your own GitHub account in VS Code (bottom-left **Accounts** icon) and that the repository you cloned is your own fork, not the original template.

## What this sets up

Everything from here on assumes this environment works. **Lab 1** is next — a pen-and-paper analysis (no tools) where you baseline a slow delivery team with the DORA metrics and set the targets the rest of the course aims at. Then **Module 2** turns your first push into an automated CI pipeline; by Lab 5 that pipeline deploys to the Azure resources you just created, and by Lab 7 it provisions new ones with Bicep. If anything here failed, flag it before Module 1 starts so it can be fixed while the material is still introductory.
