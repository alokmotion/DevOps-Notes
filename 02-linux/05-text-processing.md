# Text Processing: grep, sed, awk & friends

## Why this is a whole topic

In Linux, **everything is text**: configs, logs, command output, API responses. DevOps work is largely "find the thing in the text and do something with it". These tools are how you search 2GB of logs in seconds, extract the field you need, and edit a config file from inside a script without a text editor.

The mental model: each tool reads text line by line and emits text — so they chain together with pipes.

| Tool | One-line job |
|---|---|
| `grep` | Find lines that match |
| `sed` | Edit/replace text in a stream |
| `awk` | Work with columns; do logic and maths |
| `cut` | Extract fields (simpler awk) |
| `sort` / `uniq` | Order and deduplicate |
| `jq` | The same idea, but for JSON |

## grep — find matching lines

```bash
grep "ERROR" app.log              # lines containing ERROR
grep -i "error" app.log           # case-insensitive
grep -v "DEBUG" app.log           # INVERT: lines NOT containing DEBUG
grep -n "ERROR" app.log           # show line numbers
grep -c "ERROR" app.log           # count matching lines
grep -r "TODO" ./src              # recursive through a directory
grep -l "ERROR" *.log             # just list which FILES match
grep -w "test" file               # whole word only (won't match "testing")
grep -A 5 "ERROR" app.log         # 5 lines AFTER each match
grep -B 5 "ERROR" app.log         # 5 lines BEFORE
grep -C 5 "ERROR" app.log         # 5 lines of Context either side  ← very useful
grep -E "ERROR|FATAL" app.log     # extended regex: either word
grep -o "user=[0-9]*" app.log     # print ONLY the matching part, not the whole line
```

**`-C 5` is the debugging favourite:** an error line alone rarely tells you enough — you want what led up to it and what happened next.

**Regex basics** (enough to be dangerous):

| Pattern | Matches |
|---|---|
| `^ERROR` | lines *starting* with ERROR |
| `failed$` | lines *ending* with failed |
| `.` | any single character |
| `.*` | any number of any characters |
| `[0-9]` | any digit |
| `[a-z]` | any lowercase letter |
| `\|` (or `\|` in `-E`) | OR |
| `\.` | a literal dot |

```bash
grep "^2026-09-10" app.log              # today's entries only
grep -E "^[0-9]{3}\." access.log        # lines starting with a 3-digit number and a dot
grep -E "5[0-9]{2}" access.log          # HTTP 5xx status codes
```

**Always quote your pattern** — `grep "*.log"` without quotes gets mangled by the shell before grep ever sees it.

## sed — stream editor

Edits text as it flows past. Its main use is find-and-replace, especially inside scripts.

```bash
sed 's/old/new/' file             # replace FIRST occurrence on each line
sed 's/old/new/g' file            # replace ALL occurrences (g = global)
sed 's/old/new/gi' file           # ...case-insensitively
sed -i 's/old/new/g' file         # EDIT THE FILE IN PLACE (no output)
sed -i.bak 's/old/new/g' file     # in place, keeping file.bak as backup ← safer
sed -n '5p' file                  # print only line 5
sed -n '10,20p' file              # print lines 10–20
sed '/^#/d' config                # delete comment lines
sed '/^$/d' file                  # delete blank lines
sed 's/[0-9]\+/N/g' file          # replace any run of digits with N
sed 's|/old/path|/new/path|g' f   # use | as delimiter when the text has slashes
```

**The workflow that keeps you safe:** run it *without* `-i` first and look at the output. `sed -i` rewrites the file with no undo. On a production config, take a `.bak`.

**Where you'll really use it:** substituting values into config templates during a deploy.

```bash
sed -i "s/{{VERSION}}/${BUILD_NUMBER}/g" deployment.yaml
sed -i "s/^ *replicas:.*/  replicas: 5/" deployment.yaml
```

## awk — columns and logic

`awk` splits each line into fields (whitespace-separated by default) and gives you `$1`, `$2`, … with `$0` being the whole line. It's a full programming language, but you'll use maybe five patterns.

```bash
awk '{print $1}' file             # first column
awk '{print $1, $3}' file         # columns 1 and 3
awk '{print $NF}' file            # LAST column (NF = number of fields)
awk '{print NR, $0}' file         # prefix each line with its number
awk -F: '{print $1}' /etc/passwd  # -F sets the separator (here, colon)
awk -F, '{print $2}' data.csv     # CSV

awk '$3 > 100' file                       # lines where column 3 > 100
awk '/ERROR/ {print $1, $5}' app.log      # only matching lines, chosen columns
awk '{sum += $3} END {print sum}' file    # SUM a column
awk '{sum += $1} END {print sum/NR}' file # average
awk 'NR > 1' file                         # skip the header row
awk '{print $2}' access.log | sort | uniq -c | sort -rn   # frequency count
```

The `{sum += $x} END {print sum}` pattern is the one to memorise — "add up a column" comes up constantly.

**Real examples:**
```bash
ps aux | awk '{print $2, $11}'                    # PID and command only
ps aux | awk '$3 > 50 {print $2, $11}'            # processes using >50% CPU
df -h | awk '$5+0 > 80 {print $6, $5}'            # filesystems over 80% full
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head   # top 10 IPs
```

That last one — top IPs hitting your server — is a genuine ops one-liner you'll use.

## cut, sort, uniq, tr

```bash
cut -d: -f1 /etc/passwd       # field 1, colon-delimited (simpler than awk for this)
cut -d, -f2,4 data.csv        # fields 2 and 4
cut -c1-10 file               # characters 1–10

sort file                     # alphabetical
sort -r file                  # reverse
sort -n file                  # NUMERIC (without -n, "10" sorts before "9")
sort -rn file                 # numeric, descending
sort -h file                  # human sizes (1K, 5M, 2G) — pairs with du -sh
sort -k2 file                 # sort by column 2
sort -u file                  # sort and deduplicate

uniq file                     # remove ADJACENT duplicates — sort first!
uniq -c file                  # count occurrences
uniq -d file                  # only show duplicated lines

tr 'a-z' 'A-Z' < file         # translate: lowercase → uppercase
tr -d ' ' < file              # delete all spaces
tr -s ' ' < file              # squeeze repeated spaces into one
```

**`uniq` only removes *adjacent* duplicates.** It's almost always `sort | uniq` or `sort | uniq -c`. Forgetting the sort gives silently wrong counts.

**The counting idiom** — burn this into memory:
```bash
... | sort | uniq -c | sort -rn | head
```
"Count how many of each, show the most common first." It answers *"which error is most frequent?"*, *"which IP is hammering us?"*, *"which URL 404s most?"*

## jq — grep for JSON

Every cloud CLI and API returns JSON. `jq` is how you handle it. (Install: `apt install jq` / `brew install jq`.)

```bash
cat data.json | jq '.'                   # pretty-print
jq '.name' data.json                     # one field
jq -r '.name' data.json                  # -r = raw, no surrounding quotes
jq '.users[]' data.json                  # every element of an array
jq '.users[].name' data.json             # a field from each element
jq '.users | length' data.json           # count
jq '.users[] | select(.age > 30)' data.json          # filter
jq -r '.items[] | "\(.id): \(.name)"' data.json      # format output
jq '.a.b.c' data.json                    # nested

# Real usage
aws ec2 describe-instances | jq -r '.Reservations[].Instances[].InstanceId'
curl -s https://api.github.com/users/torvalds | jq -r '.public_repos'
kubectl get pods -o json | jq -r '.items[].metadata.name'
```

`-r` matters more than you'd think: without it you get `"value"` with quotes, which breaks the next command in your pipeline.

## Putting it together

Real one-liners of the kind you'll write:

```bash
# Top 10 IPs in an access log
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# Count errors by type
grep ERROR app.log | awk '{print $4}' | sort | uniq -c | sort -rn

# Every filesystem over 80% full
df -h | awk 'NR>1 && $5+0 > 80 {print $6 " is " $5 " full"}'

# All unique URLs that returned a 404
awk '$9 == 404 {print $7}' access.log | sort -u

# Total bytes served (column 10 of a standard access log)
awk '{sum += $10} END {print sum/1024/1024 " MB"}' access.log

# Errors in the last hour, most frequent first
journalctl --since "1 hour ago" -p err | awk '{print $5}' | sort | uniq -c | sort -rn

# Replace an image tag across every YAML file
grep -rl "image: myapp:" ./k8s | xargs sed -i "s|image: myapp:.*|image: myapp:v2.1|"
```

`xargs` in that last one takes the list of filenames from `grep -l` and feeds them as arguments to `sed`.

## Practice

**1. Set up a sample log**
```bash
mkdir -p ~/text-practice && cd ~/text-practice
cat > access.log << 'EOF'
192.168.1.10 - - [10/Sep/2026:09:00:01] "GET /home HTTP/1.1" 200 1234
192.168.1.11 - - [10/Sep/2026:09:00:05] "GET /about HTTP/1.1" 200 2341
192.168.1.10 - - [10/Sep/2026:09:01:12] "POST /login HTTP/1.1" 401 122
192.168.1.12 - - [10/Sep/2026:09:02:33] "GET /missing HTTP/1.1" 404 89
192.168.1.10 - - [10/Sep/2026:09:03:01] "GET /api/users HTTP/1.1" 500 45
192.168.1.13 - - [10/Sep/2026:09:04:22] "GET /home HTTP/1.1" 200 1234
192.168.1.10 - - [10/Sep/2026:09:05:00] "GET /missing HTTP/1.1" 404 89
EOF
```

Now answer each with a one-liner:
- How many requests total?
- How many returned a 404?
- Which IP made the most requests? (use the counting idiom)
- Print only the IP and the status code for every request.
- List every unique URL that was requested.
- What's the total bytes served (last column)?
- Show every non-200 request.

**2. grep drills**
- Every line with either 404 or 500 (one command).
- Every line *except* the 200s.
- Count lines from `192.168.1.10`.
- Show the line before and after each 500.

**3. sed**
```bash
cat > config.txt << 'EOF'
# App configuration
host=localhost
port=8080

# Database
db_host=localhost
db_port=5432
EOF
```
- Replace every `localhost` with `prod-server` — print the result first, don't touch the file.
- Now do it in place, keeping a `.bak`. Verify with `diff config.txt config.txt.bak`.
- Delete all comment lines and blank lines from the output.
- Print only lines 2–3.

**4. awk**
- From `/etc/passwd`: print every username (colon-delimited).
- From `/etc/passwd`: only users with a UID of 1000 or more.
- From `ps aux`: PID and command for the 5 biggest memory users.
- From `df -h`: any filesystem over 50% full, formatted as a readable sentence.

**5. jq**
```bash
curl -s https://api.github.com/users/torvalds > user.json
jq '.' user.json
jq -r '.name, .public_repos, .followers' user.json
curl -s https://api.github.com/users/torvalds/repos | jq -r '.[].name' | head
curl -s https://api.github.com/users/torvalds/repos | jq -r '.[] | select(.stargazers_count > 100) | "\(.name): \(.stargazers_count)"'
```

**6. Build one real pipeline**
Write a single command that reports the top 3 most-requested URLs in `access.log` along with their counts. Then extend it to exclude successful (200) requests, so you're seeing only the top failing URLs. This is exactly the shape of a real incident investigation.

**7. Clean up**
```bash
ls ~/text-practice && rm -rf ~/text-practice
```

---

**Previous:** [Users, packages & disks](04-users-packages-disks.md) · **Next:** [Networking basics](../03-networking/01-networking-basics.md)
