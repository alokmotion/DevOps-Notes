# Remotes & Team Workflows

## Remotes

A **remote** is a named URL pointing to another copy of the repository — usually on GitHub/GitLab. The default name for the one you cloned from is `origin` (just a convention, not a keyword).

```bash
git remote -v                                   # list remotes and their URLs
git remote add origin git@github.com:u/repo.git # add one
git remote set-url origin git@github.com:u/repo.git   # change the URL (e.g. HTTPS → SSH)
git remote remove origin
git remote show origin                          # detailed info about branches
```

**HTTPS vs SSH:**
```
https://github.com/user/repo.git   → prompts for credentials/token each time
git@github.com:user/repo.git       → uses your SSH key, no prompts
```
Use SSH for anything you work on regularly. (See [SSH](../03-networking/02-ssh.md).)

## fetch, pull, push

```bash
git fetch origin           # download remote changes; DON'T touch my working files
git pull origin main       # fetch + merge into my current branch
git pull --rebase origin main   # fetch + rebase instead of merge (linear history)
git push origin main       # upload my commits
git push -u origin feature/login    # push a NEW branch and set upstream tracking
git push                   # once upstream is set, this is enough
git push --tags            # push tags (they don't go automatically)
```

**`fetch` vs `pull` — an important distinction:**

- `git fetch` = "show me what's new on the remote" — updates your remote-tracking branches, changes nothing in your working directory. **Always safe.**
- `git pull` = `fetch` + `merge` — modifies your working directory immediately, and can create conflicts.

The careful habit:
```bash
git fetch origin
git log HEAD..origin/main --oneline    # what's coming?
git diff HEAD origin/main              # what will change?
git merge origin/main                  # now merge, knowing what you're getting
```

**`-u` (upstream)** links your local branch to a remote one so `git push` and `git pull` work with no arguments afterwards. You'll type `git push -u origin <branch>` once per new branch — Git's error message helpfully tells you the exact command.

## Force push — handle with care

```bash
git push --force                    # ⚠️ overwrites the remote. Can destroy others' work.
git push --force-with-lease         # ✅ refuses if someone else pushed since your last fetch
```

**Always use `--force-with-lease` instead of `--force`.** It's the same operation with a safety check: if the remote has commits you haven't seen, it aborts rather than obliterating your colleague's work.

Legitimate use: you rebased *your own* feature branch to clean it up, and need to update your PR. Never force-push to `main` — most teams protect the branch specifically to prevent this.

## Pull Requests / Merge Requests

A **PR** (GitHub) or **MR** (GitLab) is a request to merge your branch, plus a place to review it.

The flow:
```
1. git switch -c feature/thing        create a branch
2. ...work, commit...
3. git push -u origin feature/thing   push it
4. Open a PR on GitHub
5. CI runs automatically              ← tests, linting, security scans
6. Teammates review, comment
7. Address feedback, push more commits (the PR updates itself)
8. Approved + CI green → merge
9. Delete the branch
```

**Why PRs matter in DevOps:** the PR is the gate. It's where automated checks run *before* anything reaches `main`, and where a second person sees the change. It's also the audit record — many compliance regimes require exactly this: a reviewed, approved, traceable change.

**Branch protection rules** on `main` typically enforce:
- No direct pushes — everything goes through a PR
- At least one approving review
- CI must pass
- Branch must be up to date with `main`
- No force pushes

**Merge strategies at the button:**

| Strategy | Result | Good for |
|---|---|---|
| **Merge commit** | All commits kept + a merge commit | Full history, larger features |
| **Squash and merge** | All commits collapsed into one | Keeps `main` clean; most common default |
| **Rebase and merge** | Commits replayed, no merge commit | Strictly linear history |

Squash is the popular default: your branch's twelve "wip" commits become one tidy commit on `main`, which makes `git log` on main readable and `git revert` on a feature trivial.

## Branching strategies

### GitHub Flow (simple — start here)

```
main ────●────●────●────●────►  always deployable
          \        /
           ●──●──●   feature branch
```

- `main` is always deployable
- Branch off `main`, work, PR, merge back
- Deploy from `main` — often automatically on merge

Best for web apps with continuous deployment. It's simple, and simple wins. **Most teams should use this.**

### Git Flow (heavier)

Branches: `main` (production), `develop` (integration), `feature/*`, `release/*`, `hotfix/*`.

Suits software with scheduled versioned releases that must support multiple versions at once (desktop apps, on-prem software). For a web app deploying several times a day it's usually overhead you don't need — even its author has said so.

### Trunk-Based Development

Everyone commits to `main` (the trunk) at least daily; branches live for hours, not days. Unfinished work hides behind **feature flags**.

This is what elite-performing teams do. It requires strong automated testing and feature flags, but it maximises integration frequency — which is the actual point of CI.

**The common thread:** the shorter your branches live, the fewer conflicts and the smoother everything runs. Long-lived branches are the enemy.

## The everyday team workflow

```bash
# Start fresh
git switch main
git pull origin main                    # get the latest

# Branch
git switch -c feature/add-search

# Work
# ...edit...
git add -p                              # stage deliberately
git commit -m "Add search endpoint"

# Stay current (do this daily on a longer branch)
git fetch origin
git merge origin/main                   # or: git rebase origin/main

# Publish
git push -u origin feature/add-search

# → open the PR, CI runs, review happens

# Address feedback
git add . && git commit -m "Address review: validate input"
git push                                # PR updates automatically

# After merge, tidy up
git switch main
git pull origin main
git branch -d feature/add-search        # delete local
git push origin --delete feature/add-search   # delete remote (often automatic)
```

## Real situations you'll hit

**"Updates were rejected because the remote contains work you do not have"**
```bash
git pull origin main       # or: git pull --rebase origin main
# resolve any conflicts
git push origin main
```
Someone pushed while you were working. Pull first, then push.

**"Detached HEAD"** — you checked out a commit or tag rather than a branch. Commits made here belong to no branch and are easy to lose.
```bash
git switch main                 # just leave, if you made no commits
git switch -c new-branch        # or keep your work by making a branch here
```

**Committed to the wrong branch**
```bash
git log --oneline               # note your commit's hash
git reset --soft HEAD~1         # undo the commit, KEEP the changes staged
git stash
git switch correct-branch
git stash pop
git commit -m "message"
```

**Need someone else's branch locally**
```bash
git fetch origin
git switch feature/their-branch      # Git auto-tracks the remote branch
```

**Accidentally committed a secret and pushed it**
1. **Rotate the credential immediately.** This is step one and it is not optional — assume it's already scraped.
2. Then purge history with `git filter-repo` or BFG, and force-push (coordinate with the team — everyone must re-clone).
3. Add it to `.gitignore` and add a secret-scanning pre-commit hook.

Never assume a private repo makes it safe. Repos change visibility, get forked, and get cloned onto laptops.

## Git in CI/CD

Where all of this connects to the pipeline:

```yaml
# A GitHub Actions workflow, triggered by Git events
on:
  push:
    branches: [main]           # merge to main → deploy
    tags: ['v*']               # push a version tag → release
  pull_request:
    branches: [main]           # open a PR → run tests
```

Useful facts about Git in pipelines:
- CI usually does a **shallow clone** (`--depth 1`) for speed — so `git describe` and history-based tooling may need `fetch-depth: 0`.
- The **commit SHA** is the natural version identifier: tag your Docker image `myapp:$GIT_SHA` and you can always trace a running container back to exact source.
- **Tags trigger releases**, branches trigger builds.
- CI runs in **detached HEAD** at the merge commit — normal, not a problem.

## Practice

**1. Set up a real remote** (needs a GitHub account)
```bash
mkdir -p ~/remote-practice && cd ~/remote-practice
git init && echo "# Practice" > README.md
git add . && git commit -m "Initial commit"
```
Create an empty repo on GitHub, then:
```bash
git remote add origin git@github.com:YOUR_USER/practice.git
git remote -v
git push -u origin main
```
Refresh GitHub — your commit is there.

**2. fetch vs pull, felt directly**
Edit the README on GitHub's web UI and commit it there. Then locally:
```bash
git fetch origin
git status                       # "your branch is behind by 1 commit"
cat README.md                    # UNCHANGED — fetch didn't touch your files
git log HEAD..origin/main --oneline    # what's incoming
git diff HEAD origin/main              # exactly what will change
git merge origin/main
cat README.md                    # NOW it's updated
```
Write down, in one sentence each, what `fetch` did and what `merge` did.

**3. Full feature branch → PR cycle**
```bash
git switch -c feature/add-license
echo "MIT License" > LICENSE
git add . && git commit -m "Add MIT license"
git push -u origin feature/add-license
```
On GitHub: open a PR, read the diff view, merge it (try **Squash and merge**). Then locally:
```bash
git switch main
git pull origin main
git log --oneline                # note it's ONE commit, not your branch's history
git branch -d feature/add-license
```

**4. Create and resolve a real push rejection**
- Edit README on GitHub, commit there.
- Locally, edit the *same line* in README, commit.
- `git push` → rejected. Read the message carefully.
- `git pull origin main` → conflict. Resolve, commit, push.

This is the single most common day-to-day Git situation. Doing it once deliberately, calmly, is much better than meeting it for the first time under pressure.

**5. Branch protection**
In your GitHub repo: Settings → Branches → add a rule for `main` requiring a PR before merging. Then try `git push origin main` directly. Read the rejection. This is what every real repo does.

**6. Reason it out**
- Why is `--force-with-lease` safer than `--force`? Describe the scenario it protects against.
- Your team deploys 5x a day to a web app. GitHub Flow or Git Flow? Why?
- Why does squash-merging make `git revert` on `main` easier?
- A pipeline tags images with `myapp:latest`. Why is `myapp:$GIT_SHA` better?

**7. Clean up**
```bash
cd ~ && rm -rf ~/remote-practice
```
(And delete the GitHub repo if you don't want it.)

---

**Previous:** [Branching & merging](02-branching-and-merging.md) · **Next:** [Bash scripting](../05-shell-scripting/01-bash-basics.md)
