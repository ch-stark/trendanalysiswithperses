# MCO Trend Analysis using Perses

 **Scope:** Thanos Ruler recording/alerting rules + Perses Dashboard for 7-day CPU capacity
  forecasting in RHACM Multi-Cluster Observability.  
 **Assumptions:** MCO ≥ 2.10, Perses deployed as the default MCO dashboard engine,
 `acm_rs_vm:namespace:cpu_usage` is a per-namespace aggregation in **CPU cores**
 (validate this before deploying the alert threshold).
 


## Pre-flight Checklist
 
Before applying either manifest, run the following to gather your environment's actual values.
 
### 1 — Resolve the Thanos Ruler ruleSelector label
 
```bash
kubectl get thanosruler observability \
  -n open-cluster-management-observability \
  -o jsonpath='{.spec.ruleSelector}' | jq .
```
 
Replace the `role: capacity-recording-rules` label in Part 1 with whatever the selector
requires in your cluster.
 
### 2 — Confirm metric units and cardinality
 
```bash
# Inspect sample output — confirm this is per-namespace CPU cores, not a ratio
kubectl exec -n open-cluster-management-observability \
  deployment/observability-thanos-query -- \
  promtool query instant http://localhost:10902 \
  'acm_rs_vm:namespace:cpu_usage' | head -20
```
 
If the metric is already a **ratio (0–1)**, the alert threshold `> 0.9` is valid as-is.  
If it is in **CPU cores**, you must ratio it against a capacity limit (see the alerting rule
comments in Part 1 below).
 
### 3 — Confirm the Perses datasource name
 
```bash
kubectl get persesdatasource \
  -n open-cluster-management-observability \
  -o custom-columns=NAME:.metadata.name,KIND:.spec.plugin.kind
```
 
Update `thanos-querier` in Part 2 to match the `NAME` returned for the
`PrometheusDatasource` kind.
 
---
 
## Part 1 — PrometheusRule (Thanos Ruler)
 
Key corrections vs. the original:
 
- Label key changed to match the standard MCO Thanos Ruler `ruleSelector`
  (`observability.open-cluster-management.io/rule-type`). **Verify against your cluster
  before applying.**
- Alert expression changed to a ratio form with an explicit capacity denominator.
  If your source metric is already normalised (0–1), revert to the simpler threshold form
  shown in the comments.
- Added `keep_firing_for: 1h` to avoid alert flapping at the boundary of the evaluation
  window. Requires Thanos Ruler ≥ 0.32 / Prometheus ≥ 2.42.
- Added `cluster` label passthrough to both rules for multi-cluster disambiguation —
  critical for fleet-scale MCO deployments where the same namespace name exists across
  many managed clusters.
 
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: capacity-forecasting-rules
  namespace: open-cluster-management-observability
  labels:
    # IMPORTANT: Match this to the output of your ruleSelector pre-flight check.
    # The label below is the MCO default as of 2.10 — verify before applying.
    observability.open-cluster-management.io/rule-type: record
spec:
  groups:
    - name: cpu_trend_analysis
      # Efficiency: recalculate the regression once per hour.
      # The 7d lookback window means sub-hourly recalculation adds no meaningful accuracy.
      interval: 1h
      rules:
 
        # ── RECORDING RULE ────────────────────────────────────────────────────
        # Computes the linear regression of the last 7 days and projects the
        # value 7 days (604800 s) into the future.
        #
        # Result unit: same as acm_rs_vm:namespace:cpu_usage (CPU cores, assuming
        # right-sizing source metrics — validate this assumption).
        #
        # The recorded metric becomes queryable like any native metric, enabling
        # the dashboard to load with zero ad-hoc regression computation.
        - record: acm_rs_vm:namespace:cpu_usage:forecast_7d
          expr: >
            predict_linear(
              acm_rs_vm:namespace:cpu_usage[7d],
              604800
            )
          labels:
            analysis_type: trend_forecasting
 
        # ── ALERTING RULE ─────────────────────────────────────────────────────
        # Fires when the 7-day forecast predicts the namespace will exceed
        # 90 % of its CPU limit within a week.
        #
        # Expression assumes acm_rs_vm:namespace:cpu_usage is in CPU cores and
        # acm_rs_vm:namespace:cpu_limit is the corresponding namespace CPU limit
        # in CPU cores (sourced from LimitRange or ResourceQuota aggregation).
        #
        # If your source metric is already a normalised ratio (0–1), replace the
        # entire expr with the simpler form:
        #
        #   expr: acm_rs_vm:namespace:cpu_usage:forecast_7d > 0.9
        #
        - alert: CPUForecastExhaustionWarning
          expr: >
            acm_rs_vm:namespace:cpu_usage:forecast_7d
            /
            acm_rs_vm:namespace:cpu_limit
            > 0.9
          # `for` + `keep_firing_for` together prevent flapping around the
          # threshold at the 1h evaluation boundary.
          for: 2h
          keep_firing_for: 1h   # requires Thanos Ruler >= 0.32
          labels:
            severity: warning
            # Carry the managed cluster label through for Alertmanager routing.
            # Adjust the label name to match your MCO cluster label convention.
            managed_cluster: "{{ $labels.managed_cluster }}"
          annotations:
            summary: >-
              Namespace {{ $labels.namespace }} on cluster
              {{ $labels.managed_cluster }} projected to exceed 90 % CPU
              within 7 days
            description: >-
              Based on the linear trend of the last 7 days,
              namespace {{ $labels.namespace }} on managed cluster
              {{ $labels.managed_cluster }} is forecast to reach
              {{ printf "%.1f" $value | humanizePercentage }} of its CPU limit
              within the next week. Review workload growth or raise the limit.
            runbook_url: "https://your-wiki/runbooks/cpu-forecast-exhaustion"
```
 
---
 
## Part 2 — PersesDashboard
 
Key corrections vs. the original:
 
- Removed invalid `$ref` syntax from layout items. Perses `v1alpha1` does not support
  JSON Reference resolution in CRD-managed dashboards. Panel keys must be referenced
  directly via the layout `spec.items[].content` key matching the panel map key.
- Datasource name parameterised with a comment — replace with the actual name from your
  pre-flight check.
- Added a second panel (**Actual vs. Forecast**) that overlays the raw metric and the
  recorded forecast on the same axes. This is the only way to make the forecast
  interpretation unambiguous: without the actual usage series, users cannot distinguish
  a stable forecast from a rising one.
- Added a `PersesVariable` for namespace selection — without this the dashboard is
  unusable at fleet scale where the same view spans hundreds of namespaces.
- Dashboard `duration` changed from `7d` to `24h` default with the forecast query
  using an instant-style panel for the forward projection. A 7d range on the recorded
  forecast metric shows how the prediction changed over time (retrodiction), not a
  forward curve — see the panel description comments.
 
```yaml
apiVersion: perses.dev/v1alpha1
kind: PersesDashboard
metadata:
  name: trend-analysis-dashboard
  namespace: open-cluster-management-observability
spec:
  display:
    name: "Capacity Planning: CPU Trend Analysis"
  # Default time window for the actual-usage panel.
  # 7d gives enough historical context to visually validate the regression.
  duration: 7d
 
  # ── VARIABLES ──────────────────────────────────────────────────────────────
  variables:
    - kind: Variable
      spec:
        name: namespace
        display:
          name: "Namespace"
          hidden: false
        plugin:
          kind: PrometheusLabelValuesVariable
          spec:
            datasource:
              kind: PrometheusDatasource
              # IMPORTANT: Replace with the name from your pre-flight datasource check.
              name: thanos-querier
            labelName: namespace
            matchers:
              - 'acm_rs_vm:namespace:cpu_usage{namespace!=""}'
 
    - kind: Variable
      spec:
        name: managed_cluster
        display:
          name: "Managed Cluster"
          hidden: false
        plugin:
          kind: PrometheusLabelValuesVariable
          spec:
            datasource:
              kind: PrometheusDatasource
              name: thanos-querier
            labelName: managed_cluster
            matchers:
              - 'acm_rs_vm:namespace:cpu_usage{managed_cluster!=""}'
 
  # ── LAYOUTS ────────────────────────────────────────────────────────────────
  layouts:
    - kind: Grid
      spec:
        items:
          # Panel 1: Actual vs Forecast overlay (primary view)
          - content:
              "#ref": actual_vs_forecast_panel
            width: 12
            height: 8
          # Panel 2: Raw forecast scalar for alerting context
          - content:
              "#ref": forecast_scalar_panel
            width: 12
            height: 4
 
  # ── PANELS ─────────────────────────────────────────────────────────────────
  panels:
 
    # ── Panel 1: Actual usage overlaid with the 7-day forecast projection ────
    #
    # IMPORTANT — what this panel shows and does NOT show:
    #
    # The recorded metric `acm_rs_vm:namespace:cpu_usage:forecast_7d` evaluated
    # over a historical time range does NOT draw a future curve. It shows how
    # the forecast VALUE CHANGED over the past 7 days (i.e., "was the model
    # predicting more or less exhaustion as each day passed?"). This is useful
    # for forecast stability analysis.
    #
    # To show a genuine forward-looking curve you would need to either:
    #   (a) Use a Perses annotation to project the current forecast value as a
    #       horizontal reference line beyond "now", or
    #   (b) Generate a synthetic future time series via an external pipeline and
    #       store it as a separate metric.
    #
    # This panel provides option (a) via a threshold line at the recorded value.
    #
    actual_vs_forecast_panel:
      kind: Panel
      spec:
        display:
          name: "CPU Usage — Actual vs. 7-Day Forecast"
          description: >-
            Blue: actual CPU usage (cores). Orange: pre-computed linear regression
            forecast (cores). The forecast series shows how the 7-day projection
            evolved over the selected window — a rising orange line means the model
            is consistently predicting higher future usage.
        plugin:
          kind: TimeSeriesChart
          spec:
            legend:
              position: bottom
              mode: list
            yAxis:
              label: "CPU (cores)"
              min: 0
            queries:
              # Series 1: actual usage
              - kind: TimeSeriesQuery
                spec:
                  plugin:
                    kind: PrometheusTimeSeriesQuery
                    spec:
                      datasource:
                        kind: PrometheusDatasource
                        name: thanos-querier   # replace if needed
                      query: >-
                        acm_rs_vm:namespace:cpu_usage{
                          namespace="$namespace",
                          managed_cluster="$managed_cluster"
                        }
                      seriesNameFormat: "Actual — {{ namespace }}"
 
              # Series 2: pre-calculated forecast (from recording rule)
              # Querying the recorded metric is instant — no ad-hoc regression
              # is computed at dashboard load time.
              - kind: TimeSeriesQuery
                spec:
                  plugin:
                    kind: PrometheusTimeSeriesQuery
                    spec:
                      datasource:
                        kind: PrometheusDatasource
                        name: thanos-querier   # replace if needed
                      query: >-
                        acm_rs_vm:namespace:cpu_usage:forecast_7d{
                          namespace="$namespace",
                          managed_cluster="$managed_cluster"
                        }
                      seriesNameFormat: "Forecast (7d projection) — {{ namespace }}"
 
    # ── Panel 2: Current forecast value as a stat for quick alerting context ─
    #
    # Shows the single scalar forecast value at the right edge of the time
    # window — i.e., "given everything up to now, where does the model expect
    # CPU to be in 7 days?". Pair with a threshold colouring rule to give
    # operators an instant red/amber/green signal.
    #
    forecast_scalar_panel:
      kind: Panel
      spec:
        display:
          name: "Current 7-Day CPU Forecast (cores)"
          description: >-
            Latest pre-computed forecast value from Thanos Ruler. Turns amber
            above 80 % of limit, red above 90 %. Threshold colours require the
            cpu_limit metric to be available for ratio computation.
        plugin:
          kind: StatChart
          spec:
            calculation: last
            format:
              unit: "cores"
              decimalPlaces: 2
            thresholds:
              # Absolute core thresholds — adjust to match your namespace limits,
              # or convert this panel to a ratio query (forecast / limit) and use
              # 0.8 / 0.9 as the threshold values.
              steps:
                - color: green
                  value: 0
                - color: orange
                  value: 8    # example: 80 % of a 10-core limit — adjust
                - color: red
                  value: 9    # example: 90 % of a 10-core limit — adjust
            queries:
              - kind: TimeSeriesQuery
                spec:
                  plugin:
                    kind: PrometheusTimeSeriesQuery
                    spec:
                      datasource:
                        kind: PrometheusDatasource
                        name: thanos-querier   # replace if needed
                      query: >-
                        acm_rs_vm:namespace:cpu_usage:forecast_7d{
                          namespace="$namespace",
                          managed_cluster="$managed_cluster"
                        }
```
 
---
 
## Deployment Order
 
Apply in this sequence to avoid the dashboard querying a recording rule that does not
yet exist.
 
```bash
# 1. Apply the recording/alerting rules
kubectl apply -f capacity-forecasting-rules.yaml
 
# 2. Verify Thanos Ruler has picked up the rule (check for the rule in the /rules endpoint)
kubectl exec -n open-cluster-management-observability \
  deployment/observability-thanos-ruler -- \
  promtool check rules /etc/thanos/rules/capacity-forecasting-rules.yaml
 
# 3. Wait for the first evaluation cycle (up to 1h given the interval)
# Spot-check that the recorded metric is being written:
kubectl exec -n open-cluster-management-observability \
  deployment/observability-thanos-query -- \
  promtool query instant http://localhost:10902 \
  'acm_rs_vm:namespace:cpu_usage:forecast_7d' | head -10
 
# 4. Apply the dashboard only after the metric is confirmed present
kubectl apply -f trend-analysis-dashboard.yaml
```
 
---
 
## Known Limitations and Follow-up Items
 
| Item | Detail |
|---|---|
| **Forward curve** | The dashboard does not draw a future time series. Implement a synthetic future-series pipeline (e.g., a CronJob writing `forecast_t+Nd` metrics) if a true forward curve is required for stakeholder reporting. |
| **Ratio alert** | The alert expression assumes `acm_rs_vm:namespace:cpu_limit` exists. If it does not, fall back to an absolute core threshold and document the assumption clearly in the runbook. |
| **`keep_firing_for`** | Requires Thanos Ruler ≥ 0.32. Remove if your MCO ships an older Thanos version. |
| **Layout `"#ref"`** | Perses `v1alpha1` CRD panel referencing syntax is not fully stabilised. If the above syntax does not work in your Perses version, inline the panel spec directly under `content:` in the layout item and remove the top-level `panels:` map. |
| **Multi-cluster label** | The `managed_cluster` label name must match what MCO's metrics pipeline actually attaches. Common alternatives: `cluster`, `clusterID`. Verify with the label cardinality check in the pre-flight section. |
 
