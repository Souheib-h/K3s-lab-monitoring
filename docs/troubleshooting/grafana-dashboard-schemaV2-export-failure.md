# Post-mortem — Grafana Dashboard Export Failure (Schema v2)

## Symptom

Dashboard built in Grafana 13.1 UI, exported via "Export as code" with 
"Share dashboard with another instance" toggle enabled. Import into a 
clean Grafana Docker instance (localhost) failed with:

> Dashboard using new layout cannot be imported because the feature is 
> not enabled.

## Root cause

Grafana 13.1's dashboard editor now saves dashboards internally using 
**schema v2** (`elements` + `layout` structure, replacing the classic 
`panels[]` array). This schema requires the `dashboardNewLayouts` 
feature toggle ("Dynamic Dashboards") — **not enabled by default** on 
any standard Grafana instance, including a vanilla `grafana/grafana:latest` 
Docker image.

Confirmed via Grafana GitHub issues that this feature toggle is still 
experimental and carries active bugs as of mid-2026 
(datasource references resetting on reload, provisioned dashboards 
rendering blank after save, broken CSV report rendering). Not something 
to build a public deliverable on top of.

## Fix path taken

1. Pulled the dashboard back via the classic HTTP API 
   (`GET /api/dashboards/uid/<uid>`), which returns the legacy 
   `panels[]` schema regardless of internal storage format.
2. Manually re-added the fields the classic schema (and Grafana.com's 
   upload validator) expect but that a stripped-down export omits: 
   `id: null`, `gnetId: null`, `tags: []`, `links: []`, `__elements: {}`.
3. `__requires` needs a complete list (grafana core + datasource + 
   every panel type used — `stat`, `timeseries`, etc.), not just the 
   datasource entry.
4. Templated datasource UID references (`"uid": "${DS_PROMETHEUS}"`) 
   and added the corresponding `templating.list` entry with 
   `"type": "datasource"`.

## Second symptom — Grafana.com rejects with "Old dashboard JSON format"

Even after templating correctly, Grafana.com's upload validator rejected 
the file with this specific message. Root cause: **missing `id: null` 
field**. The validator appears to use the presence/absence of this key 
as one of its format-version heuristics — omitting it (rather than 
setting it to `null`) triggers the "old format" rejection path, 
somewhat counterintuitively since `null` is normally treated as 
"absent" in most JSON tooling.

Applying the fix (re-adding `id: null` explicitly, along with the other 
missing fields listed above) resolved the rejection — the dashboard was 
accepted and assigned public ID `25537`.

## Lesson

Don't trust "Export as code" from the Grafana 13.x UI at face value for 
external portability — it reflects whatever internal schema the 
instance currently uses (v1 or v2), and the "share with another 
instance" toggle templates datasources correctly but does **not** 
downgrade the schema version itself. For any external-facing export 
(Grafana.com, sharing with a team on an older Grafana version), pull 
via the classic REST API instead and hand-verify the field set above.

## Status

**Resolved.** Dashboard published on Grafana.com as 
[dashboard 25537](https://grafana.com/grafana/dashboards/25537). 
`dashboards/prometheus-self-monitoring.json` kept in the repo, 
templated version, for reference and version tracking alongside any 
future updates.
