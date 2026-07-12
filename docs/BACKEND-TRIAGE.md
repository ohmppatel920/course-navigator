# Backend triage — Cairn hosted-LLM pilot

*Senior-backend adversarial review (5 lenses: rate-limiting/abuse, cost, security, reliability,
data-rights) of the proposed Vercel serverless `/api/chat` design, before building.*

## Verdict

**Not safe to build as drawn.** The `$10` ceiling and the 20-query cap are both *unenforceable*
as designed (state would live in serverless process memory), the paid endpoint is effectively
public (the access code was "optional"), and there is no provider-side hard cap — so any single
bug, burst, or scripted loop becomes a **real bill** on your card and/or a route to reconstruct
the private Critical Review dataset.

**The single biggest risk is bill-drain, and it's the *intersection* of three P0s** — fix all
three together: (1) optional auth, (2) in-memory counters, (3) no Anthropic Console cap.

## P0 — must fix before any deploy

| # | Risk | Fix |
|---|------|-----|
| 1 | **Optional access code → public paid Claude relay.** The `/api/chat` URL ships in the client JS; with the code optional, anyone can `curl` it in a loop to burn tokens *and* query it as an oracle to harvest CR rows. Origin/Referer checks are forgeable. | **Mandatory** per-student code, validated as the **first line** of the handler (401) before any Redis read or Claude call. 30 unique server-generated codes; store SHA-256 hashes in Redis (revocable). Optional: exchange for a short-lived HMAC token + Cloudflare Turnstile. |
| 2 | **No Anthropic Console spend cap** — the only *bug-proof* backstop is missing. Every app-side guard is self-written code; one bug removes the only thing between you and a big invoice. | Dedicated Anthropic **Workspace**, scoped key, **hard ~$15/mo spend limit**, alerts at 50/80/100%, prepaid credits, **auto-reload OFF**. This — not app code — guarantees "no surprise bills." |
| 3 | **In-memory counters enforce nothing on serverless.** Vercel fans requests across ephemeral instances; cold starts reset memory to 0. Concurrent requests each count to 20 / each read "budget ok" → caps never fire. | Move **all** counters to one external atomic store: **Upstash Redis** (free tier, single region matching the function). Rate limit via `@upstash/ratelimit` (atomic Lua sliding window); budget via an `INCRBYFLOAT` key. |
| 4 | **Budget check-then-call is a TOCTOU race**, and per-user caps can't bound *total* spend (1,000 fake identities × 20 = 20,000 "in-limit" calls). | Keep **both** a per-code rate limit **and** a global monthly spend key. **Reserve-before-spend:** atomically `INCRBYFLOAT` a worst-case estimate; proceed only if running total ≤ ~$8 app ceiling; reconcile to actual `response.usage` after. |
| 5 | **Unbounded `max_tokens` + per-query (not per-token) accounting + uncapped input.** One default-max response can be dollars; a pasted transcript inflates every call. | Hard `max_tokens` (~1024); reject bodies > few KB (413); clamp message ~1000 chars; **meter by real tokens**, not per-query. Haiku 4.5 → ~$0.008 worst-case/request. |
| 6 | **Fail-open on a store/API error = unbounded spend.** If Upstash is down or its free quota is flooded and you "call Claude anyway," you lose counting *and* spend freely. | **Fail CLOSED on spending:** any store error / failed reservation / API 5xx → serve the deterministic keyless ranker ($0). Never reach `messages.create()` without a successful reservation. |
| 7 | **API-key leak** via client bundle / `NEXT_PUBLIC_` / error passthrough / logs / committed `.env`. | Key only from `process.env` inside the function; grep the built bundle to confirm it's absent; generic 500s; `.env*` gitignored; rotate if ever committed. |
| 8–9 | **Real dataset leak** — committed to the public repo (tree + history are cloneable), or echoed via the response / client bundle / a debug route. | Dataset in a **separate private repo** backing Vercel, imported only under `/api`. Response built from a **hardcoded allow-list** (≤5 recommended courses, whitelisted fields only) — never `JSON.stringify` the candidate set. Build-time grep guard. *(Note: our current public repo already ships synthetic-only + gitignores real data — this is partly handled.)* |

## P1 — fix before the pilot enrolls students
- **Client-minted session id / raw `X-Forwarded-For` keying** is bypassable → key limits on the server-issued code; use platform-verified `request.ip` (/64 for IPv6) only as a secondary signal.
- **Non-atomic increment / eventually-consistent store** → atomic single-round-trip ops; strongly-consistent single-region Redis (or Durable Objects on Cloudflare, never Workers KV).
- **No idempotency** (double-clicks, timeout retries, SDK auto-retries double-spend) → client request-id + `SET NX`; disable send in-flight; `max_retries=1`; reconcile from `usage`.
- **Prompt injection** exfiltrates the ~15 candidate rows / system prompt → minimal fields in context, delimited user turn, post-check rejects system-prompt echoes / bulk dumps; auth + caps stop the multi-query sweep.
- **Logging student queries = FERPA-adjacent data** in third-party logs → log **telemetry only** (no query text/identity together); Brown IRB determination; PRIVACY.md + consent.
- **Sending CR data to Anthropic may exceed CR's permitted use** → send only **numeric facts + your own summaries, never verbatim CR prose**; frame Anthropic as a processor (no training by default); get written CR permission for LLM/third-party processing.
- **No Anthropic error handling / 10s timeout hard-fail at peak; streaming defeats the guardrail** → bounded backoff retries then deterministic fallback; `maxDuration=60`; **non-streaming** so you buffer → run guardrail → send.
- **No observability** → one telemetry log line/request; a code-gated `/api/stats` showing spend-vs-ceiling.
- **Public demo wired to the pilot backend** would expose it to the whole internet → keep the public demo 100% self-contained (synthetic + keyless); pilot on a separate gated URL.

## P2 — hardening
- Unique revocable codes (not one shared secret) + a per-code anomaly auto-disable.
- Persistent CR attribution in the UI ("ratings from The Critical Review; recommendations are AI-generated, not endorsed by CR"); label public-demo numbers SYNTHETIC.

## Recommended architecture (minimal + safe, solo-student / free-tier)

**Stack:** Vercel serverless (single pinned region) + Upstash Redis (free, same region) + a hard
Anthropic **Console** spend cap. Model **Haiku 4.5**, `max_tokens=1024`, **non-streaming**. Two
routes: `POST /api/chat` and a code-gated `GET /api/stats`. Real dataset in a **separate private
repo**, imported only under `/api`.

**Layered defense (each request):**
1. **Edge:** Cloudflare free plan (one rate-limit rule + Bot Fight Mode) or Vercel Firewall drops floods before the function runs.
2. **Auth (first statement):** mandatory access code → 401 if missing/invalid. No code → no spend.
3. **Rate limit:** `@upstash/ratelimit` sliding window, **keyed on the code** (atomic Lua).
4. **Budget reserve-before-spend:** global `spend:YYYY-MM` key (no TTL). `INCRBYFLOAT` worst-case; if > ~$8 → refund + serve deterministic ranker; else call Claude; reconcile to actual after.
5. **Fail closed:** any error → deterministic ranker ($0), never an uncounted Claude call.
6. **Backstop:** Anthropic Console hard cap ~$15 — the only guarantee that's not your own code.

**Data-leak controls:** allow-list response (≤5 courses, whitelisted fields); buffer → guardrail →
send; `productionBrowserSourceMaps:false`; contract test (≤N courses, whitelisted keys only);
build-time bundle grep for a known course code.

**Governance:** Brown IRB determination in writing; PRIVACY.md + one-screen consent (Anthropic +
Vercel as subprocessors, retention, deletion at pilot end); written CR permission + attribution.

## Decisions only you can make
1. **Access-gate strength:** 30 static unique codes (simplest) vs. code→short-lived token vs. + Turnstile.
2. **Budget numbers:** app ceiling (~$8), Console hard cap (~$15), per-code query cap (~20).
3. **Model:** confirm Haiku 4.5 + `max_tokens` ~1024.
4. **Edge layer:** Cloudflare free vs. Vercel Firewall (one is needed to drop floods pre-function).
5. **Log query *content* for eval?** If yes → opt-in consent + identifier stripping + retention window. If no → telemetry only (recommended).
6. **Critical Review permission:** scope (covers sending facts to an LLM?), no verbatim prose, exact attribution wording.
7. **IRB:** run Brown's human-subjects determination before enrolling students.
8. **Regions:** Vercel + Upstash in the same region; consider Anthropic Zero-Data-Retention.
