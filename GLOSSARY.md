# Glossary

Terms that show up constantly. Skim it now, come back when something's unfamiliar.

## Process & culture

| Term | Meaning |
|---|---|
| **DevOps** | Culture + practices where the people who build software also run it, supported by automation |
| **SRE** | Site Reliability Engineering — Google's implementation of DevOps; ops treated as a software problem |
| **Platform Engineering** | Building an internal platform so app teams can self-serve deploys |
| **GitOps** | Git as the single source of truth for infrastructure; an agent reconciles reality to match the repo |
| **Shift Left** | Move testing and security earlier in the process, where bugs are cheaper to fix |
| **Toil** | Manual, repetitive work that scales with usage and produces no lasting value. Automate it. |
| **Blameless postmortem** | Incident review that fixes the system rather than punishing a person |
| **Blast radius** | How much breaks if this change goes wrong |
| **Runbook** | Step-by-step instructions for handling a known operational situation |
| **On-call** | Rota for responding to production alerts |

## Delivery

| Term | Meaning |
|---|---|
| **CI** (Continuous Integration) | Merge to a shared branch frequently; every merge auto-builds and tests |
| **Continuous Delivery** | Every change is proven releasable; a human presses the deploy button |
| **Continuous Deployment** | Same, but no button — passing changes deploy automatically |
| **Pipeline** | The automated sequence: build → test → deploy |
| **Artifact** | The built output that gets deployed (jar, binary, Docker image) |
| **Build once, promote** | Build the artifact once and move that same one through environments; never rebuild per environment |
| **Rollback** | Return to the previous known-good version |
| **Blue-Green** | Two identical environments; flip all traffic at once. Instant rollback. |
| **Canary** | Send a small % of traffic to the new version first, watch, then increase |
| **Rolling deploy** | Replace instances a few at a time; no downtime |
| **Feature flag** | Ship code disabled; enable it later without deploying |
| **Smoke test** | Quick post-deploy check that the basics work |

## DORA metrics

| Term | Meaning |
|---|---|
| **Deployment Frequency** | How often you ship to production |
| **Lead Time for Changes** | Commit → running in production |
| **Change Failure Rate** | % of deploys that cause a problem |
| **MTTR** | Mean Time To Recovery — how fast you recover from failure |

## Linux

| Term | Meaning |
|---|---|
| **Shell** | The program interpreting your commands (bash, zsh) |
| **PID / PPID** | Process ID / Parent Process ID |
| **Daemon** | A long-running background service |
| **systemd / systemctl** | The service manager on modern Linux / its control command |
| **journalctl** | Reads systemd's logs |
| **SIGTERM (15)** | "Please stop cleanly" — the polite kill; can be caught |
| **SIGKILL (9)** | "Stop now" — cannot be caught; no cleanup, risks data loss |
| **Zombie process** | Finished process whose parent hasn't collected its status |
| **Load average** | Average processes wanting CPU over 1/5/15 min — compare to core count |
| **stdin / stdout / stderr** | Streams 0 / 1 / 2 — input, normal output, errors |
| **Pipe (`\|`)** | Feed one command's output into the next |
| **Inode** | Filesystem record per file; you can run out of these while disk space remains |
| **umask** | Default permission mask for newly created files |
| **SUID** | Bit making a program run with its owner's privileges |
| **Symlink** | A pointer to another file |
| **`/etc`** | All configuration | 
| **`/var/log`** | All logs — first stop when debugging |

## Networking

| Term | Meaning |
|---|---|
| **DNS** | Name → IP address translation |
| **TTL** | How long a DNS record is cached. Lower it before a migration. |
| **A record / CNAME** | Name → IP / name → another name |
| **CIDR** | `10.0.0.0/16` notation for an address range |
| **`0.0.0.0/0`** | "The entire internet" — recognise this instantly in firewall rules |
| **`127.0.0.1`** | Localhost — this machine only |
| **`0.0.0.0` (bind)** | All interfaces — reachable from outside. The container networking gotcha. |
| **Port** | Picks the service on a host (22 SSH, 80 HTTP, 443 HTTPS, 5432 Postgres) |
| **NAT** | Maps private addresses to a shared public one |
| **Reverse proxy** | Sits in front of your app; terminates TLS, routes requests (nginx) |
| **Load balancer** | Distributes traffic across healthy backends |
| **Health check** | Periodic request that decides whether an instance gets traffic |
| **502 / 503 / 504** | Bad gateway (backend down) / no healthy backends / backend too slow |
| **Bastion / jump host** | Hardened public server you SSH through to reach private ones |
| **Security Group** | Cloud-level firewall attached to an instance |
| **TLS/SSL** | Encryption for connections; the S in HTTPS |

## Git

| Term | Meaning |
|---|---|
| **Repository** | A project plus its full history |
| **Commit** | A snapshot with a message, author and unique hash |
| **Branch** | A movable pointer to a commit |
| **HEAD** | Where you currently are |
| **Detached HEAD** | You're on a commit, not a branch — commits here are easy to lose |
| **Staging area / index** | Holding pen for what goes in the next commit |
| **Remote / origin** | Another copy of the repo / the conventional name for the main one |
| **fetch vs pull** | Download only (safe) vs download + merge (changes your files) |
| **Merge** | Combine branches; may create a merge commit |
| **Rebase** | Replay commits elsewhere for linear history — **never on shared branches** |
| **Fast-forward** | Merge that just moves the pointer; no merge commit |
| **Conflict** | Both branches changed the same lines; you decide |
| **Cherry-pick** | Copy one specific commit to another branch |
| **Tag** | Immutable marker, usually a release (`v1.2.0`) |
| **SemVer** | MAJOR.MINOR.PATCH — breaking / feature / fix |
| **PR / MR** | Pull Request / Merge Request — the review-and-CI gate |
| **`--force-with-lease`** | Safe force push; aborts if someone else pushed |
| **reflog** | Log of everywhere HEAD has been — recovers "lost" commits |
| **Trunk-based development** | Everyone commits to main daily; branches live hours |

## Bash

| Term | Meaning |
|---|---|
| **Shebang** | `#!/usr/bin/env bash` — which interpreter runs the file |
| **`set -euo pipefail`** | Exit on error / on undefined variable / on any pipeline failure. Use always. |
| **Exit code** | 0 = success, nonzero = failure. What CI checks. |
| **`$?`** | Exit code of the last command |
| **`$@`** | All arguments — always quote it: `"$@"` |
| **`trap ... EXIT`** | Run cleanup whenever the script exits, including on error or Ctrl+C |
| **shellcheck** | Static analyser for shell scripts. Install it. |
| **cron** | Time-based job scheduler; minimal PATH, redirect output to a log |
| **Idempotent** | Safe to run repeatedly with the same result |

## Coming later in the roadmap

| Term | Meaning |
|---|---|
| **Container** | Isolated process with its own filesystem, sharing the host kernel |
| **Image** | The immutable template a container is started from |
| **Dockerfile** | Recipe for building an image |
| **Registry** | Where images are stored (Docker Hub, ECR) |
| **Orchestration** | Running containers across many machines (Kubernetes) |
| **Pod** | Smallest deployable unit in Kubernetes; one or more containers |
| **IaC** | Infrastructure as Code — define infrastructure in files (Terraform) |
| **Idempotence (IaC)** | Applying the same config repeatedly converges to the same state |
| **State file** | Terraform's record of what it created |
| **Immutable infrastructure** | Replace servers rather than modifying them |
| **Observability** | Metrics + logs + traces — understanding a system from its outputs |
| **SLI / SLO / SLA** | Indicator (measurement) / Objective (internal target) / Agreement (contract) |
| **Error budget** | Allowed unreliability under your SLO; spend it on shipping |
| **Secret management** | Storing credentials safely (Vault, AWS Secrets Manager) — never in Git |
