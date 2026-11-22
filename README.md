# K-Bail-

K-Bail- (Kingsley Brief Mail) is a privacy-first Gmail reader that summarizes your inbox into 1–3 clear bullet points while keeping the full email one click away.

Live application: **https://kbail.kingsleykoech.tech**

If you want to test the interface without signing into a Google account, click **Sample Mail** on the landing page to load a prebuilt demo inbox.

---

## Why it exists

Long email threads, newsletters, and automated notifications bury the few lines you actually need to read. Most AI email tools solve this by storing your mailbox on their own servers. K-Bail- takes the opposite approach: it reads your inbox on demand over OAuth, summarizes only the newest/unread items, and keeps everything in your browser rather than a server database.

---

## How the system works

### Infrastructure & traffic flow

```mermaid
flowchart LR
    Browser["User Browser\n(localStorage cache)"] -->|HTTPS :443| LB["7327-lb-01\nHAProxy + Let's Encrypt TLS\nSticky cookies (SERVERID)"]

    LB -->|HTTP :80| W1["7327-web-01\nNginx -> 127.0.0.1:8080\nExpress (kbail.service)"]
    LB -->|HTTP :80| W2["7327-web-02\nNginx -> 127.0.0.1:8080\nExpress (kbail.service)"]

    W1 & W2 -->|gmail.readonly| Gmail["Google Gmail API"]
    W1 & W2 -->|Primary| Groq["Groq API"]
    W1 & W2 -->|Fallback| Gemini["Gemini API"]
```

### Authentication & summarization flow

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
        App->>Google: Exchange code for access_token (stored in RAM session)
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

Because K-Bail- requests the `gmail.readonly` scope without undergoing Google's commercial OAuth verification review, Google displays a **"Google hasn't verified this app"** screen when you sign in:

1. Click **Advanced** at the bottom-left of the warning card.
2. Click **Go to kbail.kingsleykoech.tech (unsafe)**.
3. Grant the requested read-only permissions (`gmail.readonly` and `userinfo.email`) to open `/inbox`.

If you prefer not to sign in with your Google account, use **Sample Mail** on the landing page to test the full UI and summarization flow.

---

## Interface & features

The workspace at `/inbox` is split into three columns over a translucent glass layout:

1. **Inbox** — Lists synced messages with live search, `All` / `Today` filtering, and sorting (`Newest`, `Oldest`, `Sender`).
2. **Summarize** — Displays the 1–3 bullet AI summary for the selected email, generated with Groq first and automatically falling back to Gemini if Groq is unavailable or rate-limited.
3. **Full Mail** — Loads the full email body on demand via `GET /api/emails/:id/full` when you click **Full Read**.

Additional controls:
- **SYNC mail** triggers `POST /api/sync` to pull new messages and summarize them.
- **Settings** opens the profile modal showing the signed-in email (or `Sample Mailbox`) and a **Logout** action (`/auth/logout`).

---

## Privacy & security design

| Layer | What lives here | Lifecycle |
| :--- | :--- | :--- |
| **Browser (`localStorage`)** | Synced email metadata and generated bullet summaries | Stays on your device; cleared on logout/reset |
| **Server RAM (`express-session`)** | `mode`, `userEmail`, short-lived Google `accessToken` | Held in memory only; destroyed on `/auth/logout` |
| **Server disk / database** | Nothing | No database and no persisted user emails |

Other safeguards:
- **Minimal OAuth scopes:** Only `https://www.googleapis.com/auth/gmail.readonly` and `https://www.googleapis.com/auth/userinfo.email` are requested.
- **Clean, gated routes:** `/` serves the login page, `/inbox` checks the session and redirects unauthenticated visitors to `/`, and raw HTML files under `UIUX/html/` are not exposed as static paths.
- **HTTPS everywhere:** `7327-lb-01` redirects all HTTP (`:80`) traffic to HTTPS (`:443`) using a Let's Encrypt certificate for `kbail.kingsleykoech.tech`.
- **Sticky sessions:** HAProxy pins each client session to either `web01` or `web02` with `SERVERID` cookies so in-memory Express sessions stay consistent during OAuth and API calls.

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

### 2. Configure environment variables

Create `BACKENDS/.env` (this file is gitignored and must not be committed):

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

### 3. Run the server

```bash
npm start
```

Visit `http://localhost:8080`.

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
│   └── clouds.mp4               # Background video used on landing and inbox views
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

## Production deployment

The production setup runs across three Ubuntu servers:

| Server | Role | Configuration |
| :--- | :--- | :--- |
| **`7327-lb-01`** | Load balancer & TLS termination | HAProxy terminating HTTPS (`:443`) with Let's Encrypt certificate for `kbail.kingsleykoech.tech`, 301 redirect from `:80`, and `SERVERID` sticky session cookies |
| **`7327-web-01`** | Application node 1 | Nginx reverse proxy (`kbail.kingsleykoech.tech` -> `127.0.0.1:8080`) + Node.js 20 managed by `systemd` (`kbail.service`) |
| **`7327-web-02`** | Application node 2 | Nginx reverse proxy (`kbail.kingsleykoech.tech` -> `127.0.0.1:8080`) + Node.js 20 managed by `systemd` (`kbail.service`) |

Both web nodes share the same `SESSION_SECRET` and use `GOOGLE_REDIRECT_URI=https://kbail.kingsleykoech.tech/auth/google/callback` in `BACKENDS/.env`.

---

## Author

Built by **Kingsley Kipkoech** (`k.kipkoech@alustudent.com`)
