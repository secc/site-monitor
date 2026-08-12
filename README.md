# Site Monitor

External uptime monitoring for the Rock RMS external site, built as a GitHub Actions scheduled workflow. Checks run every 5 minutes from GitHub's infrastructure (outside our Azure environment, so it can detect problems with the front door itself) and alert the team via Microsoft Teams and email when something looks wrong.

All of our Rock (sub)domains serve the same health page path, so each run checks every domain listed in `HEALTH_DOMAINS` against the shared `HEALTH_PATH`. Every problem line is tagged with the domain it came from (e.g. `[rock.example.org] Health page returned HTTP 502`), and one failing domain doesn't stop checks on the rest — a single alert aggregates all problems found in the run.

This exists because of an incident where an expired certificate took the external site down overnight and nobody knew until a staff member stumbled onto it.

## What it checks

Each run performs these checks against the Rock server health page **on every configured domain**, in priority order:

1. **Connection failure** — DNS failure, TLS handshake failure (e.g. an expired or invalid certificate), or timeout. The request never completes.
2. **HTTP error status** — the request completes but returns a 4xx/5xx (gateway errors, app crashes).
3. **Maintenance page detection** — Rock is serving the "We're Updating our Website" / Site Maintenance page. This page returns a 200, so a plain uptime check would miss it; we detect it by content.
4. **Health block status** — the page loads but the Rock Server Health block reports `Unhealthy`, or the expected `Healthy` text is missing entirely (which usually means the page served unexpected content, such as a login redirect — see [Failure modes](#failure-modes-to-be-aware-of)).
5. **Certificate expiry** — warns when the TLS certificate has fewer than 14 days remaining (`CERT_WARN_DAYS` in the workflow). This is the check that turns a midnight outage into a calm ticket two weeks earlier.

A single run can report multiple problems; all of them are included in one alert.

## Alerting

When any check fails, two alerts are sent. Each channel is independent and skips gracefully if its secrets aren't configured:

- **Microsoft Teams** — an Adaptive Card posted to the engineering channel via a Teams Workflows incoming webhook, with a link to the failed run.
- **Email** — sent via Mailgun to the shared engineering inbox (`ALERT_EMAIL_TO` secret).

While the site is down, alerts repeat every 5 minutes until the checks pass again. That's intentional — for a small team, a repeating alert is harder to miss than a single one.

## Setup

### 1. Repository secrets

Add these under **Settings → Secrets and variables → Actions → New repository secret**:

| Secret | Value |
|---|---|
| `HEALTH_DOMAINS` | Comma-separated hostnames to check (no scheme or path), e.g. `rock.example.org, my.example.com` |
| `HEALTH_PATH` | Path of the Rock server health page, appended to every domain (the anonymously viewable page with the Server Health block) |
| `TEAMS_WEBHOOK_URL` | Teams Workflows incoming webhook URL (see below) |
| `MAILGUN_API_KEY` | A Mailgun **sending** API key (not the account password) |
| `MAILGUN_DOMAIN` | Our verified Mailgun sending domain |
| `ALERT_EMAIL_TO` | Destination address for email alerts (the shared engineering inbox) |

`HEALTH_DOMAINS` and `HEALTH_PATH` are kept as secrets so the health page addresses aren't committed to source. Note this is obscurity, not security — anyone with access to Actions logs or workflow edit rights could recover them.

### 2. Teams webhook

Microsoft retired the classic "Incoming Webhook" connector; the current mechanism is the Workflows app:

1. In the target Teams channel, click **⋯ → Workflows**.
2. Choose the template **"Post to a channel when a webhook request is received."**
3. Copy the generated URL and save it as the `TEAMS_WEBHOOK_URL` secret.

Messages post from a "Workflows" bot and cannot @-mention people — set channel notification preferences accordingly, or rely on the email alert as the attention-getter.

### 3. Mailgun

The workflow calls Mailgun's HTTP API directly. If our Mailgun account is in the EU region, change the API base URL in the workflow from `api.mailgun.net` to `api.eu.mailgun.net`. The From address is `site-monitor@<MAILGUN_DOMAIN>` so alerts are easy to recognize and filter. Make sure no inbox rules on the shared mailbox route these somewhere nobody looks.

### 4. Scheduling

Nothing to configure. Once `.github/workflows/site-monitor.yml` is on the **default branch**, GitHub registers the cron automatically (every 5 minutes, offset to :x4/:x9 minutes to dodge peak scheduler congestion at round minutes). Scheduled workflows only run from the default branch.

GitHub's scheduler is best-effort: runs may start a few minutes after their cron time under load.

> **Maintenance:** PAT for cron-job.org expires Aug 2027 — regenerate and update the cron job.

## Testing

Test the alert path **now**, not during the next outage:

1. **Green path:** Actions tab → **Site Monitor** → **Run workflow** (manual trigger). Confirm the run passes with "All checks passed."
2. **Red path:** temporarily change the `grep -q "Healthy"` string in the workflow to gibberish, run manually, and confirm the Teams card and the email both arrive. Revert the change.

Never trust an untested alert path.

## Failure modes to be aware of

- **60-day auto-disable.** GitHub disables scheduled workflows in repos with no activity for 60 days. This repo will naturally sit idle, so expect this. GitHub emails a warning first and shows a re-enable banner in the Actions tab. Any commit resets the clock.
- **Health page must stay anonymously viewable.** If a Rock upgrade or security change resets the page's Auth settings, the monitor will start alerting with "unexpected content — login redirect or error page?" That alert means the *monitor* broke, not necessarily the site — but treat it as actionable either way, because a monitor grepping a login page is a monitor that's silently blind.
- **Webfarm blind spot.** The health page is served by whichever backend node the Application Gateway routes the request to (the answering server name is logged in each run's output). A single sick node in the farm may go undetected until a check happens to land on it. A Rock-side health endpoint that aggregates all `WebFarmNode` statuses would close this gap.
- **Unhealthy string assumption.** The check greps for the literal strings `Healthy` (case-sensitive, so it won't false-match inside `Unhealthy`) and `Unhealthy`. If Rock's Server Health block renders a different word in its degraded state (e.g. "Warning"), only the fallback "Healthy missing" alert would fire. Verify the block's actual unhealthy output and adjust the greps if needed.
- **GitHub itself.** The monitor runs on GitHub Actions; if GitHub is having a bad day, checks may be delayed or skipped. There is no alert for "the monitor didn't run." Accepted trade-off at this price point.

## Billing note

GitHub Actions is free without limits for public repos. In a **private** repo, each run bills as a full minute; at 288 runs/day that's roughly 8,600 minutes/month, far beyond the Free plan's included 2,000 minutes. If this must live in a private repo, drop the cadence to every 30 minutes (`*/30 * * * *`, ~1,450 min/month).

## Files

- `.github/workflows/site-monitor.yml` — the entire monitor: checks, thresholds, and both alert channels. There is no other moving part.
