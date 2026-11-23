# PValue-True Percentage Renko Requirements

## Purpose

Build a production-ready TradingView Pine Script v6 overlay indicator named `PValue-True Percentage Renko`, with short title `PValue-True % Renko`. The indicator draws confirmed Renko-style bricks over a standard time-based chart. It supports compounding percentage bricks, fixed-price bricks, ATR-sized bricks, and standard-deviation-sized bricks without using TradingView's built-in Renko chart type.

The script must use `overlay = true`, `max_boxes_count = 500`, `max_labels_count = 500`, and `max_lines_count = 500`. It is an indicator only: do not add trade orders, buy/sell signals, or alert conditions. Its candle-count statistics panel is the sole requested analytical table.

## Calculation and Confirmation Rules

- Process brick state only on `barstate.isconfirmed`; intrabar movement must not create or revise bricks.
- Use `high` for rises and `low` for declines when the calculation basis is `High/Low`; use `close` for both directions when it is `Close`.
- Initialize the reference price from the `open` of the first confirmed bar for which the selected sizing mode has a valid box size. Do not form a brick on that initialization bar.
- Continue an existing direction before testing a reversal on the same confirmed bar. Use `while` loops so a gap or large move can create multiple sequential bricks.
- In adaptive modes, calculate the box size on every bar but use it only after the bar confirms. All thresholds and all bricks formed on that confirmed bar use that bar's one current size. Previously drawn bricks never change.
- ATR and standard-deviation modes must not initialize or draw bricks until their computed size is non-`na` and greater than zero.

### Box Size Modes

The `Box Size Type` input must provide these exact options:

| Mode | Box-size calculation | Upward next price | Downward next price |
| --- | --- | --- | --- |
| `Percentage` | `factor = 1 + Brick Size (%) / 100` | `price * factor` | `price / factor` |
| `Absolute` | `Absolute Box Size` chart-price units | `price + boxSize` | `price - boxSize` |
| `ATR Multiple` | `ta.atr(ATR Length) * ATR Multiplier` | `price + boxSize` | `price - boxSize` |
| `Standard Deviation` | `ta.stdev(close, Standard Deviation Length) * Standard Deviation Multiplier` | `price + boxSize` | `price - boxSize` |

Percentage mode uses a symmetric logarithmic grid and is the default. Absolute mode is measured directly in the symbol's chart-price units, not minimum ticks. Standard deviation is calculated from `close`; its multiplier choices are 1, 2, and 3.

### Brick and Reversal Behavior

- With no established direction, form an upward brick when the upward source reaches the next upward price from the initial price, or a downward brick when the downward source reaches the next downward price.
- In an upward direction, keep forming upward bricks while the upward source reaches the next upward price from `brickClose`. Then test reversal:
  - One-brick reversal: trigger at `brickOpen`; form a downward brick from `brickClose` to `nextDown(brickClose)`.
  - Two-brick reversal: trigger at `nextDown(brickOpen)`; form the non-overlapping downward brick from `brickOpen` to `nextDown(brickOpen)`.
- In a downward direction, keep forming downward bricks while the downward source reaches the next downward price from `brickClose`. Then test reversal:
  - One-brick reversal: trigger at `brickOpen`; form an upward brick from `brickClose` to `nextUp(brickClose)`.
  - Two-brick reversal: trigger at `nextUp(brickOpen)`; form the non-overlapping upward brick from `brickOpen` to `nextUp(brickOpen)`.
- After any reversal brick, continue forming additional bricks in the new direction while the same source price still crosses successive thresholds.

## Inputs

### Calculation

- `Box Size Type`: `Percentage` by default; options are `Percentage`, `Absolute`, `ATR Multiple`, and `Standard Deviation`.
- `Brick Size (%)`: float, default `10.0`, minimum `0.01`, step `0.5`; used only in Percentage mode.
- `Absolute Box Size`: float, default `1.0`, positive; used only in Absolute mode.
- `ATR Length`: integer, default `14`, minimum `1`; used only in ATR Multiple mode.
- `ATR Multiplier`: float, default `1.0`, positive, step `0.1`; used only in ATR Multiple mode.
- `Standard Deviation Length`: integer, default `20`, minimum `1`; used only in Standard Deviation mode.
- `Standard Deviation Multiplier`: integer, default `1`, options `1`, `2`, `3`; used only in Standard Deviation mode.
- `Brick Calculation Basis`: `High/Low` by default; options `High/Low` and `Close`.
- `Reversal Brick Count`: default `2`; options `1` and `2`.

### Display

- `Standard Chart Candles`: default `Visible`; options `Visible`, `Dimmed`, and `Hidden`.
- `Dimmed Candle Transparency (%)`: integer, default `30`, range `0` through `100`.
- `Renko Segment Fill Opacity (%)`: integer, default `85`, range `0` through `100`.
- `Show Brick Price & % Labels`: boolean, default `false`.
- `Label Text Size`: default `tiny`; options `tiny`, `small`, and `normal`.
- `Show Trading Warning`: boolean, default `true`.

### Statistics

- `Show Statistics`: boolean, default `true`.
- `Lookback Renko Boxes`: integer, default `50`, range `1` through `500`; uses the most recent completed boxes.
- `Number of Bins`: integer, default `7`, range `1` through `50`; it is the maximum equal-width bin count.
- `Statistics Table Width (%)`: integer, default `20`, range `5` through `40`; total chart-pane width allocated to the table, split 50% to candle range and 25% each to frequency and probability.
- `Statistics Table Title`: string, default `Renko Statistics`; displayed as the merged first row of the Statistics table.
- `Statistics Location`: default `Bottom Right`; options `Top Left`, `Top Center`, `Top Right`, `Center Left`, `Center Center`, `Center Right`, `Bottom Left`, `Bottom Center`, and `Bottom Right`.
- `Statistics Text Size`: default `small`; options `tiny`, `small`, and `normal`.

The candle-count distribution pools bullish and bearish completed boxes. Store each brick's start bar, completion bar, candle count, and direction in a UDT. A box's completed count includes every time-based candle from its start bar through the bar immediately before the next box begins, irrespective of where candle prices fall relative to the box. Multiple bricks created on the same chart bar share the same eventual candle count.

For the available completed sample, calculate its minimum and maximum candle counts and divide the inclusive integer range into equal-width, non-overlapping bins. Use `min(Number of Bins, maximum - minimum + 1)` effective bins to avoid zero-width ranges. Display each bin as an accurate whole-candle interval, rounding its lower edge up and its upper edge down. Frequency is the number of completed boxes in each bin; probability is `frequency / sample size`. If fewer than the requested lookback are available, use all available completed boxes. If none are available, show a waiting state.

The current developing box includes the current chart candle. Map its candle count to the completed-sample bins and show its historical bin probability. Clamp counts below or above the completed-sample range to the nearest bin.

### Midpoint Guides

- `Show Midpoint Guides`: boolean, default `true`.
- `Midpoint Guide Box Count`: integer, default `10`, range `1` through `500`; includes the developing box.
- `Midpoint Line Width`: integer, default `1`, range `1` through `4`.
- `Midpoint Line Opacity (%)`: integer, default `25`, range `0` through `100`.

Draw a dotted midpoint guide at `(brick open + brick close) / 2` for every retained brick, including the developing box. Use time-based line coordinates, retain the most recent effective guide count, and color guides with the active bullish or bearish palette color. The effective count is `max(Midpoint Guide Box Count, Lookback Renko Boxes)`. When the configured midpoint count is lower than the Statistics lookback, show a final Statistics-table warning row that the guide count was adjusted.

### Color Scheme and Help

- Provide the existing `Color Scheme` group and `Palette` selector. Its options must remain synchronized with `ref/palettes.json`; `Highcharts 5.x` is the default.
- Apply the active palette consistently to bullish and bearish brick colors, labels, candle dimming, Help panel, warning panel, and statistics table.
- Provide `Show Help`, default `false`; `Help Location`, default `Top Right`, with the same nine table-position options as Statistics Location; and `Help Text Size`, default `small`.

## Rendering and Retention

- Draw every brick with `box.new()` on the main price chart from its open/close range, using a solid border and the configured fill opacity.
- Retain each newly formed box as the active segment until the next chart bar, extending its right edge to the current bar. When a new bar begins, close the prior active segment at that bar and begin a new one.
- Keep at most 500 brick boxes and 500 labels, deleting the oldest retained object after the limit is exceeded.
- Keep at most 500 midpoint-guide lines, deleting the oldest retained guide after the effective count is exceeded.
- Keep at most 500 completed box-duration records, deleting the oldest record after the limit is exceeded.
- When enabled, labels must show the brick close using `format.mintick` and its signed percentage move `((close - open) / open) * 100`. Place labels at the close, above bullish bricks and below bearish bricks.
- Use `barcolor()` for dimmed or hidden standard candles. Hidden candles use a fully transparent color; chart settings may still control border/wick visibility.
- Display the exact, toggleable warning text: `Use this indicator as a supporting tool, not as the primary basis for trading decisions.` Render it in a theme-readable palette-derived table at the bottom center.
- Render the optional Help panel only on the last bar. It must use the selected location and text size, palette-derived readable colors, and an explicit width of 30% of the chart pane.
- Render the optional statistics table only on the last bar. Its first row must be a merged, palette-accented title row using the configured Statistics Table Title. Below it, show candle-count range, frequency, and probability columns plus a current-box row, use palette-derived readable colors, and have explicit cell widths totaling the configured Statistics Table Width (%), 20% by default and configurable from 5% to 40%. Highlight the current-box row with the palette accent color and every tied highest-probability bin row with the palette bullish color. Color the current Probability cell bullish when its bin is tied for the highest occupied frequency, bearish when it is tied for the lowest occupied frequency, and accent when it lies between those frequencies or all occupied bins have equal frequency. When midpoint-guide count is adjusted, append the adjustment warning as the final row.
- Add a `Direction Mix` subheader and summary row using the same completed-box lookback. Show bullish-box count, bearish-box count, and the bullish-to-bearish ratio. Use distinct palette-derived bullish, bearish, and accent backgrounds for those three summary cells. Color the B/B ratio bullish above 1, bearish below 1, and accent at exactly 1; show an infinite bullish ratio when there are no bearish boxes.

## Engineering Constraints

- Begin with `//@version=6` and use explicit Pine types where appropriate.
- Define the box-duration UDT before calculations and cache array sizes before loops.
- Do not split ternary expressions across lines or call plotting functions in local scope.
- Keep all loop boundaries cached before iterating arrays.
- Use `na` guards before operating on adaptive box sizes.
- Preserve the current palette encoding and selector names in sync with `ref/palettes.json`.
- Before delivery, run the repository `validation_checks.md` checklist and review `docs/pine-error-log.md` for applicable Pine v6 constraints.
