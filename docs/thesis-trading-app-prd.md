# Product Spec: Thesis-Driven Trading Workbench

**Status:** Draft v1 — captures all product decisions from founder ideation interview (June 2026)

## 1. Vision

A desktop-web trading workbench for active retail traders where a user-authored
**investment thesis** acts as the guardrail for everything they trade. The
experience splits into a **strategy layer** (the thesis) and a **tactical
layer** (daily trading sessions + durable research sessions), with trades
executed through Alpaca. Long-term, the app becomes a financial foundation:
one place to manage savings, checking, retirement, 529, and investments.

## 2. Target user (v1)

Active retail traders — trade weekly/daily, comfortable with options chains
and technical analysis. Experience tiers for long-term investors and beginners
are explicitly deferred.

## 3. Core concepts

### 3.1 Strategy layer: the investment thesis

- A chat box where the user writes their overarching investment thesis in
  plain English.
- **Hybrid evaluation model:** when the thesis is saved/edited, an LLM
  compiles it into **structured rules** (e.g., max position size, sector
  allowlist/blocklist, "no entries during earnings week", max portfolio
  concentration). The natural-language thesis is retained for qualitative
  consistency checks.
- Rules are **fetched once at session start** (trading or research session)
  and applied deterministically during orchestration — the LLM is not
  re-prompted to judge every order from scratch.

### 3.2 Guardrail enforcement: soft block + override

- Every order is evaluated against the compiled rules and the qualitative
  thesis.
- Violations trigger a **friction screen** explaining the conflict with the
  thesis; the user can override with explicit confirmation.
- Overrides are logged and surfaced in the portfolio console (thesis-drift
  reporting). Keeping the user as the final decision-maker is also a
  deliberate regulatory posture (self-directed, not advisory).

### 3.3 Research sessions (durable)

NotebookLM-style, finance-native research surfaces. Persistent until the user
archives them. A session may cover a single ticker or a **theme** (e.g., "AI
capex cycle").

Panels / capabilities:
- Real-time ticker chart with drawing tools (trendlines, levels, zones)
- Options chain
- Current positions
- Real-time news and events feed (see §5), citable by the AI
- **Source ingestion:** PDFs (10-Ks, analyst notes, newsletters) and imported
  memory/knowledge from any source, in addition to the live news add-on
- Prior trading sessions can be **ingested as learning** (see §3.5)

**Floating prompt box:** captures chart annotations + natural language and
converts them **immediately into concrete, fully-specified orders** — the
output of the prompt box is a structured trade object that plugs directly
into the Alpaca API request schema (symbol, side, qty, type, limit/stop
prices, time-in-force, option legs). The user reviews/edits the concrete
order; it is then **staged** to the research session.

Staged trades **persist until filled, expired, or cancelled** — they live
with the research session, not the trading day.

### 3.4 Trading sessions (daily reset)

- Every trading day the user signs in starts a **fresh trading session**
  (clean AI context + behavioral reset — no revenge-trading on yesterday's
  loss).
- The user **links one or more research sessions**; their staged trades and
  context become available.
- Execution paths:
  1. **One-tap confirm:** when a staged trade's entry condition triggers, the
     user is alerted and taps to execute the pre-staged order.
  2. **Plain-English orders:** the user types a trade in natural language;
     the orchestrator converts it to a concrete order, runs guardrails, and
     presents it for confirmation.
- No fully automatic execution in v1 (avoids investment-adviser registration
  questions; revisit with counsel later).

### 3.5 Session history & auditability

Previous trading sessions are **visible and archived** — never deleted. Uses:
- **Auditing:** full record of orders, guardrail evaluations, overrides, and
  AI reasoning.
- **Learning:** any past trading session can be ingested into a new or
  existing research session as a knowledge source ("what did I do last time
  NVDA gapped down?").

### 3.6 Portfolio console

Holistic portfolio management and summary: positions, P&L, exposure breakdown,
thesis-alignment/drift view (including override history), staged-trade
pipeline across research sessions.

## 4. Scope decisions (v1)

| Decision | Choice |
|---|---|
| Asset classes | US equities + options (Alpaca). Crypto/futures deferred. |
| Paper vs live | Paper trading first; live behind a waitlist. Alpaca paper API is identical to live, so one build. |
| Platform | Desktop web. Mobile confirm-companion app is a fast-follow candidate (entry triggers fire when users are away from desk). |
| Brokerage model | **Users bring their own Alpaca account** (OAuth). App is a pure software layer — lightest compliance. Alpaca Broker API onboarding deferred. |
| Household finance (savings/checking/retirement/529) | **Deferred past v1.** Roadmap: read-only aggregation (Plaid/MX) → money movement → possibly BaaS custody. |
| Monetization | Subscription. Alpaca's own charges pass through to the user since they bring their own account. Premium data tiers (e.g., low-latency news) are a future upsell. |

## 5. Data & news integrations

**v1 stack (recommended):**
- **Alpaca News API** — Benzinga-sourced, WebSocket stream + historical REST,
  ticker-keyed, free with the Alpaca account. Default news feed.
- **SEC EDGAR** (free) — real-time filings (8-K, S-1, 13-F) as first-class
  events; high signal for a thesis-driven audience.
- **Finnhub** — earnings calendar, company news, sentiment scores.

**Upgrades / alternatives:**
- **Benzinga Pro API** — lower latency and fuller content than the Alpaca
  passthrough; the natural paid-tier unlock.
- **Polygon.io** — news with clean ticker tagging; pairs well if Polygon is
  also used for market data.
- **NewsFilter.io** — SSE streaming incl. press wires (GlobeNewswire,
  AccessWire).
- **Trading Economics** — macro event calendar (CPI, FOMC).
- **Financial Modeling Prep** — earnings dates and transcripts.
- **Marketaux** — budget multi-source aggregator with entity/sentiment tags.
- **StockTwits** — optional retail-sentiment layer.

**Design requirement:** news must be **citable**, not just rendered — the AI
grounds statements against specific articles/filings inside research
sessions. Prefer providers whose licensing permits storing/excerpting, not
display-only (EDGAR and Alpaca are safe; verify Benzinga redistribution
terms).

## 6. Architecture notes

- **Alpaca MCP server** powers the agentic execution path: the trading-session
  orchestrator translates plain-English intents and one-tap confirms into MCP
  tool calls against the Alpaca API.
- **Real-time panels bypass MCP:** positions, quotes, and chart data hit
  Alpaca WebSocket streams directly; MCP is for agent-initiated actions, not
  streaming UI state.
- **Order pipeline:** floating prompt box → structured trade object (Alpaca
  request schema) → guardrail engine (compiled thesis rules, deterministic)
  → staging (research session) → trigger monitor → one-tap confirm →
  Alpaca MCP execution → audit log.
- **Session model:** research sessions are durable documents with sources
  (news, PDFs, imported knowledge, past trading sessions) and staged trades.
  Trading sessions are daily, ephemeral-context views that link research
  sessions; their transcripts are archived immutably for audit/learning.
- **Thesis compiler:** LLM pass at thesis write/edit time producing a typed
  rule set; rules versioned so past trades can be audited against the rules
  in force at execution time.

## 7. Roadmap sketch

- **v1:** Desktop web, paper trading, BYO Alpaca account, equities + options,
  thesis guardrails, research/trading sessions, Alpaca News + EDGAR +
  Finnhub, subscription billing.
- **v1.x:** Live trading off the waitlist; mobile companion for alerts +
  one-tap confirms; Benzinga Pro premium tier.
- **v2:** Read-only household aggregation (Plaid/MX) feeding the portfolio
  console; crypto (requires rethinking daily-reset semantics for 24/7
  markets); theme-level research templates.
- **v3:** Money movement, retirement/529 accounts, possibly BaaS custody;
  revisit auto-execution within guardrails with legal counsel.

## 8. Open questions

- Trigger-monitor infrastructure: server-side condition evaluation against
  Alpaca streams vs. native Alpaca stop/limit orders where expressible —
  likely a hybrid (use native order types when the condition maps cleanly;
  server-side monitor for drawn-level/multi-condition setups).
- Options UX depth in v1: single-leg only, or multi-leg strategies (verticals,
  iron condors) in the prompt-box grammar?
- How thesis rule conflicts are resolved when multiple research sessions are
  linked to one trading session.
- Subscription price point and what (if anything) is free-tier.
