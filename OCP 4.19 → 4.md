# OCP 4.19 → 4.20 升級前準備工作 Runbook

> **適用版本**：OpenShift Container Platform 4.19 → 4.20（Kubernetes 1.33）  
> **文件來源**：Red Hat 官方文件（2026-06-25 更新）  
> **重要提示**：本次升級為 **minor version 升級**，具有強制性的管理員確認機制，與 z-stream 升級流程不同，請務必完整執行所有步驟。

---

## 升級前準備總覽

```
階段 A：環境評估（升級前 1-2 週）
  ├─ A1. 確認叢集健康狀態
  ├─ A2. 檢查 Kubernetes 已移除 API 使用情況
  ├─ A3. 評估特定組件影響（MetalLB / vSphere / PerformanceProfile）
  └─ A4. 確認 Operator 相容性

階段 B：遷移與修正（升級前 3-7 天）
  ├─ B1. 遷移已移除的 Kubernetes API
  ├─ B2. MetalLB FRR-K8s 命名空間遷移（如適用）
  ├─ B3. MachineOSConfig 命名修正（如適用）
  └─ B4. Cloud Credential 更新（如適用）

階段 C：升級前最終確認（升級前 1 天）
  ├─ C1. 備份 etcd
  ├─ C2. 暫停 MachineHealthCheck
  ├─ C3. 提供管理員確認（解除升級阻擋）
  └─ C4. 切換升級 Channel

階段 D：執行升級
  └─ D1. 執行升級指令並監控進度
```

---

## 階段 A：環境評估

### A1. 確認叢集整體健康狀態

在啟動任何升級準備工作之前，必須確保叢集處於完全健康的狀態。所有 critical 告警都必須在升級前解決，因為帶著未解決的告警進行升級可能導致升級失敗或叢集不穩定。[1]

**檢查 Cluster Operators 狀態：**

```bash
# 確認所有 Cluster Operators 均為 Available=True, Progressing=False, Degraded=False
oc get clusteroperators

# 確認叢集版本與升級可用性
oc get clusterversion

# 使用新的升級預檢工具（4.20 GA）
oc adm upgrade recommend
```

**確認叢集處於 Upgradable 狀態：**

當一個或多個 Operator 的 `Upgradeable` 條件未回報為 `True` 超過一小時時，`ClusterNotUpgradeable` 告警會被觸發。對於 minor version 升級（4.19 → 4.20），**必須**解決此告警並確保所有 Operator 回報 `Upgradeable=True`，否則升級無法進行。[1]

```bash
# 檢查是否有 Upgradeable=False 的 Operator
oc get clusteroperators -o json | \
  jq -r '.items[] | select(.status.conditions[] | select(.type=="Upgradeable" and .status!="True")) | .metadata.name'
```

**確認 Critical 告警：**

在 Web Console 的 **Administrator** 視角中，前往 **Observe → Alerting**，篩選 Severity=Critical 的告警並逐一解決。

**確認節點資源充足：**

升級過程中節點會逐一重啟（預設 `maxUnavailable=1`），必須確保有足夠的備用節點容量，讓 workloads 能在節點重啟期間臨時轉移。[1]

```bash
# 確認所有節點均為 Ready 狀態
oc get nodes

# 確認所有 MachineConfigPool 均未暫停且處於 Updated 狀態
oc get mcp
```

**確認 PodDisruptionBudget 配置：**

不當的 PDB 配置可能阻止節點排空（drain），進而阻塞升級流程。需確認高可用 workloads 的 PDB 允許至少一個 Pod 暫時離線，且非高可用 workloads 不受 PDB 保護或有其他終止機制。[1]

```bash
# 列出所有 PDB 並檢查 minAvailable 配置
oc get pdb -A
```

---

### A2. 檢查 Kubernetes 已移除 API 使用情況（最重要）

OCP 4.20 移除了 `admissionregistration.k8s.io/v1beta1` 的四個 API。**所有 4.19 叢集在升級前必須完成此評估**，否則升級程序將被阻擋。[1]

已移除的 API 清單如下：

| 資源類型 | 已移除 API | 遷移目標 |
|---|---|---|
| `MutatingWebhookConfiguration` | `admissionregistration.k8s.io/v1beta1` | `admissionregistration.k8s.io/v1` |
| `ValidatingAdmissionPolicy` | `admissionregistration.k8s.io/v1beta1` | `admissionregistration.k8s.io/v1` |
| `ValidatingAdmissionPolicyBinding` | `admissionregistration.k8s.io/v1beta1` | `admissionregistration.k8s.io/v1` |
| `ValidatingWebhookConfiguration` | `admissionregistration.k8s.io/v1beta1` | `admissionregistration.k8s.io/v1` |

**步驟 1：檢查告警**

叢集中有兩個告警會在使用即將移除的 API 時觸發：
- `APIRemovedInNextReleaseInUse`：將在下一個 OCP 版本移除的 API 正在使用中
- `APIRemovedInNextEUSReleaseInUse`：將在下一個 EUS 版本移除的 API 正在使用中

若這些告警正在觸發，必須立即處理。

**步驟 2：使用 APIRequestCount 識別使用中的已移除 API**

```bash
# 列出所有 API 請求計數，重點查看 REMOVEDINRELEASE 欄位
oc get apirequestcounts

# 只顯示有 REMOVEDINRELEASE 標記的 API（更精確）
oc get apirequestcounts \
  -o jsonpath='{range .items[?(@.status.removedInRelease!="")]}{.status.removedInRelease}{"\t"}{.metadata.name}{"\n"}{end}'
```

> **注意**：以下服務帳號出現在結果中可以安全忽略，因為它們會遍歷所有已註冊 API：
> - `system:serviceaccount:kube-system:generic-garbage-collector`
> - `system:serviceaccount:kube-system:namespace-controller`
> - `system:kube-controller-manager`
> - `system:cluster-policy-controller`

**步驟 3：識別使用已移除 API 的具體 Workloads**

```bash
# 以 MutatingWebhookConfiguration 為例，查詢使用 v1beta1 的 workload
oc get apirequestcounts \
  mutatingwebhookconfigurations.v1beta1.admissionregistration.k8s.io \
  -o jsonpath='{range .status.currentHour..byUser[*]}{..byVerb[*].verb}{","}{.username}{","}{.userAgent}{"\n"}{end}' \
  | sort -k 2 -t, -u | column -t -s, -N VERBS,USERNAME,USERAGENT

# 對 ValidatingWebhookConfiguration 執行相同檢查
oc get apirequestcounts \
  validatingwebhookconfigurations.v1beta1.admissionregistration.k8s.io \
  -o jsonpath='{range .status.currentHour..byUser[*]}{..byVerb[*].verb}{","}{.username}{","}{.userAgent}{"\n"}{end}' \
  | sort -k 2 -t, -u | column -t -s, -N VERBS,USERNAME,USERAGENT
```

---

### A3. 評估特定組件影響

**MetalLB 用戶（BGP FRR-K8s 遷移）：**

```bash
# 確認是否有自訂的 FRRConfiguration 在 metallb-system namespace
oc get frrconfigurations.frrk8s.metallb.io -n metallb-system 2>/dev/null
```

若有輸出，需在升級後執行命名空間遷移（詳見階段 B2）。

**VMware vSphere 用戶：**

確認目前使用的 vSphere 版本。若使用 vSphere 7 或 VCF 4，必須計劃遷移至 vSphere 8 Update 1+ 或 VCF 5+，因為 Broadcom 已終止對這些版本的一般支援。[1]

**PerformanceProfile 用戶（RPS 行為變更）：**

```bash
# 確認是否有 PerformanceProfile 配置
oc get performanceprofile
```

若有 PerformanceProfile，需了解 4.20 中 Receive Packet Steering (RPS) 預設禁用的影響，評估是否需要在升級後重新啟用。

**Gateway API 用戶（CRD 版本確認）：**

由於 Ingress Operator 自 4.19 起管理 Gateway API CRD 的生命週期，若叢集中已有 Gateway API CRD，需確認其版本符合 OCP 4.20 要求（Gateway API Standard v1.2.1）。[1]

```bash
# 確認現有 Gateway API CRD
oc get crd | grep -F -e gateway.networking.k8s.io -e gateway.networking.x-k8s.io
```

**Kernel Module Management (KMM) 用戶：**

若叢集中使用了 KMM Modules，需在升級前執行 Preflight Validation，確認核心模組在升級後的核心版本上仍可安裝。[1]

```bash
# 取得目標版本的 driver-toolkit 映像
oc adm release info quay.io/openshift-release-dev/ocp-release:4.20.0-x86_64 \
  --image-for=driver-toolkit
```

---

### A4. 確認 Operator 相容性

所有通過 OLM 安裝的 Operator 必須更新至與目標版本（4.20）相容的版本，以確保在叢集升級後 default software catalogs 切換時有有效的更新路徑。[1]

```bash
# 列出所有已安裝的 Operator 及其版本
oc get csv -A

# 確認 Operator 訂閱狀態
oc get subscription -A
```

對於每個 Operator，需查閱其官方文件確認對 OCP 4.20 的支援狀態。特別注意：**Operator SDK 已在 4.19 中移除**，若有依賴 Operator SDK 的自訂 Operator 開發流程需提前調整。

---

## 階段 B：遷移與修正

### B1. 遷移已移除的 Kubernetes API

根據階段 A2 的評估結果，對所有使用 `admissionregistration.k8s.io/v1beta1` API 的資源進行遷移。詳細的遷移差異請參考 [Kubernetes Deprecated API Migration Guide](https://kubernetes.io/docs/reference/using-api/deprecation-guide/)。[1]

**遷移 MutatingWebhookConfiguration：**

```bash
# 列出所有使用 v1beta1 的 MutatingWebhookConfiguration
oc get mutatingwebhookconfigurations -o yaml | grep "apiVersion"

# 將 apiVersion 從 admissionregistration.k8s.io/v1beta1 改為 admissionregistration.k8s.io/v1
# 並確認 webhooks[].admissionReviewVersions 欄位包含 "v1"
```

**遷移 ValidatingWebhookConfiguration：**

```bash
# 列出所有使用 v1beta1 的 ValidatingWebhookConfiguration
oc get validatingwebhookconfigurations -o yaml | grep "apiVersion"
```

**遷移 ValidatingAdmissionPolicy 和 ValidatingAdmissionPolicyBinding：**

```bash
# 確認是否有使用 v1beta1 的 ValidatingAdmissionPolicy
oc get validatingadmissionpolicies -o yaml | grep "apiVersion"
oc get validatingadmissionpolicybindings -o yaml | grep "apiVersion"
```

> **重要**：遷移完成後，需重新確認 `oc get apirequestcounts` 的輸出，確保相關 API 的 `REQUESTSINCURRENTHOUR` 和 `REQUESTSINLAST24H` 均降至 0（排除可忽略的系統服務帳號）。

---

### B2. MetalLB FRR-K8s 命名空間遷移（如適用）

若叢集中安裝了 MetalLB Operator 且有自訂 `FRRConfiguration` CR，需在升級**後**（但建議提前規劃）將其從 `metallb-system` 遷移至 `openshift-frr-k8s` 命名空間。[1]

```bash
# 步驟 1：建立新命名空間
oc create namespace openshift-frr-k8s

# 步驟 2：執行遷移（將非 metallb- 前綴的 FRRConfiguration 遷移至新命名空間）
OLD_NAMESPACE="metallb-system"
NEW_NAMESPACE="openshift-frr-k8s"
FILTER_OUT="metallb-"

oc get frrconfigurations.frrk8s.metallb.io -n "${OLD_NAMESPACE}" -o json | \
  jq -r '.items[] | select(.metadata.name | test("'"${FILTER_OUT}"'") | not)' | \
  jq -r '.metadata.namespace = "'"${NEW_NAMESPACE}"'"' | \
  oc create -f -

# 步驟 3：驗證遷移結果
oc get frrconfigurations.frrk8s.metallb.io -n openshift-frr-k8s

# 步驟 4：確認遷移成功後，移除舊命名空間中的 CR
# oc delete frrconfigurations.frrk8s.metallb.io -n metallb-system <name>
```

---

### B3. MachineOSConfig 命名修正（如適用）

若叢集使用 on-cluster image mode（`MachineOSConfig`），需確認每個 `MachineOSConfig` 物件的名稱與其目標 machine config pool 的名稱**完全相同**。[1]

```bash
# 列出所有 MachineOSConfig 及其名稱
oc get machineosconfig

# 列出所有 MachineConfigPool 及其名稱
oc get mcp

# 若名稱不一致，需重新建立 MachineOSConfig（使用正確名稱）
```

---

### B4. Cloud Credential 更新（使用手動維護憑證的叢集）

若叢集使用手動維護的 Cloud Credentials（CCO manual mode），`Upgradeable` 狀態預設為 `False`，必須在升級前完成以下步驟。此步驟**不適用於** RHOSP 和 VMware vSphere 叢集（這些平台自動處理 cloud provider 資源變更）。[1]

**步驟 1：確認 CCO 模式**

```bash
# 通過 CLI 確認 CCO 模式
oc get cloudcredential cluster -o=jsonpath={.spec.credentialsMode}
```

**步驟 2：提取目標版本的 CredentialsRequest（以 AWS 為例）**

```bash
# 設定目標版本映像變數
RELEASE_IMAGE=$(oc get clusterversion -o jsonpath={..desired.image})
NEW_RELEASE_IMAGE="quay.io/openshift-release-dev/ocp-release:4.20.0-x86_64"

# 提取 CredentialsRequest
oc adm release extract \
  --credentials-requests \
  --cloud=aws \
  --to=<credentials_requests_directory> \
  ${NEW_RELEASE_IMAGE}
```

**步驟 3：使用 ccoctl 更新 Cloud Provider 資源**

```bash
# 提取 ccoctl 工具
CCO_IMAGE=$(oc adm release info --image-for='cloud-credential-operator' ${NEW_RELEASE_IMAGE} -a ~/.pull-secret)
oc image extract ${CCO_IMAGE} --file="/usr/bin/ccoctl.rhel9" -a ~/.pull-secret
chmod 775 ccoctl.rhel9 && mv ccoctl.rhel9 ccoctl

# 提取 bound service account signing key
mkdir -p <output_dir>
oc get secret bound-service-account-signing-key \
  -n openshift-kube-apiserver \
  -ojsonpath='{ .data.service-account\.pub }' | base64 -d > <output_dir>/serviceaccount-signer.public

# 處理 CredentialsRequest（AWS 範例）
./ccoctl aws create-all \
  --name=<cluster_name> \
  --region=<aws_region> \
  --credentials-requests-dir=<credentials_requests_directory> \
  --output-dir=<output_dir> \
  --public-key-file=<output_dir>/serviceaccount-signer.public
```

**步驟 4：標注叢集已準備好升級**

```bash
oc edit cloudcredential cluster
# 在 metadata.annotations 中加入：
# cloudcredential.openshift.io/upgradeable-to: "4.20.0"
```

---

## 階段 C：升級前最終確認

### C1. 備份 etcd（強制）

etcd 備份是升級前的必要步驟，在升級失敗的災難情境下，etcd 備份是恢復叢集的最後手段。[1]

```bash
# 登入任一 control plane 節點執行備份
# 方法一：通過 debug pod
oc debug node/<control_plane_node_name>
chroot /host
/usr/local/bin/cluster-backup.sh /home/core/assets/backup

# 方法二：直接在 control plane 節點上執行
ssh core@<control_plane_node>
sudo /usr/local/bin/cluster-backup.sh /home/core/assets/backup

# 確認備份文件已生成
ls -la /home/core/assets/backup/
```

> **注意**：etcd 備份僅作為最後手段，Red Hat **不支援**將叢集回滾至先前版本。若升級失敗，應聯繫 Red Hat Support。

---

### C2. 暫停 MachineHealthCheck 資源

升級過程中節點可能暫時不可用，`MachineHealthCheck` 可能誤判這些節點為不健康並嘗試重啟，干擾升級流程。必須在升級前暫停所有 MHC 資源。[1]

```bash
# 列出所有 MachineHealthCheck
oc get machinehealthcheck -n openshift-machine-api

# 暫停每個 MHC（替換 <mhc_name> 為實際名稱）
oc -n openshift-machine-api annotate mhc <mhc_name> cluster.x-k8s.io/paused=""

# 確認所有 MHC 已暫停
oc get machinehealthcheck -n openshift-machine-api -o yaml | grep "paused"
```

> **重要**：升級完成後必須恢復 MHC，執行以下指令移除暫停標注：
> ```bash
> oc -n openshift-machine-api annotate mhc <mhc-name> cluster.x-k8s.io/paused-
> ```

---

### C3. 提供管理員確認（解除升級阻擋）

這是 4.19 → 4.20 升級的**唯一強制步驟**，若未執行此步驟，升級程序將被阻擋無法進行。在確認已完成所有 API 遷移後，執行以下指令提供管理員確認。[1]

```bash
oc -n openshift-config patch cm admin-acks \
  --patch '{"data":{"ack-4.19-admissionregistration-v1beta1-api-removals-in-4.20":"true"}}' \
  --type=merge
```

**驗證確認已成功：**

```bash
oc get cm admin-acks -n openshift-config -o yaml
# 確認輸出中包含：
# ack-4.19-admissionregistration-v1beta1-api-removals-in-4.20: "true"
```

> **警告**：提供此確認代表管理員已完全負責確保所有已移除 API 的使用情況均已評估並遷移完畢。OCP 無法識別所有可能的使用情況，特別是閒置的 workloads 或外部工具。

---

### C4. 切換升級 Channel

將叢集的升級 channel 切換至目標版本的 channel。對於生產叢集，建議使用 `stable-4.20` channel；若需要更快取得更新，可使用 `fast-4.20`。[1]

```bash
# 切換至 stable-4.20 channel（生產環境建議）
oc adm upgrade channel stable-4.20

# 確認 channel 已切換
oc get clusterversion -o jsonpath='{.items[0].spec.channel}'

# 查看可用的推薦升級路徑
oc adm upgrade recommend
```

> **建議**：盡早切換 channel，讓 OSUS（OpenShift Update Service）有足夠時間評估並提供最佳升級路徑建議。

---

## 階段 D：執行升級

### D1. 執行升級指令

完成所有前置準備後，執行升級指令。**強烈建議**使用 OSUS 推薦的版本，避免使用 `--force` 旗標。[1]

```bash
# 方法一：升級至最新推薦版本
oc adm upgrade --to-latest=true

# 方法二：升級至指定版本（推薦，使用 oc adm upgrade recommend 輸出的版本號）
oc adm upgrade --to=4.20.x

# 監控升級進度（使用新的 GA 工具）
watch oc adm upgrade status

# 或使用傳統方式監控
watch oc get clusterversion
watch oc get clusteroperators
```

**升級進度說明：**

`oc adm upgrade status` 輸出包含三個區段：
- **Control Plane**：顯示 control plane 升級進度、Cluster Operators 狀態和預估完成時間
- **Worker Upgrade**：顯示各 worker pool 的節點升級進度
- **Update Health**：顯示升級相關的健康洞察

**升級完成後的驗證：**

```bash
# 確認叢集版本已更新至 4.20
oc adm upgrade

# 確認所有節點已更新至 Kubernetes 1.33
oc get nodes

# 確認所有 Cluster Operators 均正常
oc get clusteroperators

# 恢復之前暫停的 MachineHealthCheck
oc -n openshift-machine-api annotate mhc <mhc-name> cluster.x-k8s.io/paused-
```

---

## 升級後注意事項

升級完成後，需評估以下 4.20 新行為對現有 workloads 的影響：

**Linux User Namespace（預設可用，需主動啟用）**：Linux user namespace 支援現已 GA，但需管理員主動啟用。啟用後，使用 NFS 後端 PersistentVolume 的 Pod 可能因 ID-mapped mounts 不相容而出現存取問題。建議先在非生產環境驗證後再啟用。[1]

**RPS 預設禁用**：若叢集使用 `PerformanceProfile`，RPS 不再自動配置。若有延遲敏感的網絡 workloads 受到影響，可在 `PerformanceProfile` 中添加 `performance.openshift.io/enable-rps: "enable"` 標注重新啟用，但需了解此操作會全局降低所有 Pod 的網絡效能。[1]

**Prometheus 3.5.0**：監控堆疊已升級至 Prometheus 3.5.0，若有自訂的 alerting rules 或 recording rules，需確認其相容性（Red Hat 不保證向後相容性）。`AlertmanagerClusterFailedToSendAlerts` 告警的評估視窗已從 5m 延長至 15m。[1]

**MachineOSConfig 命名規則**：若使用 on-cluster image mode，確認 `MachineOSConfig` 物件名稱已與 machine config pool 名稱一致（階段 B3 已處理）。[1]

---

## 快速參考：關鍵指令清單

| 步驟 | 指令 | 說明 |
|---|---|---|
| 叢集健康 | `oc get clusteroperators` | 確認所有 CO 正常 |
| 升級預檢 | `oc adm upgrade recommend` | 查看推薦升級路徑 |
| API 使用檢查 | `oc get apirequestcounts` | 識別已移除 API 使用情況 |
| 暫停 MHC | `oc -n openshift-machine-api annotate mhc <name> cluster.x-k8s.io/paused=""` | 防止節點誤重啟 |
| **管理員確認** | `oc -n openshift-config patch cm admin-acks --patch '{"data":{"ack-4.19-admissionregistration-v1beta1-api-removals-in-4.20":"true"}}' --type=merge` | **解除升級阻擋（必做）** |
| 切換 Channel | `oc adm upgrade channel stable-4.20` | 切換至目標 channel |
| 執行升級 | `oc adm upgrade --to=4.20.x` | 啟動升級 |
| 監控升級 | `watch oc adm upgrade status` | 即時監控升級進度 |
| 恢復 MHC | `oc -n openshift-machine-api annotate mhc <name> cluster.x-k8s.io/paused-` | 升級完成後恢復 |

---

## 參考資料

[1]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/updating_clusters/preparing-to-update-a-cluster "Chapter 2. Preparing to update a cluster | OCP 4.20 | Red Hat Documentation"
[2]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/updating_clusters/performing-a-cluster-update "Chapter 3. Performing a cluster update | OCP 4.20 | Red Hat Documentation"
[3]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/release_notes/ocp-4-20-release-notes "Chapter 1. OpenShift Container Platform 4.20 Release Notes | Red Hat Documentation"
[4]: https://kubernetes.io/docs/reference/using-api/deprecation-guide/ "Kubernetes Deprecated API Migration Guide"
[5]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html-single/backup_and_restore/ "Backup and Restore | OCP 4.20 | Red Hat Documentation"
