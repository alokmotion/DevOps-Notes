# Caddy

## Concept

**Caddy** is a web server — the same job as Nginx or Apache. It is written in Go and ships as a **single binary with no dependencies**.

Its defining feature: **automatic HTTPS**. You write a domain name in the config, and Caddy obtains a free TLS certificate from Let's Encrypt, installs it, redirects HTTP → HTTPS, and renews it before expiry — on its own, forever. No certbot, no renewal cron job, no expired-certificate outage at 2am.

### What it actually does on a VPS

Your app (Node, Python, Java, whatever) listens on some local port like `3000`. That port should **not** be exposed to the internet — it has no TLS, no rate limiting, and no protection. Instead:

```
Internet ──► :443 Caddy ──► localhost:3000 your app
             (HTTPS)        (plain HTTP, private)
```

Caddy sits in front and handles:

| Job | What it means |
|---|---|
| **Reverse proxy** | Receives public requests, forwards them to your app on localhost |
| **TLS termination** | Decrypts HTTPS so your app only ever speaks plain HTTP internally |
| **Static file serving** | Serves an HTML/CSS/JS folder directly, no app needed |
| **Virtual hosting** | One VPS, many domains → different apps, each with its own certificate |
| **Access logging** | Records every request |

### Caddy vs Nginx

| | Caddy | Nginx |
|---|---|---|
| HTTPS | Automatic | Manual (certbot + cron) |
| Config for a basic HTTPS proxy | ~3 lines | ~30 lines + certbot setup |
| Config reload | Zero-downtime, built in | Zero-downtime, built in |
| Community & tutorials | Smaller | Enormous — it's everywhere |
| Raw performance at extreme load | Slightly behind | Slightly ahead |
| HTTP/3 | On by default | Needs building/enabling |

**Use Caddy** when you want a small number of sites up with HTTPS and minimal fuss — exactly the single-VPS case.
**Learn Nginx too**, because almost every existing production system you inherit will be running it.

### The Caddyfile

The config file is called `Caddyfile` (no extension). On a VPS installed from the official repo it lives at **`/etc/caddy/Caddyfile`**.

A complete, production-ready HTTPS reverse proxy:

```
example.com {
    reverse_proxy localhost:3000
}
```

That is the whole thing. Certificate issued, renewed, HTTP redirected, HTTP/2 and HTTP/3 enabled.

The structure is always:

```
site-address {
    directives...
}
```

- **Site address** — a domain (`example.com`), a wildcard (`*.example.com`), a domain with port (`example.com:8080`), or `:80` for "any host on port 80".
- **Directives** — one per line, in any order. Caddy sorts them internally by a fixed precedence, so ordering is not the trap it is in Nginx.

### Where Caddy keeps things

| Path | What |
|---|---|
| `/etc/caddy/Caddyfile` | Your config |
| `/var/lib/caddy/.local/share/caddy/` | Certificates and keys — **do not delete** |
| `/var/log/caddy/` | Log files (if you configure logging) |
| `/etc/systemd/system/caddy.service` | The systemd unit |

Caddy runs as the `caddy` user, not root. It is granted `CAP_NET_BIND_SERVICE` so it can still bind ports 80 and 443.

### Requirements for automatic HTTPS to work

Certificate issuance fails silently-ish if any of these are wrong — check them first when HTTPS doesn't come up:

1. The domain's **A record points at your VPS's public IP** (and has propagated).
2. **Ports 80 and 443 are open** in the firewall *and* in your provider's cloud firewall/security group. Port 80 is needed for the HTTP challenge and the redirect — don't close it.
3. Nothing else is already bound to 80/443 (a leftover Nginx or Apache is the usual culprit).
4. The domain is publicly resolvable — Let's Encrypt has to reach it from the outside. `localhost` and private IPs get a local self-signed cert instead.

---

## Commands

### Install (Ubuntu/Debian VPS)

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' \
  | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' \
  | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install caddy
```

Installing from the repo gives you the systemd service already enabled and started.

```bash
# macOS, for local practice
brew install caddy
```

### Service management (systemd, on the VPS)

```bash
sudo systemctl status caddy            # is it running?
sudo systemctl start caddy
sudo systemctl stop caddy
sudo systemctl restart caddy           # full restart — drops connections
sudo systemctl reload caddy            # apply config, ZERO downtime ← use this
sudo systemctl enable caddy            # start on boot
```

**Always prefer `reload` over `restart`.** Caddy loads the new config and swaps over without dropping a single in-flight request.

### Config handling

```bash
sudo nano /etc/caddy/Caddyfile         # edit
caddy validate --config /etc/caddy/Caddyfile   # CHECK BEFORE RELOADING
caddy fmt --overwrite /etc/caddy/Caddyfile     # auto-format/indent
sudo systemctl reload caddy            # apply
```

`caddy validate` catches syntax errors while the old config is still serving traffic. Run it every time.

### Running it by hand (local practice, no systemd)

```bash
caddy version
caddy run                              # foreground, reads ./Caddyfile — Ctrl-C to stop
caddy start                            # background
caddy stop
caddy reload                           # reload the running instance
caddy file-server --listen :8080       # instant static server, no config file at all
caddy reverse-proxy --from :8080 --to localhost:3000   # one-off proxy
```

### Logs

```bash
sudo journalctl -u caddy --no-pager | tail -50    # service + error logs
sudo journalctl -u caddy -f                       # follow live
sudo journalctl -u caddy --since "10 min ago"
sudo tail -f /var/log/caddy/access.log            # if file logging is configured
```

Caddy logs in **JSON**. Pipe through `jq` to read it comfortably:

```bash
sudo journalctl -u caddy -o cat | jq .
sudo tail -f /var/log/caddy/access.log | jq '.request.uri, .status'
```

### Firewall

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw status
sudo ss -tlnp | grep -E ':80|:443'     # confirm Caddy is the one bound
```

---

## Caddyfile recipes

**Reverse proxy to a local app**
```
api.example.com {
    reverse_proxy localhost:3000
}
```

**Serve a static site**
```
example.com {
    root * /var/www/example
    file_server
    encode gzip zstd
}
```

**Several sites on one VPS**
```
example.com {
    root * /var/www/site
    file_server
}

api.example.com {
    reverse_proxy localhost:3000
}

admin.example.com {
    reverse_proxy localhost:4000
}
```
Each gets its own certificate automatically.

**Same config for multiple domains**
```
example.com, www.example.com {
    reverse_proxy localhost:3000
}
```

**Path-based routing — one domain, two backends**
```
example.com {
    handle /api/* {
        reverse_proxy localhost:3000
    }
    handle {
        root * /var/www/frontend
        file_server
    }
}
```

**Single-page app (React/Vue) — serve index.html for unknown routes**
```
app.example.com {
    root * /var/www/app
    try_files {path} /index.html
    file_server
    encode gzip
}
```

**Access logging to a file**
```
example.com {
    reverse_proxy localhost:3000
    log {
        output file /var/log/caddy/example-access.log {
            roll_size 10mb
            roll_keep 5
        }
    }
}
```

**Load balancing across app instances**
```
example.com {
    reverse_proxy localhost:3000 localhost:3001 localhost:3002 {
        lb_policy round_robin
        health_uri /health
        health_interval 10s
    }
}
```

**Security headers**
```
example.com {
    reverse_proxy localhost:3000
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
        -Server                      # remove the Server header
    }
}
```

**WebSockets** — no config needed. `reverse_proxy` handles the upgrade automatically (a common Nginx pain point that simply doesn't exist here).

**Basic auth**
```bash
caddy hash-password        # prompts, prints a bcrypt hash
```
```
private.example.com {
    basic_auth {
        alok $2a$14$hashed...
    }
    reverse_proxy localhost:3000
}
```

**Redirect**
```
old.example.com {
    redir https://new.example.com{uri} permanent
}
```

**Local HTTPS for development** — Caddy issues its own local certificate:
```
localhost {
    reverse_proxy localhost:3000
}
```

**Environment variables** in the Caddyfile: `{$PORT}` reads `PORT` from the environment.

---

## Troubleshooting

| Symptom | Cause & fix |
|---|---|
| `bind: address already in use` | Nginx/Apache still running. `sudo systemctl disable --now nginx apache2` |
| Site serves HTTP but no certificate | DNS A record wrong or not propagated; check with `dig +short example.com`. Or port 80 blocked. |
| `could not get certificate` in logs | Let's Encrypt can't reach you. Check firewall *and* cloud security group. Check rate limits (5 failures/hour per domain). |
| `502 Bad Gateway` | Your app isn't running, or is on a different port. `curl localhost:3000` from the VPS to confirm. |
| Config change did nothing | You edited the file but didn't reload. `sudo systemctl reload caddy` |
| `caddy: command not found` after install | Shell hasn't rehashed — `hash -r` or reopen the session |
| Permission denied writing certs | `/var/lib/caddy` ownership broken. `sudo chown -R caddy:caddy /var/lib/caddy` |
| Works over IP, fails over domain | Caddy matches on the `Host` header. An IP doesn't match a `example.com { }` block — that's correct behaviour. |

**First three commands when something is wrong:**
```bash
sudo systemctl status caddy
sudo journalctl -u caddy --since "10 min ago" --no-pager
caddy validate --config /etc/caddy/Caddyfile
```

---

## Practice

**1. Static file server, zero config**
```bash
mkdir -p ~/caddy-practice && cd ~/caddy-practice
echo "<h1>Hello from Caddy</h1>" > index.html
caddy file-server --listen :8080
```
Open `http://localhost:8080`. Ctrl-C to stop. No config file existed at any point — why is that useful when you just need to check a build output?

**2. Your first Caddyfile**
```bash
cd ~/caddy-practice
cat > Caddyfile <<'EOF'
:8080 {
    root * .
    file_server
    encode gzip
}
EOF
caddy fmt --overwrite Caddyfile
caddy validate
caddy run
```
Then `curl -I -H "Accept-Encoding: gzip" http://localhost:8080` — confirm you see `Content-Encoding: gzip`.

**3. Reverse proxy to a real backend**
```bash
# terminal 1 — a fake app
python3 -m http.server 3000

# terminal 2
cat > Caddyfile <<'EOF'
:8080 {
    reverse_proxy localhost:3000
}
EOF
caddy run
```
`curl http://localhost:8080` reaches the Python server through Caddy. Now stop the Python server and curl again — what status do you get, and which of the two processes produced it?

**4. Local HTTPS**
```
localhost {
    reverse_proxy localhost:3000
}
```
Run `caddy run`, visit `https://localhost`. You have HTTPS on localhost with no certificate work. Where did that certificate come from, and why won't a browser on another machine trust it?

**5. Break the config on purpose**
```bash
echo "reverse_proxy" >> Caddyfile     # incomplete directive
caddy validate                        # read the error carefully
```
Fix it. Then practise the real habit: **validate → reload**, never edit-and-restart-blind.

**6. On your VPS**

- Print your live config: `sudo cat /etc/caddy/Caddyfile`. Identify each site block and which local port it points to.
- Confirm what's listening: `sudo ss -tlnp | grep -E ':80|:443'`
- Watch a request land in real time: run `sudo journalctl -u caddy -f` in one session, then hit your site from a browser. Read one JSON log line and find the status code, path and upstream latency.
- Check your certificate: `echo | openssl s_client -connect yourdomain.com:443 2>/dev/null | openssl x509 -noout -dates -issuer`
- Add a `header` block with HSTS and `X-Frame-Options`, then `caddy validate` → `systemctl reload caddy`, and verify with `curl -I https://yourdomain.com`.

**7. Reason it out**
- Why should your app listen on `localhost:3000` rather than `0.0.0.0:3000` once Caddy is in front of it?
- Why is `reload` safer than `restart` for a live site — what exactly happens to a request that's mid-flight during each?
- You get a `502`. List the checks you'd run, in order, and say what each one rules out.
- Your certificate failed to issue and you've retried 6 times in 20 minutes. Why is retrying again a bad idea?

**8. Clean up**
```bash
rm -rf ~/caddy-practice
```

---

**Previous:** [Bash scripting basics](../05-shell-scripting/01-bash-basics.md) · **Next up in the roadmap:** Docker
