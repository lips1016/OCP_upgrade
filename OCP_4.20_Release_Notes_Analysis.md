# OpenShift Container Platform 4.20 版本說明深度分析

> **來源文件**：[OCP 4.20 Release Notes](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/release_notes/ocp-4-20-release-notes)（發布日期：2026-06-30）  
> **適用對象**：已完成 OCP 4.19 升級，計劃升級至 4.20 的叢集管理員  
> **核心版本**：Kubernetes **1.33**，CRI-O runtime，RHSA-2025:9562

---

## 目錄

1. [4.19 → 4.20 升級注意事項（必讀）](#1-419--420-升級注意事項必讀)
2. [新功能與增強](#2-新功能與增強)
3. [Technology Preview 升級為 GA](#3-technology-preview-升級為-ga)
4. [已棄用與已移除功能](#4-已棄用與已移除功能)
5. [已知問題](#5-已知問題)
6. [參考資料](#6-參考資料)

---

## 1. 4.19 → 4.20 升級注意事項（必讀）

### 1.1 強制管理員確認（最高優先）

在 OCP 4.20 中，一個在 4.17 版本無意重新引入的 Kubernetes API 再次被正式移除。**所有 4.19 叢集在升級至 4.20 之前，管理員必須完成手動確認程序**，否則升級程序將被阻止。[1]

管理員需依序完成以下三個步驟：

1. 評估叢集中是否有 workloads、工具或自動化腳本仍依賴即將移除的 API 版本
2. 將所有受影響的 manifests、workloads 及 API clients 遷移至支援的 API 版本
3. 在叢集中提供管理員確認，以解除升級阻擋

可使用 `oc adm upgrade recommend` 指令（現已 GA）執行升級前預檢，識別潛在問題後再啟動升級。

### 1.2 必須在升級前完成的 Kubernetes API 遷移

以下四個 `admissionregistration.k8s.io/v1beta1` API 在 4.20 中被正式移除，**必須在升級前完成遷移**，否則相關 workloads 將失效。[1]

| 資源類型 | 已移除 API | 遷移目標 API |
|---|---|---|
| `MutatingWebhookConfiguration` | `admissionregistration.k8s.io/v1beta1` | `admissionregistration.k8s.io/v1` |
| `ValidatingAdmissionPolicy` | `admissionregistration.k8s.io/v1beta1` | `admissionregistration.k8s.io/v1` |
| `ValidatingAdmissionPolicyBinding` | `admissionregistration.k8s.io/v1beta1` | `admissionregistration.k8s.io/v1` |
| `ValidatingWebhookConfiguration` | `admissionregistration.k8s.io/v1beta1` | `admissionregistration.k8s.io/v1` |

可參考 [Kubernetes 官方棄用指南](https://kubernetes.io/docs/reference/using-api/deprecation-guide/) 了解各 API 的遷移差異。

### 1.3 VMware vSphere 7 / VCF 4 終止支援

Broadcom 已終止 VMware vSphere 7 和 VMware Cloud Foundation (VCF) 4 的一般支援。若現有叢集運行於這些平台，**必須計劃將 VMware 基礎架構遷移或升級**至 vSphere 8 Update 1 或更高版本，或 VCF 5 或更高版本。[1]

### 1.4 BGP FRR-K8s 配置遷移（使用 MetalLB 的用戶）

4.20 中 Cluster Network Operator（CNO）原生支援 BGP routing，frr-k8s 的管理命名空間從 `metallb-system` 遷移至 `openshift-frr-k8s`。若叢集中已安裝 MetalLB Operator 並有自訂 frr-k8s 配置，**必須手動執行遷移腳本**，將 `FRRConfiguration` CR 移至新命名空間。[1]

```bash
# 建立新命名空間
oc create namespace openshift-frr-k8s

# 執行遷移腳本（將 metallb-system 中非 metallb- 前綴的 FRRConfiguration 遷移）
OLD_NAMESPACE="metallb-system"
NEW_NAMESPACE="openshift-frr-k8s"
FILTER_OUT="metallb-"
oc get frrconfigurations.frrk8s.metallb.io -n "${OLD_NAMESPACE}" -o json | \
  jq -r '.items[] | select(.metadata.name | test("'"${FILTER_OUT}"'") | not)' | \
  jq -r '.metadata.namespace = "'"${NEW_NAMESPACE}"'"' | \
  oc create -f -
```

### 1.5 MachineOSConfig 命名規則變更

使用 on-cluster image mode 的用戶需注意：`MachineOSConfig` 物件的名稱現在**必須與目標 machine config pool 的名稱相同**。之前可使用任意名稱的做法已不再支援，以防止多個 `MachineOSConfig` 物件嘗試使用同一個 machine config pool。[1]

### 1.6 Receive Packet Steering (RPS) 預設禁用

使用 `PerformanceProfile` 的叢集需注意：RPS 不再在套用 Performance Profile 時自動配置。此變更影響在延遲敏感執行緒中直接執行網絡系統呼叫的容器。若需恢復舊行為，可在 `PerformanceProfile` manifest 中添加 `performance.openshift.io/enable-rps: "enable"` 標注，但代價是全局降低所有 Pod 的網絡效能。[1]

### 1.7 Linux User Namespace 啟用後的 NFS 相容性問題

Linux user namespace 在 4.20 中已 GA 並預設可用。**升級本身不影響現有叢集**，但一旦管理員明確啟用 user namespace 功能後，使用 NFS 後端 PersistentVolume 的 Pod 可能因 ID-mapped mounts 不相容而出現存取或權限問題。這是 Kubernetes 1.33 及以上版本的通用限制，並非 OCP 特有問題。[1]

### 1.8 oc-mirror v2 新增前置驗證

`oc-mirror` v2 現在在開始填充緩存和執行 mirroring 操作之前，會先驗證 registry credentials、DNS 名稱和 SSL 憑證。這可防止用戶在緩存已填充後才發現憑證問題，但可能影響現有的自動化腳本流程。[1]

---

## 2. 新功能與增強

### 2.1 核心平台

OCP 4.20 升級至 **Kubernetes 1.33**，帶來多項底層改進。API Server 的 self-signed loopback 憑證有效期從一年延長至**三年**，降低因憑證過期導致服務中斷的風險。此外，`oc delete istag --dry-run=server` 的誤刪問題已修復，`--dry-run` 選項現在正確連接至指令。[1]

**etcd** 方面新增多層級的 `etcdDatabaseQuotaLowSpace` 告警（`info`、`warning`、`critical`），讓管理員能更提前感知 etcd 配額不足的風險。etcd 文件亦更新，新增對 TLS 1.3 的支援說明。

### 2.2 監控堆疊組件版本更新

4.20 對監控堆疊進行了全面版本更新，具體如下：[1]

| 組件 | 版本 |
|---|---|
| Prometheus | **3.5.0** |
| Prometheus Operator | 0.85.0 |
| Metrics Server | 0.8.0 |
| Thanos | 0.39.2 |
| kube-state-metrics | 2.16.0 |
| prom-label-proxy | 0.12.0 |

值得注意的是，`AlertmanagerClusterFailedToSendAlerts` 告警的評估時間視窗從 `5m` 延長至 `15m`，以減少誤報。Red Hat 不保證 recording rules 和 alerting rules 的向後相容性，使用自訂告警規則的用戶需留意此變更。

### 2.3 網絡功能

**BGP Routing 原生支援**是 4.20 網絡領域最重要的新功能。CNO 現在直接支援啟用 Border Gateway Protocol (BGP) routing，通過 `FRRConfiguration` CR 進行管理，支援路由匯入/匯出、多宿主、鏈路冗餘和快速收斂。配合此功能，**BGP unnumbered peering** 亦從 TP 升級為 GA。[1]

**SR-IOV 網絡管理** 現在可在應用程式命名空間中直接進行，無需叢集管理員介入。用戶可在自己的命名空間中創建和管理 `SriovNetwork` 物件，提升自主性並改善安全隔離。

**Gateway API** 方面，新增對 Gateway API Inference Extension 的支援（通過 OSSM 3.1.0），並更新 Istio 至 1.26.2。需注意 Gateway API 目前仍不支援 on-premise 平台（Bare Metal、vSphere、Nutanix 等），且效能低於 HAProxy-based Ingress Controller。

此外，**br-ex bridge 遷移至 NMState** 現已 GA，允許將安裝時通過 `configure-ovs.sh` 設定的 br-ex bridge 遷移至 NMState 管理。

### 2.4 安裝與基礎架構

**VMware vSphere 多 NIC 安裝**從 TP 升級為 GA，支援為節點配置多個網絡介面控制器。**VMware vSphere Foundation 9 和 VCF 9** 現已獲得正式支援。

**Bare Metal** 方面，新增對 Dell iDRAC10 的 Redfish virtual media 支援，以及多架構支援（可從現有 x86_64 叢集通過 virtual media 部署 aarch64 節點）。

**Azure** 方面新增多項功能：Intel TDX Confidential VMs 支援、虛擬網絡加密支援，以及 etcd 專用磁碟（TP）。

### 2.5 Machine Config Operator

**On-cluster image mode** 獲得多項改進：以下配置變更不再觸發節點重啟，顯著減少維護窗口需求：修改 `/var` 或 `/etc` 目錄中的配置文件、添加或修改 systemd 服務、更改 SSH 密鑰、移除 ICSP/ITMS/IDMS 物件的 mirroring 規則，以及更新 `user-ca-bundle` configmap。[1]

**MachineConfigNode** 狀態報告現已 GA，可監控自訂 machine config pool 的更新進度。新增 `ImageBuildDegraded` 狀態欄位，以及 `oc describe mcp` 的增強輸出。

新增 `hostmount-anyuid-v2` SCC，提供與 `hostmount-anyuid` 相同的功能，但包含 `seLinuxContext: RunAsAny`，允許以任意 UID（包括 UID 0）存取主機文件系統，作為 `privileged` SCC 的替代選項。

### 2.6 節點與安全

**Linux user namespace** 現已 GA 並預設可用，包含兩個新 SCC：`restricted-v3` 和 `nested-container`，專為 user namespace 設計。此功能可緩解被攻陷容器對其他 Pod 和節點的多種安全威脅。[1]

**sigstore** 支援現已 GA（`config.openshift.io/v1` API 版本），支援 `ClusterImagePolicy` 和 `ImagePolicy` 物件，並新增 BYOPKI（Bring Your Own PKI）映像驗證支援。

**In-place Pod Resizing** 允許在不重建或重啟 Pod 的情況下，動態調整運行中容器的 CPU 和 Memory 資源。

### 2.7 儲存

多項儲存功能在 4.20 中從 TP 升級為 GA：**Volume Populators**（支援預填充 PVC）、**Azure Disk Performance Plus**（513 GiB 以上磁碟的 IOPS/吞吐量提升）、**AWS EFS One Zone**（DNS 解析失敗時可回退至 mount targets）、**Always Honor PV Reclaim Policy**（確保 PV 刪除順序不影響 reclaim policy 的執行），以及 **Manila CSI 多 CIDR 支援**。[1]

### 2.8 OpenShift CLI (oc)

`oc adm upgrade recommend` 和 `oc adm upgrade status` 兩個指令均從 TP 升級為 GA，為管理員提供升級前預檢和升級狀態監控的官方支援工具。

### 2.9 RHCOS 更新

`kdump` 現已在所有支援架構（x86_64、arm64、s390x、ppc64le）上 GA，便於診斷和解決核心崩潰問題。Ignition 更新至 2.20.0，新增對 Proxmox Virtual Environment 的支援。Butane 更新至 0.23.0，`coreos-installer` 更新至 0.23.0。[1]

### 2.10 效能與可擴展性

**NUMA Resources Operator** 現在預設啟用 HA 模式，為每個 control plane 節點創建一個 scheduler 副本（最多 3 個）。新增對可調度 control plane 節點的支援，適用於資源受限的緊湊型叢集。

**Hitless TLS 憑證輪換**確保 Kubernetes API 的 TLS 憑證輪換期間 95% 的叢集可用性，對高事務率叢集和 SNO 部署尤為重要。

---

## 3. Technology Preview 升級為 GA

以下功能在 4.20 中從 Technology Preview 升級為正式 GA，可在生產環境中使用：[1]

| 功能 | 升級版本 | 備注 |
|---|---|---|
| VMware vSphere 多 NIC 安裝 | 4.20 | — |
| Linux user namespace | 4.20 | 預設啟用，注意 NFS 相容性 |
| sigstore ClusterImagePolicy/ImagePolicy | 4.20 | API: config.openshift.io/v1 |
| `oc adm upgrade recommend` | 4.20 | — |
| `oc adm upgrade status` | 4.20 | 不支援 HCP 叢集 |
| 本地 arbiter 節點（2 control plane + 1 arbiter） | 4.20 | — |
| Volume Populators | 4.20 | — |
| Azure Disk Performance Plus | 4.20 | — |
| AWS EFS One Zone | 4.20 | — |
| Always Honor PV Reclaim Policy | 4.20 | — |
| Manila CSI 多 CIDR | 4.20 | — |
| MachineConfigNode 狀態報告 | 4.20 | — |
| PinnedImageSet | 4.19.12+ | 早期 4.19.x 仍為 TP |
| BGP unnumbered peering | 4.20 | — |
| kdump | 4.20 | 所有架構 |
| Direct OIDC 外部身份提供者認證 | 4.20.5+ | 早期 4.20.z 仍為 TP |
| br-ex bridge 遷移至 NMState | 4.20 | — |

---

## 4. 已棄用與已移除功能

### 4.1 4.20 新增棄用

以下功能在 4.20 中**新增棄用**，計劃在未來版本中移除：[1]

| 功能 | 棄用版本 | 建議替代方案 |
|---|---|---|
| AMD SEV on Google Cloud | 4.20 | AMD SEV-SNP |
| Docker v2 registries | 4.20 | OCI 規範相容 registry |
| Red Hat Marketplace | 4.20 | Red Hat Ecosystem Catalog |
| Red Hat Quay Container Security Operator | 4.20 | Red Hat Advanced Cluster Security for Kubernetes |

### 4.2 持續棄用（4.18 起）

以下功能持續處於棄用狀態，尚未移除：

| 功能 | 棄用起始版本 |
|---|---|
| iptables | 4.18 |
| `ImageContentSourcePolicy` (ICSP) | 4.18 |
| `DeploymentConfig` | 4.18 |
| Cluster Samples Operator | 4.18 |
| oc-mirror plugin v1 | 4.18 |
| SQLite database format for Operator catalogs | 4.18 |

### 4.3 4.20 確認已移除（4.19 已移除）

以下功能在 4.19 中已移除，4.20 確認不再提供：

| 功能 | 移除版本 |
|---|---|
| Package-based RHEL compute machines | 4.19 |
| cgroup v1 | 4.19 |
| Patternfly 4 | 4.19 |
| Operator SDK | 4.19 |
| Scaffolding tools for Ansible/Helm/Go-based Operator projects | 4.19 |

---

## 5. 已知問題

以下為 4.20 的主要已知問題，管理員在升級前應評估影響：[1]

**Gateway API 相關（多項）**：Gateway API 在 AWS、GCP、Azure 私有叢集中，load balancer 始終配置為外部，無支援的解決方案。Gateway API 目前不支援 on-premise 平台（Bare Metal、vSphere、Nutanix、KubeVirt、None platform type），因為這些平台沒有預設的 cloud controller manager 分配外部 IP。`HTTPRoute` 不支援 Passthrough 或 Re-encrypt TLS 終止。Gateway API 的效能（基於 Istio/Envoy）目前低於 HAProxy-based Ingress Controller，效能優化工作仍在進行中。

**Linux User Namespace + NFS**：啟用 user namespace 後，使用 NFS 後端 PV 的 Pod 可能因 ID-mapped mounts 不相容而失敗。此為 Kubernetes 1.33 的通用限制，升級本身不觸發，僅在明確啟用 user namespace 後生效。

**NUMA topo-aware-scheduler**：不支援 Kubernetes priority-based preemption。當所有 NUMA zone 被低優先級 Pod 佔滿時，高優先級 Pod 會無限期停留在 `Pending` 狀態。

**Azure 安裝**：若將 `identity.type` 設為 `None`，叢集無法從 Azure Container Registry 拉取映像。建議提供 user-assigned identity 或留空此欄位。

**SR-IOV + TuneD race condition**：在配置了 SR-IOV VF 的叢集中，節點重啟後 TuneD 服務可能因 race condition 導致 profile 降級，影響效能。解決方案是重啟 TuneD Pod。

**Guaranteed QoS + static CPU Manager**：在 `full-pcpus-only` 配置且大多數 CPU 已分配的節點上，節點重啟後 Guaranteed QoS Pod 可能不自動重啟，需手動刪除並重建。

**OSSM 2.x 與 3.x 共存**：提供 Gateway API 實現的 OSSM 3.x 無法與 OSSM 2.x 在同一叢集共存（除官方遷移程序外），可能導致 Ingress Operator 報告 `degraded` 狀態。

---

## 6. 參考資料

[1]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/release_notes/ocp-4-20-release-notes "Chapter 1. OpenShift Container Platform 4.20 Release Notes | Red Hat Documentation"
[2]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/release_notes/index "Release Notes Index | OCP 4.20 | Red Hat Documentation"
[3]: https://kubernetes.io/docs/reference/using-api/deprecation-guide/ "Kubernetes API Deprecation Guide"
[4]: https://access.redhat.com/errata/RHSA-2025:9562 "RHSA-2025:9562 - Red Hat Security Advisory"
[5]: https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.33.md "Kubernetes 1.33 Changelog"
