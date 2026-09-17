# DevOps Notes

My learning notes, written as I go. Every topic follows the same shape:

> **Concept** (what it is, in plain words) → **Commands** (what to type, what it does) → **Practice** (do it yourself)

---

## Roadmap

The order matters. Each stage assumes the one before it.

| # | Stage | Why it comes here | Status |
|---|-------|-------------------|--------|
| 01 | [DevOps Fundamentals](01-devops-fundamentals/) | What the job actually is, before any tool | ✅ |
| 02 | [Linux](02-linux/) | Every server, container, and CI runner is Linux | ✅ |
| 03 | [Networking](03-networking/) | Debugging deploys is 80% "why can't A reach B" | ✅ |
| 04 | [Git](04-git/) | The source of truth everything else triggers from | ✅ |
| 05 | [Shell Scripting](05-shell-scripting/) | Glue for all automation | ✅ |
| 06 | [Web Servers](06-web-servers/) | How traffic actually reaches your app | 🚧 |
| 07 | Docker | Package the app once, run it anywhere | ⬜ |
| 08 | CI/CD | Automate build → test → deploy | ⬜ |
| 09 | Cloud (AWS) | Where it all runs | ⬜ |
| 10 | Terraform (IaC) | Infrastructure defined in code, not clicks | ⬜ |
| 11 | Kubernetes | Run containers at scale | ⬜ |
| 12 | Monitoring & Logging | Know it broke before users tell you | ⬜ |

---

## Contents

### 01 — DevOps Fundamentals
- [What is DevOps](01-devops-fundamentals/01-what-is-devops.md)
- [SDLC & the DevOps lifecycle](01-devops-fundamentals/02-sdlc-and-lifecycle.md)

### 02 — Linux
- [Linux basics & filesystem](02-linux/01-basics-and-filesystem.md)
- [File permissions & ownership](02-linux/02-permissions.md)
- [Processes & services (systemd)](02-linux/03-processes-and-services.md)
- [Users, packages & disks](02-linux/04-users-packages-disks.md)
- [Text processing: grep, sed, awk](02-linux/05-text-processing.md)

### 03 — Networking
- [Networking basics for DevOps](03-networking/01-networking-basics.md)
- [SSH](03-networking/02-ssh.md)

### 04 — Git
- [Git basics](04-git/01-git-basics.md)
- [Branching & merging](04-git/02-branching-and-merging.md)
- [Remotes & workflows](04-git/03-remotes-and-workflows.md)

### 05 — Shell Scripting
- [Bash scripting basics](05-shell-scripting/01-bash-basics.md)

### 06 — Web Servers
- [Caddy](06-web-servers/01-caddy.md)

---

## How I study a topic

1. Read the concept section — don't touch the keyboard yet.
2. Type every command manually. Copy-paste teaches nothing.
3. Do the practice section without looking back at the notes.
4. Break it on purpose, then fix it. That's where the real learning is.

## Glossary

Quick-reference for terms that show up everywhere: [GLOSSARY.md](GLOSSARY.md)
