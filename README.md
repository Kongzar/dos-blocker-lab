# DoS Mitigation Lab — Rate Limiting, Automated IP Banning & SYN Flood Defense

A self-contained, isolated VM lab built to test and verify denial-of-service defenses at two
different layers of the stack: **application-layer** (Nginx request rate limiting +
fail2ban automated IP banning) and **transport-layer** (iptables SYN flood mitigation).
Everything below — the config, the attacks, and the results — was run and captured by me in
my own VirtualBox environment.

> **AI disclosure:** I used Claude (Anthropic) as a guided assistant throughout this project —
> for explaining concepts, walking through command syntax, and helping debug errors as they
> came up (several are documented below, typos and all). The environment, the testing, the
> debugging, and the decisions were mine. I'm including that context because I think it's more
> useful to recruiters than pretending otherwise.

---

## Why I built this

I'm working toward a cybersecurity career and wanted hands-on, practical experience with a
real attack/defense cycle rather than just reading about rate limiting in the abstract. The
brief I set myself: **stand up a target web server, prove it's vulnerable to a basic flood,
then layer on defenses and prove they work — with evidence, not just claims.**

I was specifically advised not to run flood/load-testing tools against my own machine or
network, so the entire test was built inside an **isolated VirtualBox host-only network** —
no traffic ever touched my real LAN or the internet.

---

## Lab architecture

```mermaid
flowchart LR
    subgraph HostMac["Host: macOS (VirtualBox)"]
        subgraph HostOnlyNet["Host-only Network — 192.168.56.0/24 (isolated)"]
            Kali["Kali Linux VM\n(Attacker)\n192.168.56.3"]
            Ubuntu["Ubuntu Server VM\n(Target)\n192.168.56.4\nNginx + fail2ban"]
            Kali -- "ab / flood traffic" --> Ubuntu
            Ubuntu -- "503 responses / drops" --> Kali
        end
    end
```

Both VMs ran on a **Host-only Adapter**, deliberately excluding Bridged/NAT on the attack
path — confirmed before testing by checking that the attacker VM had no route to the public
internet.

---

## Defense architecture — how a flood gets stopped

```mermaid
sequenceDiagram
    participant A as Attacker (Kali)
    participant N as Nginx (rate limiter)
    participant F as fail2ban
    participant S as Server (protected)

    A->>N: Flood: 1000 requests, 50 concurrent
    loop Each request
        N->>N: Check rate (limit_req_zone: 5r/s, burst 10)
        alt Under limit
            N->>S: Forward request
            N->>A: 200 OK
        else Over limit
            N--xA: 503 + log "limiting requests"
        end
    end
    F->>N: Tail /var/log/nginx/error.log
    Note over F: 10 violations within 60s window
    F->>F: maxretry threshold crossed
    F->>A: iptables ban (300s)
    A--xS: Connection refused — no response at all
```

---

## Tools & skills demonstrated

| Area | What I used / did |
|---|---|
| **Virtualization** | Oracle VirtualBox — isolated host-only networking, snapshotting, VM lifecycle |
| **OS admin** | Ubuntu Server (headless, CLI-only) — package management, `systemctl`, log inspection |
| **Web server config** | Nginx — `limit_req_zone`, `limit_req`, config syntax validation (`nginx -t`), service reload |
| **Intrusion prevention** | fail2ban — custom filter regex, custom jail definition, `fail2ban-client` operations |
| **Offensive tooling (lab only)** | Kali Linux, Apache Bench (`ab`) and `hping3` for load/flood generation |
| **Firewall / packet filtering** | `iptables` — SYN rate limiting, `connlimit` per-IP caps, rule counters for verification |
| **Networking fundamentals** | Host-only vs. NAT vs. Bridged adapters, IP addressing, TCP handshake mechanics (SYN/SYN-ACK/ACK), connectivity verification (`ping`, `curl`, `ss`) |
| **Linux CLI** | `nano`, file permissions, `systemctl`, `journalctl`, log tailing, regex-based log filters |
| **Debugging under pressure** | Diagnosed and fixed a broken config directive, a service-not-running error, and multiple command typos using service logs (`journalctl -xeu`) rather than guesswork |

---

## What I actually learned

- **Rate limiting ≠ blocking.** Nginx's `limit_req` throttles traffic per-IP but still processes
  every request up to the burst limit — it reduces load, it doesn't eliminate the attacker's
  access. That's *why* I added fail2ban on top.
- **fail2ban doesn't watch traffic directly — it watches logs.** It needed a custom filter
  (regex) to recognize Nginx's `"limiting requests, excess: ... client: <HOST>"` log line as a
  "violation" before it could act on it. Understanding that log-driven detection model was the
  main conceptual takeaway of this project.
- **A `reload` only works on a running service.** I hit `nginx.service is not active, cannot
  reload` because I reloaded before the service had ever successfully *started* — a config
  error earlier had prevented that. `reload` ≠ `start`, and `journalctl -xeu` was what actually
  showed me why the start had failed.
- **Isolate before you attack, even in a lab.** Keeping the attacker VM off NAT/Bridged
  networking the whole time meant I could run real flood traffic without any risk to my own
  machine or network — a habit I'd carry into any future pentesting work.
- **Application-layer and transport-layer attacks need different defenses.** Nginx's
  `limit_req` had zero effect on the SYN flood — it never even saw those packets, because
  `iptables` operates a layer below the web server. Fixing "my site is slow under load" and
  fixing "my connection table is being exhausted" are genuinely different problems with
  different tools.
- **A SYN flood doesn't need a response to do damage.** Kali's `hping3 --flood` never read a
  single reply — the attack's entire effect came from Ubuntu's kernel doing work (allocating
  half-open connection state) in response to packets the attacker could fire and forget.
  Understanding that asymmetry is the whole reason SYN floods are cheap to launch and
  expensive to absorb without rate limiting.

---

## Results

### 1. Baseline flood — Nginx rate limiting alone
`ab -n 1000 -c 50 http://192.168.56.4/`

| Metric | Value |
|---|---|
| Requests sent | 1000 |
| Failed / non-2xx (rejected by rate limiter) | 975 |
| Requests per second | 325.74 |

![Apache Bench flood test results](assets/01-ab-flood-results.jpg)
![Nginx error log showing requests being rate-limited](assets/02-nginx-rate-limit-log.jpg)

### 2. Config validation before going live
![nginx -t syntax check passing](assets/03-nginx-config-test-success.png)

### 3. fail2ban jail configured and active
![fail2ban status showing the custom nginx-req-limit jail](assets/04-fail2ban-jail-status.jpg)

### 4. Automated ban triggered by the second flood
After re-running the same flood with fail2ban active:

| Metric | Before | After |
|---|---|---|
| Total failed (rate-limit hits) | 0 | 160 |
| Currently banned | 0 | 1 |
| Banned IP | — | `192.168.56.3` (the attacker) |

![fail2ban banning the attacker's IP after repeated violations](assets/05-fail2ban-ip-banned.jpg)

Once banned, the attacker's `curl` requests stopped getting any response at all —
the difference between **throttled** (still gets a 503) and **blocked** (connection refused
at the firewall).

---

## Round 2 — SYN flood + iptables (transport-layer defense)

The first round (above) defended against an **application-layer** flood — real HTTP requests
that Nginx had to parse. This round tests a **transport-layer** attack instead: a TCP SYN
flood, which never completes the handshake and tries to exhaust the server's connection table
before Nginx is even involved. Rate-limiting at the Nginx level doesn't help here — the defense
has to live at the firewall (`iptables`), one layer below the web server.

### The attack
```bash
sudo hping3 -S -p 80 --flood 192.168.56.4
```
`-S` sends SYN-only packets as fast as possible, deliberately never sending the final ACK —
each one leaves a half-open connection sitting in the kernel's backlog.

### Defenses added
```bash
# Rate-limit new SYN packets per source (5/sec, burst 10)
sudo iptables -A INPUT -p tcp --syn -m limit --limit 5/second --limit-burst 10 -j ACCEPT
sudo iptables -A INPUT -p tcp --syn -j DROP

# Cap concurrent half-open connections per source IP
sudo iptables -A INPUT -p tcp --syn -m connlimit --connlimit-above 20 -j DROP

# Kernel-level SYN cookie protection
sudo sysctl -w net.ipv4.tcp_syncookies=1
```

### Measuring the effect
Tracked half-open connections live during each flood:
```bash
watch -n1 'ss -tan state syn-recv | wc -l'
```

| | Before iptables rules | After iptables rules |
|---|---|---|
| Half-open (SYN-RECV) connections during flood | **513** | **160** |

![Half-open connection count before iptables rules — 513](assets/06-syn-flood-before-iptables.jpg)
![Half-open connection count after iptables rules — 160](assets/07-syn-flood-after-iptables.jpg)

That's roughly a **69% reduction** — and just as important, the count stayed bounded instead of
climbing unchecked the way it did before any firewall rule existed.

### Confirming the rule actually did the work
```bash
sudo iptables -L INPUT -v -n
```
```
637    packets  ACCEPT   tcp  --  *  *  0.0.0.0/0  0.0.0.0/0   tcp flags:0x17/0x02 limit: avg 5/sec burst 10
37452  packets  DROP     tcp  --  *  *  0.0.0.0/0  0.0.0.0/0   tcp flags:0x17/0x02
0      packets  DROP     tcp  --  *  *  0.0.0.0/0  0.0.0.0/0   tcp flags:0x17/0x02 #conn src/32 > 20
```
![iptables counters showing 37,452 SYN packets dropped](assets/08-iptables-drop-counters.jpg)

**37,452 SYN packets were dropped** by the rate-limit rule — only 637 were let through as
legitimate. The `connlimit` rule never fired at all (0 packets), because the rate limiter was
already catching almost everything before any single source could open 20+ concurrent
half-open connections. That told me the rate-limit rule was doing effectively all the work in
this test, and the connlimit rule was a backstop that never got exercised.



**`nginx.conf`** (inside the `http {}` block):
```nginx
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=5r/s;
```

**`sites-available/default`** (inside `location / {}`):
```nginx
limit_req zone=mylimit burst=10 nodelay;
```

**`/etc/fail2ban/filter.d/nginx-req-limit.conf`**:
```ini
[Definition]
failregex = limiting requests, excess:.* by zone.*client: <HOST>
ignoreregex =
```

**`/etc/fail2ban/jail.local`** (appended):
```ini
[nginx-req-limit]
enabled = true
filter = nginx-req-limit
logpath = /var/log/nginx/error.log
findtime = 60
maxretry = 10
bantime = 300
```

---

## Next steps I'm considering

- Tightening the `limit_req` burst value and the `iptables` SYN rate, and re-testing to map
  both thresholds more precisely
- Isolating the effect of `tcp_syncookies` on its own (currently tested only alongside the
  iptables rate-limit rule, not in isolation)
- Comparing this against a cloud WAF (AWS WAF / Cloudflare) rate-limiting setup as part of my
  broader cloud security learning path

---

## Repo structure
```
dos-blocker-lab/
├── README.md
├── LICENSE
├── .gitignore
└── assets/
    ├── 01-ab-flood-results.jpg
    ├── 02-nginx-rate-limit-log.jpg
    ├── 03-nginx-config-test-success.png
    ├── 04-fail2ban-jail-status.jpg
    ├── 05-fail2ban-ip-banned.jpg
    ├── 06-syn-flood-before-iptables.jpg
    ├── 07-syn-flood-after-iptables.jpg
    └── 08-iptables-drop-counters.jpg
```
