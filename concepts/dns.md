# DNS (Domain Name System)

> **In one sentence:** DNS is the internet's phone book: you give it a name like `github.com` and it gives you back the number (the IP address) that computers need to talk to each other.

- **Last updated:** 2026-11-10
- **Related:** [tutorial: deploying RabbitMQ behind an Ingress](../tutorials/how-to-deploy-rabbitmq-on-the-kubernetes-cluster.md)

## The idea in plain words

Computers find each other by **IP address**, a number like `<SERVER_IP>`. People are bad at remembering numbers and good at remembering names. DNS is the system that turns a name into a number.

Think of your phone's contact list: you tap "Mom" and the phone dials the number for you. You never type the number. DNS does the same thing for every website, app and service you use, many times a minute.

DNS also lets the number change without you noticing. If a website moves to a new server, its owner updates the DNS entry, and everyone who uses the name reaches the new server. Nobody has to learn a new number.

Where the phone-book analogy stops working:

- There is **no single phone book**. The world's names are split into millions of small books, each kept by a different owner (more on this below).
- An entry does not always hold a number. It can also point to another name, say which server handles e-mail, or hold a short note of text. Each kind of entry is called a **record type**.

## How it works

### The names form a tree

Read a name **from right to left**. Each dot separates a level, from the most general to the most specific:

```
www.github.com.
│   │      │  └── "." the root: the top of the tree (usually hidden)
│   │      └───── com: a top-level domain (TLD)
│   └──────────── github: a domain someone registered under "com"
└──────────────── www: a name inside that domain
```

```
                     . (root)
           ┌─────────┼─────────┐
          com       org        br       ← top-level domains (TLDs)
           │                   │
        github               com.br
           │
          www
```

Each owner is responsible only for their own branch, which is called a **zone**. The root servers know who runs `com`. The `com` servers know who runs `github.com`. GitHub's own servers know what `www.github.com` is. Nobody has to know everything.

### Who is involved

| Role | What it does | Everyday comparison |
|---|---|---|
| **Stub resolver** | The small part of your computer's operating system that asks "what is the IP of this name?" and waits for the answer. Its server list lives in `/etc/resolv.conf` on Linux. | You, asking a question |
| **Recursive resolver** | Does the hard work: walks down the tree on your behalf, and remembers the answers (cache). At home this is usually your router; on the internet, services like `1.1.1.1` or `8.8.8.8`. | A librarian who finds the answer for you |
| **Root and TLD servers** | They do not know the final answer, but they tell you who does. | Signs in the library: "computers → 3rd floor" |
| **Authoritative server** | The official source for one zone. Its answer is the truth. | The book that actually has the answer |

### One lookup, step by step

When you open `www.github.com` and nobody has the answer cached:

```
your computer ──"www.github.com?"──▶ recursive resolver (your router)
                                        │
                                        ├─▶ root server:   "I don't know. Ask the .com servers."
                                        ├─▶ .com server:   "I don't know. Ask github.com's servers."
                                        └─▶ github.com's authoritative server:
                                                           "www.github.com is another name for github.com,
                                                            and github.com is <IP>."
your computer ◀────── "<IP>" ───────────┘
```

You can watch this happen with `dig +trace www.github.com` (see [Trying it yourself](#trying-it-yourself)).

### Cache and TTL: why the whole trip is rare

Walking the tree for every request would be slow. So each answer comes with a **TTL** (time to live): the number of seconds the answer may be remembered. During that time, resolvers and computers reuse the saved answer, which is called the **cache**.

```
github.com.   60   IN   A   <IP>
              └─ TTL: "you may remember this for 60 seconds"
```

- A **long TTL** (hours) means fast lookups and less traffic, but changes take that long to reach everyone.
- A **short TTL** (seconds), or `0`, means changes show up right away, but there are more lookups.

When people say "DNS takes time to propagate", what they really mean is: the old answer is still sitting in caches until its TTL runs out.

Failed lookups ("this name does not exist") are cached too, for the time set in the zone's SOA record. If you look up a name *before* creating it, you may keep getting "not found" for a while after you create it.

### DNS at home: names that only exist on your network

A home router often plays two roles at once:

1. **Recursive resolver** for the internet: it looks up `github.com` for you.
2. **Authoritative server** for a private local zone, such as `<your-local-domain>`. It answers for names that exist only on your network, like `rabbitmq.<your-local-domain>`.

Those local names do not exist on the internet. If you ask a public resolver for them, it answers "not found" (`NXDOMAIN`). This is why a new service needs an entry in the router's DNS before its name works on the LAN.

## Record types

A single name can hold several records, of different **types**. Each type answers a different question.

| Type | Question it answers | Example |
|---|---|---|
| **A** | What is the **IPv4** address of this name? | `rabbitmq.<your-local-domain>  A  <LOADBALANCER_IP>` |
| **AAAA** | What is the **IPv6** address of this name? (IPv6 addresses are 4 times longer than IPv4: 128 bits against 32, hence the four A's.) | `example.com  AAAA  2606:2800:...` (some sites have none: GitHub currently returns no AAAA) |
| **CNAME** | Is this name an **alias** for another name? The resolver then looks up the other name. | `www.github.com  CNAME  github.com` |
| **MX** | Which servers receive **e-mail** for this domain? The number is a priority: lower is tried first. | `gmail.com  MX  5 gmail-smtp-in.l.google.com` |
| **NS** | Which servers are **authoritative** for this zone? This is how the tree hands you to the next level. | `github.com  NS  ns-520.awsdns-01.net` |
| **TXT** | Free-form **text**. Used to prove you own a domain, and for e-mail anti-spoofing (SPF, DKIM, DMARC). | `github.com  TXT  "v=spf1 ... ~all"` |
| **PTR** | The reverse question: which **name** belongs to this **IP**? | `8.8.8.8  →  dns.google` |
| **SOA** | Basic facts about the zone: its main server, a version number, how long to cache "not found". | One per zone |
| **SRV** | On which **host and port** does a service run? The name has the form `_service._protocol.name`. | Kubernetes creates them for named Service ports: `_amqp._tcp.rabbitmq.messaging.svc.cluster.local` |
| **CAA** | Which certificate authorities may issue **TLS certificates** for this domain? | `example.com  CAA  0 issue "letsencrypt.org"` |

### Three rules about CNAME that surprise people

1. **A name with a CNAME cannot have any other record.** If `app` is a CNAME, it cannot also have an A or a TXT record.
2. Because of rule 1, **the top of a zone (`github.com` itself, called the apex) cannot be a CNAME**: the apex must hold NS and SOA records. Some DNS providers offer "ALIAS" or "CNAME flattening" to work around this.
3. Every CNAME costs one more lookup. Keep chains short.

### A or CNAME for services behind one Ingress?

In a homelab, many web UIs often sit behind the same Ingress controller (Traefik), so they all share one IP. You have three options:

| Option | How | Good | Bad |
|---|---|---|---|
| One **A** record per service | `argocd → <IP>`, `rabbitmq → <IP>`, ... | Simple, explicit | If the IP changes, you edit every record |
| One **A** + **CNAME**s | `traefik → <IP>`, then `argocd CNAME traefik`, ... | One place to change the IP | One extra lookup |
| A **wildcard** | `*.<your-local-domain> → <IP>` | New services work without touching DNS | Typos also resolve, and land on the Ingress' 404 page |

## How Linux decides where to look

Before any DNS question is sent, Linux checks `/etc/nsswitch.conf` to see **where to look and in what order**. A common line:

```
hosts:  files  mdns4_minimal [NOTFOUND=return]  dns
```

1. **`files`**: look in `/etc/hosts` first. This is a plain text file that maps names to IPs on this one computer only. It is handy for quick tests, but every machine needs its own copy.
2. **`mdns4_minimal`**: **mDNS** (multicast DNS), a serverless system in which devices on the LAN announce their own names (printers, Macs, `raspberrypi.local`). It handles names that end in **`.local`**. `[NOTFOUND=return]` means: if mDNS cannot find a `.local` name, **stop here** and do not ask DNS.
3. **`dns`**: only now ask the servers listed in `/etc/resolv.conf`.

Two practical effects:

- **Do not use `.local` for names in your router's DNS.** `.local` is reserved for mDNS (RFC 6762), and with the line above your computer never asks DNS about it. Use another suffix such as `.lan`, or `home.arpa`, the suffix officially reserved for home networks (RFC 8375). `.local` is fine in `/etc/hosts`, because `files` is checked before mDNS.
- **`dig`, `host` and `nslookup` skip this file.** They ask a DNS server directly and ignore `/etc/hosts` and mDNS. Browsers, `curl`, `ping` and `getent` follow `nsswitch.conf`. So a name that is only in `/etc/hosts` opens in the browser, but `dig` says it does not exist. Both are right; they are asking different sources.

## DNS inside Kubernetes

Pods do not use your router. A Kubernetes cluster runs its own DNS server, **CoreDNS**, and gives every Service a name:

```
<service>.<namespace>.svc.cluster.local
rabbitmq.messaging.svc.cluster.local
```

A pod's `/etc/resolv.conf` looks like this:

```
nameserver <CLUSTER_DNS_IP>
search messaging.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

- **`search`** lists suffixes to try. If a name has fewer dots than **`ndots`** (5 here), each suffix is added in turn. That is why, inside the `messaging` namespace, plain `rabbitmq` works. From another namespace, use `rabbitmq.messaging`.
- Side effect: a lookup for `github.com` first tries `github.com.messaging.svc.cluster.local`, then `github.com.svc.cluster.local`, and so on, getting "not found" each time before reaching the real name. Those `NXDOMAIN` lines in `nslookup` output inside a pod are normal. A trailing dot (`github.com.`) marks a name as complete and skips the search list.

## Trying it yourself

Test from the bottom up. The first step that fails shows where the problem is.

### 1. Does the name resolve?

```bash
getent hosts rabbitmq.<your-local-domain>     # what your apps will get (follows nsswitch.conf and /etc/hosts)
dig +short rabbitmq.<your-local-domain>       # only the answer from the DNS server
dig rabbitmq.<your-local-domain>              # the full answer: status, TTL, which server replied
dig @<DNS_SERVER_IP> rabbitmq.<your-local-domain>   # ask one specific server
dig @1.1.1.1 rabbitmq.<your-local-domain>     # a public resolver: NXDOMAIN proves the name is local-only
dig github.com AAAA                           # pick the record type: A, AAAA, MX, TXT, NS, CNAME...
dig -x 8.8.8.8                                # reverse lookup (PTR)
dig +trace www.github.com                     # the full trip: root → TLD → authoritative
```

How to read `dig` output:

```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 1587
;; ANSWER SECTION:
rabbitmq.<your-local-domain>.  0  IN  A  <LOADBALANCER_IP>
;; SERVER: <DNS_SERVER_IP>#53
```

| You see | It means |
|---|---|
| `status: NOERROR` with an ANSWER | Found it |
| `status: NOERROR` with **no** ANSWER | The name exists, but has no record of the type you asked for |
| `status: NXDOMAIN` | The name does not exist (on that server) |
| `status: SERVFAIL` | The server could not get an answer (a problem upstream, or a DNSSEC failure) |
| `;; SERVER:` | Which server replied: check you asked the one you meant to |

`host` and `nslookup` are simpler alternatives (`nslookup` also exists on Windows).

### 2. Is there a route to the IP?

```bash
ip -br addr                        # your IP on each network interface
ip route                           # routing table; "default via" is your gateway
ip route get <LOADBALANCER_IP>     # which interface and gateway a packet to that IP would use
```

### 3. Does the host answer?

```bash
ping -c 3 rabbitmq.<your-local-domain>        # host alive? how fast?
traceroute rabbitmq.<your-local-domain>       # each hop on the way: where does the path stop?
```

A failed ping **does not prove** the host is down: many firewalls block ping (ICMP). The port test below is the one that counts.

### 4. Is the port open?

```bash
nc -zv rabbitmq.<your-local-domain> 80
#   "succeeded" → the port is open
#   "refused"   → the host is up, but nothing listens on that port
#   timeout     → blocked by a firewall, or no route

ss -tlnp     # on the server itself: which TCP ports are listening, and which process owns them
```

### 5. Does the application answer?

```bash
curl -I http://rabbitmq.<your-local-domain>/        # headers only: is it HTTP 200?
curl -v http://rabbitmq.<your-local-domain>/        # everything: resolved IP, connection, request, response
curl -s -o /dev/null -w '%{http_code} %{time_total}s\n' http://rabbitmq.<your-local-domain>/

# Test the Ingress WITHOUT DNS:
curl --resolve rabbitmq.<your-local-domain>:80:<LOADBALANCER_IP> http://rabbitmq.<your-local-domain>/
curl -H 'Host: rabbitmq.<your-local-domain>' http://<LOADBALANCER_IP>/
```

An Ingress controller picks the backend service by the **`Host` header**, not by the IP, because many services share that one IP. The last two commands send the right name straight to the right IP. If they work but the plain URL does not, the problem is DNS.

### 6. From inside the cluster

```bash
kubectl run dns-test --rm -it --restart=Never --image=busybox:1.37 -- nslookup rabbitmq.messaging
```

### A quick checklist

```bash
getent hosts <name>                                   # 1. resolves?          no → DNS or /etc/hosts
nc -zv <name> 80                                      # 2. port open?         no → network, load balancer, Ingress controller
curl -s -o /dev/null -w '%{http_code}\n' http://<name>/   # 3. 200?   404 → Ingress host rule; 502/503 → Service or pod
```

## Common confusions

| You might think... | But actually... |
|---|---|
| "`dig` says the name does not exist, so it is broken." | `dig` only asks DNS. If the name is in `/etc/hosts`, your apps still find it. Use `getent hosts` to see what apps get. |
| "I changed the record, but the old IP keeps coming back: DNS is broken." | The old answer is cached until its TTL runs out. Wait, or flush the cache (browser, OS, router). |
| "I can use `.local` for my home services in the router." | `.local` belongs to mDNS; many Linux and macOS machines never ask DNS about it. Use `.lan` or `home.arpa`. |
| "Ping fails, so the service is down." | Ping may be blocked. Test the port with `nc` or the app with `curl`. |
| "A CNAME redirects the browser to the other name." | No. A CNAME only tells the *resolver* to look up the other name. The browser still shows, and sends, the original name. |
| "One IP can only serve one website." | Many names can point to the same IP. The server (e.g. an Ingress controller) tells them apart by the `Host` header. |
| "`NXDOMAIN` lines inside a pod mean CoreDNS is failing." | They are usually the `search` list being tried. The last attempt is the one that matters. |

## Summary

- DNS turns names into IP addresses (and other data), using a tree split into zones, each with its own authoritative servers.
- A recursive resolver walks the tree for you and caches the answers for their TTL.
- Record types answer different questions: A/AAAA (IP), CNAME (alias), MX (e-mail), NS (delegation), TXT (text), PTR (reverse), SRV (host and port).
- On Linux, `nsswitch.conf` decides the order (`/etc/hosts`, mDNS, DNS). `getent` follows it; `dig` does not.
- Test in layers: name → route → host → port → application.

## References

- RFC 1034, *Domain Names - Concepts and Facilities*: https://www.rfc-editor.org/rfc/rfc1034
- RFC 6762, *Multicast DNS* (the `.local` suffix): https://www.rfc-editor.org/rfc/rfc6762
- RFC 8375, *Special-Use Domain 'home.arpa.'*: https://www.rfc-editor.org/rfc/rfc8375
- Kubernetes, *DNS for Services and Pods*: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- Cloudflare Learning Center, *What is DNS?*: https://www.cloudflare.com/learning/dns/what-is-dns/
- `man dig`, `man nsswitch.conf`, `man resolv.conf`
