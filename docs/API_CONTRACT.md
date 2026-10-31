# Portfolio Research API — Contract v1

> **Audience:** frontend teammates and their coding agents. This file is the source of truth.
> Backend implements exactly this. Changes are **additive only** (new optional fields / new event types);
> nothing here gets renamed or removed without a heads-up in the team chat.

## What the product does

User fills an intake (amount, horizon, risk) → backend runs a team of Gemini agents →
streams their progress live → returns a **verified two-level portfolio**:

- **Inner ring:** asset classes — `STOCKS` / `CRYPTO` / `GOLD` with %.
- **Outer ring:** holdings inside each class (e.g. `NVDA 14%`, `BTC-USD 15%`, `GLD 25%`).
- Every holding has a rationale + evidence links from the agents.
- A backtest compares the portfolio to benchmarks; a Verifier may **reject** round 1 and
  the Orchestrator rebalances in round 2. **The UI should make that rejection visible** — it's the demo moment.

## Agents (fixed set — render one card/lane per agent)

| `AgentId`      | Model           | Job |
|----------------|-----------------|-----|
| `DISCOVERY`    | Gemini 3.8 Flash (`gemini-3.8-flash`) | Finds candidate stocks + crypto (Google Search grounding) |
| `FINANCIALS`   | Gemini 3.8 Flash (`gemini-3.8-flash`) | Scores each stock 1–10 from real statements (code execution) |
| `LANDSCAPE`    | Gemini 3.8 Flash (`gemini-3.8-flash`) | Competitors / market position (Search grounding) |
| `NEWS`         | Gemini 3.8 Flash (`gemini-3.8-flash`) | Recent news per holding (Search grounding) |
| `SOCIAL`       | Gemini 3.8 Flash (`gemini-3.8-flash`) | Reddit/X sentiment (Search grounding) |
| `ORCHESTRATOR` | Gemini 4 Argon (`gemini-4-argon`) | Builds the allocation from all findings |
| `VERIFIER`     | Gemini 4 Argon (`gemini-4-argon`) | Runs backtest in Gemini sandbox, cross-checks numbers, approves/rejects |

Order: `DISCOVERY` first → `FINANCIALS`, `LANDSCAPE`, `NEWS`, `SOCIAL` **in parallel** →
`ORCHESTRATOR` → `VERIFIER` → (if rejected) `ORCHESTRATOR` → `VERIFIER` again. Max 2 rounds.

---

## Endpoints

Base path: `/api`. In Vite dev, add `"/api"` to `PROXY_PATHS` in `vite.config.ts`.
Production base URL comes from `VITE_API_BASE_URL` (Cloud Run URL, shared in team chat).

### `POST /api/runs` — start a run

Request body: `IntakeInput`. Response `202`: `Run` (status `QUEUED`, `report: null`).

```http
POST /api/runs
Content-Type: application/json

{ "amountUsd": 10000, "horizonYears": 2, "riskTolerance": "MODERATE", "theme": "AI infrastructure" }
```

### `GET /api/runs/{runId}/events` — live stream (SSE)

`text/event-stream`. Each message has:
- `event:` = the event `type` (see `RunEvent` below) — use with `useSSE` named handlers.
- `id:` = `"{runId}:{seq}"` — supports `Last-Event-ID` reconnect; backend replays missed events.
- `data:` = JSON of the event.

Stream closes after `run.completed` or `run.failed`. Connecting after the run finished replays everything, then closes.

### `GET /api/runs/{runId}` — snapshot

Response `200`: `Run`. `report` is non-null once `status === "COMPLETED"`.
Use this for page refresh / deep links; use the stream for the live view.

### `GET /api/health`

`200 { "status": "ok" }`

### Errors (every endpoint, same shape)

```ts
interface APIError {
  error: { code: ErrorCode; message: string; details?: unknown };
}
type ErrorCode = "VALIDATION_ERROR" | "RUN_NOT_FOUND" | "RATE_LIMITED" | "INTERNAL";
```

| Status | `code` | When |
|--------|--------|------|
| 422 | `VALIDATION_ERROR` | Bad intake. `details` = `{ field: message }` map — show inline on the form |
| 404 | `RUN_NOT_FOUND` | Unknown `runId` |
| 429 | `RATE_LIMITED` | A run is already in progress (we cap concurrency to protect the hackathon quota) |
| 500 | `INTERNAL` | Anything else. `message` is safe to show |

---

## Types (copy into `src/types/portfolio.ts`)

```ts
// ---------- Input ----------
export type RiskTolerance = "CONSERVATIVE" | "MODERATE" | "AGGRESSIVE";

export interface IntakeInput {
  amountUsd: number;        // 100 – 10_000_000
  horizonYears: number;     // integer 1 – 10
  riskTolerance: RiskTolerance;
  theme?: string;           // optional focus, ≤ 200 chars, e.g. "AI infrastructure"
}

// ---------- Run ----------
export type RunStatus = "QUEUED" | "RUNNING" | "COMPLETED" | "FAILED";

export interface Run {
  runId: string;
  status: RunStatus;
  intake: IntakeInput;
  createdAt: string;        // ISO 8601
  report: Report | null;
  error: APIError["error"] | null;
}

// ---------- Agents ----------
export type AgentId =
  | "DISCOVERY" | "FINANCIALS" | "LANDSCAPE" | "NEWS" | "SOCIAL"
  | "ORCHESTRATOR" | "VERIFIER";

export interface Source { title: string; url: string }

export interface Finding {
  id: string;               // "f_12" — referenced by Holding.findingIds
  agent: AgentId;
  symbol: string | null;    // null = market-wide finding
  claim: string;            // one sentence, human readable
  sentiment: "POSITIVE" | "NEGATIVE" | "NEUTRAL";
  sources: Source[];        // may be empty for FINANCIALS (data from statements)
}

// ---------- Allocation (the sunburst) ----------
export type AssetClass = "STOCKS" | "CRYPTO" | "GOLD";

export interface Holding {
  symbol: string;           // "NVDA", "BTC-USD", "GLD"
  name: string;             // "NVIDIA Corp."
  weightPct: number;        // % of WHOLE portfolio, 1 decimal
  amountUsd: number;        // weightPct * intake.amountUsd / 100
  rationale: string;        // 1–3 sentences
  findingIds: string[];     // evidence → look up in Report.findings
  financialScore: number | null;  // 1–10, stocks only
  sentimentScore: number | null;  // -1.0 … 1.0 (SOCIAL + NEWS blend)
}

export interface AssetClassSlice {
  assetClass: AssetClass;
  weightPct: number;        // = sum of holdings[].weightPct
  rationale: string;
  holdings: Holding[];
}

export interface Allocation {
  slices: AssetClassSlice[]; // all slice weightPct sum to 100
}

// ---------- Backtest ----------
export interface Metrics {
  totalReturnPct: number;
  cagrPct: number;
  volatilityPct: number;    // annualized
  sharpe: number | null;    // null if undefined
  maxDrawdownPct: number;   // negative number, e.g. -28.4
  worstMonthPct: number;    // negative number
}

export type SeriesKey = "PORTFOLIO" | "SPY" | "SIXTY_FORTY" | "EQUAL_WEIGHT";

export interface Backtest {
  startDate: string;        // "2021-10-29"
  endDate: string;
  metrics: Record<SeriesKey, Metrics>;
  equityCurve: {
    dates: string[];        // weekly points, ~260 for 5y
    series: Record<SeriesKey, number[]>; // growth of $1, same length as dates
  };
  crossCheck: {             // sandbox result vs. our independent recompute
    sandboxTotalReturnPct: number;
    independentTotalReturnPct: number;
    isMatch: boolean;       // |diff| < 0.5pp
  };
}

// ---------- Verification ----------
export interface Check { name: string; isPassed: boolean; detail: string }

export interface Verification {
  isApproved: boolean;
  checks: Check[];          // e.g. "Max drawdown within MODERATE limit (-35%)"
  changesRequested: string[]; // empty when approved
}

export interface Round {
  round: 1 | 2;
  allocation: Allocation;
  backtest: Backtest;
  verification: Verification;
}

// ---------- Report (final output) ----------
export interface Report {
  runId: string;
  generatedAt: string;
  intake: IntakeInput;
  summary: string;          // 3–5 sentence executive summary, markdown allowed
  finalAllocation: Allocation;   // = last round's allocation
  rounds: Round[];          // 1 or 2 — show round 1 → 2 diff when length is 2
  findings: Finding[];
  disclaimer: string;       // always render
}

// ---------- SSE events ----------
interface Base { runId: string; seq: number; ts: string }

export type RunEvent =
  | (Base & { type: "run.started"; agents: AgentId[] })
  | (Base & { type: "agent.started"; agent: AgentId; model: string; task: string })
  | (Base & { type: "agent.progress"; agent: AgentId; message: string })   // "Searching: NVDA earnings"
  | (Base & { type: "agent.finding"; finding: Finding })
  | (Base & { type: "agent.completed"; agent: AgentId; summary: string; durationMs: number })
  | (Base & { type: "agent.failed"; agent: AgentId; message: string })     // run continues without it
  | (Base & { type: "allocation.proposed"; round: 1 | 2; allocation: Allocation })
  | (Base & { type: "backtest.completed"; round: 1 | 2; backtest: Backtest })
  | (Base & { type: "verification.completed"; round: 1 | 2; verification: Verification })
  | (Base & { type: "run.completed"; report: Report })
  | (Base & { type: "run.failed"; error: APIError["error"] });
```

### Example event sequence (abridged)

```
event: run.started          data: {"agents":["DISCOVERY","FINANCIALS",...]}
event: agent.started        data: {"agent":"DISCOVERY","model":"gemini-3.8-flash","task":"Find candidates for AI infrastructure"}
event: agent.progress       data: {"agent":"DISCOVERY","message":"Searching: top AI infrastructure stocks 2026"}
event: agent.finding        data: {"finding":{"id":"f_1","agent":"DISCOVERY","symbol":"NVDA",...}}
event: agent.completed      data: {"agent":"DISCOVERY","summary":"Found 5 stocks, 3 coins","durationMs":8200}
event: agent.started        data: {"agent":"FINANCIALS",...}     ← these four interleave
event: agent.started        data: {"agent":"NEWS",...}
...
event: allocation.proposed  data: {"round":1,"allocation":{...}}
event: backtest.completed   data: {"round":1,"backtest":{...}}
event: verification.completed data: {"round":1,"verification":{"isApproved":false,"changesRequested":["Cut crypto below 30%: max drawdown -41% exceeds MODERATE limit -35%"]}}
event: allocation.proposed  data: {"round":2,...}
event: backtest.completed   data: {"round":2,...}
event: verification.completed data: {"round":2,"verification":{"isApproved":true,...}}
event: run.completed        data: {"report":{...}}
```

A full example `GET /api/runs/{runId}` response (a completed `Run` with a 2-round `Report`) lives in
[`fixtures/report.example.json`](fixtures/report.example.json). Equity curves are synthetic, but the shape is real. Build the
report page against it before the backend is live.

---

## UI notes for frontend agents

- **Sunburst:** ECharts `type: "sunburst"` is already a dependency. Inner = `slices`, children = `holdings`, `value` = `weightPct`.
- **Agent lanes:** one card per `AgentId`, states `idle → running → done | failed` driven by `agent.*` events; show latest `agent.progress.message`; findings append live.
- **Round diff:** when `rounds.length === 2`, show round-1 sunburst greyed with the Verifier's `changesRequested`, then round 2. This is the money shot.
- **Evidence:** clicking a holding shows its `rationale` + `findingIds` → findings with source links.
- **Backtest:** line chart of `equityCurve` (4 series) + metrics table; show `crossCheck.isMatch` as a "✓ Verified" badge.
- Reuse `src/hooks/useSSE.ts` for the stream.
- All numbers arrive pre-rounded (1 decimal for %, 2 for sharpe). Don't recompute weights client-side.
