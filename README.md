# TableFlow (working name)

QR table ordering, live kitchen display, floor map, waiter calls, bill splitting and pay-at-table for restaurants.
"TableFlow" is a **placeholder brand**: change it with the `BRAND` environment variable (no code edits needed).

No npm packages are used at runtime. You only need **Node.js 22.5 or newer**.

## Run it locally (2 minutes)

```bash
cd tableflow
DEMO_MODE=1 PAYMENT_PROVIDER=mock npm start
```

The first start prints the admin, waiter and kitchen passwords **once**. Save them.
Open http://localhost:3000 — the landing page has a "Try the guest experience" button (a demo table).

| Screen | URL | Who |
|---|---|---|
| Guest app (from the QR code) | `/t/<table-token>` | Guests |
| Waiter floor map | `/floor` | waiter, admin |
| Kitchen display | `/kitchen` | kitchen, waiter, admin |
| Admin | `/admin` | admin |

`PAYMENT_PROVIDER=mock` **simulates** card / Apple Pay / Google Pay. It is blocked automatically when `NODE_ENV=production`.

## Tests

```bash
npm test                                   # 25 API + security tests (in-memory database)
node tests/e2e.js <url> <waiterPw> <kitchenPw> <adminPw> <outDir>   # browser test, needs Playwright + Chrome
```

## Configuration (environment variables)

See `.env.example`. Important ones:

* `BASE_URL` — your public https address. QR codes, sitemap and canonical links use it.
* `ADMIN_EMAIL`, `ADMIN_PASSWORD` — first-run owner account (a random password is printed if empty).
* `TRUST_PROXY=1` — only when behind nginx / Caddy / Cloudflare.
* `PAYMENT_PROVIDER=stripe` + `STRIPE_SECRET_KEY` + `STRIPE_WEBHOOK_SECRET` — enables card, Apple Pay and Google Pay (via Stripe Checkout).
* `RESEND_API_KEY` + `MAIL_FROM` — enables emailed receipts. Without them, receipts are still available by link.

Never put these in frontend code. They are only read by the server.

## Going live (checklist)

1. Get a small Linux server (1 vCPU / 1 GB RAM is enough for one restaurant) and install Node 22 LTS or newer.
2. Copy the project, create `.env` from `.env.example`, set `NODE_ENV=production`, `BASE_URL=https://your-domain`, `TRUST_PROXY=1`.
3. Run it under a process manager (systemd example below).
4. Put **Caddy** (automatic HTTPS) in front:
   ```
   your-domain.com {
     reverse_proxy 127.0.0.1:3000
   }
   ```
   Caddy obtains and renews the TLS certificate by itself.
5. DNS: create an `A` record for your domain pointing to the server IP (and `AAAA` for IPv6 if used).
6. Sign in at `/login`, then in Admin: set restaurant name, currency, tax, Wi-Fi, review link; edit the menu; arrange tables on the floor layout.
7. **Print the QR codes** from Admin → Tables & QR. Scan one with a real phone first.
8. Stripe (optional): create the account, add a webhook endpoint `https://your-domain/api/webhooks/stripe` for events `checkout.session.completed` and `checkout.session.expired`, copy the signing secret into `STRIPE_WEBHOOK_SECRET`.

systemd unit (`/etc/systemd/system/tableflow.service`):
```
[Unit]
Description=TableFlow
After=network.target
[Service]
WorkingDirectory=/opt/tableflow
EnvironmentFile=/opt/tableflow/.env
ExecStart=/usr/bin/node --no-warnings src/server.js
Restart=always
User=tableflow
[Install]
WantedBy=multi-user.target
```

## Backup and recovery

All data is in one SQLite file (`data/app.db`, plus `-wal`/`-shm` files while running).
Nightly backup (safe while running):
```
sqlite3 /opt/tableflow/data/app.db ".backup '/backups/app-$(date +%F).db'"
```
Copy backups off the server (object storage or another machine). To restore: stop the service, replace `data/app.db`, start the service.

## Monitoring recommendations

* Uptime check on `https://your-domain/api/health` (UptimeRobot, Better Stack, etc.).
* `journalctl -u tableflow` for logs. Add Sentry or similar if you want error alerts.
* Keep Node.js and the OS updated.

## Security model (short)

* Passwords: scrypt. Sessions: random 192-bit ids stored hashed, HttpOnly + SameSite=Strict cookies (Secure on https).
* Roles: admin / waiter / kitchen, enforced on the server for every endpoint.
* Guests have no accounts: the **table QR token** is the key. Staff must approve every order before the kitchen sees it. Admin can issue a new QR code any time ("New code").
* Prices always come from the server. All SQL is parameterised. All user text is rendered with `textContent`. Strict Content-Security-Policy.
* Rate limits on login, ordering, calls and payments (in memory, single server).
