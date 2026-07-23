# IITH Sanchari — Progress Log

**Read this whole file before touching the code.** It's the compressed memory of every session that's worked on this project. Don't rely on chat history from a previous conversation being available to you — it won't be. This file is how continuity actually happens.

## Rule for every agent (Claude, Gemini, Codex, or human) touching this repo

**After you make changes, append a new dated entry to the top of the [Log](#log-newest-first) section below — newest first, never delete or rewrite past entries.** Use the template at the bottom of this file. Be specific: what changed, why, what's still broken. A vague entry is worse than no entry — the next agent needs to make real decisions from what you write.

If you're a human reading this and an agent just finished work without logging it, ask it to add an entry before you close the session.

---

## Project at a glance

- **What:** ticket booking + live schedule app for campus shuttle and outstation buses (Patancheru, Miyapur) serving the IIT Hyderabad community. Developed in coordination with the IIT Hyderabad Transport Department.
- **Live at:** [transport.iith.online](https://transport.iith.online)
- **Repo:** [github.com/mschandan/iith-transport](https://github.com/mschandan/iith-transport) → Hostinger Git auto-deploy (push to `main` = live within seconds, but see the CDN caching gotcha below)
- **Local path:** `~/claude/transport`
- **Design system:** [`docs/DESIGN.md`](./DESIGN.md) — read before touching any UI
- **Original full spec:** [`docs/BUILD_SPEC.md`](./BUILD_SPEC.md) — historical, partially superseded (palette changed entirely, nav/accounts scope changed — this log is more current than that doc for anything it disagrees with)

## Stack (locked, don't relitigate without strong reason)

- **Frontend:** static HTML/CSS/JS. No framework, no build step, no npm, no bundler.
- **Backend:** PHP 8, no framework, no Composer. Raw `cURL` to the Razorpay REST API — chosen specifically because Hostinger shared hosting doesn't guarantee Composer/SSH access.
- **Database:** MySQL — **not yet provisioned.** This is the single biggest open item; see below.
- **Hosting:** Hostinger shared hosting, deployed via GitHub → Hostinger Git integration.
- **Payments:** Razorpay Standard Checkout, orders created server-side (fare is never trusted from the client).
- **Font:** SF Rounded (`ui-rounded` stack) — see DESIGN.md.

## Folder structure

```
transport/
├── index.html, ticket.html, legal.html   ← served pages, stay at repo root
├── assets/                                ← served: app.css, app.js, logo.svg/png, hero.png, qrcode.js
├── api/                                   ← served PHP endpoints
│   └── config.php                         ← git-ignored, real secrets live here ONLY (see below)
├── docs/                                  ← this file, DESIGN.md, historical planning docs
└── inspirations/                          ← design-reference/ (locked mockups), raw-drops/ (git-ignored
                                              source images — duplicates of what's in assets/, kept
                                              locally for reference only, never committed)
```

**Do not move `index.html`/`ticket.html`/`legal.html`/`assets/`/`api/` out of repo root** — Hostinger's Git deploy serves the repo root as `public_html` directly. Moving them requires a manual hPanel deploy-path change first, or the live site goes down.

## Current state (as of 2026-07-24)

### Done
- Full home screen: internal shuttle schedule (live countdown), Patancheru/Miyapur route cards with real fare/schedule data, live-tracker links, expandable schedule panels
- Complete visual identity: "Sunrise IITH" palette, logo, SF Rounded typography — see DESIGN.md
- Razorpay Standard Checkout: order creation (server-side fare lookup, never client-trusted), signature verification, all live-tested against the real Razorpay API
- Boarding-pass ticket page (`ticket.html`) — real scannable QR (vendored `qrcode.js`, no CDN), journey duration + estimated arrival (not a stale countdown), icon-labeled meta grid, "Verified" stamp
- `legal.html` — privacy/terms/payments page, linked from footer
- "Coming Next" preview grid (static, non-interactive) previewing roadmap features

### Not done / open items, roughly in priority order
1. **MySQL database not provisioned.** Blocks: persistent "My Tickets" (right now a ticket only exists at its one-time URL — close the tab and it's gone), driver check-in / single-use QR verification, ticket-by-email. This is the next big unlock — most other open items depend on it.
2. **Razorpay account still "under review."** Test keys have died/been invalidated multiple times for reasons outside our control (not a caching or code issue — confirmed via direct `curl` against Razorpay's API each time). International cards are blocked (domestic-only account); UPI didn't appear in one checkout attempt, likely same review-status cause. **The full paid flow has never been confirmed to work end-to-end with a real successful payment** — everything up to and including the `handler` callback is built and tested, but the actual "payment succeeds → redirect fires" moment hasn't happened yet.
3. No accounts/login — guest checkout only, by design for now (do not add name/signup fields without being asked; this was a deliberate scope cut).
4. "Buy" → "Show Ticket" button state not implemented (needs either localStorage as a device-local stopgap, or the database for a real solution).
5. Ticket is not emailed. Razorpay may send its own generic payment receipt if it collects an email, but that's not our ticket — building real delivery needs a transactional email service (Hostinger's PHP `mail()` is unreliable/spam-flagged), which needs its own API key/account decision.
6. Driver-scan / check-in screen not built (needs #1).

### Constraints and decisions — don't re-litigate these without a real reason
- No Composer/npm/build step, ever, on this hosting.
- Razorpay account is Chandan's **personal KYC**, not an official IIT Hyderabad entity. `legal.html` deliberately says "developed in coordination with the IIT Hyderabad Transport Department," not "official" or "endorsed" — don't strengthen that claim without Chandan confirming the institute relationship has formalized.
- No name field collected at checkout (Razorpay only collects phone by default) — ticket shows phone, not a passenger name.
- "Departs" on the ticket says "Scheduled," never "On time" — no live tracking feed exists to back up an on-time claim.
- Don't have an agent regenerate the logo/hero artwork — those are provided assets from Chandan, not agent-generated. If a change is needed, ask for a new file.
- Bento-grid layout was tried and explicitly reverted ("the previous model was better") — don't re-propose it.

---

## Log (newest first)

### 2026-07-24 — Repo reorganization + persistent docs
- Moved `BUILD_SPEC.md`, `AGENT_PROMPT.md` → `docs/`; `design-reference/` → `inspirations/design-reference/`; raw source image drops → `inspirations/raw-drops/` (git-ignored, duplicates of what's already in `assets/`)
- Added `docs/DESIGN.md` (full design system reference) and this file
- Reason: session context had grown very large; this is the persistence mechanism so any future session/agent can pick up without replaying the whole conversation

### 2026-07-24 — Ticket page icon/badge polish
- Added icon badges + subtitles ("Approx.", "Scheduled", weekday) to the ticket meta grid, per a reference screenshot Chandan provided
- Green "Paid" pill badge instead of plain "· Paid" text
- Dropped the `+91` prefix from the displayed contact number (implied for India) and fixed it wrapping awkwardly onto two lines

### 2026-07-24 — Ticket: journey time instead of a frozen countdown
- The ticket previously baked in a live "~X min" countdown at purchase time, which goes stale immediately since the ticket page never re-renders after load. Replaced with journey duration (Patancheru ~1 hr, Miyapur ~1 hr 40 min, confirmed by Chandan) and a computed estimated-arrival time — both evergreen, correct no matter when the ticket is viewed later
- Added a rotated "VERIFIED" stamp seal next to the QR

### 2026-07-24 — Bento-grid redesign, then reverted
- Tried restructuring the home screen top section into a bento-style tile grid (shuttle rows as independent side-by-side cards). Chandan: "I don't like this... revert back." Reverted via `git revert`, confirmed clean.

### 2026-07-24 — Ticket screen + Razorpay checkout wiring
- Built the full boarding-pass ticket page: real QR (vendored `qrcode-generator` by kazuhikoarase, MIT, no CDN), signed HMAC token (separate secret from the Razorpay secret), fetches phone number from Razorpay's Payments API post-verification
- No database yet, so the ticket is **stateless by design** — everything needed to render it is encoded in the signed token itself, round-tripped through Razorpay's own order `notes` field
- Wired the Buy button's success handler to redirect to `ticket.html?t=<token>` instead of showing an alert

### 2026-07-24 — Razorpay integration debugging saga
- Multiple rounds of test API keys dying (`expired`, then `Authentication failed`) — confirmed via direct `curl` against Razorpay's API each time, independent of our code/config/caching. Root cause never fully explained by Razorpay; likely tied to the account's "under review" status.
- Found (via Razorpay's own payment records, fetched with a working key) that a real test payment attempt failed with `international_transaction_not_allowed` — the account is domestic-cards-only, and the commonly-used generic test card (`4111 1111 1111 1111`) is flagged international. Fix: use the actual domestic test card (`5104 0155 5555 5558`) or test UPI (`test@razorpay`).
- **Lesson for future debugging:** don't guess — query Razorpay's API directly (`curl -u key:secret https://api.razorpay.com/v1/payments`) to see the real error Razorpay recorded, rather than reasoning from the frontend symptom alone.

### 2026-07-23/24 — Razorpay Standard Checkout built
- `api/create-order.php`, `api/verify-payment.php`, `api/razorpay.php` (shared cURL/config helpers), `api/fares.php` (server-side fare table — frontend never dictates price)
- `api/config.php` (real secrets) is git-ignored; `api/config.sample.php` is the committed template
- **Manual step every time credentials change:** the server's `api/config.php` must be edited directly via Hostinger File Manager — it is NOT deployed by git push, by design (it's git-ignored). This has repeatedly been the actual cause of "it's still broken" when the code itself was already fixed.

### 2026-07-23/24 — Full visual retheme: "Sunrise IITH"
- Replaced the original Phase 1 "Ivory & Oxblood" palette entirely with a red-orange/gold palette sampled from the IITH crest and the hero illustration
- New logo (provided by Chandan, not agent-generated), new hero illustration, ticket cards redesigned as full-bleed gradient "tickets" with a corrected tear-off perforation (see DESIGN.md for the exact math — it was wrong on the first attempt, looked like plain circles instead of clean semicircles)
- Font switched to SF Rounded after a side-by-side comparison of 5 options
- Bottom nav removed (Cabs/Tickets/Me were unreachable placeholders); added a static "Coming Next" 3×3 grid instead, listing actual roadmap items (Cab Sharing, Lost & Found, Tourist Places, Railway Station, Airport, Complaints, Request a Trip, Rewards, Account Management)
- Added `legal.html` (privacy/terms/payments), linked from footer, contact `transport@iith.ac.in`
- Fixed a Hostinger CDN caching bug: static assets were cached 7 days and didn't bust on deploy, so pushed code silently wasn't reaching visitors. Fixed by adding `?v=N` cache-busting query params to `assets/app.css`/`assets/app.js` in `index.html`/`legal.html` — **bump the version number on every future CSS/JS change**, or the CDN will serve stale files again.

### 2026-07-23 — Phase 1: initial build
- Static HTML/CSS/JS home screen built from a locked design reference (`inspirations/design-reference/`): internal shuttle schedule, Patancheru/Miyapur route cards with real fare/schedule data (fares: ₹30 / ₹100), live-tracker links
- Original palette "Ivory & Oxblood" (since fully replaced, see above)
- Pushed to GitHub (`mschandan/iith-transport`) via a repo-scoped SSH deploy key; connected to Hostinger Git auto-deploy

---

## Entry template — copy this for your new entry

```markdown
### YYYY-MM-DD — Short title
- What changed
- Why
- What's still open / broken / next, if relevant
```
