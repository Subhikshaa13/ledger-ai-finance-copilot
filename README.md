# Ledger — AI Financial Copilot

A personal finance dashboard: load transactions (CSV or the built-in sample), get
AI-assisted categorization, see deterministic spend/income totals and rule-based
alerts, track savings goals, and ask a chat assistant questions about your own
numbers.

This version is wired for real deployment: **the Anthropic API key lives only on
the server**, never in the browser. The frontend (`public/index.html`) calls your
own backend (`server.js`), which calls Anthropic on its behalf.

## How it's structured

```
ledger-app/
├── server.js         Express server: serves the frontend + proxies AI calls
├── package.json
├── .env.example       Copy to .env and fill in your key
└── public/
    └── index.html     The whole frontend (HTML/CSS/JS in one file)
```

- All money math (totals, category breakdown, anomaly rules, goal projections) is
  computed in plain JavaScript in the browser — never sent to the model to
  "calculate." The model only reasons over numbers it's given.
- Two endpoints do the AI work:
  - `POST /api/categorize` — takes a list of transactions, returns a category per
    transaction.
  - `POST /api/ask` — takes a question plus a JSON snapshot of your computed
    summary, returns a grounded answer.
- Data currently lives in the browser's `localStorage` (per device/browser) so it
  survives a refresh. There's no multi-user database or login yet — see
  "Next steps for a real multi-user product" below if you want that.

## Run it locally

Requires Node.js 18+.

```bash
cd ledger-app
npm install
cp .env.example .env
# edit .env and paste your real Anthropic API key
npm start
```

Then open `http://localhost:3000`.

Get an API key at https://console.anthropic.com — the free/trial tier is enough
to try this out; production use is billed per token by Anthropic.

## Deploying it

Any Node host works. A few straightforward options:

### Render / Railway / Fly.io (easiest)
1. Push this folder to a GitHub repo.
2. Create a new "Web Service" (Render) or project (Railway/Fly) pointing at the
   repo.
3. Set the build command to `npm install` and the start command to `npm start`.
4. Add environment variables in the host's dashboard: `ANTHROPIC_API_KEY` (required)
   and optionally `ANTHROPIC_MODEL`.
5. Deploy. The platform gives you a public HTTPS URL.

### Any VPS (e.g. a $5 droplet)
```bash
git clone <your-repo>
cd ledger-app
npm install --production
cp .env.example .env   # fill in ANTHROPIC_API_KEY
npm start               # or run it under pm2 / systemd so it survives reboots
```
Put it behind a reverse proxy (nginx/Caddy) for HTTPS and a real domain.

### Docker (optional)
If your host prefers a container, a minimal Dockerfile is:
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
ENV PORT=3000
EXPOSE 3000
CMD ["npm", "start"]
```

## Environment variables

| Variable            | Required | Default          | Notes                                  |
|---------------------|----------|------------------|-----------------------------------------|
| `ANTHROPIC_API_KEY`  | Yes      | —                | Never expose this to the browser.       |
| `ANTHROPIC_MODEL`    | No       | `claude-sonnet-5`| Swap to a cheaper model for high volume.|
| `PORT`               | No       | `3000`           | Port the server listens on.             |

## Security & production notes

- **Rate limiting**: `server.js` includes a very basic in-memory per-IP limiter
  (20 requests/minute) so a single user can't rack up unbounded API spend. For
  real traffic, replace it with something backed by Redis (e.g.
  `rate-limiter-flexible`) since in-memory state doesn't share across server
  instances.
- **CORS** is currently wide open (`cors()` with no options) since the frontend
  and backend are served from the same origin. If you split them onto different
  domains, lock this down to your actual frontend origin.
- **Input limits**: both endpoints cap payload size and reject empty/oversized
  input before calling the model, so malformed requests can't waste API calls.
- The model is instructed to answer only from the JSON snapshot it's given and
  to say so when the data isn't there — but it can still misread or misstate
  data, so treat its answers as an assistant, not a source of truth. All the
  actual totals and alerts come from local deterministic JS.

## Next steps for a real multi-user product

Right now every visitor to the same deployment shares nothing (data is per
browser via `localStorage`), which is fine for a personal-use tool but not for
multiple people to use safely. If you want that:
- Add authentication (e.g. a simple email/password or OAuth flow).
- Add a real database (Postgres/SQLite) and move transactions/goals server-side,
  scoped per user.
- Move the rate limiter to be per-user rather than per-IP.
