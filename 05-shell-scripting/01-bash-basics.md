# Bash Scripting Basics

## Why

Automation is the core of DevOps, and bash is the glue. Deploy scripts, health checks, backups, container entrypoints, CI steps — all bash. The rule of thumb: **if you've done it manually twice, script it the third time.**

Bash isn't a great general-purpose language, and that's fine. Use it for orchestrating other commands. When a script exceeds ~200 lines or needs real data structures, switch to Python.

## Your first script

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "Hello, $(whoami)!"
echo "Running on $(hostname) at $(date)"
```

```bash
chmod +x script.sh      # make it executable (see permissions notes)
./script.sh             # run it
bash script.sh          # run it without the executable bit
```

### The shebang

`#!/usr/bin/env bash` on line 1 tells the OS which interpreter to use. `env bash` finds bash on `$PATH` — more portable than hardcoding `#!/bin/bash`, which matters on macOS (where `/bin/bash` is an ancient version 3.2) and in minimal containers.

If you genuinely only need POSIX features, use `#!/bin/sh` — but then don't use bash-only syntax like arrays or `[[ ]]`.

### `set -euo pipefail` — put this in every script

This one line prevents most bash disasters:

| Flag | Effect |
|---|---|
| `set -e` | **Exit immediately if any command fails.** Without it, the script blindly carries on after an error. |
| `set -u` | **Error on undefined variables.** Catches typos. |
| `set -o pipefail` | **A pipeline fails if *any* command in it fails**, not just the last one. |
| `set -x` | Print each command before running it — add temporarily for debugging. |

**Why this matters, concretely:**

```bash
cd /app/data        # what if this directory doesn't exist?
rm -rf *            # ...you just deleted everything in your CURRENT directory
```

Without `set -e`, the failed `cd` prints an error and the script continues to `rm -rf *` in whatever directory it happened to be in. This exact bug has destroyed real production systems. With `set -e`, the script stops at the failed `cd`.

And `set -u` catches the equally famous:
```bash
rm -rf "$MYDIR/"    # if MYDIR is unset/typo'd, this becomes rm -rf "/"
```

## Variables

```bash
NAME="Alok"                # NO SPACES around = — "NAME = x" is a syntax error
COUNT=5
FILES=$(ls | wc -l)        # capture command output
readonly API_URL="https://api.example.com"    # constant

echo "$NAME"               # ALWAYS use double quotes
echo "${NAME}_suffix"      # braces when the name touches other characters
echo '$NAME'               # single quotes = literal, prints $NAME
```

**Quote every variable.** `"$var"` not `$var`. Unquoted variables get word-split on spaces:

```bash
FILE="my document.txt"
rm $FILE      # ✗ tries to delete TWO files: "my" and "document.txt"
rm "$FILE"    # ✓ correct
```

This is the number one bug in beginner bash scripts.

**Defaults and required variables:**
```bash
PORT="${PORT:-8080}"                    # use $PORT if set, else 8080
NAME="${1:-world}"                      # first argument, or "world"
: "${API_KEY:?API_KEY must be set}"     # exit with an error if unset ← great for scripts
```

**Environment variables:**
```bash
export DATABASE_URL="postgres://..."    # available to child processes
echo "$PATH" "$HOME" "$USER" "$PWD"
env                                     # list all
```

**Special variables:**

| Variable | Meaning |
|---|---|
| `$0` | script name |
| `$1`, `$2`… | positional arguments |
| `$@` | all arguments (as separate words — use `"$@"`) |
| `$#` | number of arguments |
| `$?` | **exit code of the last command** (0 = success) |
| `$$` | current PID |
| `$(cmd)` | command substitution |

## Exit codes

**0 means success. Anything else means failure.** This is how every tool, script, and CI pipeline decides whether a step passed.

```bash
ls /tmp
echo $?              # 0

ls /nonexistent
echo $?              # 2 (nonzero = failed)

exit 0               # success
exit 1               # generic failure
```

Your script's exit code is what CI checks. A script that prints "ERROR!" but exits 0 will show as a **green, passing** pipeline step while having done nothing. Always `exit 1` on failure.

```bash
if command; then ...        # tests the exit code directly
command && echo "ok"        # run only if the previous succeeded
command || echo "failed"    # run only if it failed
command || exit 1           # bail out on failure
```

## Conditionals

```bash
if [[ "$COUNT" -gt 10 ]]; then
    echo "big"
elif [[ "$COUNT" -eq 10 ]]; then
    echo "exactly ten"
else
    echo "small"
fi
```

Use `[[ ]]` in bash — it's safer than the older `[ ]` (handles empty variables and doesn't word-split).

**Numeric comparison:**

| Operator | Meaning |
|---|---|
| `-eq` | equal |
| `-ne` | not equal |
| `-gt` / `-ge` | greater than / or equal |
| `-lt` / `-le` | less than / or equal |

**String comparison:**
```bash
[[ "$A" == "$B" ]]        # equal
[[ "$A" != "$B" ]]        # not equal
[[ -z "$A" ]]             # empty (zero length)
[[ -n "$A" ]]             # NOT empty
[[ "$A" == prefix* ]]     # pattern match
```

**File tests** — used constantly in real scripts:
```bash
[[ -f "$FILE" ]]          # exists and is a regular file
[[ -d "$DIR" ]]           # exists and is a directory
[[ -e "$PATH" ]]          # exists (any type)
[[ -r "$F" ]] / [[ -w ]] / [[ -x ]]    # readable / writable / executable
[[ -s "$F" ]]             # exists and is NOT empty
```

**Combining:**
```bash
[[ -f "$FILE" && -r "$FILE" ]]     # AND
[[ "$A" == "x" || "$B" == "y" ]]   # OR
[[ ! -f "$FILE" ]]                 # NOT
```

## Loops

```bash
# Over a list
for env in dev staging prod; do
    echo "Deploying to $env"
done

# Over files ← note the quotes; this handles spaces in filenames
for file in /var/log/*.log; do
    echo "Processing $file"
done

# Numeric range
for i in {1..5}; do echo "$i"; done
for ((i=0; i<5; i++)); do echo "$i"; done

# Over arguments
for arg in "$@"; do echo "$arg"; done

# While
count=0
while [[ $count -lt 5 ]]; do
    echo "$count"
    ((count++))
done

# Read a file line by line ← the correct idiom
while IFS= read -r line; do
    echo "Line: $line"
done < input.txt

# Retry until something succeeds
until curl -sf http://localhost:8080/health; do
    echo "Waiting for service..."
    sleep 2
done
```

`while IFS= read -r line` is the right way to read lines: `IFS=` preserves leading/trailing whitespace and `-r` stops backslashes being interpreted. Don't use `for line in $(cat file)` — it splits on spaces, not lines.

That `until curl` loop is a genuinely common pattern — waiting for a service to become healthy before continuing a deploy.

## Functions

```bash
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

log_error() {
    echo "[ERROR] $*" >&2        # errors go to stderr
}

deploy() {
    local env="$1"              # 'local' keeps it out of the global scope
    local version="$2"

    log "Deploying $version to $env"

    if [[ -z "$version" ]]; then
        log_error "Version required"
        return 1
    fi

    # ...do the work...
    return 0
}

deploy "prod" "v1.2.3"      # call it — space-separated, no parentheses
if deploy "dev" "v1.0"; then echo "worked"; fi
```

Always use `local` for function variables — without it everything is global, and two functions using `i` will silently corrupt each other.

Send errors to **stderr** (`>&2`) so they can be separated from normal output when the script's output is piped or logged.

## Error handling

```bash
#!/usr/bin/env bash
set -euo pipefail

# Run a cleanup function whenever the script exits, for any reason
cleanup() {
    echo "Cleaning up..."
    rm -f /tmp/myapp.lock
}
trap cleanup EXIT

# Catch errors with the line number
trap 'echo "Error on line $LINENO"' ERR

# Fail fast with a clear message
command -v docker >/dev/null 2>&1 || { echo "docker not installed"; exit 1; }

if ! curl -sf "$URL"; then
    echo "Health check failed" >&2
    exit 1
fi
```

**`trap cleanup EXIT` is the most valuable pattern here.** It runs on normal exit, on error, and on Ctrl+C — so temp files, lock files, and port-forwards always get cleaned up. Without it, a script that dies halfway leaves debris that breaks the next run.

## A realistic deploy script

Everything above, put together:

```bash
#!/usr/bin/env bash
#
# deploy.sh — deploy an application version to an environment
# Usage: ./deploy.sh <environment> <version>

set -euo pipefail

readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly LOG_FILE="/tmp/deploy-$(date +%Y%m%d-%H%M%S).log"

log()     { echo "[$(date '+%F %T')] $*" | tee -a "$LOG_FILE"; }
err()     { echo "[$(date '+%F %T')] ERROR: $*" | tee -a "$LOG_FILE" >&2; }
die()     { err "$*"; exit 1; }

usage() {
    cat <<EOF
Usage: $(basename "$0") <environment> <version>

  environment   dev | staging | prod
  version       e.g. v1.2.3

Example: $(basename "$0") staging v1.2.3
EOF
    exit 1
}

cleanup() {
    local code=$?
    [[ $code -ne 0 ]] && err "Deploy failed with exit code $code"
    rm -f /tmp/deploy.lock
    log "Log written to $LOG_FILE"
}
trap cleanup EXIT

check_prerequisites() {
    for cmd in docker curl; do
        command -v "$cmd" >/dev/null 2>&1 || die "$cmd is not installed"
    done
}

health_check() {
    local url="$1"
    local retries=30

    log "Health checking $url"
    for ((i=1; i<=retries; i++)); do
        if curl -sf "$url/health" >/dev/null 2>&1; then
            log "Healthy after ${i} attempts"
            return 0
        fi
        sleep 2
    done
    return 1
}

main() {
    [[ $# -eq 2 ]] || usage

    local environment="$1"
    local version="$2"

    case "$environment" in
        dev|staging|prod) ;;
        *) die "Invalid environment: $environment" ;;
    esac

    check_prerequisites

    if [[ "$environment" == "prod" ]]; then
        read -rp "Deploy $version to PRODUCTION? (yes/no) " confirm
        [[ "$confirm" == "yes" ]] || die "Aborted by user"
    fi

    log "Deploying $version to $environment"
    # docker pull "myapp:$version"
    # docker compose up -d

    health_check "http://localhost:8080" || die "Health check failed — investigate"

    log "Deploy of $version to $environment complete ✓"
}

main "$@"
```

Patterns worth stealing from this:
- `main "$@"` at the bottom — logic in functions, one entry point
- `die()` for fail-with-message
- `trap cleanup EXIT` reporting the failure
- Validating arguments and prerequisites *before* doing anything
- An extra confirmation for production
- A health check that actually gates success
- Logging to both console and a file with `tee`

## Debugging

```bash
bash -n script.sh      # syntax check WITHOUT running it
bash -x script.sh      # print every command as it runs ← the main tool
set -x                 # ...or turn it on partway through
set +x                 # ...and off again

shellcheck script.sh   # STATIC ANALYSIS — install it, seriously
```

**`shellcheck` is genuinely the best thing you can add to your bash workflow.** It catches unquoted variables, useless `cat`, wrong test operators, and dozens of subtle bugs — before they hit production. `apt install shellcheck` / `brew install shellcheck`, and there's a VS Code extension. Run it in CI on every `.sh` file.

## Scheduling with cron

```bash
crontab -e          # edit your crontab
crontab -l          # list
```

```
┌───── minute (0-59)
│ ┌─── hour (0-23)
│ │ ┌─ day of month (1-31)
│ │ │ ┌─ month (1-12)
│ │ │ │ ┌─ day of week (0-6, Sun=0)
│ │ │ │ │
* * * * * command
```

```cron
0 2 * * *      /opt/scripts/backup.sh              # 2am daily
*/5 * * * *    /opt/scripts/health-check.sh        # every 5 minutes
0 0 * * 0      /opt/scripts/weekly-cleanup.sh      # midnight Sunday
0 9 * * 1-5    /opt/scripts/workday-report.sh      # 9am weekdays
```

**Cron gotchas — these catch everyone:**

1. **Cron has a minimal `PATH`** (often just `/usr/bin:/bin`). Your script works in your shell and fails in cron. Use **absolute paths** for every command and file, or set `PATH=` at the top of the crontab.
2. **No environment variables** from your `.bashrc`. Source what you need explicitly.
3. **Output is emailed, not logged.** Redirect it: `0 2 * * * /opt/backup.sh >> /var/log/backup.log 2>&1`
4. **Times are in the server's timezone** — check with `date`. Usually UTC on cloud VMs.
5. **Test the script as the cron user** first: `sudo -u www-data /opt/scripts/backup.sh`

A cron job whose output goes nowhere is one that's been silently failing for six months. Always redirect to a log.

## Practice

**1. First script with proper structure**
```bash
mkdir -p ~/bash-practice && cd ~/bash-practice
cat > info.sh << 'EOF'
#!/usr/bin/env bash
set -euo pipefail

echo "User:     $(whoami)"
echo "Host:     $(hostname)"
echo "Date:     $(date)"
echo "Uptime:   $(uptime -p 2>/dev/null || uptime)"
echo "Disk:     $(df -h / | awk 'NR==2 {print $5}') used"
EOF
chmod +x info.sh
./info.sh
```

**2. Feel what `set -e` prevents** — do this one, it's the important lesson
```bash
cat > danger.sh << 'EOF'
#!/usr/bin/env bash
cd /nonexistent-directory
echo "STILL RUNNING — this is where rm -rf * would have run!"
EOF
chmod +x danger.sh && ./danger.sh      # note it continues past the failure

# now with the guard
sed -i.bak '1a set -euo pipefail' danger.sh
./danger.sh                             # stops at the failed cd ✓
```

**3. Arguments, validation, exit codes**
Write `greet.sh` that:
- Exits 1 with a usage message if given no arguments
- Prints `Hello, <name>!` otherwise
- Accepts an optional second argument for the greeting word, defaulting to "Hello"

Test it: `./greet.sh`, `./greet.sh Alok`, `./greet.sh Alok Hi`, and check `echo $?` after each.

**4. Quoting — see the bug happen**
```bash
touch "my file.txt"
FILE="my file.txt"
ls $FILE          # FAILS — two arguments
ls "$FILE"        # works
rm "$FILE"
```

**5. Loops and conditionals**
Write `check-disk.sh` that reads `df -h` output and prints a warning line for any filesystem above 50% usage. (Hint: `df -h | awk 'NR>1 {print $5, $6}'` piped into a `while read` loop; strip the `%` with `${var%\%}`.)

**6. Functions and traps**
```bash
cat > trapdemo.sh << 'EOF'
#!/usr/bin/env bash
set -euo pipefail

cleanup() { echo "CLEANUP RAN (exit code $?)"; rm -f /tmp/demo.lock; }
trap cleanup EXIT

touch /tmp/demo.lock
echo "Working..."
sleep 2
false          # deliberate failure
echo "never reached"
EOF
chmod +x trapdemo.sh && ./trapdemo.sh; echo "script exited with $?"
ls /tmp/demo.lock 2>&1     # gone — trap cleaned it up even though we failed
```
Run it again and press Ctrl+C during the sleep. The cleanup still runs.

**7. A retry loop**
Write a script that curls `http://localhost:8000` every 2 seconds, up to 10 times, exiting 0 as soon as it succeeds and 1 if it never does. Test it: run the script first (it should retry), then in another terminal start `python3 -m http.server 8000` and watch it succeed. This is a real deploy health-check pattern.

**8. shellcheck**
```bash
shellcheck *.sh        # install it if you don't have it
```
Fix every warning it reports. Read the explanations — this teaches you more bash than any tutorial.

**9. Write something you'll actually use**
Take the manual task you wrote down back in [What is DevOps](../01-devops-fundamentals/01-what-is-devops.md) practice #1 and turn it into a real script with `set -euo pipefail`, argument validation, logging, and a cleanup trap. That's the whole point of this section.

**10. Clean up**
```bash
cd ~ && rm -rf ~/bash-practice
```

---

**Previous:** [Remotes & workflows](../04-git/03-remotes-and-workflows.md) · **Next:** [Caddy](../06-web-servers/01-caddy.md)
