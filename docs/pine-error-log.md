# Pine Script Error Log

## CE10271: `ta.sum` not found in Pine v6

- Cause: The rolling sliding-sum function is in the `math` namespace in Pine v6.
- Fix: Use `math.sum(source, length)` for rolling sums, including conditional numeric series.
- Reference: https://www.tradingview.com/pine-script-reference/v6/

## Confirmed-bar gating for range/Renko signals

- Gotcha: Conditions using the live bar's `close` can change before a non-time-based candle is confirmed.
- Fix: Gate signal conditions and drawing-state updates with `barstate.isconfirmed`; preserve the prior confirmed guide during the live bar when needed.

## RE10140: Script creates too many plots

- Cause: Pine counts every plot-producing call at compile time, including inactive constant-size `plotshape()` branches. A `plotcandle()` call also consumes four plot counts for OHLC values.
- Fix: For configurable signal sizes in plot-constrained scripts, use bounded `label.new()` objects rather than one `plotshape()` call per size. Set `max_labels_count` and delete oldest labels from an array as new signal labels are created.

## NCO state invalidation can suppress all entries

- Cause: Resetting an armed short when NCO merely falls below the 50 zone, or an armed long when it merely rises above it, cancels normal continuation before the required two A.A.V.C. entry candles can form.
- Fix: After an NCO confirmation, retain the armed state through continuation movement. Reset only for price-bias failure or movement to the diagonally opposite NCO extreme (0 zone for shorts, 100 zone for longs).
