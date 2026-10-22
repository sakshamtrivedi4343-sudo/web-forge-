# ⚡ FlowForge — Visual API Workflow Automation

> **Web Forge 2026 · Zenith, MANIT Bhopal · Problem Statement 04 — FlowForge**
> Live app: **https://flowforge-uj64.onrender.com** (free server — the first visit after a quiet period can take about a minute to wake up)

FlowForge lets anyone connect web services **without writing code**. You build a workflow on a drag-and-drop canvas — *"when this happens → call this API → if the answer looks like that → send a message"* — then run it by hand, on a schedule, or when another app calls a webhook. Every run is recorded step by step, so you can always see exactly what went in, what came out, and why something failed.

It is not a mock-up: a real **DAG execution engine** runs your workflows on a **BullMQ / Redis** queue, stores every step in **PostgreSQL**, and streams live progress to the browser over **Server-Sent Events**.

---

## Contents

1. [What you can build](#1-what-you-can-build)
2. [Quick tour (2 minutes)](#2-quick-tour-2-minutes)
3. [Using FlowForge — feature guide](#3-using-flowforge--feature-guide)
4. [Connect your own accounts (step by step)](#4-connect-your-own-accounts-step-by-step)
5. [Ready-made templates](#5-ready-made-templates)
6. [Reliability, limits and quotas](#6-reliability-limits-and-quotas)
7. [Security](#7-security)
8. [Architecture](#8-architecture)
9. [Set it up on your computer](#9-set-it-up-on-your-computer)
10. [Deploy it (Render)](#10-deploy-it-render)
11. [Tests](#11-tests)
12. [Troubleshooting](#12-troubleshooting)
13. [Requirement checklist (PS 04)](#13-requirement-checklist-ps-04)
14. [Known limits and future scope](#14-known-limits-and-future-scope)

---

## 1. What you can build

| Example | Blocks used |
|---|---|
| Post a message in **Discord / Slack** whenever any app calls a link (webhook) | Trigger → HTTP |
| Create a real **Google Calendar event with a Google Meet link** at a time you choose, then tell Discord when the meeting starts | Trigger → HTTP → Condition → Delay → HTTP |
| **Schedule a YouTube video** to go public at an exact time, or post a **comment** on a video | Trigger → HTTP |
| Announce a **new YouTube video** in Discord — only when there really is a new one | Trigger → HTTP → Watch → HTTP |
| Call any public API, branch on the answer (e.g. temperature above 30?), and notify | Trigger → HTTP → Condition → HTTP / Transform |

If a service has an HTTP API, FlowForge can talk to it.

## 2. Quick tour (2 minutes)

1. Open the app and **Register** (any email + password).
2. On the home page click **+ Sample workflow**. A small graph appears: *Trigger → Get user → "Pro tier?" → notify / shape*.
3. Press **▶ Run**. The steps turn blue → green one by one. One branch of the condition runs, the other is greyed out as *skipped*.
4. Click any step: you see exactly **what was sent** and **what came back**, plus its log lines.
5. Try **⏰ Schedule** (repeat every N minutes, or run once at a date/time) and click the **Start** node to copy its **Webhook URL**.

---

## 3. Using FlowForge — feature guide

### 3.1 The canvas and the six blocks

Drag blocks from the left palette (or click one to add it); connect them by dragging from the dot on the right of a block to the dot on the left of the next. Changes **autosave** every ~1.5 seconds.

| Block | What it does |
|---|---|
| **Trigger** | The start of every workflow. Holds the data the run was started with (`{{input.…}}`). |
| **HTTP Request** | Calls any web address (GET / POST / PUT / PATCH / DELETE / HEAD). Has query parameters, headers, a saved-login picker and a body. |
| **Condition** | An if/else: sends the flow down the **true** or **false** exit. Operators: equals, not equals, greater/less (and or-equal), contains, starts/ends with, exists, is truthy. |
| **Transform** | Builds new data from earlier steps (name = value rows, or a text template). |
| **Delay** | Waits N seconds/minutes (max 2 min), **or waits until a date & time** (up to 30 min ahead). |
| **Watch for change** | Remembers a value (e.g. the latest video id) and takes the **green** exit only when it differs from last time — so you act on *new* things only, never duplicates. |

### 3.2 The Inspector (right panel) — built for non-programmers

- **Quick setup** — pick *Send a Discord message*, *Send a Slack message*, *Post on X*, *Create a Google Meet event*, *Get a YouTube video*, or any GET/POST. FlowForge fills in method, address and body for you.
- **Simple mode for messages** — for Discord / Slack / X steps you only see what matters: the **webhook link** (or your **X login**) and the **message**. Everything else (method, query, headers…) is tucked under **More options**.
- **Message box** — type plain text. Buttons insert **@everyone**, **@here**, **@channel**, tag a person or role, or any **data from earlier steps** (click the chips). X shows a 280-character counter.
- **Fields mode / Custom JSON** — for other APIs: rows of name/value, or raw JSON for power users. The JSON preview is hidden under "Technical details".
- **🧪 Test this step** — runs just that one step and shows what came back (steps that change things ask you to confirm).
- **Plain-English hints** everywhere ("What are query parameters?", "What are headers?").

### 3.3 Passing data between steps (templating)

Anything in double curly braces is replaced with real data when the run happens:

| Write | Meaning |
|---|---|
| `{{steps.fetch.output.body.name}}` | the `name` field returned by the step with id `fetch` |
| `{{steps.fetch.output.body.items.0.title}}` or `…items[0].title` | first item of a list |
| `{{input.message}}` | a field sent in through the webhook / Run input |
| `{{now.iso}}`, `{{now.date}}`, `{{now.in30min}}`, `{{now.in1h}}` | the run's start time (and offsets) |

You never have to type these by hand: open a step and click the **chips** under *Data from earlier steps*. If a `{{name}}` points at nothing, the run log says so in plain words.

### 3.4 Starting a workflow — four ways

| Start it with | How |
|---|---|
| **Manual** | **▶ Run** (optionally with `{ } Input` JSON). |
| **Repeating schedule** | **⏰ Schedule → Repeat**: every N minutes, or a cron expression with a **time zone** (e.g. `0 9 * * *` in `Asia/Kolkata`). |
| **One time, at an exact moment** | **⏰ Schedule → Run once** (date + time + time zone). Survives restarts. |
| **Webhook** | Click the **Start** node → copy the secret URL. Any app can `POST` JSON to `/hooks/<token>`; its fields become `{{input.field}}`. |

**Run vs Schedule:** *Run* = now, once. A schedule never fires at the moment you create it — the panel shows the *Next run*, and **History** labels each run *You pressed Run / Scheduled / Webhook*. Once scheduled, you do **not** need to press Run again (pressing Run as well gives an extra run).

### 3.5 Watching what happened

- **Live run view** — steps colour as they execute (SSE), with a streaming log.
- **Run history** — every run per workflow, with who/what started it.
- **Errors in plain words** — e.g. *"The service says that address does not exist (404)… the Discord webhook was probably deleted"*, with a suggested fix.
- **↻ Retry from failed step** — re-runs only the failed part; earlier successful steps are reused.
- **Cancel** a run that is in progress.

### 3.6 Saving, versions, duplicates

Workflows autosave. **Versions** are immutable snapshots you can restore. **Duplicate** copies a workflow in one click. Before a run the graph is **validated** (cycles, unconnected steps, missing URLs, placeholders) with friendly messages — a workflow that still contains a `PASTE_…` placeholder refuses to run, so nothing half-configured can fire.

---

## 4. Connect your own accounts (step by step)

FlowForge works **the n8n way: each user brings their own keys.** There is no shared app and no list of allowed test users — whoever uses the site connects their own Google / X / Discord / Slack.

Secrets are stored **encrypted** and never shown again. Open **Credentials** in the top bar.

### 4.1 Discord and Slack (free, 1 minute) — recommended first

- **Discord:** channel ⚙ → Integrations → Webhooks → New Webhook → **Copy Webhook URL**.
- **Slack:** api.slack.com/apps → Create app → Incoming Webhooks → Add New Webhook → **Copy**.
- In a workflow: add an **HTTP Request** → Quick setup → *Send a Discord message* → paste the link → type your message → **Test this step**.

### 4.2 Google (Calendar + Meet, YouTube)

1. In **console.cloud.google.com**: create a project → *APIs & Services → Library* → enable **Google Calendar API** (and **YouTube Data API v3** if you use YouTube templates).
2. *OAuth consent screen* → choose **External**, add yourself under **Test users**.
3. *Credentials → Create credentials → OAuth client ID → Web application*. Under **Authorized redirect URIs** add the address FlowForge shows you on its **Credentials** page (it looks like `https://<your-site>/api/oauth/callback`). Copy the **Client ID** and **Client secret**.
4. In FlowForge → **Credentials**: give it a name, paste the Client ID and secret, click **Connect with Google**, and press *Allow*. You land back in FlowForge with the login saved.
   - For **YouTube** features, type this in the *Scope* box first: `https://www.googleapis.com/auth/youtube.force-ssl`.
5. In your workflow, pick that login in the **Authentication** box of the Google step.

### 4.3 X (Twitter)

1. At **developer.x.com** create a project and app; enable **OAuth 2.0** with *Read and write* permission and add the same redirect address as above.
2. In FlowForge → **Credentials** → paste the Client ID and secret → **Connect with X**.
3. 💳 **Important:** posting through X's API is **paid** — X charges credits. Add credit in your X developer account first, otherwise X answers *"402 Payment required"* (FlowForge explains this in plain words).

### 4.4 Any other API

Credentials supports **Bearer token**, **Basic**, **API key (custom header)**, **Custom header** and **OAuth2 (refresh token)**. Pick it in the step's *Authentication* box — the secret is injected only at run time and masked in logs.

---

## 5. Ready-made templates

Click one on the home page, then fill in the single **⚙ Settings** step, Save, and Run.

| Template | What it does | You need |
|---|---|---|
| **Create a real Google Meet (at a set time)** | Creates a Calendar event with a Meet link for the start/end you type; if you gave a Discord link it posts it **when the meeting starts** (waits up to 30 min). | Google login; Discord link optional |
| **Schedule a YouTube video to go public** | Sets the publish time of an already-uploaded **private** video; optional Discord notice. | Google login with YouTube scope |
| **Post a comment on a YouTube video** | Posts a comment from your account. | Google login with YouTube scope |
| **New YouTube video → Discord** | Checks a channel and posts only when a *new* video appears (Watch block). | YouTube API key, uploads playlist id, Discord link |
| **Post a tweet on X** | One post. | X login + X API credit |
| **Webhook → Discord message** | Whatever an app POSTs as `{"message":"…"}` appears in Discord. | Discord link |
| 3 **offline demos** | Meet at a set time, new video → post, webhook → alert — run against FlowForge's built-in demo API, no accounts needed. | nothing |

> Uploading a *new* video file to YouTube is not possible from FlowForge (it needs a binary upload); scheduling and commenting are.

---

## 6. Reliability, limits and quotas

- **Retries** per step with exponential back-off, plus **timeouts** (default 15 s per step).
- **Rate limits (HTTP 429):** FlowForge waits (honouring the service's `Retry-After`) and retries on its own up to 2 extra times; if the service still refuses, the run shows a plain explanation of the limit. GET requests also retry automatically on 502/503/504.
- **Every provider has its own quota** (YouTube Data API ≈ 10,000 units/day, Discord webhooks ≈ 30 messages/min, X's free tier is small, Slack ≈ 1 message/sec). The **Watch** block exists so you check often but act only on real changes. A warning appears if a schedule repeats more often than every 5 minutes.
- **Limits in the engine:** at most 100 steps per run, 1 MB response per call, a Delay up to 2 minutes (or *wait until* a time up to 30 minutes ahead; for later times use **Schedule → Run once**).
- **Real-world HTTP behaviour handled:** redirects are followed (with the SSRF check at every hop), gzip/deflate/brotli responses are decoded, JSON sent as plain text is recognised, form-style bodies work, and URLs typed without `https://` are accepted.
- **Free hosting note:** the free server sleeps after ~15 min without visits; an uptime pinger (e.g. UptimeRobot on `/api/health`) and a built-in self-ping keep it awake. Schedules are rebuilt from the database on every start and re-checked every 5 minutes, so a restart does not lose them.

## 7. Security

- **SSRF protection** — every outbound URL is resolved and each IP is checked against private / loopback / link-local / **cloud-metadata (169.254.169.254)** / reserved ranges before connecting; the request is pinned to the validated IP (defeats DNS-rebinding). Hard-off for private addresses in production.
- **Credential vault** — AES-256-GCM at rest, never returned to the browser, injected only at execution time, and masked in logs (API keys, Discord/Slack webhook tokens).
- **Auth** — JWT; every query is scoped to the owner (tenant isolation tested).
- **Validation** on every endpoint (Zod), rate limiting on sign-in and OAuth endpoints, helmet security headers.
- **No `eval`** — conditions are parsed by a hand-written expression parser; templates can only read the run context, never the server's environment.
- Webhook URLs contain long unguessable tokens. The server refuses to start in production without `JWT_SECRET` and `CREDENTIAL_KEY`.

## 8. Architecture

```
 React 18 + Vite + TypeScript + React Flow          Node 20 + Express + TypeScript
┌──────────────────────────────┐   HTTPS / SSE   ┌───────────────────────────────────┐
│ Editor (canvas, inspector)   │ ──────────────▶ │ REST API · SSE stream · Zod       │
│ Live run view · Credentials  │ ◀────────────── │ Auth (JWT) · Versions · OAuth     │
└──────────────────────────────┘                 │ DAG executor ─▶ BullMQ worker     │
                                                 │ SSRF guard · Credential vault     │
                                                 └───────────┬───────────┬───────────┘
                                                             ▼           ▼
                                                        PostgreSQL     Redis
                                              (workflows, versions,   (queue, cron + one-time
                                               runs, logs, creds)      scheduled jobs)
```

- `backend/src/engine/` — `dag.ts` (topological sort, cycle detection, validation), `executor.ts` (run loop, retries, branch skipping, retry-from-failed), `nodes/` (block implementations), `httpClient.ts` + `ssrfGuard.ts`, `expr.ts` (safe conditions), `context.ts` (templating).
- `backend/src/routes/` — `auth`, `workflows`, `runs`, `schedules`, `credentials`, `oauth`, `hooks` (public webhooks), `mock` (offline demo API).
- `backend/src/queue.ts` — BullMQ queue, worker, cron and one-time job schedulers. `backend/src/db/schema.sql` — tables.
- `frontend/src/` — `pages/` (Workflows, Editor, RunView, RunHistory, Credentials), `components/` (Inspector, BodyEditor, SchedulePanel, WebhookPanel…), `templates.ts`, `presets.ts`, `friendlyError.ts`.

In production one service serves **both** the API and the built website (same origin, no CORS setup).

---

## 9. Set it up on your computer

**Prerequisites:** Node 18+ (20 recommended), PostgreSQL and Redis running. On Linux / WSL2:
`sudo service postgresql start && sudo service redis-server start`

```bash
git clone https://github.com/sakshamtrivedi4343-sudo/web-forge-.git
cd web-forge-
npm run setup     # once: installs packages, creates backend/.env, checks Postgres + Redis
npm run dev       # every time: backend on :4000 + website on :5173 (Ctrl+C stops both)
```

Open **http://localhost:5173**. Both commands explain in plain words how to fix anything missing (wrong Postgres password, Redis not running, port in use).

First-time Postgres login (only if `npm run setup` asks for it):

```bash
sudo -u postgres psql -c "CREATE USER vs WITH PASSWORD 'vs' SUPERUSER;"
sudo -u postgres psql -c "CREATE DATABASE flowforge OWNER vs;"
```

The backend ships a **mock API** at `/mock/*` (users, orders, weather, flaky, rate-limited, slow, calendar, YouTube-shaped data…) so demos run with no outside services.

**Connect with Google/X while running locally:** the Credentials page shows the exact redirect address to register in the Google / X console — copy the one it displays.

## 10. Deploy it (Render)

FlowForge deploys as **one service** (website + API). `render.yaml` describes it all.

1. Push the repo to GitHub. Create a free account at <https://render.com> and sign in with GitHub.
2. **New + → Blueprint** → pick the repo → **Apply**. Render creates the web service, a Postgres database and a Redis (Key Value) store, and wires the addresses and random secrets automatically.
3. Wait for the build (3–6 min) until it says **Live**, then open the `https://….onrender.com` link and register.
4. **Save `CREDENTIAL_KEY`:** Render → service → *Environment* → copy its value somewhere safe (it unlocks saved logins; if lost, saved credentials cannot be read).
5. Add the site's redirect address (`https://<your-site>/api/oauth/callback`) in your Google / X developer console, then connect accounts on the **Credentials** page.
6. **Updating:** push to the connected branch and Render redeploys. If your Render service watches a fork, click **Sync fork → Update branch** on GitHub first. For a clean rebuild: Render → *Manual Deploy → Clear build cache & deploy*.
7. **Keep it awake:** add a free UptimeRobot HTTP monitor on `https://<your-site>/api/health` every 5 minutes.

Required environment variables in production: `NODE_ENV=production`, `JWT_SECRET`, `CREDENTIAL_KEY`, `DATABASE_URL`, `REDIS_URL` (the Blueprint sets all of them). Free-tier notes: 750 instance-hours per month on Render's free plan, free Postgres is time-limited, and free Redis keeps data in memory only (schedules are rebuilt from the database after a restart). More detail: [`docs/DEPLOY-AND-GOOGLE-MEET.md`](docs/DEPLOY-AND-GOOGLE-MEET.md).

Optional split hosting: frontend on Vercel (root `frontend`, env `VITE_API_BASE` = backend URL) and backend on Render with `CORS_ORIGINS` = the Vercel URL.

## 11. Tests

With the API running (`cd backend && npm run dev`):

```bash
cd backend
npm run test:api        # 118 end-to-end checks: auth, tenant isolation, branching, retries, timeouts,
                        # cancel, cycles, schedules, versions, credentials, rate limits, OAuth connect, SSRF
npm run test:scenarios  # 68 "judge-style" workflows: redirects, gzip, text/plain JSON, arrays, bracket paths,
                        # every condition operator, merging branches, fan-out/in, 12-step chain, wait-until, Watch
npm run test:ssrf       # 28 IP-classification cases (IPv4, IPv6, IPv4-mapped / NAT64 / 6to4 tricks)
```

## 12. Troubleshooting

| You see | Do this |
|---|---|
| "FlowForge is waking up…" | The free server was asleep or restarting. Wait a minute and reload. |
| Run button disabled, "Finish setup" | Replace the `PASTE_…` values in the ⚙ Settings step (or connect the missing login) and Save. |
| **401 / 403** from Google or X | Re-connect the login on Credentials; check the API is enabled and the permission/scope is right. |
| **402 Payment required** (X) | X's API is paid. Add credit in the X developer console. Discord and Slack are free. |
| **404** on Discord/Slack | The webhook was deleted — create a new one and paste its link. |
| **429 Too many requests** | The service's limit. Run less often, or wait and press **↻ Retry from failed**. |
| Event at the wrong time in Calendar | Make a **new** workflow from the template — old saved workflows keep their old settings. Set start, end and time zone in ⚙ Settings. |
| `Cannot reach the server` while developing | Start the backend (`npm run dev`) — port 4000. |

## 13. Requirement checklist (PS 04)

| # | Requirement | How FlowForge does it |
|---|---|---|
| 1 | Visual editor with API / condition / transform / delay / trigger blocks | React Flow canvas with six block types (plus Watch) |
| 2 | Configure method, headers, auth, body, params | Full HTTP inspector, Quick setup, saved-login picker |
| 3 | Directed graphs, branching, data passing | Topological DAG engine; `{{steps.x.output…}}` templating; true/false exits |
| 4 | Manual **and** scheduled execution | Run button, repeat / cron / run-once schedules, webhooks |
| 5 | Validate before running | Cycles, unconnected steps, missing config, placeholders — with friendly messages |
| 6 | Retries, timeouts, failure handling, re-run failed steps | Per-step retries + back-off, timeouts, automatic 429 handling, **Retry from failed** |
| 7 | Live status, step logs, errors, history | SSE live view, per-step input/output/logs, plain-English errors, run history |
| 8 | Save, edit, duplicate, version | Autosave, immutable versions with restore, one-click duplicate |
| ★ | Security constraint | SSRF guard with IP pinning, AES-256-GCM credential vault, masked logs |

## 14. Known limits and future scope

- A single step waits at most 2 minutes (or up to 30 minutes with *wait until a date & time*); longer waits use **Schedule → Run once**.
- Binary uploads (e.g. uploading a video file) are not supported — blocks exchange JSON / text.
- Each provider's quotas and pricing apply to *your* account (e.g. X posting is paid).
- Future: per-workflow daily run budgets and quota dashboards, more ready-made connectors, team sharing, a visual version-diff, loops over lists.

---

**Squad — Web Forge 2026.** Built for Zenith, MANIT Bhopal. Team lead: Laksh Bagdi.
