# Sports Probability Platform · Technical Co-founder & Full-Stack Developer

## TL;DR
Co-built and operate a subscription platform that computes live scoreline probabilities and trading indicators for football matches, powered by three independent probability engines and an in-product AI assistant with internal tool calling. **Seven-figure lifetime revenue (BRL)** with **thousands of paying subscribers** across roughly 3 years of continuous operation. I lead architecture, the real-time data pipeline, the AI assistant, and the multi-gateway billing stack.

## Context
Sports trading (buying and selling positions on live matches, similar to financial trading) is a growing vertical in Brazil. A content creator with an engaged audience in this niche saw consistent demand from his followers for accessible, reliable scoreline probability estimates, and proposed building a product around it.

The product was a subscription from the start, and we deliberately launched narrow: pre-match scoreline probabilities. The scope then expanded over time into live in-match calculations, trading indicators, an AI assistant, and Telegram automation. The architecture described below is what the platform grew into. I joined as the technical lead under a project contract, and after about ten months I stepped into a 10% partnership and went full-time. The product launched in **August 2023** and remains in active operation.

## Constraints
- A lean technical team, with limited hands for both building and operating the product.
- Bootstrap budget: no external funding, no runway beyond the product's own revenue.
- Live data had to stay fresh within seconds of real-world events, on a sub-minute cadence (about every 10 seconds) that scheduled crons alone couldn't provide.
- Audience majority Brazilian with an international segment, so PIX, local installments, and per-region fee structure mattered for unit economics from day one.
- Live football data was a single-vendor dependency that couldn't easily be swapped.

## Decision and rationale
I went with **Next.js + Firebase** from the start. For the initial scope of pre-match scoreline probabilities, it was a clean fit that let us ship fast on a bootstrap budget, which for an early-stage product with a lean team mattered more than chasing a theoretically-optimal architecture.

The architecture: **Next.js on Vercel** for the frontend, **Firebase (Firestore + Auth)** for application data, and **Google Cloud Functions** for live data ingest and probability computation (~30 functions deployed today). Live football data comes from a paid sports-data API.

As the scope grew into live calculations and an AI assistant, the early choice kept paying off instead of forcing a rewrite:
- **Firestore real-time listeners** fit the live-update use case without us building WebSocket infrastructure, genuinely non-trivial work a lean team got for free.
- **Cloud Functions on crons and triggers** let the ingest layer scale independently of the web tier, so a signup-traffic spike doesn't affect live calculation latency for active subscribers.
- **Next.js ISR + locale-prefixed routing** handles marketing pages and authenticated dashboards in the same codebase, with internationalization built in.
- **GCP free tier + pay-as-you-go** matched our budget, so we didn't pay for idle capacity.

Conscious trade-off: vendor lock-in on Google Cloud and Firestore in exchange for shipping speed and near-zero infra ops overhead. After ~3 years it's been worth it, and we've never been blocked on infra.

## Implementation highlights

- **Event-driven, sub-minute live pipeline.** A Firestore trigger on the live-scores collection detects state transitions (half-time, full-time, and so on) and fans out in parallel: a backtesting recompute, a results-propagation job over product-facing lists, and notification triggers. We don't poll for "which matches just ended"; the match ending *is* the event. In-play data needed a roughly 10-second refresh, finer than the one-minute floor of scheduled crons, so rather than stand up a long-running worker (Cloud Run / Cloud Tasks), each 1-minute invocation runs an internal 10-second loop, multiplying one trigger into several refreshes on the same Cloud Functions footprint. Pre-match opportunity detection runs on 30-minute crons.

- **Three independent probability engines**, each tuned to a different decision moment rather than one generic predictor: a **pre-match engine** and a **live in-play engine** that translate model probabilities into fair odds (`100/p`) and Expected Value against the betting exchange to surface value opportunities automatically, plus a **dutching engine** for multi-outcome staking. The models themselves are the product's core IP, so I'm keeping their internals out of this writeup.

- **AI assistant as a function-calling agent over 15 internal tools, not vector embeddings.** It answers both pre-match and live in-play questions. One tool reads the current state of a match in progress and returns a fair-odds estimate for that exact moment (backed by the in-play engine above), while others pull match info, team stats, score predictions, EV vs market, and so on. Each tool is an internal API route resolving against our own data. I chose **structured RAG over a vector store** because the domain has deterministic lookups, and embeddings would add latency, cost, and a hallucination surface for queries that resolve cleanly through code. Multi-model routing across OpenAI families, per-model pricing tabled in code, streaming responses over SSE.

- **Atomic credit accounting for AI calls.** A transaction reserves the user's credit *before* the LLM call, generates a transaction id, and refunds the credit if the call fails before the response starts streaming. This closes the "I was charged for a request that errored mid-stream" complaint class that LLM products commonly leak on.

- **Defense-in-depth on the AI input/output surface.** An input filter screens for known prompt-injection and jailbreak patterns (DAN-style attacks, "ignore previous instructions", system-prompt extraction, API-key probes), and an output filter sanitizes responses against leaked secret patterns and environment-variable references. The assistant can't be easily steered into exfiltrating credentials or its own system prompt.

- **Three-gateway billing behind an internal subscription layer.** An international processor for recurring card subscriptions, plus two Brazilian providers covering PIX and local installments. Each provider has its own renewal semantics, and an internal layer absorbs most of those differences so product code can treat a subscription uniformly. Routing across providers also gives us redundancy and per-region fee optimization.

- **Telegram as a delivery surface, kept in sync with billing.** Part of the subscription is delivered through Telegram channels to make access easier for users, so channel membership has to track entitlement automatically. A daily cron cross-references Telegram membership against current subscription status across all products and revokes access from users whose entitlement lapsed, with multi-channel support and a dry-run mode. A separate cron sends timed pre-match notifications, with an "already-notified" flag per notification type so a user never receives duplicate pings even if it re-fires.

## Results
- **Lifetime revenue:** seven figures (BRL) over ~3 years of continuous operation.
- **Subscribers:** thousands of paying users.
- **Production footprint:** ~30 Cloud Functions, ~260 API routes, ~165 UI components.
- **Live calculation latency:** seconds from upstream data event to subscriber dashboard.

## What I'd do differently

- **Unify the entitlement model from day 1.** The payment providers were integrated at different times, and each left its own shape on how subscription access is represented internally. I'd define one entitlement contract up front and adapt every provider into it, rather than reconciling those differences later.

- **Retire old code paths sooner.** As the live pipeline evolved, newer versions of jobs shipped while older ones lingered, leaving duplication that serves history more than current traffic. A clear deprecation policy from the start would have kept the codebase leaner.

- **Invest in observability earlier.** When a live function misbehaved during a match, debugging meant digging through raw logs at the exact moment speed mattered most. Structured logging and alerting from the start would have made incident response far faster than reading logs by hand.

---

**Stack:** Next.js · TypeScript · Vercel · Firebase (Firestore + Auth) · Google Cloud Functions · Vercel AI SDK · OpenAI · multi-gateway payments (international cards + PIX + local installments) · Telegram Bot API · next-international / i18next
