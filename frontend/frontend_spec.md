> ⚠️ **API and product changed.** Use [`../docs/API_CONTRACT.md`](../docs/API_CONTRACT.md) for the product, types, endpoints and screens.
> Still valid from this file: §1 stack and setup (Next.js, Tailwind, shadcn, Recharts), §8 visual style, light theme, and "reliability beats features".
> Ignore: §3 types, §4 API client, §5 pages, §6 presets, §7 mocks, and the `/py/*` rewrite. Use `NEXT_PUBLIC_BACKEND_URL` directly.

# VibeTrader — Frontend Spec (hand-off for coding agent)

> Attached: `wireframes.html` — two screens, ① Build and ② Results. Open it in a browser and screenshot it if you need images.
> Red tags = Build-flow components, blue tags = Results components, dashed boxes = P1 (only if time allows). Follow the layouts closely; polish the visual style beyond the wireframe.

---

## 0. Context (read first)

We are building **VibeTrader** at a one-day hackathon (Gemini 4 Argon × Imperial FinTech & Blockchain Society). Judges: Google DeepMind, Vercel, an Imperial ML professor (uncertainty/trustworthy ML), and a fintech judge.

**Product in one line:** *"The backtester that tells you the truth."* A user describes a trading strategy in plain English → the AI turns it into rules and code → a deterministic engine backtests it → an **audit** checks for look-ahead leaks, overfitting (in-sample vs out-of-sample) and statistical significance (bootstrap 95% CI on Sharpe) → the user gets an honest verdict: **Edge / Inconclusive / No edge**.

**The frontend never calls Gemini directly.** All AI and backtesting happens in a Python FastAPI backend reached through `/py/*`. The frontend only renders.

**The demo is 3 minutes, live, on a projector.** The Results page is what judges stare at longest, so it must be big, clear and beautiful. Reliability beats features.

**Scope is the MVP: one page, one flow.** Prompt → rules → run → results, plus a demo mode that works with the backend off. Anything that does not serve that flow is cut (see section 12).

---

## 1. Stack & setup

- **Next.js (App Router) + TypeScript + Tailwind**
- **shadcn/ui** components: `button card badge textarea select skeleton sonner`
- **Recharts** for charts
- No other UI or state libraries. Use React state and `fetch`. No auth, no database, no Gemini SDK on the frontend.

```bash
npx create-next-app@latest web --ts --tailwind --app --eslint
cd web
pnpm add recharts
npx shadcn@latest init
npx shadcn@latest add button card badge textarea select skeleton sonner
```

### Backend proxy — `next.config.ts`
```ts
const nextConfig = {
  async rewrites() {
    return [{ source: "/py/:path*", destination: `${process.env.BACKEND_URL ?? "http://localhost:8000"}/:path*` }];
  },
};
export default nextConfig;
```

### Env vars — `.env.local`
```
BACKEND_URL=http://localhost:8000      # or a cloudflared tunnel URL from the backend laptop
NEXT_PUBLIC_USE_MOCK=true              # true until backend is live
```

---

## 2. Folder structure

```
web/
  app/
    layout.tsx            # nav bar (logo + tagline), Toaster, footer disclaimer
    page.tsx              # Build + Results (single page; results appear below after run)
  components/
    build/  PromptInput.tsx  PresetChips.tsx  AssetSelect.tsx  SupportBanner.tsx
            RuleCards.tsx  ConfirmButton.tsx  ProgressStepper.tsx
    results/ EquityChart.tsx  MetricsTable.tsx  VerdictBadge.tsx
            AuditCard.tsx  SharpeCIBar.tsx  CodePanel.tsx
            RefineInput.tsx  TrustScoreRing.tsx      # P1
  lib/
    types.ts              # section 3
    api.ts                # section 4
    presets.ts            # section 6
    mocks/ parse.json  run.json  demo/{dip,rsi,trend}.json
```

---

## 3. Types — `lib/types.ts` (this IS the API contract; do not change field names)

Fields marked *ignored* stay in the contract so the backend does not have to change, but the MVP frontend does not render them.

```ts
export type Asset = "BTC" | "ETH" | "SPY";   // UI offers BTC and SPY only
export type Timeframe = "1d" | "1h";         // UI always sends "1d"
export type Verdict = "edge" | "inconclusive" | "no_edge";
export type Support = "supported" | "approximated" | "unsupported";

export interface Rules {
  asset: Asset;
  timeframe: Timeframe;
  entry: string[];                 // plain-English rule lines, e.g. "z-score(close, 20) < -2"
  exit: string[];
  params: Record<string, string | number>;
}

export interface ParseRequest { prompt: string; asset: Asset; timeframe: Timeframe; }
export interface ParseResponse {
  rules: Rules;
  support: Support;
  note: string | null;             // shown in SupportBanner when approximated/unsupported
  question: string | null;         // ignored
  question_options: string[];      // ignored
}

export interface Metrics { sharpe: number; cagr: number; max_dd: number; trades: number; }
export interface EquityPoint { date: string; strategy: number; buyhold: number; }

export interface Audit {
  lookahead: { passed: boolean; fixed: boolean; attempts: number; detail: string | null };
  sharpe_ci: [number, number];     // 95% CI of out-of-sample Sharpe
  verdict: Verdict;
  trust_score: number;             // 0–100; rendered only by the P1 TrustScoreRing
  checks: string[];                // e.g. ["Fees 10 bps applied", "70/30 split"]
}

export interface RunResult {
  strategy_id: string;
  version: number;
  name: string;
  rules: Rules;
  code: string;                    // Python source of signals(df)
  equity: EquityPoint[];
  split_date: string;              // boundary between in-sample and out-of-sample
  metrics: { in_sample: Metrics; out_sample: Metrics };
  audit: Audit;
  elapsed_s: number;
}

export interface RunRequest { rules: Rules; prompt: string; }

export interface ApiError { error: string; }
```

### Endpoints
| Method | Path | Body | Returns |
|---|---|---|---|
| POST | `/py/parse` | `ParseRequest` | `ParseResponse` |
| POST | `/py/run` | `RunRequest` | `RunResult` (can take **20–60 s**) |

Errors return non-2xx with `{ "error": "message" }`. The `/py/refine` and `/py/gallery` endpoints are cut; the backend does not need to build them.

---

## 4. API client — `lib/api.ts`

- One function per endpoint: `parse()`, `run()`.
- If `NEXT_PUBLIC_USE_MOCK === "true"`, return the JSON from `lib/mocks/` after a fake delay (parse 800 ms, run 6 s) so loading states can be designed properly.
- Real calls: `fetch("/py/...")` with a **120 s timeout** (AbortController). On non-2xx, throw `Error(body.error)`.
- **Demo mode (P0):** if the URL has `?demo=1` and the prompt matches a preset (section 6), return the cached `lib/mocks/demo/*.json` result instead of calling the backend. This is our safety net if the backend fails on stage. Demo mode must work with the backend turned off.

---

## 5. Pages & components

### 5.1 Layout
- Top nav: **VibeTrader** logo + grey tagline *"The backtester that tells you the truth"* on the left. No nav links (single page).
- Footer, small grey: `Not investment advice · Data: Binance via ccxt, Yahoo Finance`.
- `<Toaster />` for errors.

### 5.2 Build section (`app/page.tsx`) — wireframe ①

States: `idle → parsing → parsed → running → done` (+ `error`). Keep all of this in one page component.

| Component | Requirements |
|---|---|
| **PromptInput** | Large textarea (≥ 3 rows, 18 px text). Placeholder: *"Describe your strategy in plain English…"*. Cmd/Ctrl+Enter = Parse. |
| **PresetChips** | 3 pill buttons that fill the textarea with the preset prompt (section 6). Dashed border style. |
| **AssetSelect** | One select: BTC / SPY, default BTC. No timeframe select; always send `"1d"`. |
| **Parse button** | Primary black. Calls `parse()`. Show spinner while parsing. |
| **SupportBanner** | Shown after parse. Text only, no buttons. `supported` → small green line "✓ Fully supported". `approximated` → **yellow** banner with ⚠ and the `note`. `unsupported` → **red** banner with ✕ and the `note`; the Confirm button is disabled. |
| **RuleCards** | Three read-only cards in a row: **Entry**, **Exit**, **Params**. Each rule is one line. To change rules, the user edits the prompt and parses again. |
| **ConfirmButton** | Big primary "✓ Confirm & Run backtest", bottom right. Calls `run({ rules, prompt })`. |
| **ProgressStepper** | Shown while running. 5 steps: *Generate code → Sandbox run → Look-ahead test → Backtest → Audit*. Backend gives no progress events, so **advance steps on a timer** (~4 s each, stay on the last step until the response arrives), then mark all ✓ and show "done in {elapsed_s}s". |

After `run()` resolves, render the **Results section** (5.3) below the stepper and smooth-scroll to it. The prompt area collapses to a one-line summary with an "Edit" link that re-opens it.

### 5.3 Results section — wireframe ② (top priority for polish)

Layout: header row with strategy name + "asset · 1d"; then a 2-column grid (left ≈ 60 %, right ≈ 40 %).

| Component | Requirements |
|---|---|
| **EquityChart** (left, top) | Recharts `LineChart`, height ~300 px. Two lines: **Strategy** (solid, dark, 3 px) and **Buy & hold** (dashed, grey). `ReferenceArea` from `split_date` to the end with light grey fill + `ReferenceLine` at `split_date`; labels "IN-SAMPLE" / "OUT-OF-SAMPLE". Legend top right. Year ticks only on the X axis. |
| **MetricsTable** (left, bottom) | Rows: Sharpe, CAGR, Max drawdown, Trades. Columns: In-sample, Out-of-sample. Colour an out-of-sample value **red with ↓** if it is meaningfully worse (Sharpe or CAGR drops by > 50 %, or drawdown is > 1.5× worse). Format: Sharpe 2 decimals, CAGR/DD as %. |
| **VerdictBadge** (right, top) | Card tinted by verdict. Huge text **✅ Edge** / **⚠️ Inconclusive** / **❌ No edge** plus one-line subtitle: edge → "Out-of-sample results are statistically distinguishable from luck." · inconclusive → "Some signal, but not enough evidence yet." · no_edge → "No evidence this beats buy & hold after fees." |
| **AuditCard** (right, middle) — *the selling point, make it prominent* | (1) **Look-ahead row:** `passed && !fixed` → green "✓ No look-ahead leak". `fixed` → yellow box "⚠→✓ Look-ahead leak found & fixed ({attempts} attempts)" + `detail` in grey. `!passed` → red "✕ Look-ahead leak detected". (2) **SharpeCIBar** (below). (3) `checks[]` as ✓ items. |
| **SharpeCIBar** | SVG horizontal axis, dashed vertical line at **0** labelled "0", thick rounded bar from CI low to CI high, circle marker at the point estimate (out-of-sample Sharpe). Bar is **red if the CI crosses 0**, green if entirely above 0. Caption under it: crosses 0 → "Interval crosses 0 → can't distinguish from luck." else "Entire interval above 0." |
| **CodePanel** (right, bottom) | Collapsed by default: monospace preview `def signals(df) -> pd.Series: …` + "▸ Show code". Expanded: `<pre>` with the full `code`, horizontal scroll. |

#### P1 — only after everything in P0 works end to end

| Component | Requirements |
|---|---|
| **TrustScoreRing** | Circular SVG ring (stroke-dasharray) showing `trust_score`, placed left of the verdict text inside VerdictBadge. |
| **RefineInput** | Full-width input below the grid: "Refine: e.g. only trade when price is above the 200-day average…" + primary **Refine**. No new endpoint: call `parse()` with the original prompt + `"\nRefinement: <instruction>"`, then `run()` with the returned rules. The new result **replaces** the current one (no version tabs); show "v{n}" next to the strategy name. |

---

## 6. Presets & demo cache — `lib/presets.ts`

| Chip label | Prompt (fills textarea) | Story in the demo |
|---|---|---|
| 🎯 Buy the dip | `Buy when price is unusually low compared to normal, sell when it recovers` | **approximated** at parse; look-ahead leak caught & fixed → **no_edge** |
| 📅 RSI on Tuesdays | `Buy when RSI is below 28, sell when it is above 71, only trade on Tuesdays` | Great in-sample, collapses out-of-sample (overfit) → **no_edge** |
| 📈 200-day trend | `Buy when price is above its 200-day moving average, otherwise stay in cash` | Modest but honest → **inconclusive** or **edge** |

Create `lib/mocks/demo/dip.json`, `rsi.json`, `trend.json` as full `RunResult` objects matching the stories above (backend will overwrite these with real cached runs later; keep the file names).

---

## 7. Mock data — what to put in `lib/mocks/`

- `parse.json`: the **Buy the dip** case with `support: "approximated"`, note *"'Unusually low' is ambiguous — we'll use a 20-day z-score below −2. Not the full-history mean (that would leak the future)."*, `question: null`, `question_options: []`, rules exactly as in wireframe ①.
- `run.json`: the wireframe ② numbers → name "Buy the dip (z-score)", IS Sharpe 2.10 / CAGR 0.45 / DD −0.12 / 40 trades; OOS Sharpe 0.12 / CAGR 0.02 / DD −0.30 / 15 trades; `lookahead: {passed: true, fixed: true, attempts: 2, detail: "Original code used the full-history mean — future data."}`; `sharpe_ci: [-0.4, 0.6]`; `verdict: "no_edge"`; `trust_score: 32`; `checks: ["Fees 10 bps applied", "70/30 split"]`; ~250 daily equity points 2021→2026 (generate with a seeded random walk in a small script; strategy climbs in-sample, flattens out-of-sample); `split_date` at 70 %.

---

## 8. Visual style

- Clean, light theme. Background `#fafafa`, cards white with 1–2 px `#e5e5e5` borders, radius 10–12 px, generous spacing.
- **Verdict colours (use everywhere):** edge `#1a8f4c` / bg `#e5f6ec` · inconclusive `#b88a00` / bg `#fff6d6` · no_edge `#d62828` / bg `#fde8e8`. Primary buttons black.
- **Projector-first:** base font 16 px, headings bold, verdict text ≥ 30 px, chart lines ≥ 2.5 px, high contrast. Check at **1280×720**: the whole Results section should fit with at most one scroll.
- Never show a blank screen: every async area needs loading (Skeleton), empty and error states.

---

## 9. Priorities & order of work

**P0 — by 12:30 (all with mocks)**
1. Setup, types, mocks, `api.ts` with mock toggle **and demo mode**, layout.
2. Build flow: PromptInput, PresetChips, AssetSelect, Parse → SupportBanner + RuleCards → Confirm → ProgressStepper.
3. Results: EquityChart, VerdictBadge, MetricsTable, AuditCard + SharpeCIBar, CodePanel.
4. Deploy to Vercel once (even with mocks) to prove the pipeline.

**13:00 — switch `NEXT_PUBLIC_USE_MOCK=false`, integrate with the real backend.**

**P1 — by 14:45, only if P0 works end to end against the real backend**
5. TrustScoreRing.
6. RefineInput.

**15:45 — code freeze. Bug fixes only.**

---

## 10. Acceptance checklist

- [ ] With mocks: clicking each preset → Parse → Confirm reaches a full Results view with no console errors.
- [ ] The 3 support states (supported, approximated, unsupported) and 3 verdicts (edge, inconclusive, no_edge) all render correctly; unsupported disables Confirm.
- [ ] Sharpe CI bar turns red when the interval crosses 0, green otherwise.
- [ ] `?demo=1` works with the backend turned off.
- [ ] Looks good at 1280×720; nothing overlaps; text is readable from the back of a room.
- [ ] Deployed on Vercel with `BACKEND_URL` set.

## 11. Don'ts
- Don't call Gemini, ElevenLabs or any AI API from the frontend; don't put API keys in client code.
- Don't rename contract fields. If something must change, tell the team first and update `docs/CONTRACT.md`.
- Don't add auth, a database, or extra state libraries.
- Don't block the demo on the backend: mock mode and demo mode must always work.
- Don't build anything from section 12.

## 12. Cut from the MVP (do not build)

| Feature | Why it was cut |
|---|---|
| Voice button | No demo value. |
| ClarifyQuestion + re-parse loop | Extra round trip and failure point on stage; the `note` in SupportBanner covers ambiguity. |
| Inline-editable rule cards | Edit the prompt and parse again instead. |
| SupportBanner *Edit* / *Use this* buttons | Confirm button and the "Edit" link already cover both actions. |
| 1h timeframe, ETH | Halves backend data work; BTC (ccxt) and SPY (Yahoo) only, daily bars. |
| 🐦 Elon tweets → DOGE preset | DOGE is not in the `Asset` type; Buy the dip already demos the approximated state. |
| Gallery page + Save to Gallery | Not part of the honesty story; save needs backend persistence. |
| VersionTabs, compare overlay | Version state management; Refine replaces the result instead. |
| ⓘ Tooltips | Invisible on a projector; the presenter explains verbally. |
| `/py/refine`, `/py/gallery` endpoints | Not needed by the frontend anymore. |
| Dark mode | Light theme only. |
