# OCP 4.18 → 4.19 升級完整操作手冊

> **適用版本**：Red Hat OpenShift Container Platform 4.18.x → 4.19.x
> **文件性質**：升級前準備（Pre-upgrade）+ 升級中監控 + 升級後驗證（Post-upgrade Health Check）
> **重要提醒**：本次升級包含多項**強制性前置作業**，若未完成將導致升級被系統自動封鎖。

---

## 一、升級路徑確認

在執行任何升級動作之前，必須先確認升級路徑的可行性。Red Hat 的 OpenShift Update Service（OSUS）會根據叢集特性提供建議的升級路徑，並可能因為新發現的問題而動態調整。[1]

**確認升級路徑的方式：**

```bash
# 查看目前叢集版本與可用升級路徑
oc adm upgrade

# 查看建議的升級路徑（4.19.x 中的最新穩定版）
oc adm upgrade recommend
```

亦可使用 [Red Hat OpenShift Update Path 工具](https://access.redhat.com/labs/ocpupgradegraph/update_path) 線上確認從目前版本到 4.19 的完整升級路徑。若升級路徑顯示為 Conditional Update（條件式更新），應先評估風險後再決定是否繼續。[2]

---

## 二、升級前強制性前置作業（4.18 → 4.19 特有）

以下四項是 4.18 升級至 4.19 的**強制性前置作業**，任何一項未完成都會導致升級被封鎖。

### 2.1 【必做】管理員確認 Kubernetes API 移除（Admin Acknowledgment）

OCP 4.19 採用 Kubernetes 1.32，移除了以下已棄用的 API。[1] [3] 叢集從 4.18.13 起要求管理員必須手動確認已完成評估，才能啟動升級。

| 資源類型 | 移除的 API 版本 | 遷移目標 | 有重大變更？ |
|----------|----------------|----------|-------------|
| `FlowSchema` | `flowcontrol.apiserver.k8s.io/v1beta3` | `flowcontrol.apiserver.k8s.io/v1` | 否 |
| `PriorityLevelConfiguration` | `flowcontrol.apiserver.k8s.io/v1beta3` | `flowcontrol.apiserver.k8s.io/v1` | 是 |

**步驟 1：檢查叢集是否仍在使用被移除的 API**

```bash
# 列出所有將在 Kubernetes 1.32 被移除且仍在使用的 API
oc get apirequestcounts -o jsonpath='{range .items[?(@.status.removedInRelease!="")]}{.status.removedInRelease}{"\t"}{.metadata.name}{"\n"}{end}'

# 查看哪些工作負載正在使用被移除的 API
oc get apirequestcounts flowschemas.v1beta3.flowcontrol.apiserver.k8s.io \
  -o jsonpath='{range .status.currentHour..byUser[*]}{..byVerb[*].verb}{","}{.username}{","}{.userAgent}{"\n"}{end}' \
  | sort -k 2 -t, -u | column -t -s, -NVERBS,USERNAME,USERAGENT

# 同樣檢查 PriorityLevelConfiguration
oc get apirequestcounts prioritylevelconfigurations.v1beta3.flowcontrol.apiserver.k8s.io \
  -o jsonpath='{range .status.currentHour..byUser[*]}{..byVerb[*].verb}{","}{.username}{","}{.userAgent}{"\n"}{end}' \
  | sort -k 2 -t, -u | column -t -s, -NVERBS,USERNAME,USERAGENT
```

> **注意**：`system:serviceaccount:kube-system:generic-garbage-collector`、`system:kube-controller-manager` 等系統帳號出現在結果中可以安全忽略。

**步驟 2：完成 API 遷移後，提交管理員確認**

```bash
oc -n openshift-config patch cm admin-acks \
  --patch '{"data":{"ack-4.18-kube-1.32-api-removals-in-4.19":"true"}}' \
  --type=merge
```

---

### 2.2 【必做】確認叢集使用 cgroup v2

cgroup v1 在 OCP 4.19 中已**完全移除**，若叢集仍使用 cgroup v1，升級將被以下訊息封鎖：[4]

```
clusteroperator/machine-config is not upgradeable because Cluster is using deprecated cgroup v1,
which is removed in 4.19. Please update the 'cgroupMode' in the 'cluster' object of
nodes.config.openshift.io resource type to 'v2'.
```

**步驟 1：確認目前 cgroup 版本**

```bash
# 查看目前 cgroup 模式設定
oc get nodes.config.openshift.io cluster -o jsonpath='{.spec.cgroupMode}'

# 在節點上直接確認（應顯示 cgroup2）
oc debug node/<node-name> -- chroot /host stat -fc %T /sys/fs/cgroup
```

**步驟 2：若仍使用 cgroup v1，在 4.18 上切換至 v2**

```bash
oc patch nodes.config.openshift.io cluster \
  --type=merge \
  -p '{"spec":{"cgroupMode":"v2"}}'
```

> **重要**：切換 cgroup 版本會觸發所有節點的滾動重啟，請在維護時間執行，並確認所有工作負載與 cgroup v2 相容。Java 應用程式（特別是 JDK 8 以下版本）可能需要更新。[4]

---

### 2.3 【必做】移除套件式 RHEL 計算節點

OCP 4.19 已完全移除對套件式 RHEL（Package-based RHEL）worker nodes 的支援。[5] 若叢集中存在 RHEL worker nodes，必須在升級前移除。

**步驟 1：確認是否有 RHEL worker nodes**

```bash
# 查看節點的作業系統類型
oc get nodes -o wide
oc get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.osImage}{"\n"}{end}'
```

若輸出中出現 `Red Hat Enterprise Linux` 而非 `Red Hat Enterprise Linux CoreOS`，表示存在 RHEL worker nodes。

**步驟 2：移除 RHEL worker nodes**

```bash
# 將節點標記為不可排程
oc adm cordon <rhel-node-name>

# 驅逐節點上的工作負載
oc adm drain <rhel-node-name> --ignore-daemonsets --delete-emptydir-data

# 從叢集移除節點
oc delete node <rhel-node-name>
```

---

### 2.4 【必做】處理 Gateway API CRD（若有使用）

OCP 4.19 起，Ingress Operator 接管 Gateway API CRD 的生命週期管理。若叢集中已存在舊版 Gateway API CRD，升級時會出現 `Gateway API CRDs have been detected` 訊息，需要進行處理。[5]

**步驟 1：確認是否存在 Gateway API CRD**

```bash
oc get crd | grep -F -e gateway.networking.k8s.io -e gateway.networking.x-k8s.io
```

**步驟 2：備份現有 Gateway API 資源**

```bash
# 備份所有 Gateway API 物件
for resource in gateways httproutes grpcroutes referencegrants gatewayclasses; do
  oc get ${resource} -A -o yaml > backup_${resource}.yaml 2>/dev/null
done
```

**步驟 3：提交 Gateway API 管理確認**

```bash
oc -n openshift-config patch configmap admin-acks \
  --patch '{"data":{"ack-4.18-gateway-api-management-in-4.19":"true"}}' \
  --type=merge
```

---

### 2.5 【建議】Boot Image Management 選擇退出（AWS / GCP 叢集）

OCP 4.19 在 AWS 和 GCP 叢集上預設啟用 Boot Image Management，升級後會自動更新所有 Machine Set 的開機映像。若不希望此行為，需在升級前選擇退出。[5]

```bash
# 查看目前 Boot Image Management 狀態
oc get machineconfigurations.operator.openshift.io cluster -o yaml | grep -A5 managedBootImages

# 選擇退出（設定為空列表）
oc patch machineconfigurations.operator.openshift.io cluster \
  --type=merge \
  -p '{"spec":{"managedBootImages":{"machineManagers":[]}}}'
```

---

## 三、升級前通用健康檢查（Pre-upgrade Health Check）

以下檢查適用於所有 OCP 版本升級，建議在升級前 **24-48 小時**內完成，並在**升級前 1 小時**再次確認。

### 3.1 叢集版本與升級狀態

```bash
# 確認目前叢集版本與升級狀態
oc get clusterversion

# 查看詳細的 Upgradeable 條件
oc get clusterversion -o jsonpath='{range .items[0].status.conditions[*]}{.type}{"\t"}{.status}{"\t"}{.message}{"\n"}{end}'

# 確認沒有進行中的升級
oc adm upgrade
```

**預期結果**：`Upgradeable` 條件應為 `True`；若為 `False`，必須先解決問題才能升級。

---

### 3.2 Cluster Operators 狀態

```bash
# 檢查所有 Cluster Operators 狀態
oc get co

# 找出 Degraded 或 not Available 的 Operator
oc get co -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}Available:{.status.conditions[?(@.type=="Available")].status}{"\t"}Degraded:{.status.conditions[?(@.type=="Degraded")].status}{"\n"}{end}'
```

**預期結果**：所有 Operator 的 `AVAILABLE` 應為 `True`，`DEGRADED` 應為 `False`，`PROGRESSING` 應為 `False`。

---

### 3.3 節點狀態

```bash
# 檢查所有節點狀態
oc get nodes

# 找出非 Ready 狀態的節點
oc get nodes | grep -v " Ready "

# 確認節點資源使用情況
oc adm top nodes

# 批次查看所有節點的資源分配情況
for i in $(oc get nodes -o name | awk -F/ '{print $2}'); do
  echo "==== $i ===="
  oc describe node $i | grep -A6 "Allocated resources:"
  echo
done
```

**預期結果**：所有節點應為 `Ready` 狀態，無 `SchedulingDisabled`。CPU 和記憶體使用率建議低於 80%，以確保升級期間有足夠的緩衝空間。

---

### 3.4 etcd 健康狀態

etcd 是 OCP 控制平面的核心元件，升級前必須確認其健康狀態。[5]

```bash
# 確認 etcd pods 狀態
oc get pods -n openshift-etcd -l app=etcd

# 進入 etcd pod 執行健康檢查
ETCD_POD=$(oc get pods -n openshift-etcd -l app=etcd -o name | head -1)
oc exec -n openshift-etcd $ETCD_POD -- etcdctl endpoint health \
  --cluster \
  --cacert /etc/kubernetes/static-pod-certs/configmaps/etcd-serving-ca/ca-bundle.crt \
  --cert /etc/kubernetes/static-pod-certs/secrets/etcd-all-certs/etcd-peer-$(hostname).crt \
  --key /etc/kubernetes/static-pod-certs/secrets/etcd-all-certs/etcd-peer-$(hostname).key

# 確認 etcd 成員狀態
oc exec -n openshift-etcd $ETCD_POD -- etcdctl member list \
  --cacert /etc/kubernetes/static-pod-certs/configmaps/etcd-serving-ca/ca-bundle.crt \
  --cert /etc/kubernetes/static-pod-certs/secrets/etcd-all-certs/etcd-peer-$(hostname).crt \
  --key /etc/kubernetes/static-pod-certs/secrets/etcd-all-certs/etcd-peer-$(hostname).key
```

**預期結果**：所有 etcd 成員應顯示 `healthy`，且 `isLearner` 應為 `false`。

---

### 3.5 備份 etcd 資料

在升級前備份 etcd 是強烈建議的操作，雖然 OCP 不支援回滾，但 etcd 備份可作為災難恢復的最後手段。[5]

```bash
# 在任一 control plane 節點上執行 etcd 備份
oc debug node/<control-plane-node> -- chroot /host /usr/local/bin/cluster-backup.sh /home/core/assets/backup

# 確認備份檔案已建立
oc debug node/<control-plane-node> -- chroot /host ls -la /home/core/assets/backup/
```

---

### 3.6 Machine Config Pools 狀態

```bash
# 檢查所有 MachineConfigPool 狀態
oc get mcp

# 確認沒有 MCP 處於 paused 狀態
oc get mcp -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}Paused:{.spec.paused}{"\n"}{end}'
```

**預期結果**：`UPDATED` 應為 `True`，`UPDATING` 和 `DEGRADED` 應為 `False`，`MACHINECOUNT` 應等於 `READYMACHINECOUNT`，`DEGRADEDMACHINECOUNT` 應為 `0`。

---

### 3.7 Pod 狀態

```bash
# 找出所有非 Running/Completed/Succeeded 狀態的 Pod
oc get pods -A | grep -Ev 'Running|Completed|Succeeded'

# 查看有問題的 Pod 詳情
oc get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded

# 確認 openshift-* 命名空間中的 Pod 都正常
oc get pods -n openshift-kube-apiserver
oc get pods -n openshift-kube-controller-manager
oc get pods -n openshift-kube-scheduler
oc get pods -n openshift-machine-config-operator
```

---

### 3.8 PersistentVolume 與 PersistentVolumeClaim

```bash
# 檢查所有 PV 和 PVC 狀態
oc get pv,pvc -A

# 找出非 Bound 狀態的 PV
oc get pv | grep -v Bound

# 找出非 Bound 狀態的 PVC
oc get pvc -A | grep -v Bound

# 找出卡在 Terminating 狀態的 PVC
oc get pvc -A | grep Terminating
```

---

### 3.9 PodDisruptionBudget 檢查

PDB 配置不當可能導致節點無法被 drain，進而阻塞升級流程。[2]

```bash
# 列出所有 PDB
oc get pdb -A

# 找出 ALLOWED DISRUPTIONS 為 0 的 PDB（可能阻塞升級）
oc get pdb -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}AllowedDisruptions:{.status.disruptionsAllowed}{"\n"}{end}' | grep "AllowedDisruptions:0"
```

**重點**：若發現 `ALLOWED DISRUPTIONS` 為 `0` 的 PDB，需評估其是否會阻塞節點 drain。通常需要確保相關工作負載有足夠的副本數。

---

### 3.10 憑證簽署請求（CSR）

```bash
# 確認沒有待處理的 CSR
oc get csr

# 若有 Pending 的 CSR，批准它們
oc get csr -o name | xargs oc adm certificate approve
```

---

### 3.11 告警狀態

```bash
# 查看所有 Critical 和 Warning 告警
oc get alerts -n openshift-monitoring 2>/dev/null || \
  oc exec -n openshift-monitoring $(oc get pods -n openshift-monitoring -l app.kubernetes.io/name=alertmanager -o name | head -1) \
  -- amtool alert query --alertmanager.url=http://localhost:9093

# 查看 Warning 事件
oc get events -A --field-selector type=Warning --sort-by=".lastTimestamp" | tail -30
```

**預期結果**：升級前不應有 Critical 告警。若有 Warning 告警，需評估其是否會影響升級。

---

### 3.12 control-plane 節點標籤確認（4.19 特有）

OCP 4.19 升級時需要 `node-role.kubernetes.io/control-plane` 標籤，若叢集是從較舊版本升級而來，此標籤可能缺失，導致升級期間出現 `machine-config-nodes-crd-cleanup` pod 卡在 `Pending` 狀態。[2]

```bash
# 確認 control plane 節點是否有正確標籤
oc get nodes -l node-role.kubernetes.io/master -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels}{"\n"}{end}' | grep -o "control-plane"

# 若缺少標籤，手動添加
for node in $(oc get nodes -l node-role.kubernetes.io/master -o name); do
  oc label $node node-role.kubernetes.io/control-plane= --overwrite
done
```

---

### 3.13 Prometheus v3 遷移前置檢查（4.19 特有）

由於 OCP 4.19 將 Prometheus 從 v2 升級至 v3，需要在升級前審查監控配置。

```bash
# 掃描所有 PrometheusRule 中使用整數 le 值的規則（升級後可能失效）
oc get prometheusrule -A -o yaml | grep -E 'le="[0-9]+"' | grep -v '\.'

# 掃描所有 PrometheusRule 中使用整數 quantile 值的規則
oc get prometheusrule -A -o yaml | grep -E 'quantile="[0-9]+"' | grep -v '\.'

# 確認外部 Alertmanager 配置是否使用 v1 API
oc get configmap cluster-monitoring-config -n openshift-monitoring -o yaml 2>/dev/null | \
  grep -A 10 "additionalAlertmanagerConfigs"

oc get configmap user-workload-monitoring-config -n openshift-user-workload-monitoring -o yaml 2>/dev/null | \
  grep -A 10 "additionalAlertmanagerConfigs"
```

---

### 3.14 Operator 版本相容性確認

```bash
# 列出所有已安裝的 Operator
oc get csv -A

# 確認 Operator 訂閱的升級頻道
oc get subscription -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.spec.channel}{"\n"}{end}'
```

使用 [Red Hat OpenShift Operator Update Information Checker](https://access.redhat.com/labs/ocpupgradegraph/update_channel) 確認所有已安裝 Operator 與 OCP 4.19 的相容性。特別注意：

- **OpenShift Data Foundation（ODF）**：需先確認 ODF 版本相容性，並在 OCP 升級前確保 ODF 叢集健康。
- **OpenShift Virtualization**：需確認 CNV 版本與 OCP 4.19 相容。
- **第三方 Operator**：需向各廠商確認相容性。

---

## 四、升級前最終確認清單

在執行升級指令前，請逐一確認以下項目：

| 項目 | 指令/方法 | 預期結果 | 狀態 |
|------|-----------|----------|------|
| 管理員 API 確認已提交 | `oc get cm admin-acks -n openshift-config -o yaml` | 包含 `ack-4.18-kube-1.32-api-removals-in-4.19: "true"` | ☐ |
| cgroup v2 已啟用 | `oc get nodes.config.openshift.io cluster -o jsonpath='{.spec.cgroupMode}'` | 輸出 `v2` | ☐ |
| 無 RHEL worker nodes | `oc get nodes -o wide` | 所有節點均為 RHCOS | ☐ |
| Gateway API CRD 已處理（若適用） | `oc get cm admin-acks -n openshift-config -o yaml` | 包含 `ack-4.18-gateway-api-management-in-4.19: "true"` | ☐ |
| 所有節點 Ready | `oc get nodes` | 全部 `Ready` | ☐ |
| 所有 CO Available | `oc get co` | 全部 `True/False/False` | ☐ |
| 無 Degraded MCP | `oc get mcp` | `DEGRADEDMACHINECOUNT` 全為 `0` | ☐ |
| 無 Paused MCP | `oc get mcp -o jsonpath=...` | 全部 `Paused: false` | ☐ |
| etcd 健康 | `etcdctl endpoint health` | 全部 `healthy` | ☐ |
| etcd 已備份 | `cluster-backup.sh` | 備份檔案存在 | ☐ |
| 無 Pending CSR | `oc get csr` | 無 Pending 項目 | ☐ |
| 無阻塞性 PDB | `oc get pdb -A` | 無 `ALLOWED DISRUPTIONS: 0` | ☐ |
| 無 Critical 告警 | Web Console / amtool | 無 Critical 告警 | ☐ |
| 升級路徑已確認 | `oc adm upgrade` | 顯示可用的 4.19.x 版本 | ☐ |
| Prometheus 規則已審查 | 手動掃描 | 無整數 `le`/`quantile` 值 | ☐ |
| control-plane 標籤存在 | `oc get nodes -l node-role.kubernetes.io/control-plane` | 顯示 3 個 master 節點 | ☐ |

---

## 五、執行升級

完成所有前置作業後，可透過 Web Console 或 CLI 執行升級。

### 5.1 透過 CLI 升級

```bash
# 確認可用的升級版本
oc adm upgrade

# 升級至最新穩定的 4.19.x 版本（建議）
oc adm upgrade --to-latest=true

# 或升級至指定版本
oc adm upgrade --to=4.19.35
```

### 5.2 升級進度監控

```bash
# 持續監控升級進度（每 30 秒刷新）
watch -n 30 oc get clusterversion

# 監控 Cluster Operators 升級進度
watch -n 30 oc get co

# 監控節點升級進度
watch -n 30 oc get nodes

# 監控 MachineConfigPool 進度
watch -n 30 oc get mcp
```

升級過程通常分為兩個階段：
1. **控制平面升級**：kube-apiserver、etcd、kube-controller-manager、kube-scheduler 等逐一滾動更新
2. **計算節點升級**：依 MachineConfigPool 的 `maxUnavailable` 設定（預設為 1）逐一更新節點

---

## 六、升級後健康檢查（Post-upgrade Health Check）

升級完成後，應立即執行以下驗證步驟。

### 6.1 確認版本升級成功

```bash
# 確認叢集版本
oc get clusterversion

# 確認所有 CO 已升級至 4.19
oc get co | grep -v "4.19"
```

**預期結果**：`oc get clusterversion` 應顯示 `Completed` 狀態，所有 CO 版本應為 4.19.x。

---

### 6.2 節點狀態驗證

```bash
# 確認所有節點 Ready 且版本已更新
oc get nodes

# 確認節點 Kubernetes 版本
oc get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.kubeletVersion}{"\n"}{end}'
```

---

### 6.3 Cluster Operators 最終狀態

```bash
# 確認所有 CO 狀態正常
oc get co

# 確認沒有 Degraded 的 CO
oc get co | grep -E "True\s+False\s+True|False\s+\S+\s+\S+"
```

---

### 6.4 Prometheus v3 升級後驗證

```bash
# 確認 Prometheus 版本已升級至 3.x
oc get pods -n openshift-monitoring -l app.kubernetes.io/name=prometheus -o jsonpath='{.items[0].spec.containers[0].image}'

# 確認 PrometheusPossibleNarrowSelectors 告警是否觸發
oc exec -n openshift-monitoring \
  $(oc get pods -n openshift-monitoring -l app.kubernetes.io/name=alertmanager -o name | head -1) \
  -- amtool alert query --alertmanager.url=http://localhost:9093 | grep PrometheusPossibleNarrowSelectors

# 確認 Prometheus 日誌無 scrape 錯誤
oc logs -n openshift-monitoring \
  $(oc get pods -n openshift-monitoring -l app.kubernetes.io/name=prometheus -o name | head -1) \
  --tail=100 | grep -i "error\|failed\|content-type"
```

---

### 6.5 關鍵工作負載驗證

```bash
# 確認所有 Pod 正常運行
oc get pods -A | grep -Ev 'Running|Completed|Succeeded'

# 確認 etcd 仍然健康
ETCD_POD=$(oc get pods -n openshift-etcd -l app=etcd -o name | head -1)
oc exec -n openshift-etcd $ETCD_POD -- etcdctl endpoint health \
  --cluster \
  --cacert /etc/kubernetes/static-pod-certs/configmaps/etcd-serving-ca/ca-bundle.crt \
  --cert /etc/kubernetes/static-pod-certs/secrets/etcd-all-certs/etcd-peer-$(hostname).crt \
  --key /etc/kubernetes/static-pod-certs/secrets/etcd-all-certs/etcd-peer-$(hostname).key

# 確認 MachineConfigPool 已完成更新
oc get mcp
```

---

### 6.6 MCO 憑證輪換確認

OCP 4.19 升級後會立即觸發 Machine Config Server（MCS）憑證輪換。[5]

```bash
# 確認 MCS 憑證已更新
oc get configmap machine-config-server-ca -n openshift-machine-config-operator -o yaml | grep -A2 "ca-bundle.crt"

# 確認 MCO 正常運行
oc get pods -n openshift-machine-config-operator
```

> **注意**：若叢集使用 VMware vSphere UPI 或其他不使用 Machine Sets 的平台，需手動輪換 MCS 憑證。參考 [Regenerating CA certificates for the Machine Config Server](https://access.redhat.com/articles/regenerating_cluster_certificates)。

---

### 6.7 Web Console 驗證

登入 OCP Web Console，確認：
- 主控台可正常存取
- 版本號顯示為 4.19.x
- 預設視角已統一（Developer 視角不再預設顯示）
- 無異常告警或錯誤訊息

---

## 七、已知升級後問題與處理方式

| 問題 | 症狀 | 處理方式 |
|------|------|----------|
| IPsec must-gather 資訊不完整 | 從 4.14 升級的叢集，`ipsecConfig` 為空 | 執行 `oc patch networks.operator.openshift.io cluster --type=merge -p '{"spec":{"defaultNetwork":{"ovnKubernetesConfig":{"ipsecConfig":{"mode":"Full"}}}}}'` |
| Prometheus 告警規則失效 | `PrometheusPossibleNarrowSelectors` 告警觸發 | 更新使用整數 `le`/`quantile` 值的 PromQL 查詢 |
| Gateway API 負載均衡器問題 | AWS 私有叢集 LB 卡在 pending | 目前無解決方案，等待 Red Hat 修復 |
| vSAN Files NFS 無法掛載 | PVC 無法掛載 | 確認 VMware ESXi 和 vSAN 版本 ≥ 8.0 P05 |
| TuneD 設定檔降級（SR-IOV 叢集） | 節點重啟後效能降低 | 重啟 TuneD pod |

---

## 參考資料

[1]: https://access.redhat.com/articles/7112216 "Preparing to upgrade to OpenShift Container Platform 4.19 - Red Hat Customer Portal"

[2]: https://access.redhat.com/solutions/7004992 "OpenShift 4 cluster upgrade pre-checks requirements - Red Hat Customer Portal"

[3]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/updating_clusters/preparing-to-update-a-cluster "Chapter 2. Preparing to update a cluster - OCP 4.19 Documentation"

[4]: https://access.redhat.com/solutions/7071862 "Cgroups version 2 on Red Hat OpenShift Container Platform - Red Hat Customer Portal"

[5]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/release_notes/ocp-4-19-release-notes "OpenShift Container Platform 4.19 Release Notes - Red Hat Documentation"
