<div align="center">

# K-Bail-

K-Bail- (Kingsley Brief Mail) is a privacy-first Gmail reader that turns long emails into 1–3 bullet summaries, with a full-read view one click away.

**Live on:** [kbail.kingsleykoech.tech](https://kbail.kingsleykoech.tech) </b>

**Demo video:** [my  video link](https://www.youtube.com/watch?v=ii9NKMLRwTU)

</div>

> [!TIP]
> Want to try the app without signing into a Google account? Click **Sample Mail** on the landing page to load the demo inbox immediately.

---

## Product preview

<table>
  <tr>
    <td width="34%" valign="top" align="center">
      <strong>Landing &amp; Login (<code>/</code>)</strong><br />
      Envelope login card with Google OAuth and Sample Mail demo mode.<br /><br />
      <img src="assets/landing-preview.png" alt="Kbail landing and login page" />
    </td>
    <td width="66%" valign="top" align="center">
      <strong>3-Pane Workspace (<code>/inbox</code>)</strong><br />
      Searchable inbox on the left, 1–3 bullet AI summary in the center, and on-demand full message on the right.<br /><br />
      <img src="assets/inbox-preview.png" alt="Kbail 3-pane inbox workspace" />
    </td>
  </tr>
</table>

---

## Why it is built this way

Long threads, newsletters, and automated alerts hide the two sentences that actually matter. Instead of storing your mailbox in a cloud database, K-Bail- reads messages on demand over read-only OAuth, summarizes only the newest/unread items, and keeps cached mail strictly inside your browser.

---

## System architecture

```mermaid
flowchart LR
    Browser["User Browser\n(localStorage cache)"] -->|HTTPS :443| LB["7327-lb-01\nHAProxy + Let's Encrypt TLS\nSticky cookies (SERVERID)"]

    LB -->|HTTP :80| W1["7327-web-01\nNginx -> 127.0.0.1:8080\nExpress (kbail.service)"]
    LB -->|HTTP :80| W2["7327-web-02\nNginx -> 127.0.0.1:8080\nExpress (kbail.service)"]

    W1 & W2 -->|gmail.readonly| Gmail["Google Gmail API"]
    W1 & W2 -->|Primary| Groq["Groq API"]
    W1 & W2 -->|Fallback| Gemini["Gemini API"]
```

### Request & OAuth flow

```mermaid
sequenceDiagram
    actor User
    participant Browser
    participant LB as HAProxy (lb-01)
    participant App as Express (web-01 / web-02)
    participant Google as Google OAuth & Gmail
    participant AI as Groq / Gemini

    User->>Browser: Open https://kbail.kingsleykoech.tech/
    Browser->>LB: GET /
    LB->>App: Forward request (sets SERVERID cookie)
    App-->>Browser: Serve landing.html

    alt Sign in with Google
        User->>App: GET /auth/google
        App-->>Browser: Redirect to Google consent screen
        Google-->>App: GET /auth/google/callback?code=...
        App->>Google: Exchange code for access_token (RAM session)
        App-->>Browser: Redirect to /inbox
    else Sample Mail
        User->>App: GET /auth/sample
        App-->>Browser: Set sample session and redirect to /inbox
    end

    Browser->>App: GET /api/emails
    App->>Google: Fetch inbox message list and metadata
    App->>AI: Summarize new emails into 1-3 bullets
    App-->>Browser: Return emails + summaries (cached in localStorage)

    User->>Browser: Click "Full Read"
    Browser->>App: GET /api/emails/:id/full
    App-->>Browser: Return full email body
```

---

## Google Sign-In note (Unverified App screen)

> [!IMPORTANT]
> Because K-Bail- requests the `gmail.readonly` scope without undergoing Google's commercial OAuth verification review, Google displays a **"Google hasn't verified this app"** screen during sign-in:
>
> 1. Click **Advanced** at the bottom-left of the warning card.
> 2. Click **Go to kbail.kingsleykoech.tech (unsafe)**.
> 3. Grant the read-only permissions (`gmail.readonly` and `userinfo.email`) to open `/inbox`.

---

## Interface & controls

The `/inbox` workspace is organized into three columns over a translucent glass layout:

1. **Inbox** — Lists synced emails with live search, `All` / `Today` filtering, and sorting (`Newest`, `Oldest`, `Sender`).
2. **Summarize** — Shows the 1–3 bullet AI summary for the selected email, generated with Groq first and falling back to Gemini if Groq is unavailable or rate-limited.
3. **Full Mail** — Fetches the complete email body on demand via `GET /api/emails/:id/full` when you click **Full Read**.

Top bar controls:
- **SYNC mail** triggers `POST /api/sync` to fetch new messages and summarize them.
- **Settings** opens the profile modal showing the active account (or `Sample Mailbox`) and a **Logout** button (`/auth/logout`).

---

## Privacy & security model

| Layer | Stored data | Lifecycle |
| :--- | :--- | :--- |
| **Browser (`localStorage`)** | Synced email metadata and AI bullet summaries | Stays on your device; cleared on logout/reset |
| **Server RAM (`express-session`)** | `mode`, `userEmail`, short-lived Google `accessToken` | Held in memory only; destroyed on `/auth/logout` |
| **Server disk / database** | Nothing | No database and no persisted user emails |

Key safeguards:
- **Minimal OAuth scopes:** Only `gmail.readonly` and `userinfo.email`.
- **Clean, gated routes:** `/` serves the login page, `/inbox` redirects unauthenticated visitors to `/`, and raw HTML files under `UIUX/html/` are not served statically.
- **HTTPS everywhere:** `7327-lb-01` redirects `:80` to `:443` using a Let's Encrypt certificate for `kbail.kingsleykoech.tech`.
- **Sticky sessions:** HAProxy pins each browser session to `web01` or `web02` via `SERVERID` cookies so in-memory Express sessions stay consistent.

---

## Local setup

### Prerequisites
- Node.js 20+ and npm
- A Google Cloud project with the **Gmail API** enabled and an **OAuth 2.0 Web Client**
- A **Groq** API key (and optionally a **Gemini** API key for fallback)

### 1. Clone and install

```bash
git clone https://github.com/kingsleykipkoech/K-Bail-.git
cd K-Bail-/BACKENDS
npm install
```

### 2. Configure `BACKENDS/.env`

Create `BACKENDS/.env` (gitignored):

```env
PORT=8080
HOST=127.0.0.1
SESSION_SECRET=replace-with-a-random-secret

GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
GOOGLE_REDIRECT_URI=http://localhost:8080/auth/google/callback

GROQ_API_KEY=your-groq-api-key
GEMINI_API_KEY=your-gemini-api-key

GMAIL_MAX_RESULTS=100
MAX_SUMMARIZE_PER_SYNC=5
```

| Variable | Purpose | Default |
| :--- | :--- | :--- |
| `PORT` | Port Express listens on | `8080` |
| `HOST` | Bind address | `127.0.0.1` |
| `SESSION_SECRET` | Secret used to sign session cookies | `kbail-dev-secret` |
| `GOOGLE_CLIENT_ID` | Google OAuth client ID | Required for Google login |
| `GOOGLE_CLIENT_SECRET` | Google OAuth client secret | Required for Google login |
| `GOOGLE_REDIRECT_URI` | Callback URL registered in Google Cloud | Required for Google login |
| `GROQ_API_KEY` | Primary AI summarizer API key | Required for live AI summaries |
| `GEMINI_API_KEY` | Fallback AI summarizer API key | Optional |
| `GMAIL_MAX_RESULTS` | Maximum messages fetched per sync | `100` |
| `MAX_SUMMARIZE_PER_SYNC` | Maximum messages summarized per sync | `5` |

### 3. Start the server

```bash
npm start
```

Open `http://localhost:8080`.

---

## API & route reference

| Method | Route | Auth | Description |
| :--- | :--- | :--- | :--- |
| `GET` | `/` | Public | Serves `landing.html` |
| `GET` | `/inbox` | Session | Serves `home_page.html` or redirects to `/` if not signed in |
| `GET` | `/auth/google` | Public | Redirects to Google OAuth 2.0 consent screen |
| `GET` | `/auth/google/callback` | Public | Exchanges authorization code for token and redirects to `/inbox` |
| `GET` | `/auth/sample` | Public | Starts a demo session using `sample_emails.json` and redirects to `/inbox` |
| `GET` | `/auth/logout` | Session | Destroys Express session, clears cookie, redirects to `/` |
| `GET` | `/api/health` | Public | Returns `{"status":"ok","storage":"browser-local"}` |
| `GET` | `/api/me` | Required | Returns `{ mode, email }` for the active session |
| `GET` | `/api/emails` | Required | Fetches emails and generates bullet summaries |
| `POST` | `/api/sync` | Required | Refreshes inbox and summarizes new emails |
| `GET` | `/api/emails/:id/full` | Required | Returns the full body of a single email |

---

## Project structure

```text
K-Bail-/
├── assets/
│   ├── clouds.mp4               # Background video used on landing and inbox views
│   ├── landing-preview.png      # Landing page screenshot
│   └── inbox-preview.png        # 3-pane inbox screenshot
├── BACKENDS/
│   ├── package.json             # Express, express-session, dotenv
│   ├── sample_emails.json       # Sample mailbox dataset for demo mode
│   └── server.js                # Express app, OAuth, Gmail API, Groq/Gemini summarizer
└── UIUX/
    ├── html/
    │   ├── landing.html         # Login view (served at /)
    │   └── home_page.html       # 3-pane inbox view (served at /inbox)
    ├── scripts/
    │   └── home_page.js         # Inbox state, search/filter/sort, localStorage caching
    └── styles/
        ├── base.css             # Shared reset, typography, glass layers, background video
        ├── landing.css          # Envelope login card and Sample Mail callout
        └── home_page.css        # 3-column layout, summary/full-mail panes, profile modal
```

---

## External APIs & credits

K-Bail- relies on three external APIs, with client-side `localStorage` caching and automatic provider fallback for resilience:

| API | Provider | Purpose in K-Bail- | Credit / Docs |
| :--- | :--- | :--- | :--- |
| **Gmail API & OAuth 2.0** | Google LLC | Read-only inbox listing, message headers, and on-demand full body retrieval | [developers.google.com/gmail/api](https://developers.google.com/gmail/api) |
| **Groq Chat Completions** | Groq, Inc. | Primary LLM used to generate 1–3 bullet email summaries | [console.groq.com/docs](https://console.groq.com/docs) |
| **Gemini GenerateContent** | Google LLC | Automatic fallback LLM if Groq fails or hits rate limits | [ai.google.dev](https://ai.google.dev/) |

Error handling:
- If Groq returns an error or rate-limit response, the backend automatically falls back to Gemini.
- Expired or invalid OAuth sessions return `401` from `/api/*` and redirect the client cleanly back to `/`.
- Missing or failed upstream API responses surface readable error messages without crashing the Express process.

---

## Production deployment

The production environment runs across three Ubuntu 20.04 servers:

| Server | Role | Stack |
| :--- | :--- | :--- |
| **`7327-lb-01`** | Load balancer & TLS termination | HAProxy (`:80` -> `:443` 301), Let's Encrypt SSL, `SERVERID` sticky cookies |
| **`7327-web-01`** | Application node 1 | Nginx (`:80` -> `127.0.0.1:8080`) + Node.js 20 (`kbail.service`) |
| **`7327-web-02`** | Application node 2 | Nginx (`:80` -> `127.0.0.1:8080`) + Node.js 20 (`kbail.service`) |

### 1. Deploy application nodes (`7327-web-01` and `7327-web-02`)

Run the following on **both** web servers:

1. **Install Git and Node.js 20 LTS, then clone and install dependencies:**
   ```bash
   sudo apt update && sudo apt install -y git curl ca-certificates
   curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
   sudo apt install -y nodejs
   git clone https://github.com/kingsleykipkoech/K-Bail-.git ~/K-Bail-
   cd ~/K-Bail-/BACKENDS && npm install --omit=dev
   ```

2. **Create `~/K-Bail-/BACKENDS/.env`** on both nodes with identical `SESSION_SECRET`, API keys, and:
   ```env
   GOOGLE_REDIRECT_URI=https://kbail.kingsleykoech.tech/auth/google/callback
   ```

3. **Run Express as a `systemd` service (`/etc/systemd/system/kbail.service`):**
   ```ini
   [Unit]
   Description=K-Bail Node.js Backend
   After=network.target

   [Service]
   Type=simple
   User=ubuntu
   WorkingDirectory=/home/ubuntu/K-Bail-/BACKENDS
   ExecStart=/usr/bin/node /home/ubuntu/K-Bail-/BACKENDS/server.js
   Restart=on-failure
   Environment=NODE_ENV=production

   [Install]
   WantedBy=multi-user.target
   ```
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable --now kbail
   ```

4. **Configure Nginx reverse proxy (`/etc/nginx/sites-available/kbail`):**
   ```nginx
   server {
       listen 80;
       server_name kbail.kingsleykoech.tech;

       location / {
           proxy_pass http://127.0.0.1:8080;
           proxy_http_version 1.1;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto https;
       }
   }
   ```
   ```bash
   sudo ln -sf /etc/nginx/sites-available/kbail /etc/nginx/sites-enabled/kbail
   sudo nginx -t && sudo systemctl reload nginx
   ```

### 2. Configure load balancer (`7327-lb-01`)

1. **Issue the Let's Encrypt SSL certificate and combine it into an HAProxy PEM:**
   ```bash
   sudo systemctl stop haproxy
   sudo certbot certonly --standalone -d kbail.kingsleykoech.tech
   sudo mkdir -p /etc/haproxy/certs
   sudo sh -c 'cat /etc/letsencrypt/live/kbail.kingsleykoech.tech/fullchain.pem /etc/letsencrypt/live/kbail.kingsleykoech.tech/privkey.pem > /etc/haproxy/certs/kbail.kingsleykoech.tech.pem'
   sudo chmod 600 /etc/haproxy/certs/kbail.kingsleykoech.tech.pem
   sudo systemctl start haproxy
   ```

2. **Configure `/etc/haproxy/haproxy.cfg`** with SNI TLS termination, HTTP-to-HTTPS redirect, round-robin balancing, and `SERVERID` sticky cookies (so Express in-memory sessions stay pinned to the same web server during OAuth and API requests):
   ```haproxy
   frontend http-in
       bind *:80
       bind *:443 ssl crt /etc/haproxy/certs/kbail.kingsleykoech.tech.pem
       reqadd X-Forwarded-Proto:\ https
       redirect scheme https code 301 if !{ ssl_fc }
       default_backend web-backend

   backend web-backend
       mode http
       balance roundrobin
       cookie SERVERID insert indirect nocache
       server 7327-web-01 35.178.212.47:80 check cookie web01
       server 7327-web-02 18.130.235.76:80 check cookie web02
   ```

3. **Validate and reload HAProxy:**
   ```bash
   sudo haproxy -c -f /etc/haproxy/haproxy.cfg && sudo systemctl reload haproxy
   ```

### 3. Verify load balancing & end-to-end flow

1. **Confirm round-robin distribution across both web servers** (each fresh request without a `SERVERID` cookie alternates between `web01` and `web02` and returns `200 OK`):
   ```bash
   curl -sI https://kbail.kingsleykoech.tech/api/health
   curl -sI https://kbail.kingsleykoech.tech/api/health
   ```
   Observed response headers:
   - First request: `HTTP/1.1 200 OK` with `set-cookie: SERVERID=web01; path=/`
   - Second request: `HTTP/1.1 200 OK` with `set-cookie: SERVERID=web02; path=/`

2. **Confirm end-to-end application flows over HTTPS:**
   - `GET /` serves the login page (`200 OK`) and `GET /inbox` redirects unauthenticated requests to `/`.
   - **Sample Mail** (`/auth/sample`) and **Continue with Google** (`/auth/google`) both authenticate into `/inbox`, load `/api/me` and `/api/emails` (`200 OK`), populate AI summaries, fetch `/api/emails/:id/full` on **Full Read**, and clear the session via `/auth/logout`.

---

## Challenges & how they were solved

| Challenge | Solution |
| :--- | :--- |
| **OAuth & session state across 2 web nodes** | Express stores OAuth tokens and sessions in RAM (`express-session`), so round-robin requests without affinity would lose login state between `web-01` and `web-02`. Solved by enabling HAProxy `cookie SERVERID insert indirect nocache` sticky sessions on `7327-lb-01`. |
| **Google OAuth callback rejecting raw IPs** | Google OAuth requires a domain name and HTTPS in production rather than a raw load-balancer IP. Solved by pointing `kbail.kingsleykoech.tech` to `lb-01`, issuing a Let's Encrypt certificate, and configuring SNI + `:80 -> :443` redirect in HAProxy. |
| **LLM rate limits & latency on inbox sync** | Summarizing every email on every page load is slow and hits API quotas. Solved by capping summaries per sync (`MAX_SUMMARIZE_PER_SYNC=5`), caching emails/summaries in browser `localStorage`, and adding automatic Gemini fallback behind Groq. |

---

