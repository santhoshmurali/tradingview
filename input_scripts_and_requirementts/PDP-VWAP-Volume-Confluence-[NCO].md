# Role & Objective
Act as an expert TradingView Pine Script v6 developer. Build a comprehensive, non-repainting, multi-component indicator named **"PDP-VWAP Volume Confluence [NCO]"** with `overlay=true`.

## Accepted Implementation Decisions
- NCO remains integrated as a calculation and entry confirmation only. NCO and Volume + SMA chart panels are intentionally omitted because they are supplied separately and would consume Pine's finite plot budget.
- Signal visuals use bounded `label.new()` objects rather than `plotshape()` so configurable sizes do not multiply the compile-time plot count. Retain at most 500 recent labels (user-configurable).
- Entry labels use `SE` for bearish/short entries and `LE` for bullish/long entries. A separate diamond label denotes a high-probability entry.

## Critical Integration Requirement (NCO Logic)
- The reference Pine Script for the **Normalized Candles Oscillator (NCO)** is attached at @`input_scripts_and_requirementts\NCO.pine`
- Extract, optimize, and integrate the NCO calculation logic directly into this script using Pine Script v6 best practices.
- Wrap the integrated NCO logic block clearly with prominent header and footer comments:
  - `// ============================================================================`
  - `// START: NORMALIZED CANDLES OSCILLATOR (NCO) LOGIC`
  - `// ============================================================================`
  - *(...NCO code...)*
  - `// ============================================================================`
  - `// END: NORMALIZED CANDLES OSCILLATOR (NCO) LOGIC`
  - `// ============================================================================`

---

## Core Definitions & Calculations
1. **A.A.V.C. (Above Average Volume Candle):** Any candle where `volume > ta.sma(volume, 20)`.
2. **Strong Close:** Let `candleRange = high - low` (guard against `candleRange == 0`).
   - **Bullish Strong Close:** `close > open` AND (`(close - open) / candleRange >= 0.85` OR `(open - low) / candleRange >= 0.65`).
   - **Bearish Strong Close:** `close < open` AND (`(open - close) / candleRange >= 0.85` OR `(high - open) / candleRange >= 0.65`).
3. **PDP (Previous Day Price) & Liquidation Zone:**
   - Fetch non-repainting daily values using `request.security(syminfo.tickerid, "D", ..., lookahead=barmerge.lookahead_off)` referencing `[1]` (previous completed day).
   - Depending on user selection (`Open`, `Close`, `High/Low`, `(H+L)/2`, `(H+L+C)/3`):
     - If `High/Low` is selected, calculate **two separate levels**: Previous Day High (`PDH`) and Previous Day Low (`PDL`).
     - Each PDP level has a symmetrical liquidation zone defined by `PDP Threshold %`: `[PDP * (1 - threshold/100), PDP * (1 + threshold/100)]`.
4. **Daily VWAP & Standard Deviation Bands:**
   - Calculate session-anchored Daily VWAP (`ta.vwap`) along with Upper and Lower Standard Deviation bands (`+1SD/-1SD`, `+2SD/-2SD`, or `+3SD/-3SD` based on user selection).
5. **NCO Threshold Zones:**
   - Based on `NCO Threshold` (`T`, default `0.0`):
     - `50` zone becomes `[50 - T, 50 + T]`.
     - `100` zone becomes `[100 - T, 100]` (clamped at 100).
     - `0` zone becomes `[0, 0 + T]` (clamped at 0).

---

## User Configuration Inputs (Grouped Cleanly)
1. **PDP Settings:**
   - `Previous Day Price Source`: String options `["Open", "Close", "High/Low", "(H+L)/2", "(H+L+C)/3"]` | Default: `"Close"`.
   - `PDP Threshold (%)`: Float input | Default: `0.05`, Min: `0.0`, Max: `10.0`, Step: `0.01`.
   - `PDP Zone Visibility`: String options `["Previous Day Only", "Past N Days"]` | Default: `"Previous Day Only"`.
   - `Number of Past Days (N)`: Integer input | Default: `5`, Min: `1`.
2. **VWAP & NCO Settings:**
   - `VWAP Standard Deviation`: String/Int options `["1SD", "2SD", "3SD"]` | Default: `"1SD"`.
   - `NCO Threshold`: Float input | Default: `0.0`, Min: `0.0`, Max: `25.0`, Step: `0.1`.
3. **Signal Visuals & Theme:**
   - Color theme compatibility (adapt to chart theme by default, with color overrides for PDP zones, VWAP/bands, Bullish signals, and Bearish signals).
   - Size selectors (`size.tiny`, `size.small`, `size.normal`, `size.large`) and color overrides for:
     - `Bullish Prepare Indicator` & `Bullish Entry Indicator`
     - `Bearish Prepare Indicator` & `Bearish Entry Indicator`
   - `Maximum Signal Labels`: Integer input, default `500`, limiting retained signal labels.
4. **Statistics Table Settings:**
   - `Table Position`: Options (`Bottom Right`, `Bottom Left`, `Top Right`, `Top Left`, `Middle Right`, `Middle Left`) | Default: `"Bottom Right"`.
   - `Table Text Size`: Options (`Tiny`, `Small`, `Normal`, `Large`) | Default: `"Small"`.
   - `Aggregation Period 1`: Period (`1H` default) + Mode (`Current` / `Previous`, default: `Current`).
   - `Aggregation Period 2`: Period (`1D` default) + Mode (`Current` / `Previous`, default: `Current`).
   - `Aggregation Period 3`: Period (`1W` default) + Mode (`Current` / `Previous`, default: `Current`).
   - `Aggregation Period 4`: Period (`1M` default) + Mode (`Current` / `Previous`, default: `Current`).

---

## Trading Signal State Machine & Entry Rules

### Short Setup Logic
1. **Bias & Location Filter:**
   - Price is trading at or below PDP (specifically `close <= PDP_Upper_Zone`; if `High/Low` is active, evaluate relative to the active/nearest PDP level or `PDH`/`PDL` upper zone).
   - Price is trading below Daily VWAP (`close < Daily_VWAP`).
   - **High Probability Condition:** Price is also trading below the selected lower VWAP Standard Deviation band (`close < Lower_VWAP_SD`).
2. **Bearish Prepare Signal (State Armed):**
   - While bias filters hold, detect a counter-trend pullback: **minimum 1 Bullish A.A.V.C.** (`close > open` and `volume > SMA(20, volume)`).
   - Trigger the **Bearish Prepare** visual on this pullback candle (or upon confirmation) and enter the `Bearish_Armed` state.
3. **NCO Threshold Test:**
   - Anywhere within the active setup window (from pullback initiation up to and including the Entry candle), `NCO High` must touch or enter the **50 zone** (`[50 - T, 50 + T]`) or **100 zone** (`[100 - T, 100]`).
4. **Bearish Entry Signal (Trigger):**
   - While `Bearish_Armed` is active and the NCO Threshold test has been satisfied in the window, detect trend continuation: **minimum 2 consecutive Bearish A.A.V.C.s** (`close < open` and `volume > SMA(20, volume)`), where **at least one** of those consecutive candles is a **Bearish Strong Close**.
   - Plot the **Bearish Entry** signal on the candle that completes this condition with a **Bearish Strong Close** (if the 2nd+ consecutive Bearish A.A.V.C. is a Strong Close, trigger there; if the 1st was a Strong Close and the 2nd consecutive Bearish A.A.V.C. confirms the count, trigger upon completion).
   - Reset `Bearish_Armed` state after an Entry triggers.
5. **Invalidation / Reset Rules for Bearish Prepare:**
   - Immediately cancel/reset `Bearish_Armed` if:
     - Price crosses back above Daily VWAP OR above the PDP upper threshold zone.
    - NCO reaches the diagonally opposite `0` threshold zone `[0, T]`. Movement below the 50 zone after a valid test is treated as normal bearish continuation and does not reset the setup.

### Long Setup Logic (Exact Mirror of Short)
1. **Bias & Location Filter:**
   - Price is trading at or above PDP (`close >= PDP_Lower_Zone`).
   - Price is trading above Daily VWAP (`close > Daily_VWAP`).
   - **High Probability Condition:** Price is also trading above the selected upper VWAP Standard Deviation band (`close > Upper_VWAP_SD`).
2. **Bullish Prepare Signal (State Armed):**
   - While bias filters hold, detect a counter-trend pullback: **minimum 1 Bearish A.A.V.C.** (`close < open` and `volume > SMA(20, volume)`).
   - Trigger the **Bullish Prepare** visual and enter the `Bullish_Armed` state.
3. **NCO Threshold Test:**
   - Anywhere within the active setup window before/on the Entry candle, `NCO Low` must touch or enter the **50 zone** (`[50 - T, 50 + T]`) or **0 zone** (`[0, T]`).
4. **Bullish Entry Signal (Trigger):**
   - While `Bullish_Armed` is active and the NCO Threshold test is satisfied, detect **minimum 2 consecutive Bullish A.A.V.C.s** where **at least one** is a **Bullish Strong Close**.
   - Plot the **Bullish Entry** signal upon completion with the Strong Close condition, then reset `Bullish_Armed`.
5. **Invalidation / Reset Rules for Bullish Prepare:**
   - Immediately cancel/reset `Bullish_Armed` if:
     - Price crosses back below Daily VWAP OR below the PDP lower threshold zone.
    - NCO reaches the diagonally opposite `100` threshold zone `[100 - T, 100]`. Movement above the 50 zone after a valid test is treated as normal bullish continuation and does not reset the setup.

---

## Plots, Overlays & Statistics Table
1. **Main Chart Overlays:**
   - **PDP Lines & Zones:** Plot PDP horizontal levels and semi-transparent shaded threshold zones (`box.new` / `line.new`) for either the Previous Day only or the Past `N` Days.
   - **Daily VWAP & SD Bands:** Plot Daily VWAP and the selected upper/lower SD bands cleanly.
2. **NCO & Volume Displays:**
   - Do not render NCO or Volume + 20 SMA panels in this indicator. Users add those separate indicators when required; their values remain available internally for signal calculations.
3. **Signal Labels (`label.new`):**
   - **Bearish Prepare:** bearish `PREP` label above the bar, configurable size.
   - **Bearish Entry (Standard):** bearish `SE` arrow label above the bar, configurable size.
   - **Bearish Entry (High Probability - below `-VWAP SD`):** stacked accent-color diamond label above the `SE` label.
   - **Bullish Prepare:** bullish `PREP` label below the bar, configurable size.
   - **Bullish Entry (Standard):** bullish `LE` arrow label below the bar, configurable size.
   - **Bullish Entry (High Probability - above `+VWAP SD`):** stacked accent-color diamond label below the `LE` label.
4. **Multi-Timeframe Statistics Table (`table.new`):**
   - Render on `barstate.islast` at the user-configured position and text size.
   - Columns for the 4 configurable aggregation periods (defaults: `Current 1H`, `Current Day`, `Current Week`, `Current Month`—each switchable between `Current` and `Previous` completed period).
   - Rows tracking:
     - `Bullish Prepare` (count)
     - `Bullish Entry` (count)
     - `Bullish Conversion %` (`(Bullish Entry / Bullish Prepare) * 100`, safe-divided to show `0.0%` when Prepare is 0)
     - `Bearish Prepare` (count)
     - `Bearish Entry` (count)
     - `Bearish Conversion %` (`(Bearish Entry / Bearish Prepare) * 100`, safe-divided)
5. **Alerts:**
   - Include `alertcondition()` hooks for Bullish Prepare, Bullish Entry, High-Prob Bullish Entry, Bearish Prepare, Bearish Entry, and High-Prob Bearish Entry.
