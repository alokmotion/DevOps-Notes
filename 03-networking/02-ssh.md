# SSH

## Concept

**SSH (Secure Shell)** gives you an encrypted connection to a remote machine's shell. It is *the* way you reach servers. It also underpins `git push` over SSH, `scp`/`rsync` file transfer, Ansible, and port forwarding.

Default port: **22**.

## Password auth vs key auth

**Passwords are bad here:** guessable, brute-forceable (a public server on port 22 gets thousands of automated attempts per day), and un-automatable.

**Key-based auth** uses a pair of mathematically linked files:

| Key | File | Where it lives | Shareable? |
|---|---|---|---|
| **Private** | `id_ed25519` | your machine, **never leaves it** | ❌ Never. Not in Git, not in Slack. |
| **Public** | `id_ed25519.pub` | copied onto every server you access | ✅ Safe to share freely |

Anything encrypted with the public key can only be decrypted with the private one. The server challenges you; only the holder of the private key can answer. The private key never crosses the network.

**Rule:** if a private key is ever exposed — pasted in a ticket, committed to a repo — it's compromised. Generate a new pair and remove the old public key from every server. There's no way to "un-leak" it.

## Generating keys

```bash
ssh-keygen -t ed25519 -C "alok@laptop"           # modern default — use this
ssh-keygen -t rsa -b 4096 -C "alok@laptop"       # if something old needs RSA
ssh-keygen -t ed25519 -f ~/.ssh/prod_key -C "prod access"   # a named key
```

- **ed25519** — shorter, faster, more secure. The default choice today.
- **`-C`** is just a comment/label; it helps you identify keys later in an `authorized_keys` file.
- **Passphrase:** you're prompted for one. Use one on a laptop key — it encrypts the private key at rest, so a stolen laptop isn't instant server access. Use `ssh-agent` so you only type it once per session.

Files created in `~/.ssh/`:
```
id_ed25519       ← PRIVATE. Must be chmod 600.
id_ed25519.pub   ← public. Safe to copy anywhere.
```

## Copying your key to a server

```bash
ssh-copy-id user@server                          # the easy way
ssh-copy-id -i ~/.ssh/prod_key.pub user@server   # a specific key

# Manually (when ssh-copy-id isn't available)
cat ~/.ssh/id_ed25519.pub | ssh user@server "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

The public key gets appended to `~/.ssh/authorized_keys` on the server. That file is simply a list of public keys allowed to log in as that user — one per line. Revoking someone's access means deleting their line.

## Connecting

```bash
ssh user@server                       # basic
ssh user@192.168.1.50
ssh -p 2222 user@server               # non-standard port
ssh -i ~/.ssh/prod_key user@server    # specific key
ssh -v user@server                    # verbose — essential for debugging auth
ssh user@server "df -h"               # run one command and exit ← great in scripts
ssh user@server "sudo systemctl restart nginx"
```

**`ssh -v`** prints every step of the negotiation: which keys it offered, what the server accepted or rejected. When key auth mysteriously fails, this is how you find out why (usually "we sent a publickey packet, wait for reply" then a fallback to password).

## The SSH config file — the biggest quality-of-life win

`~/.ssh/config`:

```
Host prod
    HostName 54.221.10.5
    User ubuntu
    IdentityFile ~/.ssh/prod_key
    Port 22

Host staging
    HostName staging.example.com
    User deploy
    IdentityFile ~/.ssh/staging_key

Host github.com
    User git
    IdentityFile ~/.ssh/github_key

# Reach a private server through a bastion/jump host
Host db-private
    HostName 10.0.2.15
    User ubuntu
    IdentityFile ~/.ssh/prod_key
    ProxyJump prod

# Apply to all hosts
Host *
    ServerAliveInterval 60      # keepalive so idle sessions don't drop
    ServerAliveCountMax 3
    AddKeysToAgent yes
```

Now `ssh prod` replaces `ssh -i ~/.ssh/prod_key -p 22 ubuntu@54.221.10.5`. `scp` and `rsync` respect this file too. Set it up on day one at any job.

## Transferring files

```bash
# scp — simple copy
scp file.txt user@server:/tmp/                 # local → remote
scp user@server:/var/log/app.log ./            # remote → local
scp -r mydir/ user@server:/opt/                # a directory
scp -i ~/.ssh/key file.txt user@server:/tmp/

# rsync — smarter: only transfers differences, resumable
rsync -avz ./local/ user@server:/remote/       # -a archive, -v verbose, -z compress
rsync -avz --delete ./local/ user@server:/remote/    # mirror exactly (deletes extras!)
rsync -avz --exclude 'node_modules' ./app/ user@server:/opt/app/
rsync -avzP bigfile.iso user@server:/tmp/      # -P shows progress and allows resume
```

**Prefer `rsync` for anything repeated or large** — re-syncing a directory transfers only what changed, which is dramatically faster than scp re-copying everything.

**The trailing slash matters in rsync:** `rsync -av src/ dst/` copies the *contents* of `src` into `dst`. `rsync -av src dst/` copies the *directory itself*, giving you `dst/src/`. Get this wrong and you end up with nested duplicates.

**`--delete` is destructive** — it removes files at the destination that no longer exist at the source. Do a `--dry-run` first:
```bash
rsync -avz --delete --dry-run ./local/ user@server:/remote/
```

## Port forwarding (tunnels)

Reach a service that isn't publicly accessible, through your SSH connection.

**Local forwarding** (the one you'll use): bring a remote port to your machine.
```bash
ssh -L 5432:localhost:5432 user@server
# now localhost:5432 on YOUR machine → port 5432 on the server
# connect your local DB client to localhost:5432

ssh -L 8080:internal-service:80 user@bastion
# YOUR localhost:8080 → internal-service:80, as seen FROM the bastion
```
This is how you reach a private RDS instance from your laptop without exposing the database to the internet.

**Remote forwarding:** expose a local port on the remote machine.
```bash
ssh -R 8080:localhost:3000 user@server    # server's :8080 → your local :3000
```

**Dynamic (SOCKS proxy):**
```bash
ssh -D 1080 user@server     # point a browser at SOCKS localhost:1080 to browse via the server
```

Add `-N` (no shell) and `-f` (background) for a tunnel you just want running:
```bash
ssh -fN -L 5432:localhost:5432 user@server
```

## Bastion / jump hosts

Production servers usually sit in a private subnet with no public IP. You SSH to a hardened **bastion** (jump host), and from there to the private machines.

```bash
ssh -J user@bastion user@private-server       # jump through, in one command
ssh -J bastion db-private                     # with ~/.ssh/config entries
```

Or set `ProxyJump` in the config as shown above, and just type `ssh db-private`.

**Never copy your private key onto the bastion** to make this easier. Use `ProxyJump` — your key stays on your laptop and the authentication is forwarded properly. (Agent forwarding, `-A`, also works but is riskier: anyone with root on the bastion can use your agent.)

## Hardening an SSH server

Edit `/etc/ssh/sshd_config`:

```
PermitRootLogin no              # never SSH in as root
PasswordAuthentication no       # keys only — kills brute force dead
PubkeyAuthentication yes
Port 2222                       # non-standard port cuts automated noise (mild benefit)
AllowUsers deploy alok          # explicit allowlist
MaxAuthTries 3
ClientAliveInterval 300
```

```bash
sudo sshd -t                    # TEST the config for syntax errors — always do this
sudo systemctl restart sshd
```

**The safety rule:** keep your current SSH session open, open a *second* terminal, and confirm you can still connect before closing the first. If you've broken the config, that open session is your only way back in. Locking yourself out of a cloud VM by disabling password auth before your key works is a rite of passage best skipped.

**Also add `fail2ban`** — it bans IPs after repeated failed attempts. Cheap and effective.

## Debugging SSH

```bash
ssh -v user@server        # verbose
ssh -vvv user@server      # very verbose
```

| Error | Cause & fix |
|---|---|
| `Permission denied (publickey)` | Server doesn't have your public key, or wrong user, or wrong key offered. Check `authorized_keys` on the server; try `-i` explicitly. |
| `WARNING: UNPROTECTED PRIVATE KEY FILE` | Your private key is world-readable → `chmod 600 ~/.ssh/id_ed25519` |
| `Connection timed out` | Firewall or security group. Port 22 not open, or wrong IP. |
| `Connection refused` | Reached the host, but sshd isn't running (or is on another port). |
| `Host key verification failed` | The server's identity changed (rebuilt VM — or a real MITM). If expected: `ssh-keygen -R hostname`. |

**Permissions SSH demands** — it refuses to work otherwise, by design:
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519       # private key
chmod 644 ~/.ssh/id_ed25519.pub
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/config
```
A large share of "my key doesn't work" cases are just this.

## ssh-agent

Holds your decrypted private key in memory so you type the passphrase once.

```bash
eval "$(ssh-agent -s)"           # start it
ssh-add ~/.ssh/id_ed25519        # add a key (prompts for passphrase once)
ssh-add -l                       # list loaded keys
ssh-add -D                       # remove all
```

## Practice

**1. Generate a key and inspect it**
```bash
ssh-keygen -t ed25519 -f ~/.ssh/practice_key -C "practice"
ls -l ~/.ssh/practice_key*        # note the permissions — which is 600 and why?
cat ~/.ssh/practice_key.pub       # safe to look at
```
Which of these two files could you safely post publicly? Why does the answer matter so much?

**2. Test against GitHub** (a real server you already have access to, no VM needed)
```bash
ssh -T git@github.com             # if your GitHub key is set up: a greeting, no shell
ssh -vT git@github.com 2>&1 | grep -i "offering\|accepted"    # which key was used?
```

**3. Write an SSH config**
Create `~/.ssh/config` with a `Host github.com` block pointing at your GitHub key. Confirm `ssh -T github.com` still works. Then add a fake entry for a server you don't have and read what `ssh -v` says when it fails — practise reading the output before you need it in an incident.

**4. Break and fix permissions** — the most common real failure
```bash
chmod 644 ~/.ssh/practice_key
ssh -i ~/.ssh/practice_key git@github.com     # read the UNPROTECTED KEY error
chmod 600 ~/.ssh/practice_key
```

**5. Local port forwarding, demonstrated locally**
```bash
python3 -m http.server 9999 &          # a "remote" service
ssh -fN -L 7777:localhost:9999 localhost   # tunnel local 7777 → 9999
curl -I http://localhost:7777          # served through the tunnel
pkill -f "ssh -fN -L 7777"
kill %1
```
(Requires SSH to localhost. If that isn't enabled, just work through what each part of the `-L 7777:localhost:9999` triple means and write it down.)

**6. rsync trailing slash**
```bash
mkdir -p ~/rsync-test/src ~/rsync-test/dst
touch ~/rsync-test/src/{a,b,c}.txt
rsync -av ~/rsync-test/src ~/rsync-test/dst/     # no trailing slash
ls -R ~/rsync-test/dst                            # note: dst/src/
rm -rf ~/rsync-test/dst/* 
rsync -av ~/rsync-test/src/ ~/rsync-test/dst/    # WITH trailing slash
ls -R ~/rsync-test/dst                            # files directly in dst
rm -rf ~/rsync-test
```

**7. Reason it out**
- Why does `PasswordAuthentication no` improve security so dramatically on an internet-facing server?
- You need to reach a database in a private subnet from your laptop. Describe two ways, and why the tunnel is better than giving the DB a public IP.
- A colleague pastes their private key into a Slack channel. What are the exact steps now?

**8. Clean up**
```bash
rm -f ~/.ssh/practice_key ~/.ssh/practice_key.pub
```

---

**Previous:** [Networking basics](01-networking-basics.md) · **Next:** [Git basics](../04-git/01-git-basics.md)
