# Implementation TODO — M1 & M2

Companion to [DESIGN.md](./DESIGN.md). Work items are ordered; each has an explicit **Verify** step so we never move on to the next item without evidence the previous one works.

Legend: `[ ]` not started · `[~]` in progress · `[x]` done

Testing layers used throughout:
- **Unit** — `pytest`, no network, fixtures recorded from real responses.
- **Live check** — a small script hitting the real dependency (OpenTable, Groq, Sendblue), run manually.
- **E2E** — a real text from Hanlin's phone (`ALLOWLISTED_PHONE`) to the Sendblue number, asserting on the reply.

---

## Phase 0 — Project skeleton

- [ ] **0.1 Scaffold the Python project.** `pyproject.toml` (uv or pip-tools), `app/` package, `pytest`, `ruff`, `mypy` configured. Layout: `app/main.py` (FastAPI), `app/messaging/`, `app/agent/`, `app/availability/`, `app/booking/`, `app/scheduler/`, `app/store/`, `app/config.py`.
  - **Verify:** `ruff check .`, `mypy app`, and `pytest` all pass on an empty test suite; `uvicorn app.main:app` boots and `GET /health` returns 200.
- [ ] **0.2 Config loading.** `app/config.py` reads every var from `.env.example` via pydantic-settings, fails fast with a clear message on missing required vars.
  - **Verify:** unit test asserts a missing `LLM_API_KEY` raises a readable error; app boots with the real `.env`.
- [ ] **0.3 Restaurant config.** `restaurants.yaml` with Four Kings and Seven Hills: display name, aliases, OpenTable `rid`, timezone. `rid`s discovered from each restaurant's OpenTable URL.
  - **Verify:** unit test loads the file and resolves "four kings", "Four Kings SF", "seven hills" to the right `rid`s.

## Phase 1 — Availability via a cloud browser (build before the agent; it's the riskiest piece)

**Why this shape:** OpenTable's Akamai edge blocks this VM's IP for plain HTTP *and* for real Chrome running locally (verified against both restaurants). The live browser is the primary path, not a fallback — but it must originate from residential-grade IPs, i.e. a hosted cloud browser.

- [x] **1.0 Confirm the block and pick the approach.** Tested `httpx`, browser-like headers, local Chrome, and a server-side fetch API. Local paths return Akamai `Access Denied`; a third-party server-side fetch reached one restaurant but not the other, and exposed no slot data.
  - **Verified:** screenshots of `Access Denied` for both restaurants from local Chrome.
- [ ] **1.1 Stand up a cloud browser session.** Sign up for Browserbase (or Steel), store `BROWSER_API_KEY`, connect Playwright over CDP: `chromium.connect_over_cdp(session.connect_url)`, reusing `browser.contexts[0].pages[0]`.
  - **Verify:** a script loads the Four Kings OpenTable page through the cloud browser and prints the page title — i.e. no `Access Denied`. This is the go/no-go gate for the whole approach; if it fails, try the other provider and enable its stealth/proxy options before anything else.
- [ ] **1.2 Capture OpenTable's own availability call (Mode 1).** Attach a `page.on("response")` listener filtering for the availability GraphQL operation, navigate to the restaurant page with date/party size in the query string, and record the raw JSON as a fixture.
  - **Verify:** captured JSON contains slot times and a slot hash for a date with known availability; times match what the website shows for the same date/party size.
- [ ] **1.3 `CloudBrowserProvider.fetch(rid, day, party_size)`** normalizing the captured payload to `Slot(...)`, implementing the `AvailabilityProvider` protocol.
  - **Verify:** unit tests against the recorded fixture (normal / empty / malformed); a live run returns slots for both restaurants matching the website.
- [ ] **1.4 DOM scrape backstop (Mode 2).** If no availability response is captured, read the rendered slot buttons and their `href`s in the same session. Behind `AVAILABILITY_DOM_FALLBACK`.
  - **Verify:** disable network capture and confirm the scrape returns the same slot times as Mode 1 for the same date.
- [ ] **1.5 Clearance-cookie reuse.** Export the session's Akamai cookies (`_abck`, `bm_sz`, …) plus the request signature; replay subsequent lookups with `httpx` from our own process, and fall back to a fresh browser session on 403.
  - **Verify:** after one browser session, three consecutive `httpx` lookups succeed in under a second each; artificially expiring the cookies triggers exactly one new browser session, and no request loop.
- [ ] **1.6 Window ranking.** Slots inside the requested window ranked first (by proximity to window midpoint / stated preferred time); slots within 30 minutes of either edge included as flagged fallbacks; everything else dropped.
  - **Verify:** table-driven unit tests — window 7–9pm with candidate slots at 6:25/6:35/7:15/8:45/9:20/9:45 yields exactly `[7:15, 8:45]` in-window and `[6:35, 9:20]` as flagged fallbacks, in that rank order.
- [ ] **1.7 Cost and rate control.** Cache responses for a few minutes, one in-flight lookup per restaurant, and a hard cap on browser sessions per hour.
  - **Verify:** two identical lookups in quick succession create only one browser session; log line reports sessions used per lookup.

## Phase 2 — Booking Link Builder

- [ ] **2.1 Determine the real confirm-URL shape.** In the cloud browser, click a real slot up to (not through) the confirm step and record the resulting URL and which params are required — especially any slot token/hash and its expiry. The slot `href` captured in 1.4 is the primary evidence.
  - **Verify:** documented in `DESIGN.md`; a manually constructed URL opens the confirm page pre-filled with the right restaurant, date, time, and party size.
- [ ] **2.2 Implement `build_booking_link(slot)`.**
  - **Verify:** unit tests on URL construction; **live check** — build links for 3 real slots across both restaurants, open each, confirm all fields are pre-filled and no error page. This is the single most important check in M1.

## Phase 3 — Messaging

- [ ] **3.1 `MessagingClient` protocol + `SendblueClient`** (`send`, `parse_webhook`) and a `LoopMessageClient` stub raising `NotImplementedError`.
  - **Verify:** unit tests normalize a recorded Sendblue inbound payload into `InboundMessage`; **live check** — send myself one text from the Sendblue number and confirm it arrives as a blue bubble.
- [ ] **3.2 `POST /webhook/imessage`.** Verify the provider signature/`WEBHOOK_SHARED_SECRET`, reject non-allowlisted senders with a 200 + no-op, enqueue the message for handling, respond fast.
  - **Verify:** unit tests for a good payload, a bad signature (401), and a non-allowlisted sender (200, no reply sent); **live check** — ngrok tunnel up, webhook URL set in Sendblue, text the number and see the request in the ngrok inspector.
- [ ] **3.3 Conversation store.** SQLite `conversations` table; append every inbound/outbound/tool message.
  - **Verify:** unit test round-trips a transcript; after a live text, the row is present with the right role and phone.

## Phase 4 — Agent Core (M1 brains)

- [ ] **4.1 LLM client + tool-calling loop** against Groq (`llama-3.3-70b-versatile`, OpenAI-compatible), with a bounded tool-call iteration limit and timeouts.
  - **Verify:** live check — a trivial prompt with one fake tool returns a well-formed tool call.
- [ ] **4.2 System prompt.** Today's date/timezone, restaurant list, window semantics, pre-filled-link (never auto-book) policy, SMS formatting rules (short, no markdown).
  - **Verify:** prompt-level tests (fake tools, assert the model calls `check_availability` with correctly parsed args) over a fixture set of ~15 phrasings, including relative dates ("this Friday", "tomorrow"), party size in words ("a table for four"), and missing fields.
- [ ] **4.3 Wire the three tools** — `check_availability`, `create_monitor`, `build_booking_link` — with strict JSON schemas and validation of model-supplied args.
  - **Verify:** unit tests that malformed tool args produce a clarifying question to the user, not a crash.
- [ ] **4.4 Clarification + echo behavior.** When a required field is missing the agent asks one short question; every availability reply echoes the parsed restaurant/date/party size so mistakes are visible.
  - **Verify:** scripted multi-turn tests: "any tables at seven hills?" → asks for date and party size → user answers → agent searches.
- [ ] **4.5 Follow-up resolution via transcript replay** — "the 7:45 one", "what about Saturday?", "make it 4 people" resolve from prior turns; no state machine.
  - **Verify:** multi-turn tests asserting the second turn's tool args inherit unchanged fields from the first; explicitly confirm no dialogue-state variable exists in the codebase.

## Phase 5 — M1 end-to-end

- [ ] **5.1 Full loop wired:** webhook → agent → availability → reply → user confirms → booking link.
  - **Verify (M1 acceptance, real device):** from Hanlin's phone —
    1. "table for 2 at four kings friday around 7:30" → reply lists ranked openings within a few seconds, with edge slots labeled.
    2. "the 7:45 one" → reply contains a booking link.
    3. Tapping the link opens OpenTable pre-filled with the correct restaurant/date/time/party size.
    4. "what about seven hills same night?" → correct new search reusing party size and date.
    5. A text from a non-allowlisted number gets no reply.
    6. A date with no availability yields a clear "nothing available, want me to watch it?" reply, not silence or an error.
- [ ] **5.2 Failure-path handling.** OpenTable timeout/blocked, Groq error, Sendblue send failure → user gets a plain-English apology text, error logged with context.
  - **Verify:** fault injection for each dependency, asserting a text is still sent and nothing hangs.

## Phase 6 — M2 monitoring

- [ ] **6.1 Jobs store.** `monitors` and `notifications` tables per DESIGN.md, with de-dup on `slot_signature` and expiry.
  - **Verify:** unit tests for create/list/cancel/expire and de-dup.
- [ ] **6.2 `create_monitor` tool + conversational management.** "let me know if…", "what are you watching?", "stop watching seven hills".
  - **Verify:** multi-turn tests; live text creates a row with correctly parsed criteria.
- [ ] **6.3 APScheduler daily job** at 09:00 `America/Los_Angeles`: iterate active monitors, check availability, de-dup, notify on hit.
  - **Verify:** unit test the cron trigger computes the right next-run instant across a DST boundary; run the job function manually and confirm it texts on a hit and stays silent on a miss.
- [ ] **6.4 Hit → pick → link flow** reuses the M1 confirmation path from a proactively initiated conversation.
  - **Verify:** trigger the job manually against a monitor known to match; reply to the proactive text with a choice and confirm a valid booking link comes back.
- [ ] **6.5 Idempotency & quiet behavior.** Re-running the job the same day re-notifies nothing; expired monitors are skipped.
  - **Verify:** run the job twice in a row; exactly one notification.
- [ ] **6.6 M2 acceptance (real device):** create a monitor by text on day N; on day N+1 at 9:00 AM PST a proactive text arrives for a matching opening and the reply path returns a link.

## Phase 7 — Hardening & handoff

- [ ] **7.1 Structured logging + a `/health` check** covering DB, Groq reachability, and last successful OpenTable call.
- [ ] **7.2 README** with local run instructions (uv sync, `.env` from `.env.example`, `uvicorn`, ngrok, setting the Sendblue webhook).
- [ ] **7.3 CI** — GitHub Actions running ruff, mypy, pytest on PRs.
  - **Verify:** CI green on a PR.
- [ ] **7.4 Reserve a static ngrok domain** so the Sendblue webhook URL stops changing between dev sessions. *(Hanlin — ngrok dashboard.)*

---

## Open items needed from Hanlin

| Item | Needed for | Status |
| --- | --- | --- |
| **Cloud browser account + API key** (Browserbase ~$39/mo, or Steel free tier to start) | all of Phase 1 | **Blocking** — hard gate on real availability |
| Sendblue inbound webhook URL set to the ngrok URL + `/webhook/imessage` | 3.2 onward | Blocked until the app runs; I'll supply the URL |
| Sendblue billing active | any real send | Please confirm |
| Static ngrok domain (optional) | stable webhook URL | Optional, recommended |

## Known risks carried from DESIGN.md

1. OpenTable's protection is confirmed hostile to our infrastructure. Item 1.1 is a genuine go/no-go: if neither cloud browser provider gets through, the remaining options are running lookups from a residential IP or dropping to opening-hours data, both of which change the product.
2. The confirm-URL slot token (2.1) may be short-lived, which would make links expire; if so we surface the slot details in the text so the user can still book manually.
3. Sendblue rate limits / ToS — keep test-send volume low during E2E runs.
