# Branching & Merging

## Concept

A **branch** is just a movable pointer to a commit. That's genuinely all it is — a 41-byte file containing a hash. This is why branching in Git is instant and cheap, unlike older version control systems where a branch meant copying the whole codebase.

```
        A───B───C───D  main
                 \
                  E───F  feature/login
```

`main` points at D. `feature/login` points at F. `HEAD` points at whichever branch you're currently on.

**Why branch:** work on a feature without destabilising `main`. `main` stays deployable at all times; your half-finished work lives on its own branch until it's ready and reviewed.

## Branch commands

```bash
git branch                        # list local branches (* marks current)
git branch -a                     # include remote branches
git branch -v                     # with last commit of each
git branch feature/login          # create (but stay where you are)
git switch feature/login          # switch to it (modern)
git switch -c feature/login       # create AND switch ← what you'll actually use
git checkout -b feature/login     # same thing, older syntax
git switch main                   # back to main
git switch -                      # previous branch (like cd -)
git branch -d feature/login       # delete (safe: refuses if unmerged)
git branch -D feature/login       # force delete ⚠️
git branch -m new-name            # rename current branch
```

`switch`/`restore` are the newer, clearer commands: `checkout` historically did both jobs (switching branches *and* discarding file changes), which caused real accidents. Use `switch` for branches and `restore` for files.

**Naming conventions** (pick one, be consistent):
```
feature/user-authentication
bugfix/login-timeout
hotfix/payment-crash
release/v2.1.0
chore/update-deps
```
Many teams wire CI to the prefix — e.g. `hotfix/*` triggers a fast-track pipeline.

## Merging

Bring another branch's work into yours.

```bash
git switch main            # 1. go to the branch that RECEIVES the changes
git merge feature/login    # 2. merge the other branch INTO it
```

Direction matters and is a common confusion: you switch to the *destination* first.

### Fast-forward merge

If `main` hasn't moved since you branched, Git just slides the pointer forward:

```
Before:  A───B───C  main
                  \
                   D───E  feature

After:   A───B───C───D───E  main, feature
```

No merge commit — the history stays linear.

### Three-way merge

If both branches have new commits, Git creates a **merge commit** with two parents:

```
Before:  A───B───C───F  main
                  \
                   D───E  feature

After:   A───B───C───F───M  main
                  \     /
                   D───E
```

```bash
git merge --no-ff feature/login    # FORCE a merge commit even when fast-forward is possible
```

`--no-ff` is a common team policy: it keeps a visible record that "these commits were a feature", so you can revert the whole feature as a unit. Fast-forward loses that grouping.

## Merge conflicts

A conflict happens when both branches changed **the same lines of the same file**. Git can't decide, so it asks you.

Conflicts are normal. They are not an error or a sign you did something wrong.

```bash
git merge feature/login
# CONFLICT (content): Merge conflict in app.py
# Automatic merge failed; fix conflicts and then commit the result.
```

The file now contains markers:

```
<<<<<<< HEAD
timeout = 30          ← what's on YOUR current branch (main)
=======
timeout = 60          ← what's on the branch you're MERGING IN (feature/login)
>>>>>>> feature/login
```

**Resolving:**
1. `git status` — lists every conflicted file.
2. Open each one. Decide what the code *should* be — it might be either side, or a combination, or something new.
3. **Delete all three marker lines** (`<<<<<<<`, `=======`, `>>>>>>>`).
4. `git add <file>` — this is how you tell Git "resolved".
5. `git commit` (message is pre-filled).

```bash
git status                    # which files conflict?
git diff                      # see the conflicts
# ...edit files...
git add app.py
git commit                    # complete the merge
git merge --abort             # ← bail out entirely, back to before the merge
```

**`git merge --abort` is the escape hatch.** If a merge goes badly, this returns you to exactly where you were. Knowing it exists makes merging much less stressful.

**Leaving a marker in the file** is the classic beginner mistake — `<<<<<<< HEAD` gets committed and the code won't even parse. Search for `<<<<` before you commit, and let CI catch it as a backstop.

**Reducing conflicts:** merge `main` into your branch frequently rather than letting it drift for weeks. Small, short-lived branches conflict far less — which is the whole argument for Continuous Integration.

## Rebase

An alternative to merging that **rewrites history to be linear**.

```
Merge:                          Rebase:
A───B───C───F───M  main         A───B───C───F───D'───E'  main
          \     /                            
           D───E                (D and E REPLAYED on top of F,
                                 as brand-new commits D' and E')
```

```bash
git switch feature/login
git rebase main              # replay my commits on top of the latest main
git rebase --abort           # bail out
git rebase --continue        # after resolving a conflict
```

**Merge vs rebase:**

| | Merge | Rebase |
|---|---|---|
| History | True, with branch points | Linear, clean |
| Commit hashes | Unchanged | **Rewritten** |
| Safe on shared branches | ✅ Yes | ❌ **No** |
| Extra commit | Yes (merge commit) | No |
| Conflicts | Once | Potentially once per commit |

**THE GOLDEN RULE OF REBASE:**

> **Never rebase commits that exist outside your local repository.**

Rebasing changes commit hashes. If someone else has those commits, their history and yours are now incompatible, and the cleanup is unpleasant for everyone. Rebase your *own unpushed* branch to tidy it before opening a PR — never rebase `main` or a branch a colleague is also working on.

### Interactive rebase — cleaning up before a PR

```bash
git rebase -i HEAD~3        # edit the last 3 commits
```

An editor opens:
```
pick a1b2c3d Add login form
pick d4e5f6g fix typo
pick h7i8j9k fix typo again
```

Change the verbs:

| Command | Effect |
|---|---|
| `pick` | keep as-is |
| `reword` | change the message |
| `squash` | merge into the previous commit, combine messages |
| `fixup` | merge into the previous, **discard** this message |
| `drop` | delete the commit |
| `edit` | pause to amend it |

```
pick a1b2c3d Add login form
fixup d4e5f6g fix typo
fixup h7i8j9k fix typo again
```

Result: one clean commit instead of three, with the two "fix typo" commits absorbed. This is how you turn a messy work-in-progress branch into a reviewable history before opening a pull request.

## Cherry-pick

Take **one specific commit** from another branch.

```bash
git cherry-pick <commit-hash>
git cherry-pick <hash1> <hash2>
git cherry-pick -n <hash>          # apply without committing
```

The real use case: a hotfix was committed to `develop`, and you need exactly that fix on `release/v2.0` without pulling in everything else on develop.

Use it sparingly — it duplicates commits, so the same change ends up in history twice with different hashes, which can confuse later merges.

## Tags

Named markers for releases. Unlike branches, tags don't move.

```bash
git tag                                  # list
git tag v1.0.0                           # lightweight tag
git tag -a v1.0.0 -m "Release 1.0.0"     # ANNOTATED — has author, date, message ← use this
git tag -a v1.0.0 <commit-hash>          # tag an older commit
git show v1.0.0
git push origin v1.0.0                   # tags are NOT pushed by default!
git push origin --tags                   # push all tags
git tag -d v1.0.0                        # delete locally
git push origin --delete v1.0.0          # delete on remote
git checkout v1.0.0                      # look at the code at that release
```

**Tags are how releases work in DevOps.** Pushing tag `v1.2.0` typically triggers a pipeline that builds an artifact, tags a Docker image `myapp:v1.2.0`, and deploys it. The tag ties a running production version to an exact commit — which is what makes "roll back to the previous release" a precise operation.

**Semantic versioning** — `MAJOR.MINOR.PATCH`:

| Bump | When |
|---|---|
| **MAJOR** (2.0.0) | Breaking change — consumers must update their code |
| **MINOR** (1.3.0) | New feature, backwards compatible |
| **PATCH** (1.2.4) | Bug fix, backwards compatible |

**A forgotten `git push --tags` is a classic:** the release tag exists on your laptop, the pipeline never fires, and you spend twenty minutes wondering why the deploy didn't start.

## Practice

**1. Branch and fast-forward merge**
```bash
mkdir -p ~/branch-practice && cd ~/branch-practice
git init && echo "# App" > README.md
git add . && git commit -m "Initial commit"

git switch -c feature/greeting
echo "print('hello')" > app.py
git add . && git commit -m "Add greeting"
git log --oneline --graph --all      # see the branch

git switch main
git merge feature/greeting           # fast-forward — no merge commit
git log --oneline --graph --all      # note: perfectly linear
git branch -d feature/greeting
```

**2. Three-way merge**
```bash
git switch -c feature/farewell
echo "print('bye')" > bye.py
git add . && git commit -m "Add farewell"

git switch main
echo "## Docs" >> README.md
git add . && git commit -m "Update README"     # main moved too!

git merge feature/farewell            # now you get a MERGE COMMIT
git log --oneline --graph --all       # see the diamond shape
```

**3. Create a conflict on purpose and resolve it** — the most valuable exercise here
```bash
echo "timeout = 30" > config.py
git add . && git commit -m "Add config"

git switch -c feature/timeout
echo "timeout = 60" > config.py
git add . && git commit -m "Increase timeout to 60"

git switch main
echo "timeout = 45" > config.py
git add . && git commit -m "Increase timeout to 45"

git merge feature/timeout             # CONFLICT
git status                            # read what it tells you
cat config.py                         # see the markers
```
Now: open `config.py`, pick a value, delete all three marker lines, then `git add config.py && git commit`. Verify with `git log --oneline --graph --all`.

Then do it again and use `git merge --abort` instead — confirm you're back exactly where you started.

**4. Interactive rebase — clean up a messy branch**
```bash
git switch -c feature/messy
echo "a" > f.txt && git add . && git commit -m "Add feature"
echo "b" >> f.txt && git add . && git commit -m "fix"
echo "c" >> f.txt && git add . && git commit -m "fix again"
git log --oneline                     # three ugly commits

git rebase -i HEAD~3
# change the 2nd and 3rd lines from 'pick' to 'fixup', save and close
git log --oneline                     # ONE clean commit
```
Note the commit hash changed — this is exactly why you must not do this to pushed commits.

**5. Tags**
```bash
git switch main
git tag -a v1.0.0 -m "First release"
git tag
git show v1.0.0                       # who, when, which commit
echo "new feature" >> README.md && git add . && git commit -m "Add feature"
git tag -a v1.1.0 -m "Second release"
git log --oneline --decorate          # tags visible in history
git checkout v1.0.0                   # time travel — note the detached HEAD warning
git switch main                       # back to normal
```

**6. Cherry-pick**
```bash
git switch -c hotfix
echo "critical fix" > fix.txt && git add . && git commit -m "Critical security fix"
git log --oneline                     # copy the hash
git switch main
git cherry-pick <that-hash>
ls                                    # fix.txt is here
git log --oneline                     # the commit exists here too, with a NEW hash
```

**7. Reason it out**
- Why is rebasing a shared branch dangerous? Describe concretely what breaks for a teammate.
- When would you choose `--no-ff` over a fast-forward merge?
- You tagged `v2.0.0` and pushed, but the release pipeline never ran. What's the most likely cause?
- Your feature branch is three weeks old and merging is a mess of conflicts. What should you have done differently?

**8. Clean up**
```bash
cd ~ && rm -rf ~/branch-practice
```

---

**Previous:** [Git basics](01-git-basics.md) · **Next:** [Remotes & workflows](03-remotes-and-workflows.md)
