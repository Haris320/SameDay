# plan.md: SameDay build plan

Companion to `CLAUDE.md`. CLAUDE.md says *what* we're building and the rules; this file says *what to do, in what order*, with the exact API details gathered from each sponsor's docs on Oct 24, 2026. APIs marked beta can change: if a call fails, re-check the linked doc before guessing.

---

## 0. Before 8:30 AM (do these at home or on the commute)

### Accounts and keys

- [ ] **Stripe:** create an Individual account (organizers asked for this). Find the test and live secret keys. Install the Stripe CLI if you can (`stripe login`), useful for forwarding webhooks.
- [ ] **Whop sandbox:** sign up at `sandbox.whop.com` (separate from production). You need **one** business, not three:
  - create the **SameDay** business (the platform; it holds the aid balance)
  - complete its sandbox **verification (KYC)** if the dashboard offers it. The docs say the sending account must be verified to make transfers.
  - create an API key at `sandbox.whop.com/dashboard/developer` with the Admin role (or at least `payout:transfer_funds` and `company:create_child`)
  - copy SameDay's `biz_` ID from the browser address bar (`sandbox.whop.com/dashboard/biz_XXXX/...`)
  - The agent creates the payees ("Con Edison (demo)", "Demo Landlord LLC") itself through the API. No extra accounts needed.
- [ ] **Terac:** create a researcher account at terac.com and an organization. Generate an API key in Organization Settings → API Keys. Install the MCP in Claude Code so your coding agent can inspect Terac directly:
  `claude mcp add --transport http terac https://terac.com/api/mcp` then `/mcp` to authenticate.
- [ ] **Anthropic API key**, **Supabase project** (URL + service key), **Lovable account**, **Exa key** (optional).
- [ ] **Public HTTPS URL for your backend.** Terac webhooks require a public `https` host and refuse localhost. Pick one now: deploy to Render/Railway, or use a tunnel (`cloudflared tunnel --url http://localhost:8000` or ngrok). Test that it works before you leave.
- [ ] **Join the event Slack.**

### Demo data

- [ ] Write 5–6 fake applicants (names, household size, income, savings, flags like children or disability).
- [ ] Make fake documents for each: shutoff notice (with amount, account number, shutoff date), pay stub, ID card, rent ledger for one eviction case. Plain PDFs or PNGs from a doc template are fine. Plant **one deliberate mismatch** (wrong address or amount) in one case so the demo shows the agent catching it.

### Questions to ask at kickoff (or in Slack)

1. Is Stripe for our projects' payments or for prize payouts? Live mode or test mode?
2. How do we get Terac researcher access and an API key today (self-serve sign-up routes to a demo booking)? How do credits work, and can we get a **Terac sandbox organization** for testing webhooks? (Sandbox orgs have to be set up by Terac.)
3. (Whop team) Do sandbox connected accounts receive balance transfers without KYC, and how does the sending account get verified in sandbox?
4. Any judging criteria or sponsor prizes beyond the brief?
5. Any rules on pre-written code?

---

## 1. What the docs say: integration notes

### Terac (the host, the most important integration)

**How it actually works:** you create an *opportunity* (a paid task), Terac recruits verified people, and each person gets a *submission*. If your task lives on your own site, Terac sends the participant to your `task_url` with their submission ID attached; when they finish, your page redirects them back to Terac.

- **Base URL:** `https://terac.com/api/external/v2` (v2, beta)
- **Auth:** `Authorization: Bearer $TERAC_API_KEY`. Every POST needs a JSON body and `Content-Type: application/json`, even `-d '{}'`; a bodyless POST returns `415`.
- **Rate limit:** 100 requests, then a 60-second quiet period resets it. `429` includes `Retry-After`.
- **Shortcut:** the "Get organization context" endpoint (path in the API reference) returns a markdown briefing on your org, balance, and how to build an opportunity. Call it first.
- **Flow:** create a project → `POST /opportunities` (draft) → `POST /opportunities/{id}/launch`.
- **Create opportunity, required fields:** `title`, `project_id`, `num_participants` (1–1000), `business_type` (`"b2c"` or `"b2b"`), `tasks` (at least one: `sequence`, `task_type`, `review_type`, `task_url`, `duration_minutes`).
  - You must send either `filters` or `unrestricted_audience: true`, or the create is rejected.
  - **A study cannot launch without screening questions**, and each must have an answer someone can pass with. Add one simple qualifier (for example, "Have you ever reviewed financial or benefits documents?" with a passing answer).
  - Recruitment window `expected_days_to_complete` is at least 5 days, but people can start right away.
- **Review type:** use `manual_review` (API-only). A completed redirect then waits for *you* (the agent) to approve or reject, and only an approval pays the reviewer. This is exactly the zero-human story: the agent hires a human, checks their work came back, and releases their pay.
- **Task URL (your reviewer page):** Terac appends `teracSubmissionId`, `submissionId`, and `taskId` as query params. When the reviewer finishes, redirect their browser to:
  `https://terac.com/api/external/callback?teracSubmissionId={id}&result=completed`
  (other `result` values: `screened_out`, `quota_full`, `rejected`).
- **Submissions:** `GET /opportunities/{id}/submissions`, `GET /submissions/{id}`, `POST /submissions/{id}/approve`, `POST /submissions/{id}/reject`.
- **Webhooks:** `POST /hooks/subscriptions` with `target_url` + `event_types` (`submission.status.change`, `submission.approved`), then confirm with `POST /hooks/subscriptions/{id}` and body `{}`. Verify `X-Terac-Request-Signature` = base64(HMAC-SHA256(secret, `X-Terac-Request-Timestamp` + raw body)). Deduplicate on `X-Event-ID`. Respond `2xx` within 10 seconds, then do the work.
- **Sandbox:** a separate Terac-provisioned org where nobody is recruited or paid. `POST /opportunities/{id}/test-submissions` plays a fake participant through your flow (sandbox key only). Great for testing; ask for one at kickoff.
- **Docs:** terac.com/docs/developers · OpenAPI: terac.com/docs/developers/openapi.json

### Whop (aid out)

- **Sandbox:** API `https://sandbox-api.whop.com/api/v1`, dashboard `sandbox.whop.com`. Sandbox and production keys look identical, so name them differently in `.env`.
- **Python SDK:** `pip install whop-sdk`; `Whop(token=..., environment=WhopEnvironment.SANDBOX)`.
- **Sandbox limitation:** **payouts to bank accounts are not available in sandbox.** That's fine: we use **transfers**, which move money between Whop balances.
- **Design: SameDay is a platform, payees are connected accounts.**
  - **Create a payee:** `POST /accounts` (SDK `accounts.create`) with SameDay's Account API key. Body: `email` (required for connected accounts; use Gmail aliases like `you+conedison@gmail.com`), `title`, `metadata` (put `external_id` = our payee ID). Returns a `biz_` ID. Permission `company:create_child`. Creation is keyed on email + `external_id`, so retries return the same account.
  - **Who must be verified:** the docs say the **sending** account must complete KYC and have enough balance. They don't state a KYC requirement for the recipient of a balance transfer; recipient KYC is tied to *withdrawing* (payouts). Test this in the first hour.
  - Fallbacks if connected accounts don't work in sandbox: payees as separate hand-made sandbox businesses → claim links → `MOCK_WHOP=1`.
- **Create transfer:** `POST /transfers`, permission `payout:transfer_funds`.
  - Balance transfer body: `type: "balance"`, `origin_id` (your `biz_`), `destination_id` (payee `biz_`/`user_`), `amount` (number, e.g. `25.00`), `currency: "usd"`, plus `idempotence_key`, `notes`, `metadata` (put `case_id` here).
  - Status can come back `processing`; final is `succeeded` or `failed`. Listen for `transfer.completed` / `transfer.failed` webhooks or poll `GET /transfers/{id}`.
  - **Claim links** (`type: "claim_link"`, `origin_id`, `amount`, optional `expires_at`): a funded link anyone can redeem. Useful fallback when a payee isn't on Whop: "the landlord gets a link and claims the payment."
- **Funding the sandbox aid balance:** create a product + checkout in your sandbox account and buy it with test card `4242 4242 4242 4242` (any future date, any CVC). Do this in the first hour, then check that a transfer succeeds. Purchased funds may sit in **pending** balance; a transfer from pending still works but credits the recipient's pending balance. This balance stands for the program budget; Stripe donations are tracked in our own `funding` ledger (Stripe money doesn't flow into Whop).
- **Docs:** docs.whop.com/developer/concepts · docs.whop.com/developer/guides/sandbox · docs.whop.com/api-reference/beta/transfers/create-transfer · docs.whop.com/api-reference/beta/accounts/create-account · docs.whop.com/developer/platforms/enroll-connected-accounts · docs.whop.com/developer/platforms/collect-payments-for-connected-accounts

### Stripe (money in, required by organizers)

- Create two **Payment Links** in the dashboard: "Fund a case" (customer chooses the amount) and "Run your program: per-case fee" (fixed price). Payment Links need no frontend code; show them as QR codes.
- Webhook: `checkout.session.completed` (Payment Links produce Checkout Sessions). Verify the signature with `stripe.Webhook.construct_event(payload, sig_header, STRIPE_WEBHOOK_SECRET)` using the raw body. Put `link_type` in the Payment Link's metadata to tell the two apart.
- Local testing: `stripe listen --forward-to localhost:8000/webhooks/stripe`.

### Lovable (frontend)

- Builds a React app on Supabase and publishes it publicly. Use it for: applicant intake, program dashboard, public "Fund a case" page, and the **Terac reviewer page** (it must be public because reviewers open it). Lovable can also install the Terac MCP, but keep Terac calls in the backend.

### Exa (optional, cut first)

- Only if time allows: search for related assistance programs to show in screening. Otherwise use a static list: SNAP, Medicaid, HEAP.

---

## 2. Build phases

Each phase ends with a check. Don't start the next phase until the check passes.

### Phase 1: Setup (8:30–9:30)
- Repo per CLAUDE.md layout, `.env` with all keys, Supabase tables, backend deployed or tunneled at a public URL.
- Whop sandbox: confirm SameDay is verified, fund the aid balance with a 4242 test purchase, create "Con Edison (demo)" as a connected account with `POST /accounts`.
- **Check:** Terac's organization context call returns your org; a $1 Whop sandbox transfer to the new connected account reaches `succeeded`; Stripe CLI forwards a test event. If the Whop check fails, go down the fallback list now, not at 3 PM.

### Phase 2: Documents → structured case (9:30–11:00)
- `extract.py` with Claude vision on each fake document; `verify.py` cross-document checks.
- **Check:** all seeded cases extract correctly; the planted mismatch is flagged with exact fields.

### Phase 3: Reviewer page + Terac launch (11:00–12:00)
- Reviewer page reads `teracSubmissionId` from the URL, assigns the next unreviewed case to that submission, shows the case packet (documents, extracted fields, flags, rubric result), and collects approve / reject / needs info + a reason. On submit: save to `approvals`, then redirect to the Terac callback with `result=completed`.
- Create the opportunity: `num_participants` = number of seeded cases, `task_type` matching a value the API accepts (check the reference), `review_type: "manual_review"`, `task_url` = your reviewer page, one screening question, filters (US adults) or `unrestricted_audience`. Subscribe to webhooks.
- **By 12:00: launch it.** Real people take hours.
- **Check:** opportunity status is `active`; webhook subscription is confirmed.

### Phase 4: Rubric, triage, state machine (12:00–2:30)
- `rubric.py` (deterministic), `triage.py`, `run.py` state machine, `decisions` logging.
- **Check:** `python -m pipeline.run --all` takes every seeded case to `queued_for_review` with priorities and logged reasons.

### Phase 5: Money (2:30–4:00)
- Stripe webhook → `funding` ledger. On a Terac reviewer decision: the agent compares it with its own result, approves the Terac submission (paying the reviewer), and if the case is approved and funds are available, sends the Whop transfer to the payee with `idempotence_key = case_id`.
- **Check:** a real Stripe payment updates the balance; one approved case gets a `succeeded` Whop transfer.

### Phase 6: Dashboard (4:00–5:30)
- Lovable: case board, live decision log (Supabase realtime), case detail, money in/out, metrics, public "Fund a case" page with the QR code.
- **Check:** a case moving through the pipeline updates the dashboard live.

### Phase 7: Screening + end-to-end (5:30–6:30)
- `screen.py`, metrics, full run on every seeded case, rehearse one live case.
- **Check:** one fresh case goes from intake to payment without you touching anything except the upload.

### Phase 8: Freeze (6:30–8:30)
- No new features. Slides, demo script from CLAUDE.md, rehearse three times. Record a backup screen video of a full run in case the network fails during demos.

**Cut list if behind (in order):** Exa → related-program screening → claim links → triage vulnerability flags → case detail polish.

---

## 3. Demo-day risks and fallbacks

| Risk | Fallback |
|---|---|
| Terac reviewers don't finish in time | Show any completed real reviews; run the rest with `MOCK_TERAC=1` and say so |
| Whop transfer refused because SameDay isn't verified | Finish sandbox verification in the dashboard; ask the Whop team on site (it's their office) |
| Connected accounts don't work in sandbox | Hand-made payee businesses (Gmail aliases), then claim links, then `MOCK_WHOP=1` and say so |
| Whop sandbox transfer fails | Use a claim link; if that fails, `MOCK_WHOP=1` and say so |
| Stripe webhook not reaching you | Poll recent Checkout Sessions on dashboard load |
| Wi-Fi dies during demo | Backup screen recording |
| Organizers change rules | Check Slack hourly; update CLAUDE.md first |
