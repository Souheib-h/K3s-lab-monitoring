# Phase 3 — NOC Configuration (Grafana Datasources)

## Prometheus datasource — ✅ Connected

**Connections → Add new data source → Prometheus**

![Adding a new data source](img/grafana-add-nez-source.png)

```
URL: http://10.20.0.12:9090
Auth: None
```

![Prometheus URL configuration](img/grafana-prometheus-url-add.png)

![Connection settings](img/grafana-connection.png)

Save & test confirmed working immediately.

![Prometheus datasource successfully added](img/grafana-succesfully-added-prometheus.png)

Verified beyond the test button itself by browsing **Drilldown → Metrics**: 319 metrics available from Prometheus's own self-scrape (`go_gc_*`, runtime stats), rendering live graphs over the last 6 hours — confirms the datasource is not just reachable but actively returning queryable time series data.

![Prometheus metrics pulled into Grafana](img/grafna-prometheus-metrics-pulled.png)

## Zabbix datasource — ⏸️ Blocked (unresolved)

**Attempted setup:**

```
URL: http://10.20.0.10/zabbix/api_jsonrpc.php
Plugin: alexanderzobnin-zabbix-app (Zabbix data source by Grafana Labs)
```

### Installation troubleshooting

The plugin install itself required several non-obvious fixes before it would even load:

1. **`grafana-cli` / `grafana cli` broken on this build** — both the deprecated and new CLI syntax either failed with `Could not find config defaults` or launched a full foreground Grafana server instead of running the install command. Worked around by forcing `--homepath "/usr/share/grafana"` explicitly.
2. **Ownership bug** — plugin files installed via `grafana-cli` were owned by `root:root` instead of `grafana:grafana`, silently preventing the `grafana-server` process (running as the `grafana` user) from loading the plugin. Fixed with `chown -R grafana:grafana`.
3. **Plugin registered but not enabled** — even with correct ownership and a valid signature, the plugin doesn't auto-activate. Found under **Administration → Plugins and data → Plugins → Zabbix → Enable**. This step is easy to miss since the plugin appears installed and even runs as a backend process without it.

![Zabbix plugin installed in Grafana](img/grafana-zabbix-plugin-installed.png)

![Enabling the Zabbix plugin](img/grafana-zabbix-enabling.png)

### Connection troubleshooting — root cause not resolved

Once enabled, datasource setup was attempted:

![Zabbix datasource configuration](img/grafana-zabbix-datasourcce-add.png)

**Save & test** consistently returns one of two errors depending on the auth method:

- **User and password auth**: hangs for exactly 60 seconds, then fails with `Zabbix authentication error: context deadline exceeded` — a query timeout, not a credentials issue.
- **API token auth**: fails immediately with `Could not connect to given url`.

**Eliminated as causes** (each independently verified):

| Hypothesis | Test | Result |
|---|---|---|
| Network/firewall between Grafana-srv and Zabbix-srv | `curl -X POST` to the API endpoint from Grafana-srv | 13ms response, valid JSON-RPC result |
| Wrong credentials | Manual login to Zabbix web UI with same `Admin` credentials | Successful |
| Zabbix account locked (brute-force protection) | Direct web login test | Not locked |
| File permission issue on plugin binary | `ls -la` on the datasource binary | Correct `-rwxr-xr-x`, owned by `grafana` |
| Unsigned plugin blocked | Checked plugin signature status in UI | Signed, `grafana` signature confirmed |
| No internet access for plugin version checks | `curl` to `grafana.com/api/plugins/...` from Grafana-srv | 200 OK, fast response |
| Plugin version incompatibility (6.4.0) | Downgraded to plugin v6.2.0 | Same error |
| Grafana version incompatibility (13.1.0) | Downgraded Grafana to 12.4.5 | Same error |

A matching public report was found on the plugin's GitHub repository — [grafana/grafana-zabbix#1957](https://github.com/grafana/grafana-zabbix/issues/1957): *"After updating Grafana from 11.4.1 to 11.5.0 the Zabbix plugin can no longer connect and returns 'Could not connect to given url'"* — confirming this is a known upstream compatibility issue between the plugin and recent Grafana versions, not a misconfiguration on this lab's side.

### Status

Paused after exhausting the standard troubleshooting surface (network, auth, permissions, signature, two plugin versions, two Grafana versions). Grafana restored to 13.1.0 (latest stable) rather than staying on an arbitrary downgrade that didn't resolve the issue.

**Possible paths forward, not yet attempted:**
- Direct DB Connection (bypasses the broken JSON-RPC API path entirely, queries Zabbix's MySQL database directly via a native MySQL datasource)
- YAML provisioning instead of the UI form (may take a different code path)
- File an issue on the plugin repo with this reproduction case
- Wait for an upstream fix and retry

## Dashboard imports

Deferred until the Zabbix connection issue is resolved or worked around — Cluster View and Node View dashboards depend on both datasources being functional.
