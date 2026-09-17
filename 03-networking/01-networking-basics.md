# Networking Basics for DevOps

## Why this matters

A huge share of production problems reduce to one question: **why can't A reach B?** The service is up, the code is fine, but the request never arrives. You don't need a network engineer's depth — you need to understand the path a request takes and be able to test each hop.

## The journey of a request

When you type `https://example.com` into a browser:

```
1. DNS      example.com → 93.184.216.34        ("what's the address?")
2. TCP      3-way handshake to 93.184.216.34:443  ("open a connection")
3. TLS      certificate exchange, encryption set up   ("make it private")
4. HTTP     GET / → response 200 + HTML         ("give me the page")
```

**Any of those four can fail, and each fails differently.** Learning to tell them apart is the actual skill:

| Symptom | Likely stage |
|---|---|
| "could not resolve host" | DNS |
| "connection refused" | TCP — reached the host, nothing listening on that port |
| "connection timed out" | TCP — packets vanished, almost always a firewall |
| "certificate has expired / name mismatch" | TLS |
| "502 Bad Gateway" | HTTP — the proxy is up, the thing behind it isn't |

## IP addresses

**IPv4:** four numbers 0–255, e.g. `192.168.1.10`. About 4 billion possible, which ran out — hence NAT and IPv6.

**Public vs private.** These ranges are *private* — reserved for internal networks and not routable on the internet:

| Range | CIDR | Common use |
|---|---|---|
| `10.0.0.0` – `10.255.255.255` | `10.0.0.0/8` | AWS/cloud VPCs |
| `172.16.0.0` – `172.31.255.255` | `172.16.0.0/12` | Docker's default bridge |
| `192.168.0.0` – `192.168.255.255` | `192.168.0.0/16` | Home routers |

Special addresses: `127.0.0.1` (localhost — this machine), `0.0.0.0` (means "all interfaces" when binding a server).

**`127.0.0.1` vs `0.0.0.0` — the containers gotcha.** If your app binds to `127.0.0.1:8080`, it accepts connections *only from the same machine*. In a container, that means nothing outside the container can reach it, even with ports published. Bind to `0.0.0.0:8080` to accept from anywhere. This catches nearly everyone the first time they containerise an app.

### CIDR notation

`192.168.1.0/24` — the `/24` says the first 24 bits are the network, the rest are host addresses.

| CIDR | Addresses | Meaning |
|---|---|---|
| `/32` | 1 | one specific host |
| `/24` | 256 | a typical small subnet |
| `/16` | 65,536 | a large subnet |
| `/8` | 16,777,216 | huge |
| `/0` | all | **`0.0.0.0/0` = "the entire internet"** |

`0.0.0.0/0` is the one to recognise instantly: in a security group or firewall rule it means *open to the whole world*. Correct for a public web server's port 443; a serious incident on port 22 or a database port.

You'll use CIDR constantly when designing VPCs and writing security group rules.

## Ports

An IP address gets you to the machine; the **port** picks the service on it. 0–65535.

| Port | Service |
|---|---|
| 22 | SSH |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 27017 | MongoDB |
| 8080 | HTTP alternate (dev servers) |
| 3000 | Node/React dev |
| 9090 | Prometheus |

Ports below 1024 are privileged — binding to them requires root. That's why dev servers use 3000/8080, and why containers often run the app on 8080 with a proxy publishing 80.

## DNS

Translates names to IP addresses. Resolution order on a Linux box: `/etc/hosts` first, then the configured DNS servers.

**Record types:**

| Type | Purpose |
|---|---|
| `A` | name → IPv4 address |
| `AAAA` | name → IPv6 address |
| `CNAME` | name → another name (alias) |
| `MX` | mail servers |
| `TXT` | arbitrary text — domain verification, SPF |
| `NS` | which nameservers are authoritative |

**TTL** (time to live) is how long resolvers cache a record. A 3600s TTL means a DNS change can take an hour to reach everyone. **Before a planned migration, lower the TTL to 60s a day ahead** — then the cutover is fast. Forgetting this is why DNS changes seem to "not work" for hours.

```bash
dig example.com                 # full DNS query with details
dig +short example.com          # just the answer
dig example.com MX              # a specific record type
dig @8.8.8.8 example.com        # ask a specific DNS server (bypass local cache)
nslookup example.com            # simpler, works on Windows too
host example.com                # simplest
cat /etc/resolv.conf            # which DNS servers am I using?
cat /etc/hosts                  # local overrides — checked FIRST
```

`/etc/hosts` is a useful trick: point a domain at a staging server on your machine only, to test before switching real DNS.

## Essential debugging commands

### Is the host reachable?

```bash
ping google.com           # ICMP echo — is it alive and how far?
ping -c 4 google.com      # only 4 packets (otherwise it runs forever)
traceroute google.com     # every hop along the path
mtr google.com            # traceroute + ping combined, live — the best of the three
```

**Ping failing doesn't mean the host is down.** Plenty of firewalls and cloud security groups block ICMP while HTTP works fine. Never conclude "the server is down" from ping alone.

### Is the port open? ← the most useful test

```bash
nc -zv example.com 443        # netcat: can I open a TCP connection to this port?
nc -zv 10.0.1.5 5432          # can I reach the database?
telnet example.com 80         # older equivalent
timeout 5 bash -c "</dev/tcp/example.com/443" && echo open   # no tools needed
```

This single test separates "network/firewall problem" from "application problem". If the port opens but the app misbehaves → application. If the port won't open → network, firewall, or the service isn't running.

### What's listening on this machine?

```bash
ss -tulpn                 # modern: TCP/UDP listening ports with process names
ss -tulpn | grep :8080    # what's on port 8080?
netstat -tulpn            # older equivalent, same flags
lsof -i :8080             # which process holds port 8080?
sudo lsof -i -P -n | grep LISTEN     # everything listening
```

Flags: `t`=TCP, `u`=UDP, `l`=listening, `p`=process, `n`=numeric (don't resolve names — much faster).

**"Address already in use"** when starting your app → `ss -tulpn | grep :PORT` or `lsof -i :PORT` finds the process squatting on it. Then kill it.

Also check the **bind address** in that output: `127.0.0.1:8080` means local-only; `0.0.0.0:8080` means reachable from outside.

### HTTP testing with curl

```bash
curl https://example.com                 # fetch the body
curl -I https://example.com              # headers ONLY (fast health check)
curl -v https://example.com              # verbose: DNS, TCP, TLS, headers — great for debugging
curl -s https://api.example.com | jq     # silent + pretty JSON
curl -L https://example.com              # follow redirects
curl -X POST -H "Content-Type: application/json" -d '{"k":"v"}' https://api.example.com/items
curl -H "Authorization: Bearer $TOKEN" https://api.example.com/me
curl -o file.zip https://example.com/f.zip     # save to a file
curl -w "\ntime: %{time_total}s\n" -o /dev/null -s https://example.com   # how slow is it?
curl -k https://self-signed.local         # skip cert verification (debugging only!)
curl --resolve example.com:443:1.2.3.4 https://example.com   # test a specific server before DNS changes
```

`curl -v` is the best single debugging command here — it narrates every stage, so you see exactly where it fails.

### Interface and routing info

```bash
ip a                      # all interfaces and their IPs  (old: ifconfig)
ip r                      # routing table                 (old: route -n)
ip r get 8.8.8.8          # which route would this take?
hostname -I               # just my IP addresses
curl -s ifconfig.me       # my PUBLIC IP (as the internet sees me)
```

The difference between `ip a` (your private IP) and `curl ifconfig.me` (your public IP) is NAT in action.

## Firewalls

| Layer | Tool | Where |
|---|---|---|
| Host firewall | `ufw`, `firewalld`, `iptables` | on the machine |
| Cloud firewall | Security Groups, NACLs | in front of the machine |

```bash
sudo ufw status verbose       # Ubuntu
sudo ufw allow 22/tcp
sudo ufw allow from 10.0.0.0/8 to any port 5432
sudo ufw enable

sudo firewall-cmd --list-all  # RHEL/CentOS
sudo iptables -L -n -v        # raw rules
```

**Always allow SSH (22) before enabling a firewall on a remote machine.** Enabling `ufw` with a default-deny policy and no SSH rule locks you out of a cloud VM permanently — the classic self-inflicted outage.

**In the cloud, check both layers.** A perfectly configured `ufw` still won't help if the AWS security group blocks the port. Both must allow the traffic. When something's unreachable, check the security group first — it's the more common culprit.

## The "can't connect" checklist

Work outward, one layer at a time:

```bash
# 1. Does the name resolve?
dig +short example.com

# 2. Is the host reachable at all? (may be blocked — not conclusive)
ping -c 3 example.com

# 3. Is the PORT open?   ← the key test
nc -zv example.com 443

# 4. Is the service actually listening ON THE SERVER, and on which address?
ss -tulpn | grep :443

# 5. Is the process alive?
systemctl status nginx

# 6. Firewall — both layers
sudo ufw status
# ...and the cloud security group in the console

# 7. Does the application respond correctly?
curl -v https://example.com
```

Most incidents resolve at step 3 or 4. And a genuinely common finding at step 4: the app is bound to `127.0.0.1` instead of `0.0.0.0`.

## Load balancers, proxies, and 5xx codes

- **Reverse proxy** (nginx): sits in front of your app, terminates TLS, serves static files, forwards the rest.
- **Load balancer**: spreads requests across many backend instances and stops sending traffic to unhealthy ones.
- **Health check**: the LB requests something like `/health` every few seconds; fail it and you're removed from rotation.

**Decoding the 5xx codes** — this tells you *where* to look:

| Code | Means | Where the problem is |
|---|---|---|
| `500` | Internal Server Error | **your application** — check app logs |
| `502` | Bad Gateway | proxy is up, backend gave an invalid/no response — is the app running? |
| `503` | Service Unavailable | no healthy backends, or overloaded — check health checks |
| `504` | Gateway Timeout | backend took too long — slow query, deadlock, or the timeout is too short |

A 502 straight after a deploy nearly always means the new app version failed to start. Check the app's logs, not nginx's.

## Practice

**1. Trace a request end to end**
```bash
dig +short github.com                 # what IP(s)?
ping -c 3 github.com
nc -zv github.com 443                 # port open?
curl -I https://github.com            # status code?
curl -v https://github.com 2>&1 | head -30    # read the stages: DNS → TCP → TLS → HTTP
```
In `curl -v`, find the exact lines showing (a) which IP it connected to, (b) TLS being negotiated, (c) the HTTP request being sent.

**2. Your own machine**
```bash
ip a || ifconfig              # private IP
curl -s ifconfig.me; echo     # public IP — why are they different?
ss -tulpn || sudo lsof -i -P -n | grep LISTEN     # what's listening?
cat /etc/resolv.conf
```
Pick one listening port and identify the process behind it.

**3. Produce each failure mode deliberately** — the point is to recognise the error messages
```bash
curl http://this-domain-does-not-exist-12345.com     # DNS failure
curl http://localhost:9999                           # connection refused
curl -I https://expired.badssl.com                   # TLS failure
curl -I https://httpstat.us/500                      # HTTP error
curl -I https://httpstat.us/502
```
Write down the distinguishing phrase in each error. That vocabulary is what lets you diagnose fast later.

**4. Prove the 127.0.0.1 vs 0.0.0.0 rule**
```bash
python3 -m http.server 8000 --bind 127.0.0.1 &
curl -I http://127.0.0.1:8000          # works
curl -I http://$(hostname -I | awk '{print $1}'):8000   # fails — local-only bind
kill %1
python3 -m http.server 8000 --bind 0.0.0.0 &
curl -I http://$(hostname -I | awk '{print $1}'):8000   # works now
kill %1
```
This is the single most common containerised-app networking bug. Seeing it once makes it obvious forever.

**5. Read a CIDR**
Answer without a calculator: how many addresses in `/24`? in `/16`? What does `0.0.0.0/0` mean in a security group rule? Would you ever allow `0.0.0.0/0` on port 22, and what would you use instead?

**6. Ports in use**
```bash
python3 -m http.server 8000 &
python3 -m http.server 8000        # fails: address already in use
lsof -i :8000                      # find the culprit
kill %1
```

---

**Next:** [SSH](02-ssh.md)
