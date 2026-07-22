# 10 — NOC: Prometheus scraping the K3s cluster

Prometheus (10.20.0.12, monitoring-net) collects Kubernetes metrics from the
K3s cluster (k3s-net) across OPNsense. Unlike the usual in-cluster setups,
Prometheus runs **outside** the cluster, which shapes every design choice
below: NodePort exposure instead of ClusterIP discovery, static targets
instead of `kubernetes_sd`, and explicit RBAC with a long-lived token.

## 1. Metric sources

| Source | What it provides | Endpoint | Auth |
|---|---|---|---|
| kube-state-metrics | Cluster *state*: nodes Ready, pods, deployments, PVC… | NodePort :30080 | none |
| kubelet | Node-level runtime metrics | :10250 /metrics (HTTPS) | Bearer token |
| kubelet / cAdvisor | Per-container CPU/RAM *consumption* | :10250 /metrics/cadvisor (HTTPS) | Bearer token |

## 2. kube-state-metrics deployment

Deployed from the upstream standard manifests into `kube-system`. The standard
Service is **headless** (`clusterIP: None`) and cannot be patched into a
NodePort; a second Service is created for external exposure:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kube-state-metrics-ext
  namespace: kube-system
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: kube-state-metrics
  ports:
    - name: http-metrics
      port: 8080
      targetPort: http-metrics
      nodePort: 30080
```

Two doors, one pod: both Services select the same singleton. **Scrape it
through one door only** — configuring several NodePort targets duplicates
every series (distinct `instance` label) and doubles all aggregates. One
target; availability is an alerting concern, not a scraping one.

## 3. RBAC — Prometheus's identity in the cluster

Kubelets refuse anonymous scrapes. Prometheus is given a minimal read-only
identity:

```
ServiceAccount  kube-system/prometheus
ClusterRole     prometheus-scrape
                ├── nonResourceURLs: ["/metrics"]          verbs: ["get"]
                └── resources: nodes, nodes/metrics,
                    nodes/proxy                            verbs: [get, list, watch]
ClusterRoleBinding prometheus-scrape → the ServiceAccount
Secret          prometheus-token (type service-account-token, long-lived JWT)
```

Since K8s 1.24 ServiceAccounts get no automatic token; the annotated Secret
materializes one. Least privilege: a leaked token reads metrics, nothing else.
Sanity check:

```
kubectl auth can-i get nodes/metrics \
  --as=system:serviceaccount:kube-system:prometheus   # yes
```

Note: `kubectl create clusterrole` cannot mix resource and non-resource rules
in one flag set — the role must be applied as YAML.

## 4. Prometheus configuration

The token is stored root-only on prometheus-srv and referenced by path — never
inlined in the config:

```
/etc/prometheus/k8s-token   (prometheus:prometheus, 0600)
```

```yaml
  - job_name: 'kube-state-metrics'
    static_configs:
      - targets: ['10.10.0.11:30080']

  - job_name: 'kubelet'
    scheme: https
    tls_config: { insecure_skip_verify: true }
    authorization: { credentials_file: /etc/prometheus/k8s-token }
    static_configs:
      - targets: ['10.10.0.11:10250','10.10.0.12:10250','10.10.0.13:10250',
                  '10.10.0.31:10250','10.10.0.32:10250','10.10.0.33:10250']

  - job_name: 'kubelet-cadvisor'
    scheme: https
    metrics_path: /metrics/cadvisor
    tls_config: { insecure_skip_verify: true }
    authorization: { credentials_file: /etc/prometheus/k8s-token }
    static_configs:
      - targets: ['10.10.0.11:10250','10.10.0.12:10250','10.10.0.13:10250',
                  '10.10.0.31:10250','10.10.0.32:10250','10.10.0.33:10250']
```

Static targets are deliberate: six fixed nodes, no in-cluster discovery
available from outside, and the config stays readable. `insecure_skip_verify`
is accepted for the lab (kubelet serves a self-signed cert); the Bearer token
still authenticates the client. Kubelets are six legitimate targets (each node
serves its own metrics); kube-state-metrics is a cluster-wide singleton — one
target regardless of how many doors exist.

Network path: monitoring-net → k3s-net on TCP 30080 and 10250, allowed through
OPNsense.

## 5. A documented pitfall — do not scrape a singleton through two doors

The first working config listed **two** NodePort targets for
kube-state-metrics (one per node, "for redundancy"). Both point at the same
single pod, so every metric was scraped twice and stored under two different
`instance` labels — doubling every count and every sum:

![kube_node_status_condition returning 12 series instead of 6](img/prometheus-duplicate-series-12.png)

Removing the second target from `static_configs` and reloading is not enough
to see the fix immediately: Prometheus keeps stale series **up to ~5 minutes**
after their last successful scrape, so `count(up{job="kube-state-metrics"})`
still read `2` right after the config was corrected and the service
restarted:

![count(up{...}) still showing 2 right after the fix](img/prometheus-duplicate-still-stale.png)

The reliable check during that window is **Status → Targets** (the active
scrape config), not a query over `up` or any other series (which reflects
history, including the fading-out stale target). Redundancy for a singleton
resource is an alerting concern (watch that `up == 1`), not a "scrape it
twice" concern — never duplicate a cluster-wide singleton across targets.

## 6. Validation

13 targets up — 1 kube-state-metrics, 6 kubelet, 6 cadvisor, plus Prometheus
scraping itself:

![Prometheus targets all up](img/prometheus-k3s-targets-up.png)

Witness queries:

```
kube_node_status_condition{condition="Ready",status="true"}   # 6 series = 1
sum(rate(container_cpu_usage_seconds_total{job="kubelet-cadvisor"}[5m])) by (instance)
```

First visual validation via an imported community dashboard (K3S cluster
monitoring), fed live by the new pipeline — series start exactly at the
timestamp the duplicate-target fix was applied:

![K3s cluster dashboard in Grafana, fed by the new pipeline](img/grafana-k3s-cluster-dashboard.png)

This import serves as a data-validation baseline; the custom dashboard suite
(see `dashboard-suite-design.md`) builds on these sources. Note: this
dashboard's "Total" cluster capacity panels reflect pod requests/limits, not
node allocatable capacity — a distinction the custom Cluster dashboard (P1
tier) makes explicit.

## 6. Files under version control

```
configs/k8s/
├── kube-state-metrics-ext-svc.yaml
├── prometheus-clusterrole.yaml
├── prometheus-token-secret.yaml
└── (upstream ksm manifests referenced by URL, not vendored)
```

The JWT token itself is never committed — only the Secret *definition* that
generates it.
