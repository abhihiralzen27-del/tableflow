# Project report — TableFlow (working name)

Date: 2026-11-21. Scope taken from your pitch: guests scan a table QR, see the full menu (allergies, spice, sizes, deals), order from their phone; staff approve with one tap; kitchen display with live orders, timers and stations; Wi-Fi, rewards and games for guests; one-tap waiter call with a live floor map; bill split, tip, pay at the table (cash, bank transfer, card, Apple Pay, Google Pay); receipt by email; one-tap review. Brand names were removed as you asked; "TableFlow" is only a placeholder.

## 1. Status legend

| Status | Meaning |
|---|---|
| ✅ Completed and tested | Automated test or browser run passed in this session |
| 🟡 Implemented, not tested | Code exists, but it needs outside accounts/keys or a real device that I did not have |
| 🔧 Needs external configuration | Works only after you provide keys, a domain or a server |
| ❌ Not implemented | Not built |

## 2. Features

| Feature | Status |
|---|---|
| QR codes per table (own generator, printable, "new code" to revoke) | ✅ decoded back correctly: 200/200 with OpenCV's Aruco decoder, 193/200 with its classic decoder (decoder weakness, not data errors). **Not yet scanned with a real phone camera or from paper.** |
| Guest menu: categories, search, veg/vegan/non-veg filter, allergen hide-filter, spice level, sizes, special requests, deal prices | ✅ |
| Cart and order placement, prices always calculated on the server | ✅ |
| Staff approve / reject order, then it goes to the kitchen | ✅ |
| Kitchen display: tickets, live timers, late warning colours, stations, start → ready, sound alert | ✅ (sound not verified by ear) |
| Live status for guests ("Cooking", "Ready") without reloading | ✅ |
| Waiter call (waiter / water / clean / bill) and live floor map with colour-coded status | ✅ |
| Bill: whole bill, equal split, pick-your-own items, tip presets and custom tip, tax | ✅ |
| Cash and bank transfer, confirmed by staff | ✅ |
| Card / Apple Pay / Google Pay | ✅ in demo mode (simulated). 🟡🔧 real payments through Stripe Checkout are written but **never run** (no keys, no network). |
| Receipt page by private link | ✅ |
| Receipt email | 🟡🔧 stored in an outbox and sent through Resend only if you add keys; the sending code was not run |
| One-tap star review, optional text, link to Google review | ✅ |
| Wi-Fi card with copy button | ✅ |
| Rewards: points for paying, games, quiz; redeem; staff mark as given | ✅ |
| Games: tic-tac-toe between two tables (live), food quiz (answers kept on server) | ✅ |
| Admin: overview, menu CRUD, deals, rewards, tables, drag-and-drop floor layout, QR print, staff accounts, settings, enquiries | ✅ (drag with a finger on a touch screen not tested; keyboard arrows are supported) |
| Landing page with working contact form | ✅ |
| Hindi interface | ❌ (English only, as you chose) |
| UPI / Razorpay | ❌ see "Problems" |
| Dish photos / image upload | ❌ emoji placeholders (uploads were left out on purpose to avoid upload vulnerabilities; can be added safely later) |
| Multiple restaurants on one installation (multi-tenant SaaS) | ❌ one restaurant per installation |
| Refunds, cancelling an item after it was sent, inventory | ❌ |

## 3. Pages

`/` landing · `/t/<token>` guest app (Menu, Cart, My table, Play, More, Pay sheet) · `/r/<token>` receipt · `/login` · `/floor` · `/kitchen` · `/admin` (Overview, Menu, Deals & rewards, Tables & QR, Staff, Settings) · 404 page · `robots.txt` · `sitemap.xml`.

## 4. Technology

Node.js 22 (built-in HTTP server and SQLite), no runtime dependencies; vanilla JavaScript front end (no framework, no build step); Server-Sent Events for live updates; scrypt password hashing. I chose this because the sandbox had no network for installing packages, and it also keeps the attack surface and hosting cost very small. Trade-off: one server process, one SQLite file.

## 5. Domain verification

**Not performed.** No brand/domain was chosen, and I never claim availability without checking a registrar or RDAP source. Once you pick a name, I can check it. I have not bought or registered anything.

## 6. Hosting recommendation

One small Linux VPS (1 vCPU, 1 GB) with Caddy for automatic HTTPS, systemd to keep it running, nightly SQLite backups copied off the server. Details are in `README.md`.

## 7. Security audit (self-audit by me, plus automated tests; not an independent third-party audit)

| # | Area | Result |
|---|---|---|
| 1 | SQL injection | All queries are parameterised; injection strings in table tokens and inputs were tested ✅ |
| 2 | XSS | All dynamic text uses `textContent`; script/`<img onerror>` payloads in notes were tested in browser and did not run; strict CSP (no inline script or style) ✅ |
| 3 | CSRF | SameSite=Strict cookies + Origin check + JSON-only bodies; cross-origin and form-type posts rejected in tests ✅ |
| 4 | Broken authentication | scrypt, generic error, constant-time compare, login rate limit ✅ |
| 5 | Broken access control | Role checks on every staff/admin endpoint; tests confirm kitchen cannot use floor/admin APIs ✅ |
| 6 | Insecure endpoints | Unknown routes 404, wrong methods 405, body size limit 64 KB ✅ |
| 7 | Sensitive data exposure | Password hashes and secrets never returned; no stack traces to clients; guests only get their own table's data ✅ |
| 8 | File uploads | No upload feature exists ✅ (n/a) |
| 9 | Weak passwords | Min 10 chars with letters and numbers for staff accounts ✅ |
| 10 | Rate limiting | Login, orders, calls, payments, new visitors, live connections ✅. Limits are in memory (reset on restart; one server only). |
| 11 | Session management | Random 192-bit ids stored hashed, 12 h expiry, HttpOnly, Secure on https, revoked on logout / password change / disable ✅ |
| 12 | Dependencies | No runtime dependencies, so nothing to audit. Keep Node patched. ✅ |
| 13 | CORS | No CORS headers: other websites cannot call the API from a browser ✅ |
| 14 | Exposed env/keys | `.env`, source, tests and database are not served (tested path traversal) ✅. Keys only read on the server. |
| 15 | Security headers | CSP, nosniff, frame-deny, referrer, permissions, COOP present ✅; HSTS is sent only on https (not testable locally) 🟡 |
| 16 | Database access control | Single server-side connection; no direct DB exposure ✅ |
| 17 | Input validation | Every endpoint validates type, range and length; tampering tests pass ✅ |

**Unresolved risks (honest list)**

* Anyone who has a table's QR link can place orders for that table and see the Wi-Fi password. Mitigations: staff must approve every order; admin can issue a new code. A photo of the QR taken from a distance still works until it is rotated.
* Login lockout is per email address, so someone could deliberately lock an account for 15 minutes.
* No two-factor login for admins.
* Rate limits reset when the server restarts.
* Stripe webhook signature check is implemented per Stripe's documented scheme but was only tested for rejecting bad signatures, not with a real Stripe event.
* Cash / bank-transfer amounts are trusted once the waiter taps "Received".

## 8. Performance and quality results (measured locally in Chrome, no network throttling; **not Lighthouse**)

| Page | First paint | Largest paint | Layout shift | Requests | Size |
|---|---|---|---|---|---|
| Landing | 156 ms | 156 ms | 0 | 5 | 27 KB |
| Guest menu | 176 ms | 176 ms | 0.04 | 8 | 54 KB |

Also checked: no horizontal scroll on a 390 px phone; every button/input has a label; one `h1` per page; tap targets ≥ 36 px after a fix; no console errors in any page after fixes. Lighthouse, axe and real-device tests were not available offline.

Bugs found and fixed during testing: inline styles blocked by the CSP on the landing/404 pages; allergy section collapsing whenever a live update arrived; floor-map legend colours missing; unneeded `.00` on whole prices; SSE double-header bug; wrong currency validation; QR mask scoring (added the finder-pattern penalty).

## 9. SEO readiness

Title, meta description, canonical URL, Open Graph + Twitter card with a generated 1200×630 image, JSON-LD (`SoftwareApplication`), one `h1`, clean headings, `robots.txt` (blocks admin/guest/receipt pages), `sitemap.xml`, `noindex` on private pages. No ranking is promised. Only one public page exists, so there is little to rank yet.

## 10. External services and credentials still required

Domain name; a server; Stripe account + keys + webhook secret (for card/wallets); Resend (or another sender) + verified domain (for receipt emails); optional Google review link.

## 11. Estimated running cost (rough, from general knowledge, **not checked against live prices today**)

Server about US$5–10 / month; domain about US$10–15 / year (country domains vary); email service has free tiers at small volume; payment fees are charged per transaction by the payment provider (vary by country and method). Caddy, Node and SQLite are free.

## 12. Exact steps to launch

1. Choose the brand and domain (I can check availability first).
2. Decide the payment gateway for your country (see Problems below).
3. Create the server, follow `README.md` → "Going live".
4. Set `BASE_URL`, `NODE_ENV=production`, `TRUST_PROXY=1`, admin email/password.
5. Sign in, enter the real menu, tax, Wi-Fi, tables; arrange the floor.
6. Print one QR, scan it with 2–3 different phones, then print all.
7. Run a staff training session with a few test orders (one tested table, then clear it).
8. Add Stripe/email keys and do one real small payment before opening to guests.

## 13. Problems where I need your decision

1. **Payment gateway.** I built Stripe (card, Apple Pay, Google Pay). If your restaurant is in India, UPI and Razorpay are usually what owners expect, and Stripe availability may be limited there. Which country and gateway do you want? I can add a Razorpay adapter.
2. **One restaurant or many?** Right now one installation = one restaurant. If you plan to sell it to many restaurants (as the pitch suggests), a multi-tenant version needs a bigger redesign (Postgres, restaurant accounts, billing).
3. **Brand and domain**: name not chosen yet.
4. **Dish photos**: do you want safe image upload (admin only, re-encoded images)?
5. **Hosting**: do you want me to prepare a Docker setup as well?
