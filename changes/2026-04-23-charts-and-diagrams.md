# Charts and Diagrams in Chat

**Date:** 2026-04-23

## Target

- [Product: Charts and Diagrams](../product/chat/charts-and-diagrams.md)
- [Architecture: Chat — Message Rendering](../architecture/chat.md#message-rendering)

## Delta

Chart and diagram rendering in chat messages does not exist yet. The following are not implemented:

- SPA: lazy-loaded `react-vega` renderer for fenced code blocks tagged `vega-lite`
- SPA: lazy-loaded `mermaid` renderer for fenced code blocks tagged `mermaid`
- SPA: custom `code` component in `react-markdown` that dispatches `vega-lite` / `mermaid` blocks to the visualization renderers and falls through to the default code block otherwise
- SPA: theme-aware rendering (light / dark) for both Vega-Lite and Mermaid output
- SPA: inline error banner when a chart or diagram fails to parse or render
- SPA: size-limit guard — above the limit, the raw code block is shown with a notice
- Vega-Lite safety configuration: reject external `url` data sources, only allow inline `values` data
- Mermaid safety configuration: `securityLevel: "strict"`, no click handlers, no raw HTML nodes
- `hast-util-sanitize` schema extended to allow the SVG tags and attributes emitted by both renderers

## Acceptance Signal

- An agent response containing a ```mermaid``` block renders the diagram inline in chat.
- An agent response containing a ```vega-lite``` block with inline `values` data renders the chart inline in chat.
- A Vega-Lite spec that references an external `url` data source is refused with an inline error banner.
- An invalid Mermaid or Vega-Lite spec falls back to the raw code block with an inline error banner — the message is never dropped.
- Rendered charts and diagrams follow the active chat theme.
- Visualization libraries load only when a chart or diagram first appears in the viewport.
