# SDLC & the DevOps Lifecycle

## SDLC — Software Development Life Cycle

The stages every piece of software goes through, regardless of methodology:

**Plan → Design → Develop → Test → Deploy → Maintain**

### Waterfall (the old way)

Each stage finishes completely before the next begins.

```
Plan ──► Design ──► Develop ──► Test ──► Deploy
(1mo)     (1mo)      (4mo)      (2mo)    (1wk)
```

**Problem:** you find out in month 8 that you built the wrong thing. Feedback arrives far too late to be useful, and the cost of change is highest exactly when you discover you need it.

### Agile (iterate)

Same stages, but in 1–2 week loops delivering a working slice each time.

**Fixed what:** feedback now arrives every two weeks instead of every eight months.
**Didn't fix:** Agile made *development* fast, but deployment stayed a slow, manual, scary event. You'd have a feature "done" in a 2-week sprint that sat unreleased for three months.

That gap between "dev is done" and "users have it" is exactly the gap DevOps closes.

## The DevOps lifecycle (the infinity loop)

You'll see this diagram everywhere. It's a loop with no end because software is never "finished".

```
        ┌──────────────────────────────────────┐
        │                                      │
    PLAN ──► CODE ──► BUILD ──► TEST ──► RELEASE
        ▲                                      │
        │                                      ▼
   MONITOR ◄── OPERATE ◄── DEPLOY ◄────────────┘

    └── Dev side ──┘        └── Ops side ──┘
```

| Stage | What happens | Typical tools |
|---|---|---|
| **Plan** | Decide what to build; track work | Jira, Linear, GitHub Issues |
| **Code** | Write it; review it | Git, GitHub/GitLab |
| **Build** | Compile, package, build the image | Maven, npm, Docker |
| **Test** | Unit, integration, security scans | Jest, pytest, JUnit, Trivy, SonarQube |
| **Release** | Approve & version the artifact | GitHub Actions, Jenkins, artifact registries |
| **Deploy** | Push it to an environment | Argo CD, Helm, Terraform, Ansible |
| **Operate** | Keep it running, scale it | Kubernetes, cloud provider |
| **Monitor** | Watch metrics, logs, alerts | Prometheus, Grafana, Loki, Datadog |

The critical part is the arrow from **Monitor back to Plan**. Production data tells you what to build next. Without that arrow it's not a loop, it's just a conveyor belt.

## The key distinction: CI vs CD vs CD

Three terms, constantly confused. Learn them properly now — they come up in every interview.

### Continuous Integration (CI)

> Every developer merges their work into the shared main branch frequently (at least daily), and every merge automatically triggers a build + test run.

**The problem it solves:** "merge hell". If you work on a branch for three weeks, merging it back is a nightmare of conflicts. Integrating small changes daily keeps each merge trivial.

**What it looks like:** you push → a pipeline runs → tests pass or fail → you know within 10 minutes.

### Continuous Delivery (CD)

> Every change that passes CI is automatically prepared and proven to be releasable. Deploying to production is a **button someone chooses to press**.

The artifact is built, tested, and sitting ready. The only thing standing between it and production is a human decision — usually for business reasons (marketing timing, a compliance sign-off), not technical ones.

### Continuous Deployment (CD)

> Same as above, but **there is no button**. If it passes all the automated gates, it goes to production automatically.

No human in the loop at all. This requires serious confidence in your test suite and your ability to roll back.

**Memory aid:**

```
Continuous Delivery   = ready to deploy, human presses the button
Continuous Deployment = deploys itself, no human involved
```

Both are abbreviated "CD", so in conversation people say the full words when it matters.

## Environments

Code moves through a series of environments, each closer to reality:

| Environment | Purpose | Who uses it | Real data? |
|---|---|---|---|
| **Local / Dev** | Your laptop; write and debug | You | No, fake data |
| **Test / QA** | Automated + manual testing | QA, CI pipeline | No, seeded data |
| **Staging / UAT** | A production clone; final check | QA, stakeholders | Anonymised copy |
| **Production (Prod)** | The real thing | Real users | Yes — be careful |

**The rule:** staging should mirror production as closely as possible. Every difference between them is a place where a bug can hide until it reaches real users. Different OS version, different database version, different config — that's where the "worked in staging" outages come from.

**Promotion:** build the artifact **once**, then promote that *same* artifact through each environment. Never rebuild per environment — if you rebuild for prod, you're deploying something that was never tested. Config changes per environment; the artifact does not.

## Deployment strategies

How you get new code into production without breaking it:

| Strategy | How it works | Trade-off |
|---|---|---|
| **Recreate** | Stop old version, start new one | Simple, but causes downtime |
| **Rolling** | Replace instances a few at a time | No downtime; both versions run briefly |
| **Blue-Green** | Two identical environments; flip traffic from blue to green all at once | Instant rollback (flip back); costs 2x infrastructure |
| **Canary** | Send 5% of traffic to the new version, watch metrics, then increase | Safest; limits blast radius; needs good monitoring |
| **Feature flags** | Ship the code disabled, turn it on for specific users later | Decouples deploy from release; adds code complexity |

**Blast radius** is the term for "how much breaks if this goes wrong". Every one of these strategies is about shrinking it.

## Practice

1. **Map an app you know** (any app on your phone) onto the eight lifecycle stages. Who do you imagine does each stage? Which stage do you think is slowest at most companies, and why?

2. **Explain the CI / Continuous Delivery / Continuous Deployment difference out loud**, as if to a friend who doesn't code. If you get stuck, you've found the bit to re-read. This is a genuinely common interview question.

3. **Pick a deployment strategy for each scenario** and justify it in one sentence:
   - A banking app pushing a change to the payment logic
   - An internal dashboard used by 12 people
   - A social app testing a redesigned feed with real users
   - A legacy system that can only run one instance at a time

4. **Think through a failure:** your team builds the artifact separately for staging and for production. Describe a concrete bug that could reach users because of this. (This is why "build once, promote" is a rule.)

---

**Previous:** [What is DevOps](01-what-is-devops.md) · **Next:** [Linux basics](../02-linux/01-basics-and-filesystem.md)
