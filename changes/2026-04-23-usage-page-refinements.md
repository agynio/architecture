# Usage Page Refinements

## Target

- [Usage — Time Range](../product/usage/usage.md#time-range)
- [Usage — Chart Style](../product/usage/usage.md#chart-style)
- [Usage — LLM](../product/usage/usage.md#llm)
- [Usage — Compute](../product/usage/usage.md#compute)
- [Usage — Storage](../product/usage/usage.md#storage)
- [Usage — Platform](../product/usage/usage.md#platform)

## Delta

The Usage page is implemented with the previous spec. Current rendering has several issues that make the page harder to read and, for cached tokens, actively misleading.

Specifically:

- Charts render in the default monochrome theme. Series are visually indistinguishable, especially where two or three series share a bucket. Spec now defines a fixed neon palette per series (input / cached / output, CPU / RAM, storage, threads / messages, success / failure) that works on both light and dark themes.
- Cached tokens are presented as a separate category alongside input and output — the LLM summary and the daily-usage chart treat them as additive. In reality cached is a subset of input (e.g., `1000 input / 800 cached` means 800 of those 1000 input tokens came from cache). Spec now describes cached as a subset, shows it as `N of M input` on the summary card, and splits the input bar into cached + fresh rather than stacking cached as a third series.
- All time-series charts bucket by day regardless of the selected range. For a 24-hour range this collapses to one or two bars. Spec now defines grouping intervals per range: 24h → 5m, 7d → 1h, 30d → 6h, custom → auto-selected. Chart titles change from "Daily usage" to "… over time".
- Top-consumer and top-agent charts label bars with raw identity UUIDs. Spec now requires display names: agent name for agents, `@nickname` (or full name) for users, app name for apps. IDs are shown only in hover detail.
- Compute and Storage summaries render totals in core-seconds and GB-seconds — unreadable once a workload has run for more than a few minutes. Spec now renders totals and chart series in core-hours and GB-hours (underlying metering values remain in seconds; conversion is display-only).

## Acceptance Signal

- Each chart series uses the color specified in [Chart Style](../product/usage/usage.md#chart-style); the same metric has the same color across all charts on the page.
- LLM summary shows cached as `N of M input` (e.g., "800 of 1,000 input"). The time-series chart renders input and output side-by-side per bucket, with input split into cached + fresh; cached is never stacked on top of input as a third additive segment.
- Time-series charts bucket by the interval defined for the selected range (5m for 24h, 1h for 7d, 6h for 30d, auto for custom) and their titles reflect the range rather than "Daily".
- Top consumers (LLM) and Top agents (Compute, Storage) charts show display names. No raw UUIDs appear in the rendered labels.
- Compute summary cards read "CPU-core-hours" and "RAM-GB-hours"; Storage summary card reads "Storage-GB-hours". Values below one hour render with one decimal (e.g., `0.3 core-hours`). Time-series charts in these sections use the same hour-based units.
