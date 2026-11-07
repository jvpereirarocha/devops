# OpenObserve dashboards

Dashboards built in the OpenObserve UI, exported as JSON so they can be recreated after a
reinstall or migrated to another OpenObserve instance. These files are **not** Kubernetes
manifests: do not apply them with `kubectl` (`kubectl apply -R -f .` from `manifests/` would
fail on them, so exclude this folder as well).

| File | Dashboard | Folder in OpenObserve | Stream |
|---|---|---|---|
| `bot-telegram-overview.json` | Overview: interactions, error rate (gauge 0-100%), successes and errors over time, per-command table (total, errors, avg and p95 latency) | Bot Telegram | `fintracker_bot` |
| `api-fintracker-api-metrics.json` | API Metrics. Tab **Home**: number of requests, requests over time, requests by status code. Tab **Errors**: 5xx over time, 5xx error rate (gauge 0-10%), endpoints with most errors, 4xx by status code, errors by image version | API Fintracker | `fintracker_api` |

All panels use Custom SQL. The bot panels read the usage lines logged by the bot
(`bot.usage: event=<name> outcome=<ok|error> duration_ms=<n>`). The API panels read the
uvicorn access log lines (`"<METHOD> <path> HTTP/1.1" <status>`), filtered with
`log LIKE '%HTTP/1.1" %'`; the Prometheus `GET /metrics` scrapes are dropped by Fluent Bit
before they reach OpenObserve.

## Import

1. In OpenObserve, create the destination folder (**Dashboards → New Folder**, e.g. "Bot Telegram").
2. **Dashboards → Import**, choose the folder, and upload the JSON file (or paste its content).
3. Open the dashboard and check that the stream `fintracker_bot` exists in the new instance;
   the panels are empty until it receives data.

The `owner` and `created` fields were removed before saving (personal data, and OpenObserve
fills them in again on import).

## Export (after changing a dashboard in the UI)

Use **Dashboards → open the dashboard → Export** (download icon), or the API, which needs the
folder id when the dashboard is not in the `default` folder:

```bash
kubectl port-forward -n observability svc/openobserve 15080:5080 &
curl -s -u "<ADMIN_EMAIL>:<ADMIN_PASSWORD>" "http://localhost:15080/api/v2/default/folders/dashboards"   # folder ids
curl -s -u "<ADMIN_EMAIL>:<ADMIN_PASSWORD>" "http://localhost:15080/api/default/dashboards?folder=<FOLDER_ID>"
```

The API response wraps each dashboard in versioned keys (`v1` ... `v8`); the current one is the
key that matches the `version` field. Save that object, remove `owner` and `created`, and
replace the file here.
