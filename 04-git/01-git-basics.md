# Git Basics

## Why Git is central to DevOps

Git isn't just for source code. In modern DevOps, **Git is the source of truth for everything**:

- Application code
- Infrastructure (Terraform files)
- Pipeline definitions (`.github/workflows/`, `Jenkinsfile`)
- Kubernetes manifests
- Configuration

A `git push` is what *triggers* your pipeline. Every deploy traces back to a commit. When something breaks in production, "which commit did this?" is the first question — and the answer is a rollback.

That's the whole idea behind **GitOps**: if it's in Git, it's reviewable, revertible, and auditable.

## The mental model — three areas

This is the concept everything else depends on. Get it clear and Git stops feeling random.

```
  Working Directory  ──git add──►  Staging Area  ──git commit──►  Repository
   (your files,                    (marked for                   (permanent
    edited)                         the next commit)              history)
        │                                                              │
        └──────────────────── git checkout / restore ◄─────────────────┘
                                                                       │
                                                              git push │
                                                                       ▼
                                                                Remote (GitHub)
```

| Area | What it is |
|---|---|
| **Working Directory** | The actual files you're editing right now |
| **Staging Area (index)** | A holding pen — what you've chosen to include in the next commit |
| **Repository (.git)** | The committed history, stored locally |
| **Remote** | The shared copy on GitHub/GitLab |

**Why a staging area?** It lets you commit *some* of your changes. You fixed a bug and also renamed a variable — stage and commit them separately, so each commit is one logical change. That makes history readable and makes reverting a single change possible.

Git is **distributed**: your local `.git` holds the complete history. You can commit, branch, and view logs with no network at all. Only `push`, `pull`, `fetch`, and `clone` need the remote.

## One-time setup

```bash
git config --global user.name "Alok"
git config --global user.email "raj@mbgcard.com"
git config --global init.defaultBranch main
git config --global core.editor "vim"        # or "code --wait" for VS Code
git config --global pull.rebase false        # merge on pull (explicit is better)
git config --list                            # check everything
```

Your name and email get baked into every commit permanently. Set them before your first commit.

Handy aliases:
```bash
git config --global alias.st status
git config --global alias.lg "log --oneline --graph --decorate --all"
```

## Starting a repository

```bash
git init                              # turn this directory into a repo
git clone https://github.com/u/repo.git      # copy an existing one
git clone git@github.com:u/repo.git          # via SSH (no password prompts)
git clone --depth 1 <url>             # shallow: latest commit only, fast (used in CI)
```

`--depth 1` is standard in pipelines — CI doesn't need ten years of history to build one commit.

## The core daily loop

```bash
git status                # WHAT CHANGED? ← run this constantly, it's your compass
git add file.txt          # stage one file
git add .                 # stage everything in this directory and below
git add -p                # stage interactively, hunk by hunk ← makes clean commits
git commit -m "Fix login timeout"
git commit -am "message"  # add all TRACKED files and commit (skips new files)
git log                   # history
git push                  # send to remote
```

**`git status` is the most important command in Git.** It tells you what's changed, what's staged, which branch you're on, and — genuinely useful — it suggests the exact command for what you probably want to do next. Run it before and after everything.

`git add -p` walks you through each change and asks whether to stage it. It's how you produce commits that each do one thing, and it makes you re-read your own diff before committing — you'll catch debug statements this way.

## Writing good commit messages

```
Add retry logic to payment webhook

The payment provider intermittently returns 503 during their
maintenance window, causing failed orders. Retry 3 times with
exponential backoff before giving up.

Fixes #142
```

Rules that matter:
1. **First line ≤ 50 chars, imperative mood** — "Add feature", not "Added feature" or "Adds feature". Read it as *"this commit will… Add feature"*.
2. **Blank line**, then the body.
3. **Explain WHY, not what.** The diff already shows what changed. Six months from now, the *why* is the only thing you can't reconstruct.

Bad: `fix`, `update`, `changes`, `asdf`, `final fix v2`.

You will read your own commit messages during an incident at 2am. Write for that person.

## Viewing history

```bash
git log                             # full
git log --oneline                   # one line per commit ← the everyday one
git log --oneline --graph --all     # visual branch graph
git log -5                          # last 5
git log --author="Alok"
git log --since="2 days ago"
git log -- path/to/file             # history of one file
git log -p file.txt                 # history WITH the diffs
git log --grep="login"              # search commit messages
git show <commit-hash>              # everything about one commit
git blame file.txt                  # who last changed each line, and in which commit
```

**`git blame` isn't for blaming people** — it's for finding the commit that introduced a line, so you can read its message and understand *why* the line exists. It's a research tool.

## Seeing what changed

```bash
git diff                  # working directory vs staging (UNSTAGED changes)
git diff --staged         # staging vs last commit (what you're ABOUT to commit)
git diff HEAD             # everything not yet committed
git diff main feature-x   # between branches
git diff <hash1> <hash2>  # between commits
git diff --stat           # summary: files changed, lines added/removed
```

**Run `git diff --staged` before every commit.** It shows exactly what you're about to record. This habit catches accidentally-committed secrets, debug prints, and stray files.

## Undoing things — the important table

The scary part of Git. The decision is: *has it been pushed yet?*

```bash
# ── Not yet staged: discard file changes ──
git restore file.txt              # throw away edits (modern)
git checkout -- file.txt          # same, older syntax
git restore .                     # discard ALL uncommitted changes ⚠️ unrecoverable

# ── Staged but not committed: unstage ──
git restore --staged file.txt     # unstage, keep the edits
git reset HEAD file.txt           # same, older syntax

# ── Committed but not pushed ──
git commit --amend -m "better message"     # fix the LAST commit's message
git commit --amend --no-edit               # add forgotten files to the last commit
git reset --soft HEAD~1           # undo commit, KEEP changes staged
git reset --mixed HEAD~1          # undo commit, keep changes unstaged (default)
git reset --hard HEAD~1           # undo commit and DELETE the changes ⚠️

# ── Already pushed: NEVER rewrite. Revert instead. ──
git revert <commit-hash>          # creates a NEW commit that undoes that one ✅
```

**The rule that prevents disasters:**

> **Never rewrite history that others have pulled.** Once it's pushed to a shared branch, use `git revert`, not `reset` or `amend`.

`reset`/`amend` change the commit hashes. If a teammate has already pulled those commits, their history and yours diverge, and fixing it is genuinely painful for everyone. `revert` adds a new commit that undoes the change — safe, honest, and it preserves the record that the change happened and was rolled back.

**`--soft` vs `--mixed` vs `--hard`:**

| Flag | Commit undone | Changes staged | Changes kept |
|---|---|---|---|
| `--soft` | ✓ | ✓ | ✓ |
| `--mixed` | ✓ | ✗ | ✓ |
| `--hard` | ✓ | ✗ | ✗ **gone** |

`--soft HEAD~1` is the one you want when you committed too early and want to add more.

**`git reflog` is your safety net.** It logs every position HEAD has been in, including commits you "lost" with a bad reset. Almost nothing committed is truly unrecoverable for ~90 days:
```bash
git reflog                  # find the hash you were at
git reset --hard <hash>     # go back to it
```
Remember this exists — it turns a panic into a two-minute fix.

## .gitignore

Files Git should never track.

```gitignore
# Dependencies
node_modules/
venv/

# Secrets ← THE important part
.env
*.pem
*.key
credentials.json

# Build output
dist/
build/
*.log

# OS / editor cruft
.DS_Store
.idea/
.vscode/
```

**`.gitignore` only affects untracked files.** If a file is already committed, adding it to `.gitignore` changes nothing:
```bash
git rm --cached .env      # stop tracking it, keep the local file
```

**Committed secrets are the classic disaster.** Once a password or key is pushed, it's in the history forever — deleting it in a later commit does *not* remove it, anyone can `git log -p` and find it. If a repo is public, bots scrape and use exposed AWS keys within *minutes*.

The response is not "remove it in the next commit". It is:
1. **Rotate/revoke the credential immediately** — assume it's compromised.
2. Then purge it from history (`git filter-repo`, or BFG Repo-Cleaner).

Prevention is much cheaper: `.gitignore` your `.env` before the first commit, and consider a pre-commit hook with `gitleaks` or `git-secrets`.

## Stashing

Park uncommitted work temporarily.

```bash
git stash                       # shelve all changes, clean the working directory
git stash -u                    # include untracked files
git stash save "wip: login"     # with a label
git stash list                  # see all stashes
git stash pop                   # restore the most recent AND remove it from the list
git stash apply                 # restore but KEEP it in the list
git stash drop                  # delete the top stash
git stash clear                 # delete all ⚠️
```

The situation: you're mid-change and need to urgently switch branches for a hotfix. `git stash`, fix, switch back, `git stash pop`.

Don't leave things stashed for days — stashes are invisible and easy to forget. Prefer a WIP commit on a branch for anything longer than an hour.

## Practice

**1. Build a repo from nothing**
```bash
mkdir -p ~/git-practice && cd ~/git-practice
git init
git status                      # read this carefully — what is it telling you?

echo "# My Project" > README.md
git status                      # README is "untracked"
git add README.md
git status                      # now "staged" — note the wording change
git commit -m "Add project README"
git log --oneline
```

**2. Watch the three areas**
```bash
echo "line 1" >> README.md
git status                      # modified, not staged
git diff                        # ← unstaged changes
git add README.md
git diff                        # empty now! why?
git diff --staged               # ← there it is
git commit -m "Add a line to README"
```
Explain in one sentence why `git diff` went empty after `git add`.

**3. Selective staging**
```bash
echo "feature code" > feature.txt
echo "unrelated fix" > fix.txt
git add feature.txt
git commit -m "Add feature"     # only feature.txt is in this commit
git status                      # fix.txt still waiting
git add fix.txt
git commit -m "Add unrelated fix"
git log --oneline               # two clean, separate commits
```

**4. Undo, at each stage**
```bash
# discard an unstaged edit
echo "oops" >> README.md
git diff
git restore README.md
git diff                        # gone

# unstage
echo "oops2" >> README.md
git add README.md
git restore --staged README.md
git status                      # modified but unstaged
git restore README.md

# fix the last commit's message
git commit --amend -m "Add unrelated fix (typo corrected)"
git log --oneline
```

**5. Revert vs reset — do both and compare**
```bash
echo "bad change" > bad.txt
git add bad.txt && git commit -m "Add bad change"
git log --oneline               # note the hash

git revert HEAD --no-edit       # creates a NEW commit undoing it
git log --oneline               # BOTH commits are visible — history preserved
ls                              # bad.txt is gone
```
Now articulate: why is this the correct choice for a commit already pushed to `main`?

**6. Recover from a "disaster" with reflog**
```bash
git log --oneline               # note the top hash
git reset --hard HEAD~2         # nuke the last two commits
git log --oneline               # they're gone!
git reflog                      # find the hash from before the reset
git reset --hard <that-hash>    # restored
git log --oneline
```
This is the exercise that removes the fear of Git. Do it at least twice.

**7. .gitignore, including the trap**
```bash
echo "SECRET=abc123" > .env
git status                      # .env shows up — dangerous
echo ".env" > .gitignore
git status                      # ignored now ✓
git add .gitignore && git commit -m "Add gitignore"

# Now the trap:
git add -f .env && git commit -m "oops"    # forced it in
echo "irrelevant" >> .gitignore
git status                      # .env is TRACKED now; gitignore doesn't help
git rm --cached .env
git commit -m "Stop tracking .env"
```
Then answer: the commit containing the secret is still in your history. What two things must you actually do if this had been pushed to a public repo?

**8. Stash**
```bash
echo "half-finished work" >> README.md
git stash
git status                      # clean
git stash list
git stash pop
git status                      # your work is back
```

---

**Next:** [Branching & merging](02-branching-and-merging.md)
