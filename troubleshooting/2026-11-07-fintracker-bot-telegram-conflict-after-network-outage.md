# fintracker-bot logs `Conflict: terminated by other getUpdates request` after short internet outages

- **Date:** 2026-11-07
- **Namespace:** `fintracker-dev`
- **Component:** fintracker-bot (Telegram bot, python-telegram-bot 22.8), image `jvrochadev/fintracker-bot`; CoreDNS (k3s v1.34)
- **Status:** resolved (the bot recovers on its own; the logging was fixed so transient errors no longer look like failures)

## Symptom

The problem was found while setting up log shipping to OpenObserve. Reading the bot logs to
design the Fluent Bit pipeline showed long `[ERROR]` tracebacks ending in:

```
telegram.error.Conflict: Conflict: terminated by other getUpdates request; make sure that only one bot instance is running
```

Each error was about 70 log lines, starting with:

```
2026-11-06 20:01:29,695 [ERROR] telegram.ext.Application: No error handlers are registered, logging exception.
Traceback (most recent call last):
  File "/usr/local/lib/python3.12/site-packages/telegram/ext/_updater.py", line 340, in polling_action_cb
  ...
```

The message suggests that two copies of the bot are running with the same token. The bot
itself kept working: at the time of the investigation it answered commands normally.

## Diagnosis

### 1. First hypothesis: two pods during a rolling update (ruled out)

A Deployment's default strategy is `RollingUpdate`: on a new rollout Kubernetes starts the new
pod **before** stopping the old one. For a Telegram bot that means two pods polling with the
same token for a few seconds, which is exactly what `Conflict` describes.

```bash
kubectl get deploy fintracker-bot-deployment -n fintracker-dev \
  -o jsonpath='{.spec.replicas} replicas | strategy={.spec.strategy}{"\n"}'
kubectl get rs -n fintracker-dev -l app=fintracker-bot \
  -o custom-columns=NAME:.metadata.name,DESIRED:.spec.replicas,CREATED:.metadata.creationTimestamp \
  --sort-by=.metadata.creationTimestamp
kubectl get pod -n fintracker-dev -l app=fintracker-bot \
  -o custom-columns=NAME:.metadata.name,START:.status.startTime,RESTARTS:.status.containerStatuses[0].restartCount
```

```
1 replicas | strategy={"type":"Recreate"}
...
fintracker-bot-deployment-5b9c87c68c   1   2026-11-05T20:35:05Z
NAME                                         START                  RESTARTS
fintracker-bot-deployment-5b9c87c68c-wjh9n   2026-11-05T20:35:05Z   0
```

The Deployment **already used `Recreate`** (also in `k8s/` of the bot repository), had one
replica, and the running pod had started on 2026-11-05 with no restarts. The errors were from
2026-11-06, when no rollout happened. Hypothesis ruled out.

### 2. Second hypothesis: another copy of the bot somewhere else (ruled out)

```bash
kubectl get pods -A -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,IMAGE:.spec.containers[*].image | grep -i bot
docker ps -a --format '{{.Names}} {{.Image}} {{.Status}}' | grep -iE 'bot|fintracker'
pgrep -af -i 'fintracker|telegram'
```

Only one bot pod in the cluster, no Docker container and no local process running the bot.
Ruled out.

### 3. The timeline of the errors

```bash
kubectl logs -n fintracker-dev deploy/fintracker-bot-deployment --timestamps \
  | grep -E '\[ERROR\]|Conflict|NetworkError:'
```

Times are local (-03:00):

| Time (2026-11-06) | Event |
|---|---|
| 17:00:56 → 17:01:16 | 5× `telegram.error.NetworkError: httpx.ConnectError: [Errno -3] Temporary failure in name resolution` |
| 17:01:29 | `telegram.error.Conflict: Conflict: terminated by other getUpdates request` |
| 19:30:20 | `telegram.error.NetworkError: httpx.ConnectError: [Errno -3] Temporary failure in name resolution` |
| 19:30:27 | `telegram.error.Conflict: Conflict: terminated by other getUpdates request` |
| after that | no errors for about 20 hours |

Every `Conflict` came 7 to 13 seconds **after** a burst of DNS failures.

### 4. The DNS failures were an internet outage

Pods resolve names through **CoreDNS**, the cluster DNS server. Names outside the cluster
(such as `api.telegram.org`) are forwarded by CoreDNS to an upstream DNS server.

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=-1 --timestamps | grep 'plugin/errors' | cut -c1-16 | sort | uniq -c
```

```
     15 2026-11-06T17:00
     39 2026-11-06T17:01
      9 2026-11-06T19:30
```

Sample of the errors:

```
[ERROR] plugin/errors: <id> api.telegram.org A: read udp 10.42.0.234:43949->8.8.8.8:53: i/o timeout
[ERROR] plugin/errors: <id> api.telegram.org AAAA: dial udp [2001:4860:4860::8888]:53: connect: network is unreachable
[ERROR] plugin/errors: <id> argocd-redis A: read udp 10.42.0.234:51576->8.8.8.8:53: i/o timeout
```

The upstream server stopped answering for **every** external name, not only Telegram, and
only in two short windows (about 2 minutes at 17:00 and less than 1 minute at 19:30). In two
days of CoreDNS logs there were no other upstream errors. The homelab's internet connection
dropped twice; the cluster itself was fine.

### 5. The bot was healthy

```bash
kubectl exec -n fintracker-dev deploy/fintracker-bot-deployment -- python -c "<resolve api.telegram.org and count ESTABLISHED :443 sockets>"
```

```
dns api.telegram.org -> ['149.154.166.110']
established connections to :443 -> 2
```

The bot resolved Telegram, kept its long-polling connections open and logged no error after
19:30. python-telegram-bot had retried and recovered on its own.

## Cause

Two short internet outages broke the bot's long polling. While the network was coming back,
the bot opened a new `getUpdates` request before Telegram had closed the previous one, and
Telegram answered one of them with `Conflict`. It was a transient reconnection race, not a
second instance. The real defect was that the bot had **no error handler**, so
python-telegram-bot logged each of these transient errors as an `[ERROR]` with a full
traceback, indistinguishable from a real bug.

## Understanding the problem

### Long polling and `getUpdates`

A Telegram bot can receive messages in two ways:

- **Webhook:** Telegram calls an HTTPS URL of the bot for each message. Needs a public endpoint.
- **Long polling:** the bot calls `getUpdates` and Telegram keeps the request open (for up to
  `timeout` seconds) until a message arrives or the timeout expires. Then the bot calls again.
  This is what `application.run_polling()` does, and it fits a homelab with no public endpoint.

Telegram allows **one** open `getUpdates` request per bot token. When a second one arrives, it
terminates the first and returns `Conflict: terminated by other getUpdates request`. That is
why the message talks about "only one bot instance".

### Why a network drop can cause it

During an outage the bot's HTTP client gives up on requests (timeouts, DNS failures) and
retries. A request that the bot considers dead may still be open on Telegram's side if the
network was flapping instead of fully down. When the bot's retry reaches Telegram, Telegram
sees two `getUpdates` calls with the same token and terminates one. The bot retries again and
everything goes back to normal. One `Conflict` right after a network error is harmless.
A `Conflict` that repeats every few seconds means a real second consumer.

### `Recreate` vs `RollingUpdate`

| Strategy | What happens on a rollout | Good for |
|---|---|---|
| `RollingUpdate` (default) | Starts the new pod, waits until it is Ready, then stops the old one. Zero downtime. | Stateless HTTP services (the fintracker API) |
| `Recreate` | Stops all old pods first, then starts the new ones. A few seconds of downtime. | Anything that must never run twice at once: a polling bot, a process that owns a disk |

The bot already used `Recreate`, which is why the rollout hypothesis did not hold. The logs of
the rollout done during this investigation show it working: the old pod logged
`Application.stop() complete` at 18:13:14 and the new one started at 18:13:23.

### Error handlers in python-telegram-bot

When a handler or the polling loop raises an exception, python-telegram-bot passes it to the
**error handlers** registered with `application.add_error_handler()`. If there is none, it
logs `No error handlers are registered, logging exception.` at `ERROR` with the full traceback.
With logs now shipped to OpenObserve, this matters: an alert on `[ERROR]` would fire for every
internet glitch, and real bugs would be buried in noise.

### The CoreDNS upstream

CoreDNS forwards external names to the servers in its `/etc/resolv.conf` (`forward . /etc/resolv.conf`
in the Corefile). The errors show it used Google DNS (`8.8.8.8` and `2001:4860:4860::8888`),
while the host's own `/etc/resolv.conf` points to the router (`192.168.18.1`). k3s generates a
fallback resolv.conf with Google DNS when the host one is not usable for pods. This did **not**
cause the problem (the internet was down, so any upstream would have failed), but it is worth
reviewing separately.

## Solution

No change to the cluster or the Deployment was needed: the bot recovers on its own, and the
outage was outside the cluster. The fix was in the bot's logging, in the repository
`~/projects/fintracker-bot`.

### 1. Error handler (`34c1b5e`)

`bot/main.py` now registers `handle_error`:

| Error | Before | After |
|---|---|---|
| `NetworkError` (DNS failure, connection drop, `TimedOut`) | `[ERROR]` + about 70 lines of traceback | one line: `[WARNING] Telegram network error, polling will retry: ...` |
| `Conflict` | `[ERROR]` + traceback | one line: `[ERROR] Telegram polling conflict (if it repeats, another bot instance is running): ...` |
| Anything else (a real bug) | `[ERROR]` + traceback | `[ERROR]` + traceback, logging only the `update_id` |

The update itself is not logged because it carries the text of the user's messages (financial
data), which now goes to OpenObserve. Tests in `tests/test_main.py`.

### 2. Usage log (`c3dfd95`)

Right after the first deploy, a test message got an answer but nothing showed up in
OpenObserve, which looked like a new problem. The API logs showed that the bot had handled the
command:

```bash
kubectl logs -n fintracker-dev -l app=fintracker --since=15m --prefix | grep -v 'GET /metrics'
```

```
INFO:     <bot pod IP>:59440 - "POST /api/v1/users/login HTTP/1.1" 200 OK
INFO:     <bot pod IP>:47418 - "GET /api/v1/transactions/?orderBy=id&page=1&itemsPerPage=10 HTTP/1.1" 200 OK
INFO:     <bot pod IP>:47418 - "GET /api/v1/categories/?page=1&itemsPerPage=20 HTTP/1.1" 200 OK
```

(`getWebhookInfo`, a read-only call that does not compete with polling, also showed no webhook
and `pending_update_count: 0`.) The answer had arrived; the bot simply logged nothing when it
worked. A new module `bot/usage.py` wraps every handler with `track(<event>)`, which logs:

```
[INFO] bot.usage: event=ultimas outcome=ok duration_ms=412
```

`outcome=error` when the handler raises or logs an `ERROR` (handlers catch their own failures,
tell the user and return normally, so "returned" alone does not mean "worked"). The line never
contains the message text or amounts. Tests in `tests/test_usage.py`.

### Verification

- ruff, mypy and 54 tests passed; images `jvrochadev/fintracker-bot:34c1b5e` and `:c3dfd95`
  published and rolled out (one pod `Running`, 0 restarts, running the published digest).
- The usage line for `/ultimas` appeared in the OpenObserve stream `fintracker_bot`:

```sql
SELECT _timestamp, log FROM fintracker_bot WHERE log LIKE '%bot.usage%' ORDER BY _timestamp DESC
```

## Lessons

- **Check the timeline before trusting the error message.** `Conflict` says "two instances",
  but putting the errors next to the pod start time and the rollout history showed there was
  never a second one. The real signal was the burst of DNS errors just before each `Conflict`.
- **Correlate with the cluster's DNS logs.** CoreDNS errors for *every* external name at the
  same minute point to the network, not to the application.
- **Register an error handler in every python-telegram-bot application**, and log transient
  network errors as one-line warnings. Save `ERROR` for things that need attention, so an
  alert on it is meaningful.
- **Log success too, not only failures.** Without a line per interaction, "no logs" cannot be
  told apart from "the bot did not run", and centralized logs cannot answer simple questions
  such as "did the bot get my message?".
- **Never log the user's message content** from a bot that handles personal or financial data,
  especially once logs leave the cluster.
- Follow-ups (not done here): review why CoreDNS uses Google DNS instead of the router; deploy
  the bot with the commit tag instead of `:latest`, so the version is visible in the logs.
