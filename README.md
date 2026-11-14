# Corporate Actions Copilot

An operations proof of concept for a fund manager working corporate actions against a synthetic mutual-fund book. It takes a corporate-action notice from intake through evidence review, deterministic impact calculation, elections, maker-checker approval, a simulated instruction, settlement reconciliation and an audit trail.

All companies, schemes, holdings, notices, users, instructions and settlements are synthetic. The book belongs to a fictional fund house, "Arka Mutual Fund".

## What it does

- **Dashboard, Portfolio, Corporate actions, Issuers, Analysis** pages over seeded synthetic events and schemes.
- **Notice intake** (`/intake`): paste the public URL of a notice; the API captures the source, extracts terms and matches the issuer against the Arka schemes. Term extraction uses OpenAI and needs `OPENAI_API_KEY`.
- **Deterministic calculations, validation gates, role checks, workflow transitions and reconciliation classifications** run on the server, not in the browser or the model.
- **Instructions are never sent.** The only instruction status the API accepts is `SIMULATED - NOT SENT`; there is no custodian or market connection.
- **Optional model features** (live discovery and the judgement note on an event) also use `OPENAI_API_KEY` and report themselves unavailable without it.

Sample notice PDFs are in `artifacts/corporate-actions-copilot/public/demo-notices/`.

## Sign-in and roles

The web app requires sign-in. It uses an OpenID Connect flow (Replit's identity provider by default, configured by `REPL_ID` and `ISSUER_URL`), and sessions are stored in the `sessions` database table.

After sign-in, the API resolves an operational role. There are two roles:

| Role | Can do |
| --- | --- |
| Fund Manager | Intake, validation, calculation, elections, simulated instruction, reconciliation |
| Compliance | Approve or return a submitted election (maker-checker) |

How the role is resolved:

- **Demo mode** (`CORPORATE_ACTIONS_POC=true` and `NODE_ENV` not `production`): the API acts as one of two demo users, Rohan Iyer (Fund Manager, the default) or Nisha Kapoor (Compliance). There is no role switcher in the UI; switch with `POST /api/session` and body `{"actorId": "USR-005"}` (Compliance) or `{"actorId": "USR-004"}` (Fund Manager).
- **Otherwise:** the signed-in user must match exactly one entry in `CORPORATE_ACTIONS_ROLE_DIRECTORY`, for example `[{"email": "checker@company.example", "role": "Compliance"}]`. Without a match the UI shows "Access is not assigned" and mutations return 401.

## Prerequisites

- Node.js 24+
- pnpm 10+
- A PostgreSQL database

## Database setup

1. Set `DATABASE_URL` to a PostgreSQL connection string.
2. Push the schema:

   ```bash
   pnpm --filter @workspace/db run push
   ```

3. The API seeds the synthetic corporate-action scenarios on first read.

## Run locally

```bash
pnpm dev
```

This starts the API on `API_PORT` (8080 by default) and the web app on `WEB_PORT` (25700 by default). The Vite `/api` proxy uses the same `API_PORT`. Open `http://localhost:25700`. The API health check is at `http://localhost:8080/api/healthz`.

## Tests and checks

```bash
pnpm test        # calculation unit tests and database-backed API workflow tests
pnpm run build   # full workspace typecheck and build
```

The API workflow tests need a database with the schema already pushed. They restore the seeded fixtures after each test.

## Environment variables

| Variable | Required | Default | Purpose |
| --- | --- | --- | --- |
| `DATABASE_URL` | Yes | None | PostgreSQL connection string |
| `SESSION_SECRET` | Yes | None | Signs session cookies. The API refuses to start without it. |
| `REPL_ID` | For sign-in | None | OIDC client ID |
| `ISSUER_URL` | No | `https://replit.com/oidc` | OIDC issuer |
| `API_PORT` | No | `8080` | API port and Vite `/api` proxy target in `pnpm dev` |
| `WEB_PORT` | No | `25700` | Vite port in `pnpm dev` |
| `BASE_PATH` | No | `/` | Vite base path |
| `CORPORATE_ACTIONS_POC` | No | `false` | `true` enables the demo users outside production |
| `CORPORATE_ACTIONS_ROLE_DIRECTORY` | Outside demo mode | None | JSON list mapping an OIDC user `id` or `email` to `Fund Manager` or `Compliance` |
| `CORS_ALLOWED_ORIGINS` | No | Same origin only | Comma-separated extra origins allowed to call the API |
| `OPENAI_API_KEY` | For model features | None | Notice extraction, live discovery and the judgement note |
