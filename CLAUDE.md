# CLAUDE.md: SameDay (solo build)

Working name: **SameDay**. An agent-run disbursement service that gets emergency assistance to people in days or hours instead of weeks. It prepares each case, gets a human sign-off, and pays the bill directly.

- **Event:** Zero-Human Company Hackathon by Terac, Sat Oct 24, 8:30 AM to 8:30 PM, Whop HQ, Brooklyn. Solo build.
- **Demo program:** modeled on NYC HRA's Emergency Assistance ("One Shot Deal"): one-time cash help for utility shutoffs, eviction risk, and similar emergencies. Processing is supposed to take about 30 days and requires an interview.
- **Business model:** cities and nonprofits pay SameDay per case processed to run their emergency aid programs. SameDay has no caseworkers on staff; it buys human review on demand through Terac.
- **Revenue today (organizers require Stripe for collecting payments):** two Stripe payment links: (1) "Fund a case": anyone can fund emergency aid directly, and (2) "Run your program": a per-case fee for nonprofits. Money in through Stripe, aid out through Whop.

## The problem (for the pitch)

- SNAP and cash assistance applications in NYC must legally be processed within 30 days.
- FY2024 (IBO report): 65% of SNAP applications on time (goal 90.6%); 42% of cash assistance applications on time (target 96%).
- Summer 2023 peak: ~46,000 cash assistance and ~4,000 SNAP applicants waiting. The city cleared it by hiring nearly 1,000 staff.
- Many denials are for procedural reasons (missing or incomplete paperwork), not ineligibility.
- These figures are 2023–2024; present them as the problem's history.

## Golden rules

1. **One loop, end to end, working live.** Intake → prepare → triage → human sign-off → payment → related-program screening. Nothing else until this works.
2. **The agent prepares; a human approves money.** No payment without a Terac reviewer's approval recorded in `approvals`. This is a product principle, not a shortcut. Never add a bypass except the clearly labeled `MOCK_TERAC` demo fallback.
3. **Every decision is logged** to `decisions` with a one-line reason. The decision log is the demo centerpiece.
4. **Fake data only.** Fake applicants, fake documents, fake landlords and utilities. Never real personal information.
5. **Demo rubric, not real HRA rules.** Eligibility uses a simplified published rubric based on the factors HRA lists (income, household size, reason for need, savings, housing affordability, disability, plan to pay going forward). Label it "demo rubric" in the UI.
6. **Every external call has a mock mode** (`MOCK_TERAC`, `MOCK_WHOP`, `MOCK_EXA`).
7. **Never invent API shapes** for Terac, Whop, or Stripe. Read current docs before writing those clients. Use Whop's test/sandbox environment for payouts.
8. Small functions, no premature abstraction. 12 hours, one person.
9. **Backend must be reachable at a public HTTPS URL** (deploy or tunnel). Terac webhooks refuse localhost, and reviewers must be able to open the reviewer page.
10. **Whop runs in sandbox** (`https://sandbox-api.whop.com/api/v1`). Sandbox has no bank payouts, so aid moves by **transfers** between Whop balances (`POST /transfers`, `type: "balance"`). Say so in the demo. SameDay is a Whop **platform**: payees (utilities, landlords) are **connected accounts the agent creates itself** with `POST /accounts`. See "Whop setup" below.
11. **Check Slack for rule updates** (organizers post all day). If they change payment or judging rules, update this file first.

## Stack

| Layer | Choice |
|---|---|
| Frontend | Lovable (React + Supabase): applicant intake + program dashboard |
| Database | Supabase (Postgres, realtime for decision log and case board) |
| Backend | Python 3.11, FastAPI |
| LLM | Claude via Anthropic Python SDK (vision for document reading) |
| Human review | Terac API/MCP |
| Money in | Stripe (required by organizers): Payment Links or Checkout, `checkout.session.completed` webhook |
| Money out | Whop API (`whop-sdk`), sandbox base URL `https://sandbox-api.whop.com/api/v1`, balance transfers to payees, `transfer.completed` / `transfer.failed` webhooks |
| Research (optional) | Exa, only for finding related programs if it saves time; otherwise a static list |

## MCP servers (for the coding agent, not the product)

Use these to read docs and inspect accounts while building. The product's own code still calls the REST APIs directly.

```
claude mcp add --transport http terac https://terac.com/api/mcp
claude mcp add --transport http whop-docs https://docs.whop.com/mcp
claude mcp add --transport http whop https://mcp.whop.com/mcp
claude mcp add --transport http stripe https://mcp.stripe.com
claude mcp add --transport http supabase https://mcp.supabase.com/mcp
claude mcp add --transport http exa https://mcp.exa.ai/mcp
```

Then run `/mcp` in Claude Code to sign in to each one.

- **Whop API MCP may act on the production account**, not sandbox. Use it only to read; never move money through it. Use `whop-docs` freely.
- **Stripe MCP:** confirm it's in the sandbox/test account before creating anything.
- **Supabase MCP:** point it only at the hackathon project; it can change tables.

## Repo layout

```
backend/
  main.py                # FastAPI routes, Whop + Terac callbacks
  pipeline/
    intake.py            # create case from form + uploads
    extract.py           # Claude vision: documents -> structured fields
    verify.py            # cross-document consistency checks
    rubric.py            # demo eligibility rubric -> eligible/partial/ineligible + reasons
    triage.py            # priority score from urgency (days to shutoff/eviction)
    review.py            # Terac opportunity (manual_review, task_url = reviewer page), webhooks, approve submissions
    payees.py            # find or create the payee as a Whop connected account (POST /accounts)
    pay.py               # Whop balance transfer to payee after approval
    screen.py            # related programs: SNAP, Medicaid, HEAP
    run.py               # state machine driving a case through all steps
    fund.py              # Stripe webhook -> funding ledger; checks balance before payout
  clients/ terac_client.py whop_client.py stripe_client.py llm.py exa_client.py
  db.py  log.py  mocks/
frontend/                # exported from Lovable
seed/
  applicants/            # 5–6 fake applicants with fake documents (PDF/PNG)
  payees.json            # fake utility + landlord (name, email alias, external_id); Whop IDs filled in by the agent
```

## Money flow

- **In (Stripe):** "Fund a case" and "Run your program" payment links. The Stripe webhook writes to `funding` and logs a decision ("Received $X; available aid balance now $Y").
- **Out (Whop):** after human approval, the agent pays the payee from the aid balance. If the balance can't cover the case, it stays approved-but-waiting and gets paid first when new funds arrive, in priority order.
- **Two balances, one ledger:** Stripe money does not move into Whop automatically. The Whop sandbox balance is pre-funded at setup and stands for the program's budget; Stripe payments add to the internal `funding` ledger. The agent checks the ledger before paying, and the Whop transfer moves the money. Explain it that way if asked.
- **Ledger:** funds in, aid paid out, and per-case fees are all visible on the dashboard. Never mix fee revenue and aid money in the same number.
- **Fallback:** see the fallback order in "Whop setup". Stripe collection must still be real.

## Whop setup

**Chosen design: SameDay is a platform; payees are connected accounts.**

- One Whop business: **SameDay**, the platform. Its Account API key creates connected accounts and sends transfers.
- The first time a case needs a new payee, `payees.py` calls `POST /accounts` with `email`, `title`, and `metadata.external_id` (our payee ID). It returns a `biz_` ID; store it in `payees`. Creation is keyed on email + `external_id`, so a retry returns the same account.
- `pay.py` then calls `POST /transfers` with `type: "balance"`, `origin_id` = SameDay's `biz_`, `destination_id` = the payee's `biz_`, `amount`, `currency: "usd"`, `idempotence_key` = case ID, `metadata.case_id`.
- Why this beats three hand-made businesses: no extra logins, and the agent onboards its own payees, which fits the zero-human theme ("the agent registered the landlord and paid them, no human involved").

**Rules from the docs:**

- The **sending account must complete KYC and have enough balance**. Complete SameDay's sandbox verification first.
- API key permissions: `payout:transfer_funds` (transfers) and `company:create_child` (connected accounts), or the Admin role.
- A transfer can return `processing`; final status comes from `GET /transfers/{id}` or the `transfer.completed` / `transfer.failed` webhooks.
- Funds bought with a test card may be pending. A transfer from pending balance still works but credits the recipient's pending balance; that's fine for the demo.

**Fallback order** (test the first option in the first hour and move down only if it fails):

1. Connected accounts created by the agent (above).
2. Payees as separate sandbox businesses made by hand (sign up with Gmail aliases such as `you+utility@gmail.com`), then transfer to their `biz_` IDs.
3. **Claim links:** `POST /transfers` with `type: "claim_link"`, `origin_id`, `amount`. No recipient account needed; the agent sends the payee a link to claim the money.
4. `MOCK_WHOP=1`, and say so in the demo.

## Case state machine

`submitted → extracted → verified → scored → queued_for_review → approved | rejected | needs_info → paid → screened → closed`

- Any step can move to `needs_info` with a specific list of what's missing, sent back to the applicant. Catching gaps before submission is a core feature: it prevents procedural denials.
- Each transition writes a row to `decisions`.

## What makes it technical (build these well)

1. **Document extraction:** Claude vision pulls structured fields from a shutoff notice, lease/rent ledger, pay stub, and ID: names, addresses, amounts, dates, account numbers.
2. **Cross-document verification:** name and address match across documents; the amount owed on the notice matches the requested amount; pay stub dates are recent; shutoff date is in the future. Mismatches are flagged with the exact fields.
3. **Rubric scoring:** deterministic Python, not the LLM. The LLM only explains the result in plain language.
4. **Triage queue:** priority = urgency (days until shutoff or eviction) + vulnerability flags (children, disability). Reviewers see the most urgent cases first.
5. **Human-in-the-loop:** Terac reviewer gets a packet (extracted fields, flags, rubric result, source documents) and answers approve / reject / needs info with a reason.
6. **Payment:** pays the payee (utility or landlord), not the applicant, with an idempotency key derived from the case ID. Records the Whop transaction ID.
7. **Related-program screening:** after payment, check likely eligibility for SNAP, Medicaid, and HEAP and pre-fill a summary for each.

## Data model (Supabase)

- `applicants` (id, name, household_size, monthly_income, savings, flags_json)
- `cases` (id, applicant_id, type shutoff|eviction|other, amount_requested, payee_id, status, priority, created_at, paid_at)
- `documents` (id, case_id, kind, file_url, extracted_json)
- `checks` (id, case_id, name, passed, detail)
- `rubric_results` (id, case_id, outcome, score, reasons_json)
- `approvals` (id, case_id, terac_task_id, reviewer_decision, reason, decided_at)
- `funding` (id, source fund_a_case|program_fee, amount, stripe_session_id, received_at)
- `payees` (id, name, kind utility|landlord, email, external_id, whop_account_id, created_by_agent_at)
- `payments` (id, case_id, payee_id, amount, whop_id, status)
- `screenings` (id, case_id, program, likely_eligible, notes)
- `decisions` (id, case_id, step, choice, reason, created_at) ← realtime

## Dashboard (Lovable)

1. **Applicant intake:** short form + document upload, in plain language.
2. **Program dashboard:**
   - Case board by status, sorted by priority
   - Live decision log
   - Case detail: documents, extracted fields, flags, rubric reasons, reviewer decision, payment receipt
   - Headline metrics: median time from request to payment, cases caught with missing documents before submission, programs screened per person, funds raised (Stripe), aid paid out (Whop)
3. **Public "Fund a case" page** with the Stripe payment link and a live counter of funds raised and families helped.

## Terac review design

- One Terac opportunity, `review_type: "manual_review"`, `task_url` = our public reviewer page. Terac appends `teracSubmissionId`; the page assigns that reviewer the next unreviewed case and, on submit, redirects to `https://terac.com/api/external/callback?teracSubmissionId={id}&result=completed`.
- The agent then approves the Terac submission via API, which pays the reviewer. The agent hires the human and releases their pay.
- Launching requires at least one screening question, plus `filters` or `unrestricted_audience: true`.
- Full API details are in `plan.md` section 1.

## Terac timing (important)

Real reviewers take hours. At **12:00**, launch the opportunity for the 5–6 seeded cases so approvals land by mid-afternoon. In the demo, show those real approvals and run one fresh case live through intake, extraction, verification, and scoring; trigger its payment from an approval that arrived earlier, or use `MOCK_TERAC` and say so.

## Schedule

| Time | Goal |
|---|---|
| Before 8:30 | Stripe Individual account created and verified; join event Slack |
| 8:30–9:30 | Repo, keys, Supabase tables, Lovable project, Stripe payment links + webhook. Whop: verify SameDay, fund it with a 4242 test purchase, create one connected account, send a $1 transfer |
| 9:30–12:00 | Seed applicants + fake documents; extract.py and verify.py working |
| **12:00** | **Launch Terac review tasks for seeded cases** |
| 12:00–2:30 | rubric.py, triage.py, state machine, decision logging |
| 2:30–4:00 | Whop payment on approval, webhook/callback handling |
| 4:00–5:30 | Lovable dashboard wired to Supabase realtime |
| 5:30–6:30 | screen.py, metrics, end-to-end run on all seeded cases |
| 6:30–8:30 | Feature freeze, slides, rehearse demo 3 times |

Cut list if behind (in order): Exa, related-program screening, triage vulnerability flags, case detail page polish.

## Commands

```
cd backend && pip install -r requirements.txt --break-system-packages
uvicorn main:app --reload
python -m pipeline.run --case seed/applicants/maria.json
MOCK_TERAC=1 MOCK_WHOP=1 python -m pipeline.run --all   # offline rehearsal
```

## Env vars

`ANTHROPIC_API_KEY`, `TERAC_API_KEY`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `WHOP_API_KEY` (sandbox key; name it clearly, since sandbox and production keys look identical), `WHOP_BIZ_ID` (SameDay's `biz_`), `WHOP_WEBHOOK_SECRET`, `SUPABASE_URL`, `SUPABASE_SERVICE_KEY`, `EXA_API_KEY` (optional), `MOCK_TERAC`, `MOCK_WHOP`, `MOCK_EXA`. Never commit keys; never put Whop keys in frontend code.

## Demo script (2–3 minutes)

1. "In 2023, tens of thousands of New Yorkers waited past the legal 30-day deadline for help. The city's fix was hiring nearly a thousand more staff."
2. Submit a fake applicant live: shutoff notice dated four days out. Decision log streams: documents read, one mismatch caught, rubric result, priority set to urgent.
3. Show a case a real Terac reviewer approved this afternoon. The agent registers the utility as a payee on its own and pays it through Whop.
4. Show screening: same person flagged as likely eligible for SNAP and HEAP.
5. Live money: a judge scans the "Fund a case" QR code and pays through Stripe; the balance updates and the next waiting case gets paid out through Whop.
6. Metrics: request to payment in hours, not weeks; real dollars raised today.
7. "No caseworkers on staff. The agent does the paperwork, a human approves every dollar, and the bill gets paid the same day."

## Judge questions

- **"Should an AI decide who gets public money?"** It doesn't. It prepares the case and a human approves every payment.
- **"What about fraud?"** Cross-document checks, human review, and paying the utility or landlord directly instead of the applicant.
- **"Would a city really pay through Whop?"** This is a prototype of a fully automated pipeline; Stripe and Whop are the payment rails here. A city would plug in its own.
- **"Did it make real money?"** Yes: show today's Stripe payments, split into aid funding and program fees.
- **"Is this a company?"** Yes: cities and nonprofits pay per case to run their emergency aid programs on it.
- **"Privacy?"** Demo uses fake data; a real deployment needs government-grade data handling.
