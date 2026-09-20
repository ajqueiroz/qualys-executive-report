# Qualys Executive Vulnerability Report

Pulls directly from the Qualys VM API and produces a single, self-contained
HTML executive report — no Power BI, no Log Analytics, no gateway. Optionally
emails it to a distribution list.

## Table of Contents

- [What this is](#what-this-is)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick start (demo mode)](#quick-start-demo-mode)
- [Running against a real Qualys account](#running-against-a-real-qualys-account)
- [What's in the report](#whats-in-the-report)
- [Emailing the report](#emailing-the-report)
- [Configuration reference](#configuration-reference)
- [Security notes](#security-notes)
- [Rate limiting](#rate-limiting)
- [Troubleshooting](#troubleshooting)
- [Known limitations](#known-limitations)
- [Files](#files)
- [AI-assisted development](#ai-assisted-development)

## What this is

A Python program, not a Power BI file: it calls the Qualys VM API directly,
processes the results with pandas, and renders one interactive HTML report
with Plotly charts. It can also email that report to a distribution list.

```
Qualys VM API  →  pandas  →  Plotly + Jinja2  →  vulnerability_briefing.html
                                                          │
                                                          ├─ open directly in a browser, or
                                                          └─ email to a distribution list
```

## Requirements

- Python 3.10+
- A Qualys VM subscription with API access (username/password)
- (Optional) Playwright, if you want inline chart images in emails
- (Optional) An Entra app registration with `Mail.Send` permission, if
  emailing via Microsoft Graph

## Installation

```bash
git clone <this-repo>
cd qualys_report
pip install -r requirements.txt
```

To enable inline chart images in emails (optional):

```bash
pip install playwright
playwright install chromium
```

## Quick start (demo mode)

See what the report looks like before touching a real Qualys account:

```bash
python run_report.py --demo --org "Your Company Name"
```

Open `output/vulnerability_briefing.html` in any browser.

## Running against a real Qualys account

```bash
export QUALYS_USERNAME="your-username"
export QUALYS_PASSWORD="your-password"
python run_report.py --org "Your Company Name"
```

Prefer environment variables over `--username`/`--password` flags — a
password passed as a literal CLI argument is visible to other processes on
the same machine (`ps aux`). The script warns if you do this anyway.

If your subscription is on a non-US Qualys platform, add `--platform-url`
(e.g. `https://qualysapi.qualys.eu` for EU platform 3). Find your platform
URL in Qualys under **Help > About**.

## What's in the report

- **Hero number + sparkline** — current open findings, with a 60-day trend shape inline
- **Period comparison** — this month vs. last month vs. same month last year, for open findings, critical/high count, remediation speed, and fix count (colored "lower is better" for risk metrics)
- **Exposure trend** — active vs. fixed findings over time, reconstructed from each finding's first/last-detected timestamps (no separate history database needed)
- **Severity distribution** and **remediation aging** (0-30/31-60/61-90/90+ day buckets)
- **Highest-leverage fixes** — vulnerabilities affecting the most assets
- **Highest-risk assets** — severity-weighted ranking
- **Mean time to remediate**, by severity

## Emailing the report

Two send methods — pick the one that matches your mail infrastructure.

**Microsoft 365 / Exchange Online — use this one for M365 tenants:**

```bash
python run_report.py --org "Acme Corp" \
    --distribution-list distribution.txt \
    --from-address reports@acme.com \
    --send-via graph \
    --tenant-id $GRAPH_TENANT_ID \
    --client-id $GRAPH_CLIENT_ID \
    --client-secret $GRAPH_CLIENT_SECRET
```

Requires an Entra app registration with the **`Mail.Send`** application
permission (admin-consented), client-credentials auth. `--from-address`
must be a mailbox your app registration is allowed to send as — scope this
with an Exchange Online `ApplicationAccessPolicy` rather than granting
tenant-wide send rights.

**Why Graph and not SMTP for Microsoft 365:** Microsoft has deprecated
Basic Authentication for SMTP AUTH client submission in Exchange Online.
`smtp.office365.com` with a username/password will fail on most current
tenants unless an explicit legacy-auth exception was granted. Graph's
`sendMail` is the actual supported path today.

**Generic SMTP — for SendGrid, a relay, or any non-Microsoft mail system:**

```bash
python run_report.py --org "Acme Corp" \
    --email-to alice@acme.com bob@acme.com --email-cc ciso@acme.com \
    --from-address reports@acme.com \
    --send-via smtp \
    --smtp-host smtp.sendgrid.net --smtp-port 587 \
    --smtp-username apikey --smtp-password $SENDGRID_API_KEY
```

**Distribution list file** (`distribution.txt`), an alternative to
`--email-to` — one address per line, `#` starts a comment:

```
# Security leadership
ciso@acme.com
vp-security@acme.com

# SOC
soc-team@acme.com
```

**Test before actually sending** — add `--dry-run` to print what would be
sent without sending it:

```bash
python run_report.py --demo --distribution-list distribution.txt \
    --from-address reports@acme.com --send-via graph --dry-run
```

**Why every email includes both a summary AND an attachment:** email
clients strip `<script>` tags universally, so the interactive Plotly
report cannot render inline in an email body no matter what. Every send
builds a static summary (KPI table + a plain PNG trend chart) for the
inbox, and attaches the full interactive report for anyone who opens it in
a browser. Add `--no-inline-chart` to skip the PNG generation (faster, no
Playwright needed) and send a table-only summary instead.

## Configuration reference

| Flag | Env var | Purpose |
|---|---|---|
| `--username` | `QUALYS_USERNAME` | Qualys API username |
| `--password` | `QUALYS_PASSWORD` | Qualys API password |
| `--platform-url` | `QUALYS_PLATFORM_URL` | Qualys platform (default: US Platform 1) |
| `--org` | — | Organization name shown in the report header |
| `--output` | — | Output path (default `output/vulnerability_briefing.html`) |
| `--demo` | — | Use synthetic data, skip Qualys entirely |
| `--email-to` | — | Space-separated recipient addresses |
| `--email-cc` | — | Space-separated CC addresses |
| `--distribution-list` | — | Path to a text file, one address per line |
| `--send-via` | — | `smtp` or `graph` |
| `--from-address` | `REPORT_FROM_ADDRESS` | Sender address |
| `--no-inline-chart` | — | Skip the PNG chart image in the email body |
| `--dry-run` | — | Print what would be sent without sending |
| `--smtp-host` / `--smtp-port` | `SMTP_HOST` / `SMTP_PORT` | SMTP server |
| `--smtp-username` / `--smtp-password` | `SMTP_USERNAME` / `SMTP_PASSWORD` | SMTP credentials |
| `--tenant-id` / `--client-id` / `--client-secret` | `GRAPH_TENANT_ID` / `GRAPH_CLIENT_ID` / `GRAPH_CLIENT_SECRET` | Entra app registration for Graph sendMail |

## Security notes

- **XXE protection**: XML is parsed with entity resolution and network
  access disabled (`resolve_entities=False, no_network=True` in
  `qualys_client.py`), preventing XML External Entity injection attacks.
- **Credentials only ever go in the `Authorization` header**, never a URL
  or query string, so they can't leak into logs or proxy access records.
- **CLI credential warning**: passing `--password`, `--smtp-password`, or
  `--client-secret` as a literal argument triggers a warning, since that
  value is visible to other processes on the same machine (`ps aux`,
  `/proc`). Use the environment variables instead in any shared or logged
  environment.
- **No secrets are logged or printed** anywhere in normal operation.
- **TLS certificate verification is never disabled** — all requests use
  `requests`' default `verify=True`.

## Rate limiting

Qualys enforces per-API rate limits, exposed via response headers:
`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Window-Sec`,
`X-RateLimit-ToWait-Sec`, and separately `X-Concurrency-Limit-*` for
simultaneous calls. This client:

- **Proactively backs off** when `X-RateLimit-Remaining` gets low, before
  actually hitting the limit.
- **On a `409` (Qualys's rate-limit response)**, reads
  `X-RateLimit-ToWait-Sec` and waits exactly that long before retrying.
- **Retries transient 5xx/network errors** with exponential backoff (2s,
  4s, 8s, 16s, up to `MAX_RETRIES = 5`), then fails with a clear error
  rather than hanging indefinitely.
- **Does not retry** non-transient errors (bad credentials, bad
  parameters) — those fail immediately since a retry won't fix them.

If you're running this on a large subscription and see frequent
rate-limit waits, lower `truncation_limit` in `qualys_client.py`'s
`fetch_host_detections`/`fetch_knowledgebase` calls, or ask Qualys Support
to raise your subscription's API rate limit.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Charts are blank / report loads but no visuals appear | Plotly.js failed to load from a CDN | Already fixed in this codebase — Plotly.js is embedded directly in the file, not loaded externally. If you've modified `report_generator.py`'s `include_plotlyjs` settings, revert to embedding it in at least the first chart on the page |
| `Plotly is not defined` in the browser console | The chart that embeds the Plotly library isn't the first one to appear in the page's HTML | Whichever chart renders first in document order must have `include_plotlyjs=True`; every other chart on the page should have it `False` |
| Qualys API returns `401` | Bad username/password, or account locked | Verify credentials directly by logging into the Qualys UI; check for IP-based access restrictions on the account |
| Qualys API returns `409` repeatedly | Rate limit or concurrency limit exceeded | The client already retries this automatically using `X-RateLimit-ToWait-Sec`. If it keeps failing after 5 attempts, you're consistently exceeding your subscription's rate limit — lower `truncation_limit` or contact Qualys Support |
| Script hangs or times out on a large subscription | Pagination working correctly but subscription has many thousands of hosts | Expected — this can take a while. Consider running as a background/scheduled job rather than interactively |
| `smtp.office365.com` authentication fails | Microsoft has deprecated Basic Auth for SMTP AUTH on most current M365 tenants | Use `--send-via graph` instead — see [Emailing the report](#emailing-the-report) |
| Graph `sendMail` returns a 403 | The app registration doesn't have `Mail.Send` permission, isn't admin-consented, or `--from-address` isn't a mailbox it's allowed to send as | Check the app registration's API permissions in Entra, confirm admin consent was granted, and check for a restrictive `ApplicationAccessPolicy` scoping which mailboxes it can send as |
| Email arrives with a broken image / no chart in the body | Playwright isn't installed, or `playwright install chromium` was never run | The email still sends — the body falls back to a table-only summary automatically. Run `pip install playwright && playwright install chromium` to enable inline images |
| `kaleido`/Chrome errors when generating images | This codebase deliberately does NOT use `kaleido` for static image export (it requires its own separate Chrome download that can fail in restricted networks) — it uses Playwright instead | If you see a kaleido error, you're likely running code that wasn't from this repo, or a stale cached version. `chart_export.py` is the only static-image code path here |
| Report file is very large (4-5 MB) | Plotly.js is fully embedded for offline/portable reliability | This is intentional — see the note in `report_generator.py`. Switch `include_plotlyjs` back to `"cdn"` only if you've confirmed your recipients' network can reach `cdn.plot.ly` (some corporate networks block this) |
| Year-over-year comparison looks wrong or shows "—" | Fewer than ~13 months of detection history in your Qualys account/pull | Expected — the report notes this explicitly ("fewer than 12 months of detection history are available"). YoY becomes meaningful once you've been scanning long enough |
| `ModuleNotFoundError` on import | Dependencies not installed | `pip install -r requirements.txt` |

## Known limitations

- Basic Auth over HTTPS is what Qualys's VM API v2 supports; no OAuth/token
  alternative exists here. Qualys has a newer JSON-based Host List
  Detection API (`POST /vmrs/api/1.0/host/list/detections`) in limited
  release — worth migrating to once generally available, not used here
  since it may not be enabled on every subscription yet.
- No automated test suite is included yet — testing during development was
  ad-hoc (run and visually verify), not a persisted `pytest` suite.
- Scan coverage (% of assets scanned recently vs. total inventory),
  business-unit/owner risk breakdown, and CIS Controls maturity mapping
  are not included — these need data this pull doesn't fetch (a separate
  Host List/asset-inventory call, and CMDB/tagging data Qualys doesn't
  provide by default).

## Files

| File | Purpose |
|---|---|
| `qualys_client.py` | Qualys API client — auth, pagination, rate-limit handling, XML parsing |
| `data_processing.py` | Pandas transformations, executive KPIs, period comparisons |
| `report_generator.py` | Plotly charts, sparklines, Jinja2 HTML rendering |
| `templates/report.html` | The report's visual design |
| `chart_export.py` | Static PNG export for email (Playwright-based) |
| `report_distribution.py` | Email sending via SMTP or Microsoft Graph |
| `demo_data.py` | Synthetic dataset for testing without a live account |
| `run_report.py` | CLI entry point |

## AI-assisted development

This codebase was built with the assistance of Claude (Anthropic).
