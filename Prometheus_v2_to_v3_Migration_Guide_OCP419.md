# OCP 4.19：Prometheus v2 → v3 升級完整遷移指引

> **適用版本**：OpenShift Container Platform 4.19（Prometheus 從 2.55.1 升級至 3.2.1）
> **重要程度**：此升級包含多項**破壞性變更（Breaking Changes）**，升級前必須完成評估與調整。

---

## 一、升級背景

OCP 4.19 將內建的 Prometheus 從 **2.55.1 升級至 3.2.1**，這是 Prometheus 自 2017 年發布 v2 以來最重大的版本躍升。[1] 雖然 OCP 的監控堆疊核心元件（Cluster Monitoring Operator、Prometheus Operator 等）已由 Red Hat 完成必要的相容性調整，但**使用者自行管理的配置**（包含自訂告警規則、記錄規則、儀表板、relabeling 配置、外部 Alertmanager 整合）可能需要手動修改。[1]

以下將所有破壞性變更分類整理，並提供具體的檢查方法與修正範例。

---

## 二、破壞性變更全覽

以下表格彙整所有破壞性變更的影響範圍與緊急程度：

| # | 變更項目 | 影響範圍 | 緊急程度 |
|---|----------|----------|----------|
| 1 | `le` / `quantile` 標籤值正規化為浮點數 | 告警規則、記錄規則、儀表板、relabeling | **高** |
| 2 | Alertmanager v1 API 不再支援 | 外部 Alertmanager 整合 | **高** |
| 3 | 正規表達式 `.` 匹配換行符號 | PromQL 查詢、relabel configs | **中** |
| 4 | Range selector 左邊界改為開區間 | subquery、rate/increase 計算 | **中** |
| 5 | `holt_winters` 函式更名 | 使用預測函式的告警/記錄規則 | **中** |
| 6 | Scrape 協定嚴格驗證 Content-Type | 自訂 exporter、第三方 exporter | **中** |
| 7 | Remote Write HTTP/2 預設關閉 | Remote Write 配置 | **低** |
| 8 | TSDB 格式變更（降版限制） | 降版操作 | **低** |
| 9 | UTF-8 指標名稱支援 | 指標名稱驗證 | **低** |
| 10 | 日誌格式變更 | 日誌解析自動化腳本 | **低** |

---

## 三、高優先級變更（必須處理）

### 3.1 `le` / `quantile` 標籤值正規化為浮點數

**這是影響最廣泛的破壞性變更。** 在 Prometheus v3 中，classic histogram 的 `le` 標籤與 summary 的 `quantile` 標籤值，在資料攝取時會被強制正規化為浮點數表示。[1] [2]

**變更前後對比：**

| 原始指標暴露值 | Prometheus v2 儲存值 | Prometheus v3 儲存值 |
|---------------|---------------------|---------------------|
| `http_request_duration_bucket{le="1"}` | `le="1"` | `le="1.0"` |
| `http_request_duration_bucket{le="10"}` | `le="10"` | `le="10.0"` |
| `http_request_duration_bucket{le="100"}` | `le="100"` | `le="100.0"` |
| `rpc_duration_summary{quantile="0.5"}` | `quantile="0.5"` | `quantile="0.5"` ✓ |
| `rpc_duration_summary{quantile="1"}` | `quantile="1"` | `quantile="1.0"` |

> **注意**：已是浮點數格式的值（如 `0.5`、`0.99`）不受影響，只有整數格式（如 `1`、`10`、`100`）才會被轉換。[2]

**受影響的配置類型：**

以下類型的配置若使用整數格式的 `le` 或 `quantile` 值，升級後將**靜默失效**（不報錯，但查詢返回空結果或錯誤數值）：

- **PrometheusRule 告警規則**（`PrometheusRule` CR 中的 `expr` 欄位）
- **PrometheusRule 記錄規則**（`record` 欄位的 `expr`）
- **Grafana / 主控台儀表板**中的 PromQL 查詢
- **metric_relabel_configs** 中的 `source_labels` 選擇器

**修正方案 A（推薦）：直接更新為浮點數格式**

適用於升級後只查詢新資料的情境：

```yaml
# 修改前（v2）
- alert: HighLatency
  expr: histogram_quantile(0.99, rate(http_request_duration_bucket{le="1"}[5m])) > 0.5

# 修改後（v3）
- alert: HighLatency
  expr: histogram_quantile(0.99, rate(http_request_duration_bucket{le="1.0"}[5m])) > 0.5
```

**修正方案 B（跨版本過渡期）：使用正規表達式同時匹配兩種格式**

適用於升級後仍需查詢升級前歷史資料的情境（建議保留至少一個資料保留週期後再切換至方案 A）：

```yaml
# 使用正規表達式同時匹配 "10" 和 "10.0"
expr: histogram_quantile(0.99, rate(http_request_duration_bucket{le=~"10(.0)?"}[5m])) > 0.5

# 更通用的寫法（匹配任何整數或其浮點表示）
expr: rate(http_request_duration_bucket{le=~"1(\.0)?"}[5m])
```

**修正方案 C（緊急回退）：使用 metric_relabel_configs 保留舊格式**

若無法立即修改所有規則，可在 scrape 配置中使用 relabeling 將浮點數還原為整數格式。**此方案僅作為臨時措施，不建議長期使用。** [2]

```yaml
# 在 ServiceMonitor 或 PodMonitor 的 metricRelabelings 中加入：
metricRelabelings:
  - sourceLabels: [quantile]
    targetLabel: quantile
    regex: '(\d+)\.0+'
    replacement: '$1'
  - sourceLabels: [le, __name__]
    targetLabel: le
    regex: '(\d+)\.0+;.*_bucket'
    replacement: '$1'
```

**如何快速找出受影響的規則：**

OCP 4.19 新增了 `PrometheusPossibleNarrowSelectors` 告警，當 PromQL 查詢或 metric relabel 配置使用了可能過於嚴格的選擇器（即整數格式的 `le`/`quantile` 值）時，會主動觸發告警。[1] 升級後應立即檢查此告警是否觸發。

此外，可在升級前執行以下搜尋，找出所有潛在受影響的規則：

```bash
# 搜尋所有 PrometheusRule 中使用整數 le 值的規則
oc get prometheusrule -A -o yaml | grep -E 'le="[0-9]+"' | grep -v '\.'

# 搜尋所有 PrometheusRule 中使用整數 quantile 值的規則
oc get prometheusrule -A -o yaml | grep -E 'quantile="[0-9]+"' | grep -v '\.'

# 搜尋all namespace中的潛在問題的Promethus rule
oc get prometheusrule -A -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name" | tail -n +2 | while read ns name; do
  if oc get prometheusrule "$name" -n "$ns" -o yaml | grep -E 'le="[0-9]+"' | grep -q -v '\.'; then
    echo "⚠️ 發現需要檢查的規則 -> Namespace: $ns | Name: $name"
  fi
done

```

---

### 3.2 Alertmanager v1 API 不再支援

Prometheus v3 完全移除了對 Alertmanager v1 API 的支援，要求使用 **Alertmanager 0.16.0 或更新版本**的 v2 API。[1] [2]

**受影響的配置：** 若在 `cluster-monitoring-config` 或 `user-workload-monitoring-config` ConfigMap 中，透過 `additionalAlertmanagerConfigs` 配置了外部 Alertmanager 實例，且使用的是 v1 API scheme，升級後告警將無法傳送至這些外部實例。

**檢查方式：**

```bash
# 檢查 cluster-monitoring-config
oc get configmap cluster-monitoring-config -n openshift-monitoring -o yaml | grep -A5 "additionalAlertmanagerConfigs"

# 檢查 user-workload-monitoring-config
oc get configmap user-workload-monitoring-config -n openshift-user-workload-monitoring -o yaml | grep -A5 "additionalAlertmanagerConfigs"
```

**修正方式：**

```yaml
# 修改前（v1 API，不再支援）
additionalAlertmanagerConfigs:
  - scheme: http
    api_version: v1          # ← 需要移除或改為 v2
    static_configs:
      - targets:
          - "external-alertmanager:9093"

# 修改後（v2 API）
additionalAlertmanagerConfigs:
  - scheme: http
    api_version: v2          # ← 改為 v2
    static_configs:
      - targets:
          - "external-alertmanager:9093"
```

若外部 Alertmanager 版本低於 0.16.0，必須先升級 Alertmanager 再進行 OCP 升級。OCP 4.19 內建的 Alertmanager 已升級至 **0.28.1**，支援 v2 API。[1]

---

## 四、中優先級變更（建議處理）

### 4.1 正規表達式 `.` 現在匹配換行符號

在 Prometheus v3 中，PromQL 正規表達式中的 `.` 模式現在會匹配換行符號（`\n`）。[2] 這影響所有 PromQL 查詢和 relabel 配置中使用正規表達式的地方。

**受影響範例：**

```promql
# 在 v2 中，以下查詢不會匹配包含換行符號的標籤值
# 在 v3 中，.* 會匹配包含 \n 的字串
{job=~".*"}
{instance=~"host.+:8080"}
```

**修正方式：** 若要保持 v2 的行為（不匹配換行符），將 `.` 替換為 `[^\n]`：

```promql
# v2 等效行為
{job=~"[^\n]*"}
{instance=~"[^\n]+:8080"}
```

**實際影響評估：** 大多數生產環境的標籤值不包含換行符，此變更影響通常較小，但若有使用通配正規表達式的 relabeling 規則，應進行審查。

---

### 4.2 Range Selector 與 Lookback 左邊界改為開區間

Prometheus v3 將 range selector 和 lookback 的行為從**左閉右閉（left-closed, right-closed）**改為**左開右閉（left-open, right-closed）**。[2] 這使得行為更加一致，但可能影響某些查詢的結果。

**最常見的受影響場景：subquery**

```promql
# 在 v2 中，以下 subquery 在完美對齊時可能返回 2 個資料點
# 在 v3 中，只返回 1 個資料點，導致 rate() 無法計算
rate(foo[1m:1m])

# 修正：擴大時間窗口以確保至少包含 2 個資料點
rate(foo[2m:1m])
```

**影響評估：** 此變更主要影響 subquery，特別是在 query frontend 對齊查詢時間到步長倍數的情況下。若有使用 subquery 的記錄規則或告警規則，需要驗證其行為。

---

### 4.3 `holt_winters` 函式更名為 `double_exponential_smoothing`

Prometheus v3 將 `holt_winters()` 函式更名為 `double_exponential_smoothing()`，並將其移至 `promql-experimental-functions` feature flag 後方。[2]

**受影響配置：** 任何在告警規則、記錄規則或儀表板中使用 `holt_winters()` 的查詢。

**修正方式：**

```promql
# 修改前（v2）
holt_winters(metric[10m], 0.1, 0.5)

# 修改後（v3）
double_exponential_smoothing(metric[10m], 0.1, 0.5)
```

> **注意**：在 OCP 4.19 中，由於 `promql-experimental-functions` feature flag 的限制，若需使用此函式，需確認 OCP 是否已啟用該 feature flag。建議評估是否有替代的預測方法。

---

### 4.4 Scrape 協定嚴格驗證 Content-Type

Prometheus v3 對 scrape 目標回應的 `Content-Type` header 進行嚴格驗證。若目標未提供有效的 `Content-Type` header，scrape 將**直接失敗**，而非像 v2 一樣回退到預設的文字格式。[2]

**支援的 Content-Type 值：**

| Content-Type | 說明 |
|-------------|------|
| `text/plain;version=0.0.4` | 標準 Prometheus 文字格式 |
| `text/plain;version=1.0.0` | Prometheus 文字格式 v1 |
| `application/openmetrics-text;version=0.0.1` | OpenMetrics 格式 |
| `application/openmetrics-text;version=1.0.0` | OpenMetrics 格式 v1 |
| `application/vnd.google.protobuf;...` | Protobuf 格式 |

**受影響場景：** 自行開發的 exporter、較舊版本的第三方 exporter，或未正確設定 `Content-Type` 的應用程式。

**修正方式 A：更新 exporter** 確保回應包含正確的 `Content-Type` header（推薦）。

**修正方式 B：配置 `fallback_scrape_protocol`** 在 ServiceMonitor 或 PodMonitor 中為特定 scrape job 指定回退協定：

```yaml
# 在 ServiceMonitor 中
spec:
  endpoints:
    - port: metrics
      params: {}
      # 為不提供正確 Content-Type 的目標指定回退協定
      # 注意：此為 Prometheus scrape_config 層級的配置
      # 在 OCP 中需透過 additionalScrapeConfigs 配置
```

```yaml
# 透過 additionalScrapeConfigs 配置（在 cluster-monitoring-config 中）
additionalScrapeConfigs:
  - job_name: 'legacy-exporter'
    fallback_scrape_protocol: 'PrometheusText0.0.4'
    static_configs:
      - targets: ['legacy-app:8080']
```

**快速排查方式：** 升級後若發現某些 scrape target 的 `up` 指標變為 `0`，且 Prometheus 日誌中出現 `non-compliant scrape target sending blank Content-Type` 錯誤，即為此問題。

---

## 五、低優先級變更（了解即可）

### 5.1 Remote Write HTTP/2 預設關閉

Prometheus v3 將 Remote Write 的 `http_config.enable_http2` 預設值從 `true` 改為 `false`，以便多個 Remote Write 佇列能並行使用多個 TCP 連線。[2]

**影響：** 若依賴 HTTP/2 的多路複用特性進行 Remote Write，需明確啟用：

```yaml
remote_write:
  - url: "https://remote-storage:9090/api/v1/write"
    http_config:
      enable_http2: true    # 需明確設定為 true
```

---

### 5.2 TSDB 格式變更與降版限制

Prometheus v3 的 TSDB 格式僅能被 **Prometheus v2.55 或更新版本**讀取。[2] 這意味著升級至 OCP 4.19 後，若需要降版回 OCP 4.18，必須確保 OCP 4.18 使用的是 Prometheus 2.55.1（OCP 4.18 確實使用此版本）。

**實際影響：** 在 OCP 環境中，Prometheus 資料由平台管理，一般不需要手動處理 TSDB 降版問題。但若有自訂的資料備份/還原流程，需注意此限制。

---

### 5.3 UTF-8 指標名稱支援

Prometheus v3 原生支援指標名稱和標籤名稱中的 UTF-8 字元。[2] 這意味著原本被視為無效的指標名稱現在可能被接受。

**OCP 4.19 的特殊處理：** Red Hat 在 OCP 4.19 中**主動停用了 UTF-8 指標接受**（OCPBUGS-62429），以確保監控堆疊的穩定性。因此，在 OCP 4.19 中，Prometheus 不接受 UTF-8 指標，此變更對 OCP 使用者的影響被最小化。[3]

若未來需要使用 UTF-8 指標名稱，可在 Prometheus 配置中指定驗證方案：

```yaml
global:
  metric_name_validation_scheme: utf8   # 或 legacy（預設）
```

---

### 5.4 日誌格式變更

Prometheus v3 從 `go-kit/log` 切換至 `log/slog`，日誌格式發生變化。[2]

```
# v2 格式
ts=2024-10-23T22:01:06.074Z caller=main.go:627 level=info msg="Starting..."

# v3 格式
time=2024-10-24T00:03:07.542+02:00 level=INFO source=main.go:640 msg="Starting..."
```

**影響：** 若有自動化腳本解析 Prometheus 日誌（如日誌告警、日誌分析），需更新解析邏輯。

---

## 六、升級前完整檢查清單

以下是建議在升級至 OCP 4.19 前執行的完整檢查步驟：

### 步驟 1：掃描所有 PrometheusRule 中的整數 `le` / `quantile` 值

```bash
# 找出所有命名空間中的 PrometheusRule
oc get prometheusrule -A -o json | \
  jq -r '.items[] | .metadata.namespace + "/" + .metadata.name + ": " + 
  ([.spec.groups[].rules[].expr // "" | 
    select(test("le=\"[0-9]+\"") or test("quantile=\"[0-9]+\""))] | join(", "))' | \
  grep -v ": $"
```

### 步驟 2：檢查外部 Alertmanager 配置

```bash
# 檢查是否使用 v1 API
oc get configmap cluster-monitoring-config -n openshift-monitoring -o yaml 2>/dev/null | \
  grep -A 20 "additionalAlertmanagerConfigs"

oc get configmap user-workload-monitoring-config -n openshift-user-workload-monitoring -o yaml 2>/dev/null | \
  grep -A 20 "additionalAlertmanagerConfigs"
```

### 步驟 3：確認外部 Alertmanager 版本

若有外部 Alertmanager，確認其版本 ≥ 0.16.0：

```bash
# 在外部 Alertmanager 上執行
alertmanager --version
```

### 步驟 4：搜尋使用 `holt_winters` 的規則

```bash
oc get prometheusrule -A -o yaml | grep -n "holt_winters"
```

### 步驟 5：驗證自訂 exporter 的 Content-Type

```bash
# 測試 exporter 的回應 header
curl -I http://<exporter-endpoint>/metrics | grep -i content-type
```

### 步驟 6：審查 relabeling 配置中的正規表達式

重點審查使用 `.`、`.*`、`.+` 的 relabeling 規則，評估是否可能因匹配換行符而產生非預期行為。

---

## 七、升級後驗證步驟

升級至 OCP 4.19 後，建議執行以下驗證：

**立即驗證（升級後 30 分鐘內）：**

1. 檢查 `PrometheusPossibleNarrowSelectors` 告警是否觸發：
   ```bash
   oc get alerts -n openshift-monitoring | grep PrometheusPossibleNarrowSelectors
   ```

2. 確認 Prometheus pod 正常運行：
   ```bash
   oc get pods -n openshift-monitoring | grep prometheus
   oc get pods -n openshift-user-workload-monitoring | grep prometheus
   ```

3. 檢查 Prometheus 日誌中是否有 scrape 錯誤：
   ```bash
   oc logs -n openshift-monitoring -l app.kubernetes.io/name=prometheus --tail=100 | grep -i "error\|failed\|content-type"
   ```

**短期驗證（升級後 24 小時內）：**

4. 在 Grafana 或 OCP 主控台中，逐一驗證關鍵儀表板的 histogram 面板是否正常顯示資料。
5. 確認所有告警規則的 `ALERTS` 指標正常計算（無意外的空結果）。
6. 驗證外部 Alertmanager 是否正常接收測試告警。

---

## 八、OCP 4.19 特有的新監控功能

除了遷移注意事項外，Prometheus v3 升級也帶來以下新能力可供利用：

| 新功能 | 說明 |
|--------|------|
| **Remote Write 2.0** | 更高效的遠端寫入，改善資料流的持久性與效率 |
| **Native Histogram 支援增強** | 支援亂序攝取（out-of-order ingestion），適合邊緣部署 |
| **自動記憶體限制（GOMEMLIMIT）** | 自動根據容器記憶體限制設定 Go 記憶體上限，減少 OOM 風險 |
| **自動 CPU 配額（GOMAXPROCS）** | 自動根據容器 CPU 配額設定 Go 執行緒數 |
| **Metrics Collection Profiles（GA）** | 選擇預設或最小化指標收集量，降低資源消耗 |
| **外部 Alertmanager 代理支援** | 外部 Alertmanager 現在使用叢集範圍的 HTTP 代理設定 |

---

## 參考資料

[1]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/release_notes/ocp-4-19-release-notes "OpenShift Container Platform 4.19 Release Notes - Red Hat Documentation"

[2]: https://prometheus.io/docs/prometheus/latest/migration/ "Prometheus 3.0 Migration Guide - Prometheus Official Documentation"

[3]: https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.19/html-single/release_notes_for_openshift_monitoring/index "Release Notes for OpenShift Monitoring Stack 4.19 - Red Hat Documentation"
