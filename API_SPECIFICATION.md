> ⚠️ **SUPERSEDED.** The backend implements [`docs/API_CONTRACT.md`](docs/API_CONTRACT.md) instead
> (a multi-agent portfolio builder with SSE). Don't build against `/py/parse` or `/py/run`. This file is kept for history only.

# VibeTrader API Specification (MVP Finalized)

> **Version**: `2.0.0-mvp` (Aligned with `frontend/frontend_spec.md` & `frontend/wireframes.html`)  
> **Tagline**: *"The backtester that tells you the truth"*  
> **Target Audience**: Frontend Engineers, Backend & Quant Engineers, AI/LLM Pipeline Developers.

---

## 1. Architectural Overview & Conventions

### 1.1 Architecture & Proxy Routing
- **Frontend Stack**: Next.js (App Router) + TypeScript + Tailwind + shadcn/ui + Recharts.
- **Backend Stack**: Python FastAPI (`http://localhost:8000` locally or via a tunnel URL in `BACKEND_URL`).
- **Next.js Proxy Rewrite**: The frontend never calls Gemini or any AI service directly. All API calls go through Next.js rewrites:
  ```ts
  // next.config.ts
  async rewrites() {
    return [
      {
        source: "/py/:path*",
        destination: `${process.env.BACKEND_URL ?? "http://localhost:8000"}/:path*`,
      },
    ];
  }
  ```
  - Frontend calls: `POST /py/parse` and `POST /py/run`
  - FastAPI routes: `POST /parse` and `POST /run`

### 1.2 Protocol, Timeouts & Error Handling
- **Data Exchange**: Standard UTF-8 `application/json`. Payloads are **flat JSON objects** matching the TypeScript interfaces in Section 2 (no `{ "success": true, "data": ... }` wrapper).
- **Timeouts**:
  - `POST /py/parse` typically completes in `1–5 s` (mock delay: `800 ms`).
  - `POST /py/run` executes code generation, sandbox execution, look-ahead leak checks/repairs, backtesting, and bootstrap CI calculation, which can take **`20–60 s`** (mock delay: `6 s`). The frontend uses a **`120 s` timeout** (`AbortController`).
- **No Server-Sent Events (SSE)**: The backend does not stream intermediate step events. While `/py/run` is in flight, the frontend `ProgressStepper` advances through the 5 steps (*Generate code → Sandbox run → Look-ahead test → Backtest → Audit*) on a client-side timer (~4 s per step, holding on the final step until `/py/run` resolves).
- **No Authentication**: MVP is a single-page live demo flow with no user auth, session headers, or database persistence.
- **Error Format**: Any non-2xx HTTP response returns a flat JSON object matching `ApiError`:
  ```json
  {
    "error": "Human-readable error message explaining what failed."
  }
  ```

---

## 2. Shared Data Models & Contract (`lib/types.ts`)

> **IMPORTANT**: This is the strict API contract between Frontend and Backend. **Do not rename any fields.** Fields marked *ignored* remain in the payload so the backend contract stays stable even though the MVP UI does not render them.

```typescript
export type Asset = "BTC" | "ETH" | "SPY";   // UI offers "BTC" and "SPY" only
export type Timeframe = "1d" | "1h";         // UI always sends "1d"
export type Verdict = "edge" | "inconclusive" | "no_edge";
export type Support = "supported" | "approximated" | "unsupported";

export interface Rules {
  asset: Asset;
  timeframe: Timeframe;
  entry: string[];                 // Plain-English rule lines, e.g. ["z-score(close, 20) < −2", "position = 100% long"]
  exit: string[];                  // Plain-English rule lines, e.g. ["z-score(close, 20) > 0", "or stop-loss −5%"]
  params: Record<string, string | number>; // e.g. { "lookback": "20 days", "threshold": -2.0, "fees": "10 bps / trade" }
}

export interface ParseRequest {
  prompt: string;
  asset: Asset;
  timeframe: Timeframe;
}

export interface ParseResponse {
  rules: Rules;
  support: Support;
  note: string | null;             // Shown in SupportBanner when approximated or unsupported
  question: string | null;         // Ignored by MVP UI (kept for contract compatibility)
  question_options: string[];      // Ignored by MVP UI (kept for contract compatibility)
}

export interface Metrics {
  sharpe: number;                  // e.g. 2.10 (rendered to 2 decimal places)
  cagr: number;                    // Decimal ratio, e.g. 0.45 for 45%
  max_dd: number;                  // Decimal ratio, e.g. -0.12 for -12%
  trades: number;                  // Integer trade count, e.g. 40
}

export interface EquityPoint {
  date: string;                    // ISO date string "YYYY-MM-DD", e.g. "2021-01-01"
  strategy: number;                // Normalized equity starting at 1.0
  buyhold: number;                 // Normalized benchmark equity starting at 1.0
}

export interface Audit {
  lookahead: {
    passed: boolean;               // true if no leak or leak was successfully fixed
    fixed: boolean;                // true if initial code had a leak that was auto-repaired
    attempts: number;              // Number of repair attempts (e.g. 2)
    detail: string | null;         // Explanation, e.g. "Original code used the full-history mean — future data."
  };
  sharpe_ci: [number, number];     // [lower_bound, upper_bound] 95% bootstrap CI of out-of-sample Sharpe
  verdict: Verdict;                // "edge" | "inconclusive" | "no_edge"
  trust_score: number;             // 0–100; rendered by the P1 TrustScoreRing inside VerdictBadge
  checks: string[];                // Verification items, e.g. ["Fees 10 bps applied", "70/30 split"]
}

export interface RunRequest {
  rules: Rules;
  prompt: string;
}

export interface RunResult {
  strategy_id: string;             // Unique identifier, e.g. "strat_dip_01"
  version: number;                 // Strategy version (1 on initial run, increments on P1 Refine)
  name: string;                    // Short display title, e.g. "Buy the dip (z-score)"
  rules: Rules;
  code: string;                    // Python source code of `def signals(df) -> pd.Series:`
  equity: EquityPoint[];           // Daily equity curve points across full backtest period (e.g. 2021–2026)
  split_date: string;              // ISO date "YYYY-MM-DD" marking the 70/30 IS/OOS boundary
  metrics: {
    in_sample: Metrics;
    out_sample: Metrics;
  };
  audit: Audit;
  elapsed_s: number;               // Total execution time in seconds, e.g. 23
}

export interface ApiError {
  error: string;
}
```

---

## 3. Endpoints Specification

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         VibeTrader Single-Page MVP                       │
├────────────────────────────────────┬─────────────────────────────────────┤
│ Section ①: BUILD                   │ Section ②: RESULTS                  │
│  - PromptInput + 3 PresetChips     │  - ProgressStepper (5 steps, timer) │
│  - AssetSelect (BTC / SPY, 1d)     │  - EquityChart (IS vs OOS split)    │
│  - POST /py/parse                  │  - MetricsTable (IS vs OOS + ↓)     │
│  - SupportBanner (3 states)        │  - VerdictBadge (+ P1 TrustScore)   │
│  - RuleCards (Entry/Exit/Params)   │  - AuditCard + SharpeCIBar          │
│  - ConfirmButton → POST /py/run    │  - CodePanel + P1 RefineInput       │
└────────────────────────────────────┴─────────────────────────────────────┘
```

### 3.1 `POST /parse` (Frontend proxy: `POST /py/parse`)

Parses the user's plain-English strategy description into structured `Rules` (`entry`, `exit`, `params`), determines whether the strategy is `"supported"`, `"approximated"`, or `"unsupported"`, and provides an explanatory `note` when approximated or unsupported.

- **Frontend Components**: `PromptInput`, `AssetSelect`, `SupportBanner`, `RuleCards`, `ConfirmButton`
- **Request Body (`ParseRequest`)**:
```json
{
  "prompt": "Buy when price is unusually low compared to normal, sell when it recovers",
  "asset": "BTC",
  "timeframe": "1d"
}
```

#### Response Example A: `"approximated"` (Wireframe ① & `lib/mocks/parse.json`)
Returned when the prompt contains subjective or ambiguous terms (like *"unusually low"*) that the AI approximates using a leak-free quantitative rule.
```json
{
  "rules": {
    "asset": "BTC",
    "timeframe": "1d",
    "entry": [
      "z-score(close, 20) < −2",
      "position = 100% long"
    ],
    "exit": [
      "z-score(close, 20) > 0",
      "or stop-loss −5%"
    ],
    "params": {
      "lookback": "20 days",
      "threshold": -2.0,
      "fees": "10 bps / trade"
    }
  },
  "support": "approximated",
  "note": "“Unusually low” is ambiguous — we'll use a 20-day z-score below −2. Not the full-history mean (that would leak the future).",
  "question": null,
  "question_options": []
}
```

#### Response Example B: `"supported"`
Returned when the prompt maps cleanly to supported technical/price indicators on daily OHLCV bars.
```json
{
  "rules": {
    "asset": "BTC",
    "timeframe": "1d",
    "entry": [
      "close > SMA(close, 200)",
      "position = 100% long"
    ],
    "exit": [
      "close <= SMA(close, 200)",
      "position = 100% cash"
    ],
    "params": {
      "sma_window": 200,
      "fees": "10 bps / trade"
    }
  },
  "support": "supported",
  "note": null,
  "question": null,
  "question_options": []
}
```

#### Response Example C: `"unsupported"`
Returned when the strategy requires external data unavailable in daily OHLCV (e.g., live orderbook L2, options flow, social media tweets) or cannot be backtested. When `support === "unsupported"`, the frontend disables `ConfirmButton`.
```json
{
  "rules": {
    "asset": "BTC",
    "timeframe": "1d",
    "entry": [],
    "exit": [],
    "params": {}
  },
  "support": "unsupported",
  "note": "Real-time social sentiment / X (Twitter) stream data is not available in the OHLCV backtest engine. Try expressing the signal using price or volume.",
  "question": null,
  "question_options": []
}
```

---

### 3.2 `POST /run` (Frontend proxy: `POST /py/run`)

Triggered when the user clicks **`✓ Confirm & Run backtest`** (or submits a P1 refinement). Executes the end-to-end verification and backtesting pipeline:
1. **Generate code**: Synthesizes Python `def signals(df) -> pd.Series`.
2. **Sandbox run**: Executes the signal function in an isolated sandbox.
3. **Look-ahead test**: Tests whether signals at time $t$ change when future rows ($> t$) are perturbed/truncated; if a leak is detected, automatically prompts the LLM to fix it (up to $N$ attempts).
4. **Backtest**: Runs a deterministic 70/30 chronological train/test split with transaction fees (10 bps/trade).
5. **Audit**: Computes in-sample and out-of-sample metrics, 95% bootstrap confidence interval on OOS Sharpe, trust score (0–100), and final verdict (`"edge"` | `"inconclusive"` | `"no_edge"`).

- **Frontend Components**: `ProgressStepper`, `EquityChart`, `MetricsTable`, `VerdictBadge`, `TrustScoreRing` (P1), `AuditCard`, `SharpeCIBar`, `CodePanel`
- **Request Body (`RunRequest`)**:
```json
{
  "prompt": "Buy when price is unusually low compared to normal, sell when it recovers",
  "rules": {
    "asset": "BTC",
    "timeframe": "1d",
    "entry": [
      "z-score(close, 20) < −2",
      "position = 100% long"
    ],
    "exit": [
      "z-score(close, 20) > 0",
      "or stop-loss −5%"
    ],
    "params": {
      "lookback": "20 days",
      "threshold": -2.0,
      "fees": "10 bps / trade"
    }
  }
}
```

- **Response Body (`RunResult` — Wireframe ② & `lib/mocks/run.json`)**:
```json
{
  "strategy_id": "strat_dip_01",
  "version": 1,
  "name": "Buy the dip (z-score)",
  "rules": {
    "asset": "BTC",
    "timeframe": "1d",
    "entry": [
      "z-score(close, 20) < −2",
      "position = 100% long"
    ],
    "exit": [
      "z-score(close, 20) > 0",
      "or stop-loss −5%"
    ],
    "params": {
      "lookback": "20 days",
      "threshold": -2.0,
      "fees": "10 bps / trade"
    }
  },
  "code": "import pandas as pd\nimport numpy as np\n\ndef signals(df: pd.DataFrame) -> pd.Series:\n    \"\"\"Buy the dip (20-day z-score) with leak-free rolling window.\"\"\"\n    rolling_mean = df['close'].shift(1).rolling(20).mean()\n    rolling_std = df['close'].shift(1).rolling(20).std()\n    z_score = (df['close'] - rolling_mean) / rolling_std\n\n    pos = pd.Series(np.nan, index=df.index)\n    pos[z_score < -2.0] = 1.0\n    pos[z_score > 0.0] = 0.0\n    return pos.ffill().fillna(0.0)",
  "equity": [
    { "date": "2021-01-01", "strategy": 1.0, "buyhold": 1.0 },
    { "date": "2024-08-15", "strategy": 1.84, "buyhold": 1.62 },
    { "date": "2026-01-01", "strategy": 1.42, "buyhold": 2.15 }
  ],
  "split_date": "2024-08-15",
  "metrics": {
    "in_sample": {
      "sharpe": 2.10,
      "cagr": 0.45,
      "max_dd": -0.12,
      "trades": 40
    },
    "out_sample": {
      "sharpe": 0.12,
      "cagr": 0.02,
      "max_dd": -0.30,
      "trades": 15
    }
  },
  "audit": {
    "lookahead": {
      "passed": true,
      "fixed": true,
      "attempts": 2,
      "detail": "Original code used the full-history mean — future data."
    },
    "sharpe_ci": [-0.4, 0.6],
    "verdict": "no_edge",
    "trust_score": 32,
    "checks": [
      "Fees 10 bps applied",
      "70/30 split"
    ]
  },
  "elapsed_s": 23
}
```

---

### 3.3 Strategy Refinement Flow (P1 — Uses Existing `/parse` and `/run`)

There is **no `/py/refine` endpoint**. When the user submits a refinement in `RefineInput` (e.g. *"only trade when price is above the 200-day average"*):
1. The frontend appends the instruction to the prompt:
   ```text
   <original_prompt>
   Refinement: <user_refinement_instruction>
   ```
2. The frontend calls `POST /py/parse` with the combined `prompt`, `asset`, and `timeframe: "1d"`.
3. The frontend immediately calls `POST /py/run` with the newly returned `rules` and combined `prompt`.
4. The new `RunResult` **replaces** the current result in place (no version tabs) and displays `v{version}` next to the strategy name.

---

## 4. Presets, Mock Mode & Demo Safety Net (`?demo=1`)

### 4.1 Preset Strategies (`lib/presets.ts`)
The Build screen features 3 preset chips that populate `PromptInput`. Each corresponds to a pre-cached `RunResult` JSON file in `lib/mocks/demo/` for offline/stage fallback:

| Chip Label | Exact Prompt | Demo Cache File | Demo Story & Expected Outcome |
|---|---|---|---|
| `🎯 Buy the dip` | `Buy when price is unusually low compared to normal, sell when it recovers` | `lib/mocks/demo/dip.json` | `support: "approximated"` at parse; look-ahead leak caught & fixed (`attempts: 2`) → **`no_edge`** |
| `📅 RSI on Tuesdays` | `Buy when RSI is below 28, sell when it is above 71, only trade on Tuesdays` | `lib/mocks/demo/rsi.json` | Great in-sample metrics, collapses out-of-sample (classic overfitting) → **`no_edge`** |
| `📈 200-day trend` | `Buy when price is above its 200-day moving average, otherwise stay in cash` | `lib/mocks/demo/trend.json` | Modest, leak-free trend following → **`inconclusive`** or **`edge`** |

### 4.2 Mock & Demo Modes (`lib/api.ts`)
1. **Mock Mode (`NEXT_PUBLIC_USE_MOCK=true`)**:
   - `parse()` returns `lib/mocks/parse.json` after an `800 ms` simulated delay.
   - `run()` returns `lib/mocks/run.json` after a `6 s` simulated delay.
2. **Stage Demo Mode (`?demo=1` in URL)**:
   - If the URL contains `?demo=1` and the prompt matches one of the 3 presets above, `lib/api.ts` returns the corresponding cached `RunResult` from `lib/mocks/demo/{dip,rsi,trend}.json` without requiring the backend to be running.
   - The backend team can export real pre-computed `RunResult` JSON payloads directly into `lib/mocks/demo/{dip,rsi,trend}.json`—the schema must strictly match `RunResult`.

---

## 5. Frontend Rendering & Business Logic Reference

To ensure backend data aligns with UI presentation rules:

1. **`SupportBanner` States**:
   - `"supported"` → Green text: `✓ Fully supported` (`note` ignored/null).
   - `"approximated"` → Yellow banner (`#fff6d6` / `#b88a00`): `⚠ Approximated` + `note`.
   - `"unsupported"` → Red banner (`#fde8e8` / `#d62828`): `✕` + `note`; **disables `ConfirmButton`**.

2. **`MetricsTable` Degradation Highlighting**:
   - `sharpe`: formatted to 2 decimals (e.g. `2.10`).
   - `cagr` & `max_dd`: formatted as percentages (e.g. `0.45` → `45%`, `-0.12` → `−12%`).
   - Out-of-sample metric is highlighted in **red with `↓`** if:
     - `out_sample.sharpe` or `out_sample.cagr` drops by **> 50%** compared to `in_sample`, OR
     - `out_sample.max_dd` magnitude is **> 1.5× worse** than `in_sample.max_dd`.

3. **`VerdictBadge` Copy & Theme**:
   - `"edge"` (Green `#1a8f4c`, bg `#e5f6ec`): **✅ Edge** — *"Out-of-sample results are statistically distinguishable from luck."*
   - `"inconclusive"` (Yellow `#b88a00`, bg `#fff6d6`): **⚠️ Inconclusive** — *"Some signal, but not enough evidence yet."*
   - `"no_edge"` (Red `#d62828`, bg `#fde8e8`): **❌ No edge** — *"No evidence this beats buy & hold after fees."*

4. **`AuditCard` & `SharpeCIBar` Logic**:
   - **Look-ahead check (`audit.lookahead`)**:
     - `passed && !fixed` → Green: `✓ No look-ahead leak`
     - `fixed` → Yellow box: `⚠→✓ Look-ahead leak found & fixed ({attempts} attempts)` + `detail` in grey
     - `!passed` → Red: `✕ Look-ahead leak detected` + `detail`
   - **Sharpe 95% CI Bar (`audit.sharpe_ci` & `metrics.out_sample.sharpe`)**:
     - Plots `sharpe_ci[0]` to `sharpe_ci[1]` with a marker at `metrics.out_sample.sharpe` against a vertical dashed line at `0`.
     - If the interval crosses or touches `0` (`sharpe_ci[0] <= 0`), the bar is **red** with caption: *"Interval crosses 0 → can't distinguish from luck."*
     - If `sharpe_ci[0] > 0`, the bar is **green** with caption: *"Entire interval above 0."*

---

## 6. Explicitly Cut from MVP (Do Not Implement)

The following endpoints and features from earlier brainstorms have been **cut** to keep the 3-minute live demo focused and reliable:

| Cut Endpoint / Feature | Replacement / Reason |
|---|---|
| `GET /meta/presets` | Presets (`BTC`/`SPY`, `1d`, 3 chips) are static in `lib/presets.ts`. |
| `POST /strategy/voice-to-text` | Voice button cut from UI. |
| `POST /strategy/{id}/clarify` | Interactive clarification loop cut; `question` & `question_options` are ignored in `ParseResponse`, and `SupportBanner` `note` explains any approximation. |
| `PATCH /strategy/{id}/rules` | Rule cards are read-only in MVP; users edit the prompt and re-parse. |
| `GET /backtest/{id}/stream` (SSE) | `ProgressStepper` advances on a ~4 s client-side timer while `POST /py/run` awaits completion. |
| `POST /py/refine` | P1 `RefineInput` reuses `POST /py/parse` + `POST /py/run`. |
| `GET /py/gallery` & `POST /py/gallery` | Strategy Gallery (Screen 3) and publishing cut from MVP. |
| `DOGE`, `SOL`, `QQQ` & `1h` UI selector | UI only offers `BTC` (Binance via `ccxt`) and `SPY` (Yahoo Finance) on `1d` bars. |
