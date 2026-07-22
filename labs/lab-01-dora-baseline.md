# Lab 1: Baseline a Delivery Workflow with DORA

**Module:** 1 (DevOps Culture and Practices)
**Time:** 20-30 minutes
**Format:** Analysis and discussion (no tools or code yet)

---

## Concepts this lab makes real

This lab is where the Module 1 ideas stop being slides and become a scoreboard you can act on. You will practice:

- **DevOps as culture first, tools second** — "a culture and a set of practices, not a job title or a tool," across its three parts: *Culture, Practices, Toolchain*.
- **The DevOps lifecycle loop** (the `lifecycle-loop` diagram): plan, code, build, test, release, deploy, operate, monitor — and how *monitor feeds back into plan*.
- **Continuous Integration** ("merging code safely") and **Continuous Delivery** ("shipping it safely").
- **DORA** — DevOps Research and Assessment, a research program, not a vendor product.
- **The four DORA metrics** and the **DORA quadrant** diagram (speed on the left, stability on the right): *Deployment frequency* and *Lead time for changes* (speed); *Change failure rate* and *Time to restore service / MTTR* (stability). Two measure speed, two measure stability.
- **Speed and stability together** — the finding that strong teams are fast *and* stable, with no trade-off between the two.
- **Anti-patterns** — a failure mode paired with a concrete fix, including the **hero / bus-factor** problem ("only Priya can run the deploy").
- **ShipIt**, the .NET shipment-tracking API you carry from commit to monitored deploy for the rest of the course (the `ShipIt pipeline map`, where each stage is a module).

---

## Why this lab

You cannot improve what you cannot measure, and you cannot measure without a baseline. Before we build a single pipeline, you are going to look at a team that ships software the slow, painful way, score it with the four DORA metrics, and decide what to fix first. Those decisions become the reason behind everything you build for the rest of the course. On Day 2, when ShipIt is deployed and monitored, you will come back to the targets you set here and check whether the pipeline actually moved them.

> **From the slides:** The tools are the easy part. DevOps is culture first, tools second — "a DevOps tool dropped on top of a broken process just automates the broken process, now faster." The old "throw code over a wall" model made releases *big, rare, and scary*. Before you reach for a tool, you have to see the broken process clearly. That is the entire job of this lab.

> **Why:** This baseline is the "before" picture. DORA exists to replace opinion with four numbers, and this is the moment you turn Contoso's gut feelings into a scoreboard. Lab 7 re-checks these exact numbers against the pipeline you will have built, so the sheet you produce today is what the whole course is measured against.

## What you will do

1. Map a real delivery workflow onto the DevOps lifecycle and find where time is lost.
2. Estimate the four DORA metrics from a plain-English description.
3. Pick the three biggest bottlenecks and write one improvement target for each, tied to a DORA metric.
4. Predict which upcoming module will address each target.

## What you will produce

A one-page **Current State and Targets** sheet (template at the end of this lab). Keep it. You will reference it again in Lab 7.

## Prerequisites

- You have finished Module 1 and can name the four DORA metrics.
- No environment setup needed. This lab is pen, paper, and discussion.

---

## The scenario: "Contoso Parcel"

Contoso Parcel is a team of six that maintains the ShipIt shipment-tracking app. Here is how they work today.

- Developers work on their own machines for a week or two at a time before merging anything, each on a long-lived branch.
- When a developer thinks a feature is done, they build the app **on their laptop** and copy the output to a shared network folder.
- QA is a single shared test server. To use it, you post in a chat channel and wait your turn, sometimes a day or two.
- Testing is manual. A QA analyst clicks through the app against a checklist. There are almost no automated tests.
- Deploys happen **once every two weeks, on Friday afternoon**. One senior engineer, Priya, runs a script from her machine that copies files to the production server and restarts it.
- About **one in three** of those Friday deploys goes wrong: a missing config value, a file that did not copy, a change that worked on the laptop but not in production.
- When a deploy fails, the team scrambles. A bad Friday deploy is usually not fully fixed until **Monday or Tuesday**.
- Nobody is quite sure how long it takes for a finished feature to actually reach customers. The guesses range from "two weeks" to "sometimes a month or more."

This is a composite of real teams. Nothing here is exaggerated for effect.

> **From the slides:** Read this scenario as the "throw code over a wall" anti-patterns made concrete. Long-lived branches that merge every week or two are the opposite of Continuous Integration ("merging code safely"). Building on a laptop and copying files by hand is what makes releases big, rare, and scary. And "only Priya can run the deploy" is the textbook **hero / bus-factor** anti-pattern: one person is a single point of failure for the whole team's ability to ship.

---

## Step 1: Map it to the DevOps lifecycle

Draw the eight-stage loop: plan, code, build, test, release, deploy, operate, monitor.

For each stage, write one short note on how Contoso does it today. Then mark two things:

- **Handoffs** (where work passes from one person or system to another). Put a star on each.
- **Waiting** (where work sits idle). Put a clock on each.

> **From the slides:** This is the `lifecycle-loop` diagram from Module 1. Notice the loop *closes* — monitor feeds back into plan. Ask yourself: at Contoso, does anything from "operate" or "monitor" actually reach the next "plan"? If the loop is broken, the team never learns from what it shipped.

> **Why:** Handoffs and waiting are exactly where delivery time hides. Every star (a handoff) and every clock (idle waiting) is time a customer is waiting for a change that is technically already "done." Finding these is how you locate the lead-time problem before you ever measure it.

> **Prompt:** Handoffs and waiting are where delivery time hides. Contoso has at least four obvious ones. Find them.

## Step 2: Estimate the four DORA metrics

Using only the scenario, fill in a rough value and a direction (is this good or bad?) for each metric. Exact numbers do not matter; the reasoning does.

| Metric | Your estimate | Good or bad? | What in the scenario tells you |
|---|---|---|---|
| Deployment frequency | | | |
| Lead time for changes | | | |
| Change failure rate | | | |
| Time to restore service | | | |

> **From the slides:** These are the four DORA metrics, and they split two and two — *deployment frequency* and *lead time* measure **speed**; *change failure rate* and *time to restore service (MTTR)* measure **stability**. That is the left and right halves of the `DORA quadrant`. Remember: DORA is a research program, not a vendor product — these four numbers are how the industry replaces "I think we're pretty fast" with evidence.

> **Why:** This step turns Contoso's gut feelings ("two weeks... sometimes a month") into a scoreboard. You cannot improve what you cannot measure, so estimating even rough numbers is the act that makes improvement possible.

> **Prompt:** For lead time, think about the whole path from "code committed" to "running in production," including the wait for the shared QA server and the two-week deploy cadence.

## Step 3: Pick the top three bottlenecks and set targets

Choose the three problems that hurt the most. For each, write one improvement target. A good target names a **DORA metric** and a **direction of change**, and is specific enough that you would know if you hit it.

Weak target: "Deploy better."
Strong target: "Cut change failure rate from about 1 in 3 to under 1 in 10 by adding automated tests and a merge gate."

| # | Bottleneck | Improvement target (metric + direction) |
|---|---|---|
| 1 | | |
| 2 | | |
| 3 | | |

> **From the slides:** Fix stability first. Faster deploys of broken code just break things faster — speeding up a one-in-three failure rate only produces failures sooner. On the `DORA quadrant`, targets are *relative to where you start*: Contoso does not need to become elite overnight, it needs to move in the right direction from its own baseline.

> **Why:** A strong target always names a metric and a direction, because that is what makes it checkable in Lab 7. "Deploy better" cannot be verified; "cut change failure rate from 1 in 3 to under 1 in 10" can.

## Step 4: Predict the fix

For each target, guess which upcoming module will give you the tool to hit it. You do not need to be right; the point is to connect the pain to the coming solution.

| Target | Likely module that helps | Why |
|---|---|---|
| 1 | | |
| 2 | | |
| 3 | | |

> **From the slides:** Every anti-pattern you spotted in the Contoso scenario has a concrete fix built later this week — that is the `ShipIt pipeline map`, where each stage of the pipeline maps to a module. You are essentially predicting your own path through the course.

Reference (do not peek until you have guessed):

- Manual builds and no automated tests point to **Module 2 (CI with GitHub Actions)**.
- "Worked on my laptop, not in production" points to **Module 3 (containers)**.
- One-in-three failure rate and no quality checks point to **Module 4 (security and quality gates)** and Module 2.
- Friday hero deploys and slow recovery point to **Module 5 (CD, approval gates, rollback)**.
- "Not sure it is even measurable" points to **Module 7 (monitoring)**, where you finally get the numbers.

---

## Success criteria

You are done when:

- Your lifecycle map marks at least three handoffs and two waiting points.
- You have an estimate and a direction for all four DORA metrics, each justified by something in the scenario.
- You have three improvement targets, and every target names a DORA metric and a direction.
- At least one of your targets maps to automation this course will build.

## Discussion (as a group)

- Which single change would help Contoso the most, and why that one first?
- Contoso's instinct is that deploying more often would be reckless, because deploys already fail a third of the time. Use the DORA idea of speed and stability together to argue the opposite: why might deploying *more* often, in smaller pieces, actually make them *more* stable?
- Which of these problems exist on a team you have worked on?

> **From the slides:** The headline DORA finding is counterintuitive and worth sitting with: going faster, in *smaller* batches, tends to **increase** stability, not decrease it. Shipping small changes often is both faster and safer than shipping big changes rarely — because a small change is easier to test, easier to review, and trivial to roll back when something goes wrong. That is what "speed and stability together" means: they are not a trade-off.

## Stretch

- Rewrite the Contoso workflow as you would want it to look at the end of this course. Which stages become automatic? Where does a human still make a decision on purpose?
- Put rough "after" numbers next to your four metric estimates. That is the story the rest of the course will tell.

> **Why:** Those "after" numbers are the story the rest of the course tells. Sketching them now gives you a hypothesis to test in Lab 7 — the difference between the "before" you baselined today and the "after" your pipeline delivers.

---

## What this sets up

The targets you wrote are the scoreboard for the whole course. In Lab 2 you take the first concrete step against them: you move Contoso's build off a laptop and into an automated pipeline that runs on every change. In Lab 7, once ShipIt is monitored in production, you will pull the real DORA numbers and compare them to the targets on this sheet.

---

## Appendix: Current State and Targets sheet (template)

```
CONTOSO PARCEL - CURRENT STATE AND TARGETS
Prepared by: ______________________   Date: __________

DORA BASELINE (today)
  Deployment frequency ....... __________   (good / bad)
  Lead time for changes ...... __________   (good / bad)
  Change failure rate ........ __________   (good / bad)
  Time to restore service .... __________   (good / bad)

TOP THREE BOTTLENECKS
  1. ____________________________________________
  2. ____________________________________________
  3. ____________________________________________

IMPROVEMENT TARGETS (metric + direction)
  1. ____________________________________________
  2. ____________________________________________
  3. ____________________________________________

FIRST CHANGE TO MAKE, AND WHY
  ______________________________________________
```
