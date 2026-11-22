# Pine Script Development Assistant
Professional Pine Script v6 development environment for TradingView indicators and strategies. 

## Quick Start

### Indicator naming and storage info
**Important**
- **Pinescript location**: 
    - All indicators need to be stored in /indicator directory. 
    - All Strategies need to be stored in /strategy directory.
    - All the Libraries need to be stored in /library directory.
- **Pine Script name**: Always Prefix with "PValue-" followed by indicator name.
- **Pine Script filename name**: Always Prefix with "PValue-" followed by indicator name.

## Critical Syntax Rules

### Line Continuation - NEVER Split Ternary Operators

**WRONG:**
```pinescript
titleText = condition ? "value1" :
            "value2"  // ERROR!
```

**CORRECT:**
```pinescript
titleText = condition ? "value1" : "value2"
```

For expressions outside parentheses, continuation lines must use non-multiple-of-4 indentation:
```pinescript
longCondition = ta.crossover(ema50, ema200) and
     rsi < 30   // 5 spaces - NOT multiple of 4
```

Use the `https://www.tradingview.com/pine-script-reference/v6/` with topic  for complete syntax rules.

### Plot Scope Restriction

NEVER use `plot()` inside local scopes (if, for, functions):
```pinescript
// ❌ WRONG
if condition
    plot(value)  // ERROR!

// ✅ CORRECT
plot(condition ? value : na)
```

### UDT-First Architecture

For complex data, define User Defined Types before calculations:
- Bundle related data together
- Use `xloc.bar_time` for unlimited lookback (not bar_index which has 5000 bar limit)
- Store both `time` and `bar_index` when capturing events

## Essential Limits

- Max 500 bars lookback
- Max 500 plot outputs
- Max 40 security() calls
- Max 64 drawing objects
- Strings: 40,960 characters

## Best Practices

1. **Version Declaration**: Always start with `//@version=6`
2. **Strategy Alerts**: Include `//@strategy_alert_message {{strategy.order.alert_message}}` **immediately after** the `strategy()` call (TradingView now rejects it above the declaration)
3. **Avoid Repainting**: Use `lookahead=barmerge.lookahead_off` with `request.security()`
4. **Handle NA Values**: Check with `na(value)` before operations
5. **Cache Loop Boundaries**: Store `array.size()` before loops (2025 breaking change)
6. **Group Inputs**: Use `group=` parameter for organized settings
7. **Add Tooltips**: Help users understand each input
8. **No Unrequested Chrome**: Do NOT add performance/stats tables or other decorative panels by default. Ship the core logic + requested visuals only; offer extras as a follow-up.
9. **Line Continuation - NEVER Split**: do not do a code line break. always write entire line. irrespective of length.
10. **Pine Script name**: Always Prefix with "PValue-" followed by indicator name.
11.	**Persistent Warning Message: Always display a concise warning**: "Use this indicator as a supporting tool, not as the primary basis for trading decisions." Use a small, readable font with sufficient contrast to ensure visibility in both dark and light themes. User should be able switch on/off in  Indicator/Strategy configuration.
12. **Use of Tables** : When useing tables, always fix the width of the table as 30% of Chart size. We can use Word-wrap always on.
13. **Validation Checks**: Before delivery, run the applicable checks in `validation_checks.md` in addition to the Pine Script quality checklist.

## Indicator Color Schemes

Every indicator must include a `Color Scheme` input section using `group=`. Add a dropdown that exposes the exact palette names in `ref/palettes.json`, with a sensible documented default. Pine Script cannot read local JSON at runtime, so encode the current palette names and ordered color values from that file in the script and map the selected dropdown value to its corresponding colors. Apply the selected palette consistently to every user-visible visual element: plots, fills, hlines, backgrounds, bar colors, shapes, lines, boxes, labels, tables, and the Help and warning panels. A scheme change must update the complete indicator appearance, not only its primary plot. When adding or changing a palette, update `ref/palettes.json` first and keep the script's encoded palette values and selector options synchronized with it.



## Indicator Help Panel

For indicators that need user-facing documentation, include an optional on-chart Help panel controlled by a `Show Help` boolean input defaulting to `false`. The panel must include the indicator description, basic usage instructions, and a concise reference for the indicator's configuration inputs. Provide user inputs for Help location and text size, defaulting to `position.top_right` and `size.small`. Set the Help table width to 30% of the chart pane using explicit cell widths totaling 30, and use explicit `\n` line breaks to keep text wrapped within the fixed width. Use `chart.bg_color` and `chart.fg_color` or equivalent theme-aware colors for readable light- and dark-theme display. Populate or clear the table only on `barstate.islast` so it does not create unnecessary per-bar updates. Keep the Help panel compact and relevant to the specific indicator.

## Documentation
The complete dodcumentation is available in the in the url `https://www.tradingview.com/pine-script-reference/v6/`


## Error Log (consult + append)

`docs/pine-error-log.md` is a running repository of real compilation errors, warnings,
and non-obvious techniques hit in this project. **Check it before writing plots/panes/
strategy declarations, and append a new entry whenever you resolve a compiler error or
gotcha** so the same mistake is never repeated. (E.g. it records that `barcolor()` has
no `force_overlay` arg, and how to combine a chart overlay + a pane oscillator in one
script.)

## Quality Checklist

Before delivering code:
- [ ] `//@version=6` present
- [ ] No repainting (or documented)
- [ ] NA values handled
- [ ] Loop boundaries cached
- [ ] Inputs grouped with tooltips
- [ ] Indicator has a `Color Scheme` selector synchronized with `ref/palettes.json` and applies it to all visuals
- [ ] Strategy has alert annotation
- [ ] Applicable checks in `validation_checks.md` completed
- [ ] Complete code review by cross checking with the documentaion and confirm nothing is broken
