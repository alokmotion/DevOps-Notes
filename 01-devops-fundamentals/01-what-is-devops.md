# What is DevOps

## The problem it solves

Before DevOps, two teams with opposite goals:

- **Developers** are paid to ship change. More features, faster.
- **Operations** are paid to keep things stable. Change is the #1 cause of outages.

So Dev throws code over the wall, Ops refuses to deploy it, and everyone blames everyone.
The classic line: *"It works on my machine."* Ops replies: *"Then ship your machine."*
(That reply is literally where containers came from.)

## The definition

> **DevOps is a culture and set of practices where the people who build software also share responsibility for running it — supported by automation so that releasing is boring and routine.**

Three things to notice in that sentence:

1. **Culture first.** DevOps is not a tool. You cannot buy it. A team with Jenkins and Kubernetes and a "throw it over the wall" attitude is not doing DevOps.
2. **Shared responsibility.** "You build it, you run it." The dev who wrote the bug gets the 3am page. This aligns incentives fast.
3. **Automation is the enabler, not the goal.** Automation exists so that deploying 20 times a day is safe.

## The core principles

| Principle | What it means in practice |
|---|---|
| **Automate everything repeatable** | If you do it twice by hand, script it. Humans make typos; scripts don't. |
| **Small, frequent releases** | 10 small deploys are far safer than 1 huge one. Less to debug when it breaks. |
| **Everything as code** | Infrastructure, config, pipelines — all in Git. Reviewable, revertible, auditable. |
| **Fast feedback** | Tests in minutes, not days. The sooner you know it's broken, the cheaper the fix. |
| **Measure everything** | Logs, metrics, traces. You cannot improve what you can't see. |
| **Blameless culture** | When something breaks, fix the system, not the person. Blame just hides information. |

## CALMS — the standard framework

An easy way to remember what DevOps covers:

- **C**ulture — collaboration, shared ownership, no blame
- **A**utomation — CI/CD, IaC, automated testing
- **L**ean — small batches, remove waste, remove waiting
- **M**easurement — metrics on everything, data over opinions
- **S**haring — knowledge, tools, and responsibility across teams

## How success is measured: DORA metrics

The four metrics research shows actually separate high performers from low performers:

| Metric | Question it answers | Elite level |
|---|---|---|
| **Deployment Frequency** | How often do we ship to production? | On demand, multiple per day |
| **Lead Time for Changes** | Commit → running in prod, how long? | Under one hour |
| **Change Failure Rate** | What % of deploys cause a problem? | 0–15% |
| **Mean Time to Recovery (MTTR)** | How fast do we recover from failure? | Under one hour |

**The key insight:** speed and stability are *not* a trade-off. The teams that deploy most often are also the teams that break things least. Because small changes are easy to test, easy to understand, and easy to roll back.

## What a DevOps engineer actually does day-to-day

- Builds and maintains CI/CD pipelines (the conveyor belt from commit to production)
- Writes infrastructure as code (Terraform, CloudFormation)
- Manages containers and orchestration (Docker, Kubernetes)
- Sets up monitoring, alerting, and logging
- Automates the toil — the boring manual work
- Handles security, secrets, and access control
- Is on-call when it breaks

## Related terms you'll hear

- **SRE (Site Reliability Engineering)** — Google's specific implementation of these ideas. Treats operations as a software problem, adds concrete tools: SLOs, error budgets, toil budgets. Think of it as "DevOps with the maths".
- **Platform Engineering** — building an internal platform so app teams can self-serve deploys without needing to know Kubernetes. The current evolution of the role.
- **GitOps** — Git is the single source of truth for infrastructure; an agent continuously reconciles reality to match the repo.
- **Shift Left** — move testing and security *earlier* in the process (leftward on a timeline), because bugs get exponentially more expensive the later you find them.

## Practice

No terminal needed for this one — these are the questions that make the concept stick.

1. **Think of a manual task you've done more than twice** (renaming files, copying a config, restarting something). Write down the exact steps. That written list *is* the first draft of a script — you'll automate it in the shell scripting section.

2. **Answer these in your own words** (write them down, don't just think them):
   - Why is deploying 10 times a day *safer* than deploying once a month?
   - Your teammate deploys a change that takes the site down. What does a blameless response look like versus a blameful one?
   - Which DORA metric would you improve first at a company that deploys once a quarter, and why?

3. **Find one real postmortem** and read it. Cloudflare, GitHub, and AWS publish excellent public ones. Notice: they describe the *system* failure and the fix, and almost never name an individual. Search for "Cloudflare outage postmortem".

---

**Next:** [SDLC & the DevOps lifecycle](02-sdlc-and-lifecycle.md)
