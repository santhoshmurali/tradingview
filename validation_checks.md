# Pine Script Validation Checks

Run this checklist before handing off any Pine Script change. Add a new check only when a real compiler error, warning, or non-obvious platform behavior has been confirmed and resolved.

## Compile-time argument types

- [ ] Every `plotshape()` `size` argument is a compile-time constant (`size.tiny`, `size.small`, `size.normal`, `size.large`, etc.). Do not pass an `input.string` or other dynamic value directly.
- [ ] If a user-selectable signal size is required, prefer bounded `label.new()` objects. Multiple constant-size `plotshape()` branches prevent CE10123 but multiply compile-time plot counts.

## Plot-budget audit

- [ ] Count every `plot*()` call before delivery. Remember that `plotcandle()` consumes four plot counts for its OHLC series; color-series arguments can also consume plot counts.
- [ ] Keep the script below TradingView's 64-plot limit. Do not use every constant-size `plotshape()` branch merely to emulate a configurable `size`; use capped labels instead when plot capacity is constrained.

