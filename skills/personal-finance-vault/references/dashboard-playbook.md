# Optional local HTML dashboard

Use when considering or building a dashboard for an existing private vault.
The dashboard is a presentation of validated data, not a new source of truth.

## Readiness and user choice

Assess readiness per proposed panel, not by a fixed number of rows:

- A dated, validated snapshot can support an inventory or snapshot overview.
- Trend or period-comparison panels require comparable covered periods with
  consistent definitions. Do not interpolate gaps or imply an annual trend
  from a short sample.
- Cross-source totals require the vault's deduplication views and explicit
  currencies/dated conversions. Block only affected panels when an input is
  missing; do not gate an unrelated spending panel on insurance valuations.
- Identify stale, partial, unknown, and unclassified inputs before offering
  headlines. When no useful panel is supportable, explain what source would
  make one possible rather than building a misleading shell.

When useful data exists, offer a concrete choice, for example:
"There is enough validated data for a spending overview and account snapshot.
Would you like a local HTML dashboard, or keep using the tables for now?"
If the user already requested one, no repeat confirmation is needed. Record
a decline and its scope so routine imports do not trigger repeated offers;
revisit only if requested or materially changed circumstances justify it.

## Build only the accepted scope

Prefer a self-contained HTML file that opens locally, with embedded minimal
data and assets and no network requirement. A multi-file local page is fine
when the user prefers it or size warrants it; document how to open it. Do
not add a framework, server, deployment account, or hosting requirement by
default. Generating a dashboard does not authorize publishing or uploading.

Pick panels from the user's question and available evidence: cash flow,
spending, assets/liabilities, investment performance, coverage inventory, or
data freshness. Do not manufacture every panel just because the template
allows it. Label coverage amounts separately from asset values, and separate
spending from transfers, advances, settlements, and unresolved outflow.

Generate a minimal presentation dataset reproducibly from the read-only
query layer and documented derived outputs. Do not hand-join overlapping
source CSVs in JavaScript or hardcode figures in the HTML. Keep the renderer
and rebuild command in the private vault, with generation time, source as-of
dates, covered periods, currency/units, and definitions visible in the page.
Expose the query/view or evidence reference behind each metric without
embedding raw documents. Refresh only after a successful data rebuild; if
refresh fails, identify the last-good artifact as stale rather than current.

Show null as unavailable, never zero. Keep observed, derived, and assumed
values distinguishable. Display partial coverage and reconciliation warnings
near affected metrics, not only in a footer. Describe a sum as partial when
unvalued components are excluded.

## Privacy and validation

- An HTML file with embedded financial data is sensitive even offline.
  Include only fields the page needs; exclude full identifiers, raw documents,
  credentials, receipt access tokens, and unnecessary personal details.
- Avoid analytics, external fonts/scripts, and remote data requests by
  default. Escape source strings and safely serialize embedded data so a
  merchant name or note cannot become executable HTML/script.
- Check displayed totals against the authoritative queries, including after
  filtering. Test missing values, empty periods, negative amounts, and mixed
  currencies; do not aggregate incompatible units.
- Open the generated page in a browser when available. Check desktop/narrow
  layouts, readable labels, keyboard controls, filters, empty states, and
  console/network errors. If browser verification is unavailable, say so;
  static checks alone do not prove that interactive behavior works.
- Deliver the local path, supported panels, as-of/coverage limitations, and
  rebuild command. Do not claim it updates live unless an explicitly chosen
  update mechanism exists. Hosting or sharing requires a separate request
  and a privacy review; a local file is the default completion.

Fictional readiness examples: one reconciled month supports a monthly
snapshot, not a year-over-year chart; several comparable months support a
trend with gaps marked; an insurance report without surrender values supports
a coverage inventory but not an insurance-asset total.
