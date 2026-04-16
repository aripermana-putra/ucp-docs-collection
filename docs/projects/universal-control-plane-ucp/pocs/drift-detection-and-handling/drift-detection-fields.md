---
title: "Drift Detection — Fields Reference"
space: UCP
parent_page_id: "../drift-detection-and-handling.md"
---

# Drift Detection — Fields Reference

For each managed resource type, this page lists fields from both `spec.forProvider`
(desired state you declared) and `status.atProvider` (actual state GCP observed), and
whether each field should be included in drift comparison.

**Legend**

| Mark | Meaning |
|------|---------|
| ✅ | Yes |
| — | Not present on this side |
| `forProvider` ✅ | Field is currently declared in the POC resource |
| `forProvider` — | Field is not declared in the current POC resource |
| `atProvider` ✅ | Field observed in live `kubectl get` output |
| `atProvider` ⬜ | Field not verified from live data (no live resource available) |
| `Writable?` ✅ | Field exists in the CRD `forProvider` schema — user **can** declare it |
| `Writable?` — | Field is NOT in the CRD `forProvider` schema — GCP-assigned, cannot be set by user |
| `Compare?` ✅ | Include in drift comparison — drift here is meaningful |
| `Compare?` ❌ | Exclude — metadata, lifecycle, credentials, or one-time bootstrap |
| `Compare?` ⚠️ | Known normalization gap — will always show as drift (see notes) |
| `Compare?` ⬜ | TBD — pending decision |

> **Why `forProvider` is sparse:** `forProvider` is only what you explicitly declared as
> desired state in the current POC. Fields with `forProvider` — but `Writable?` ✅ are
> supported by the provider and can be added to forProvider at any time — they simply
> aren't declared yet in the current POC setup.
>
> Fields with `Writable?` — exist only in `atProvider` because GCP assigns them at
> provisioning time (IPs, IDs, self-links). There is no desired baseline to compare against.

> **Data sources:** `forProvider` and `atProvider` columns verified from live `kubectl get`
> and `kubectl get crd` output for CloudSQL, GCE, GKE Cluster, GKE NodePool, and GCS Bucket.
> OmniaDatabase had no live resources — its columns are based on provider implementation.

---

## 1. CloudSQL DatabaseInstance
**GVR:** `sql.gcp.upbound.io/v1beta2/databaseinstances`
**Composition:** `crossplane/composition/dbaas/cloudsql/xdatabase-cloudsql.yaml`

| Field | forProvider | atProvider | Writable? | Compare? | Notes |
|-------|:-----------:|:----------:|:---------:|----------|-------|
| `databaseVersion` | ✅ | ✅ | ✅ | ✅ | Version change is meaningful drift |
| `region` | ✅ | ✅ | ✅ | ✅ | Cannot change post-creation but worth detecting |
| `project` | ✅ | ✅ | ✅ | ❌ | Metadata |
| `deletionProtection` | ✅ | ✅ | ✅ | ❌ | Lifecycle setting |
| `maintenanceVersion` | — | ✅ | ✅ | ⬜ | Pin a specific DB engine version; drift means auto-update occurred |
| `masterInstanceName` | — | ✅ | ✅ | ❌ | Replica source — immutable after creation |
| `settings.tier` | ✅ | ✅ | ✅ | ✅ | Core sizing — most common drift target |
| `settings.diskSize` | ✅ | ✅ | ✅ | ✅ | Can be enlarged manually in GCP console |
| `settings.diskType` | ✅ | ✅ | ✅ | ✅ | Storage type change is significant |
| `settings.diskAutoresize` | ✅ | ✅ | ✅ | ✅ | Auto-grow policy — can be toggled in console |
| `settings.diskAutoresizeLimit` | — | ✅ | ✅ | ✅ | Reducing the cap limits cost control |
| `settings.edition` | — | ✅ | ✅ | ✅ | ENTERPRISE vs ENTERPRISE_PLUS has feature/cost impact |
| `settings.userLabels` | ✅ | ✅ | ✅ | ❌ | Label-only changes are not critical operational drift (or we can check only label "ucp-managed" for changes) |
| `settings.activationPolicy` | — | ✅ | ✅ | ✅ | ALWAYS vs ON_DEMAND is billing/availability difference |
| `settings.availabilityType` | — | ✅ | ✅ | ✅ | ZONAL vs REGIONAL — high availability difference |
| `settings.collation` | — | ✅ | ✅ | ❌ | Set at creation, cannot practically change post-creation |
| `settings.connectorEnforcement` | — | ✅ | ✅ | ✅ | REQUIRED → NOT_REQUIRED is a security regression |
| `settings.deletionProtectionEnabled` | — | ✅ | ✅ | ❌ | Redundant with top-level `deletionProtection` |
| `settings.enableGoogleMlIntegration` | — | ✅ | ✅ | ❌ | Minor optional feature flag |
| `settings.locationPreference` | — | ✅ | ✅ | ❌ | Zone preference — operational detail |
| `settings.pricingPlan` | — | ✅ | ✅ | ❌ | PER_USE is the only effective option now |
| `settings.timeZone` | — | ✅ | ✅ | ✅ | SQL Server only — changing timezone affects query results |
| `settings.backupConfiguration.enabled` | — | ✅ | ✅ | ✅ | Disabling backups is a critical risk |
| `settings.backupConfiguration.binaryLogEnabled` | — | ✅ | ✅ | ✅ | Affects point-in-time recovery capability |
| `settings.backupConfiguration.pointInTimeRecoveryEnabled` | — | ✅ | ✅ | ✅ | Disabling removes PITR capability |
| `settings.backupConfiguration.startTime` | — | ✅ | ✅ | ❌ | Backup window scheduling detail |
| `settings.backupConfiguration.transactionLogRetentionDays` | — | ✅ | ✅ | ✅ | Reducing limits PITR recovery window |
| `settings.backupConfiguration.backupRetentionSettings` | — | ✅ | ✅ | ✅ | Reducing backup count limits recovery options |
| `settings.ipConfiguration.ipv4Enabled` | ✅ | ✅ | ✅ | ✅ | Public IP toggle is a security-relevant change |
| `settings.ipConfiguration.privateNetwork` | — | ✅ | ✅ | ✅ | VPC peering network — changing is significant |
| `settings.ipConfiguration.allocatedIpRange` | — | ✅ | ✅ | ✅ | Changing IP range name affects private IP assignment |
| `settings.ipConfiguration.enablePrivatePathForGoogleCloudServices` | — | ✅ | ✅ | ✅ | Affects connectivity from GCP services |
| `settings.ipConfiguration.requireSsl` | — | ✅ | ✅ | ✅ | Deprecated but writable — disabling is security regression |
| `settings.ipConfiguration.sslMode` | — | ✅ | ✅ | ✅ | Downgrading SSL mode is a security regression |
| `settings.version` | — | ✅ | — | ❌ | GCP-internal settings change counter — not in CRD |
| `connectionName` | — | ✅ | — | ❌ | GCP-assigned `project:region:instance` — not in CRD |
| `dnsName` | — | ✅ | — | ❌ | GCP-assigned DNS name — not in CRD |
| `firstIpAddress` | — | ✅ | — | ❌ | GCP-assigned first IP — not in CRD |
| `id` | — | ✅ | — | ❌ | GCP-assigned resource ID — not in CRD |
| `instanceType` | — | ✅ | — | ❌ | GCP-assigned instance classification — not in CRD |
| `privateIpAddress` | — | ✅ | — | ❌ | GCP-assigned private IP — not in CRD |
| `pscServiceAttachmentLink` | — | ✅ | — | ❌ | GCP-assigned PSC link — not in CRD |
| `publicIpAddress` | — | ✅ | — | ❌ | GCP-assigned public IP — not in CRD |
| `selfLink` | — | ✅ | — | ❌ | GCP resource URL — not in CRD |
| `serviceAccountEmailAddress` | — | ✅ | — | ❌ | GCP-assigned service account — not in CRD |

---

## 2. GCE Compute Instance
**GVR:** `compute.gcp.upbound.io/v1beta1/instances`
**Composition:** `crossplane/composition/compute/gcp/xcomputeinstance-gce.yaml`

> Note: `bootDisk`, `shieldedInstanceConfig`, and `scheduling` are nested **objects** in
> `atProvider` (not arrays). Only `networkInterface` is an array.

| Field | forProvider | atProvider | Writable? | Compare? | Notes |
|-------|:-----------:|:----------:|:---------:|----------|-------|
| `machineType` | ✅ | ✅ | ✅ | ✅ | Primary sizing — most common change target |
| `zone` | ✅ | ✅ | ✅ | ✅ | Cannot change post-creation but worth detecting |
| `project` | ✅ | ✅ | ✅ | ❌ | Metadata |
| `canIpForward` | — | ✅ | ✅ | ✅ | Enabling IP forwarding is a security-relevant change |
| `deletionProtection` | — | ✅ | ✅ | ❌ | Lifecycle setting |
| `description` | — | ✅ | ✅ | ❌ | Cosmetic — not operational drift |
| `enableDisplay` | — | ✅ | ✅ | ❌ | Minor display feature |
| `hostname` | — | ✅ | ✅ | ❌ | Immutable after creation |
| `labels` | — | — | ✅ | ❌ | GCP resource labels — not set on current POC instances |
| `tags` | — | — | ✅ | ✅ | Network tags control firewall rules — not set on current POC instances |
| `bootDisk.initializeParams.image` | ✅ | ✅ | ✅ | ⚠️ | GCP resolves image family to versioned URL — always shows as drift |
| `bootDisk.initializeParams.size` | ✅ | ✅ | ✅ | ✅ | Disk size can be enlarged in console |
| `bootDisk.initializeParams.type` | — | ✅ | ✅ | ❌ | Disk type — set at creation, immutable |
| `bootDisk.initializeParams.enableConfidentialCompute` | — | ✅ | ✅ | ✅ | Security posture |
| `bootDisk.initializeParams.provisionedIops` | — | ✅ | ✅ | ✅ | Performance setting — changing affects I/O throughput |
| `bootDisk.initializeParams.provisionedThroughput` | — | ✅ | ✅ | ✅ | Performance setting |
| `bootDisk.autoDelete` | — | ✅ | ✅ | ✅ | Disabling means disk persists after deletion |
| `bootDisk.deviceName` | — | ✅ | ✅ | ❌ | Cosmetic — set at creation |
| `bootDisk.kmsKeySelfLink` | — | ✅ | ✅ | ❌ | Encryption key — set at creation, immutable |
| `bootDisk.mode` | — | ✅ | ✅ | ❌ | READ_WRITE is the only practical value — immutable |
| `bootDisk.source` | — | ✅ | ✅ | ❌ | Full disk URL — set at creation |
| `bootDisk.diskEncryptionKeySha256` | — | ✅ | — | ❌ | GCP-computed hash of encryption key — not in CRD |
| `shieldedInstanceConfig.enableSecureBoot` | ✅ | ✅ | ✅ | ✅ | Security posture |
| `shieldedInstanceConfig.enableVtpm` | ✅ | ✅ | ✅ | ✅ | Security posture |
| `shieldedInstanceConfig.enableIntegrityMonitoring` | ✅ | ✅ | ✅ | ✅ | Security posture |
| `scheduling.automaticRestart` | ✅ | ✅ | ✅ | ✅ | Disabling auto-restart changes availability behaviour |
| `scheduling.onHostMaintenance` | ✅ | ✅ | ✅ | ✅ | Live migration policy change is significant |
| `scheduling.provisioningModel` | ✅ | ✅ | ✅ | ✅ | STANDARD vs SPOT is a cost/availability change |
| `scheduling.preemptible` | — | ✅ | ✅ | ✅ | Writable; cost/availability significant |
| `scheduling.instanceTerminationAction` | — | ✅ | ✅ | ✅ | Affects SPOT instance termination behavior |
| `scheduling.minNodeCpus` | — | ✅ | ✅ | ❌ | Sole-tenancy CPU pinning — niche use, usually 0 |
| `networkInterface[0].network` | ✅ | ✅ | ✅ | ✅ | URL normalization handled (`default` → full URL suffix) |
| `networkInterface[0].accessConfig[0].networkTier` | ✅ | ✅ | ✅ | ✅ | Tier downgrade from console is meaningful |
| `networkInterface[0].networkIp` | — | ✅ | ✅ | ❌ | Writable for static IP but GCP assigns via DHCP if not set — no meaningful baseline unless explicitly declared |
| `networkInterface[0].subnetwork` | — | ✅ | ✅ | ❌ | Writable but GCP resolves to full URL — exclude to avoid false positives |
| `networkInterface[0].subnetworkProject` | — | ✅ | ✅ | ❌ | Writable but resolved by GCP |
| `networkInterface[0].stackType` | — | ✅ | ✅ | ✅ | IPV4_ONLY vs DUAL_STACK is significant |
| `networkInterface[0].nicType` | — | ✅ | ✅ | ❌ | Immutable after creation |
| `networkInterface[0].queueCount` | — | ✅ | ✅ | ❌ | Performance tuning — usually 0 |
| `networkInterface[0].internalIpv6PrefixLength` | — | ✅ | ✅ | ❌ | IPv6 dual-stack setting — usually 0 |
| `networkInterface[0].ipv6Address` | — | ✅ | ✅ | ❌ | GCP-assigned for dual-stack |
| `networkInterface[0].name` | — | ✅ | — | ❌ | GCP-assigned NIC name (`nic0`) — not in CRD |
| `networkInterface[0].accessConfig[0].natIp` | — | ✅ | — | ❌ | GCP-assigned external IP — not in CRD |
| `networkInterface[0].accessConfig[0].publicPtrDomainName` | — | ✅ | — | ❌ | GCP-assigned reverse DNS — not in CRD |
| `cpuPlatform` | — | ✅ | — | ❌ | GCP-assigned CPU platform string — not in CRD |
| `currentStatus` | — | ✅ | — | ❌ | Operational status (`RUNNING`) — not in CRD |
| `id` | — | ✅ | — | ❌ | GCP-assigned numeric instance ID — not in CRD |
| `instanceId` | — | ✅ | — | ❌ | GCP-assigned instance ID — not in CRD |
| `labelFingerprint` | — | ✅ | — | ❌ | GCP-assigned label etag — not in CRD |
| `metadataFingerprint` | — | ✅ | — | ❌ | GCP-assigned metadata etag — not in CRD |
| `selfLink` | — | ✅ | — | ❌ | GCP resource URL — not in CRD |
| `tagsFingerprint` | — | ✅ | — | ❌ | GCP-assigned tags etag — not in CRD |

---

## 3. GKE Cluster
**GVR:** `container.gcp.upbound.io/v1beta2/clusters`
**Composition:** `crossplane/composition/kubernetes/gke/xkubernetescluster-gke.yaml`

| Field | forProvider | atProvider | Writable? | Compare? | Notes |
|-------|:-----------:|:----------:|:---------:|----------|-------|
| `location` | ✅ | ✅ | ✅ | ✅ | Cannot change post-creation but worth detecting |
| `project` | ✅ | ✅ | ✅ | ❌ | Metadata |
| `deletionProtection` | ✅ | ✅ | ✅ | ❌ | Lifecycle setting |
| `description` | — | ✅ | ✅ | ❌ | Cosmetic |
| `resourceLabels` | — | — | ✅ | ❌ | GCP resource labels — not set on current POC cluster |
| `releaseChannel.channel` | ✅ | ✅ | ✅ | ✅ | Channel change from console has operational impact |
| `removeDefaultNodePool` | ✅ | ✅ | ✅ | ❌ | Bootstrap flag — GCP echoes it back but has no ongoing meaning |
| `initialNodeCount` | ✅ | ✅ | ✅ | ❌ | Bootstrap-only; actual node count managed by NodePool |
| `loggingService` | ✅ | ✅ | ✅ | ✅ | Logging destination change has observability impact |
| `loggingConfig.enableComponents` | ✅ | — | ✅ | ❌ | Composition declares `[]`; GCP omits the field entirely when empty — watcher treats empty desired slice as no-diff |
| `monitoringService` | ✅ | ✅ | ✅ | ✅ | Monitoring destination change has observability impact |
| `monitoringConfig.enableComponents` | ✅ | ✅ | ✅ | ✅ | Removing a component reduces observability |
| `monitoringConfig.managedPrometheus.enabled` | ✅ | ✅ | ✅ | ✅ | Disabling removes managed metrics collection |
| `network` | ✅ | ✅ | ✅ | ✅ | URL normalization handled |
| `subnetwork` | ✅ | ✅ | ✅ | ✅ | URL normalization handled |
| `networkingMode` | ✅ | ✅ | ✅ | ✅ | VPC_NATIVE is required for many features |
| `networkPolicy.enabled` | — | ✅ | ✅ | ✅ | Disabling network policy is a security regression |
| `networkPolicy.provider` | — | ✅ | ✅ | ❌ | Usually set once at creation |
| `nodeLocations` | ✅ | ✅ | ✅ | ✅ | Zone removal reduces availability |
| `nodeConfig.shieldedInstanceConfig.enableSecureBoot` | ✅ | ✅ | ✅ | ✅ | Security posture for system node pool |
| `nodeConfig.imageType` | — | ✅ | ✅ | ⬜ | Writable — changing OS image type has workload impact |
| `nodeConfig.diskType` | — | ✅ | ✅ | ✅ | Performance/cost impact |
| `nodeConfig.diskSizeGb` | — | ✅ | ✅ | ✅ | Node disk size |
| `nodeConfig.machineType` | — | ✅ | ✅ | ✅ | System node sizing |
| `nodeConfig.oauthScopes` | — | ✅ | ✅ | ✅ | Permission scope reduction could break workloads |
| `addonsConfig.gcePersistentDiskCsiDriverConfig.enabled` | ✅ | ✅ | ✅ | ✅ | Disabling breaks PVC provisioning |
| `clusterAutoscaling.autoscalingProfile` | ✅ | ✅ | ✅ | ✅ | BALANCED vs OPTIMIZE_UTILIZATION has cost/availability impact |
| `clusterAutoscaling.enabled` | — | ✅ | ✅ | ✅ | Disabling node auto-provisioning is significant |
| `enableShieldedNodes` | — | ✅ | ✅ | ✅ | Security posture — disabling weakens node integrity |
| `enableIntranodeVisibility` | — | ✅ | ✅ | ✅ | Affects intra-node networking observability |
| `enableLegacyAbac` | — | ✅ | ✅ | ✅ | Enabling ABAC is a security regression |
| `enableCiliumClusterwideNetworkPolicy` | — | ✅ | ✅ | ✅ | Security policy change |
| `enableL4IlbSubsetting` | — | ✅ | ✅ | ✅ | Affects load balancer behaviour |
| `datapathProvider` | — | ✅ | ✅ | ✅ | DATAPLANE_V1 vs ADVANCED_DATAPLANE is a networking change |
| `securityPostureConfig.mode` | ✅ | ✅ | ✅ | ✅ | Security posture mode change |
| `masterAuth.clientCertificateConfig.issueClientCertificate` | ✅ | ✅ | ✅ | ✅ | Enabling client certs is a security regression |
| `privateClusterConfig.masterGlobalAccessConfig.enabled` | ✅ | ✅ | ✅ | ✅ | Control plane access scope change |
| `privateClusterConfig.enablePrivateEndpoint` | — | ✅ | ✅ | ✅ | Disabling private endpoint exposes control plane |
| `privateClusterConfig.enablePrivateNodes` | — | ✅ | ✅ | ✅ | Disabling private nodes is a network exposure change |
| `notificationConfig.pubsub.enabled` | ✅ | ✅ | ✅ | ✅ | Disabling breaks upgrade notifications |
| `serviceExternalIpsConfig.enabled` | ✅ | ✅ | ✅ | ✅ | Enabling allows external IP on Services — security-relevant |
| `nodeVersion` | — | ✅ | ✅ | ⬜ | Writable (target version); tracking against expected channel version is useful |
| `clusterIpv4Cidr` | — | ✅ | ✅ | ❌ | Writable but immutable after creation |
| `defaultMaxPodsPerNode` | — | ✅ | ✅ | ❌ | Writable but immutable after creation |
| `enableAutopilot` | — | ✅ | ✅ | ❌ | Immutable after creation |
| `enableKubernetesAlpha` | — | ✅ | ✅ | ❌ | Immutable after creation |
| `enableTpu` | — | ✅ | ✅ | ❌ | Niche feature, set at creation |
| `ipAllocationPolicy` | — | ✅ | ✅ | ❌ | Writable at creation but immutable post-creation |
| `privateIpv6GoogleAccess` | — | ✅ | ✅ | ❌ | Niche IPv6 feature |
| `addonsConfig.dnsCacheConfig.enabled` | — | ✅ | — | ❌ | GCP default (true) — not in CRD |
| `addonsConfig.networkPolicyConfig.disabled` | — | ✅ | — | ❌ | GCP default — not in CRD |
| `endpoint` | — | ✅ | — | ❌ | GCP-assigned K8s API server IP — not in CRD |
| `id` | — | ✅ | — | ❌ | GCP-assigned full resource path — not in CRD |
| `labelFingerprint` | — | ✅ | — | ❌ | GCP-assigned label etag — not in CRD |
| `masterAuth.clusterCaCertificate` | — | ✅ | — | ❌ | GCP-managed CA cert — not in CRD forProvider |
| `masterVersion` | — | ✅ | — | ⬜ | Running control plane version — output only; use `minMasterVersion` to set |
| `selfLink` | — | ✅ | — | ❌ | GCP resource URL — not in CRD |
| `servicesIpv4Cidr` | — | ✅ | — | ❌ | GCP-assigned services CIDR — not in CRD directly |
| `tpuIpv4CidrBlock` | — | ✅ | — | ❌ | GCP default (empty) — not in CRD |

---

## 4. GKE NodePool
**GVR:** `container.gcp.upbound.io/v1beta2/nodepools`
**Composition:** `crossplane/composition/kubernetes/gke/xkubernetescluster-gke.yaml`

| Field | forProvider | atProvider | Writable? | Compare? | Notes |
|-------|:-----------:|:----------:|:---------:|----------|-------|
| `location` | ✅ | ✅ | ✅ | ✅ | |
| `project` | ✅ | ✅ | ✅ | ❌ | Metadata |
| `cluster` | ✅ | ✅ | ✅ | ❌ | Reference field — immutable after creation |
| `clusterRef` | ✅ | — | ✅ | ❌ | Crossplane cross-resource reference — locates the parent Cluster MR; not a GCP field, never in atProvider |
| `clusterSelector` | ✅ | — | ✅ | ❌ | Crossplane cross-resource selector — alternative to clusterRef; not a GCP field, never in atProvider |
| `initialNodeCount` | ✅ | ✅ | ✅ | ✅ | Manual scale-down in console is drift |
| `autoscaling.minNodeCount` | ✅ | ✅ | ✅ | ✅ | Lower bound change affects availability |
| `autoscaling.maxNodeCount` | ✅ | ✅ | ✅ | ✅ | Upper bound change affects cost |
| `autoscaling.totalMinNodeCount` | — | ✅ | ✅ | ✅ | Cross-zone minimum — lower bound affects availability |
| `autoscaling.totalMaxNodeCount` | — | ✅ | ✅ | ✅ | Cross-zone maximum — upper bound affects cost |
| `autoscaling.locationPolicy` | — | ✅ | ✅ | ❌ | GCP-managed zone balancing — rarely a meaningful drift target |
| `management.autoRepair` | ✅ | ✅ | ✅ | ✅ | Disabling is a reliability risk |
| `management.autoUpgrade` | ✅ | ✅ | ✅ | ✅ | Disabling is a security risk |
| `nodeCount` | — | ✅ | ✅ | ❌ | Writable but fluctuates with autoscaling — not a meaningful drift target |
| `version` | — | ✅ | ✅ | ⬜ | Writable (target version); useful to track against expected channel version |
| `nodeConfig.machineType` | ✅ | ✅ | ✅ | ✅ | Node sizing — primary cost/performance attribute |
| `nodeConfig.diskSizeGb` | ✅ | ✅ | ✅ | ✅ | Node disk size |
| `nodeConfig.diskType` | — | ✅ | ✅ | ✅ | Disk type change affects performance and cost |
| `nodeConfig.imageType` | — | ✅ | ✅ | ⬜ | Actual OS image (`COS_CONTAINERD`) — writable and potentially worth tracking |
| `nodeConfig.oauthScopes` | ✅ | ✅ | ✅ | ✅ | Permission scope reduction could break workloads |
| `nodeConfig.preemptible` | — | ✅ | ✅ | ✅ | Cost/availability significant — set at pool creation |
| `nodeConfig.spot` | — | ✅ | ✅ | ✅ | SPOT vs standard is a significant cost/availability change |
| `nodeConfig.serviceAccount` | — | ✅ | ✅ | ✅ | Changing service account changes node permissions |
| `nodeConfig.loggingVariant` | — | ✅ | ✅ | ❌ | DEFAULT vs MAX — niche observability tuning |
| `nodeConfig.shieldedInstanceConfig.enableSecureBoot` | ✅ | ✅ | ✅ | ✅ | Security posture |
| `nodeConfig.shieldedInstanceConfig.enableIntegrityMonitoring` | — | ✅ | ✅ | ✅ | Security posture — disabling weakens boot attestation |
| `nodeConfig.kubeletConfig.cpuManagerPolicy` | — | ✅ | ✅ | ✅ | CPU pinning policy has workload isolation impact |
| `nodeConfig.kubeletConfig.cpuCfsQuota` | — | ✅ | ✅ | ✅ | Affects container CPU throttling behavior |
| `nodeConfig.kubeletConfig.podPidsLimit` | — | ✅ | ✅ | ✅ | Affects container process isolation |
| `nodeConfig.kubeletConfig.cpuCfsQuotaPeriod` | — | ✅ | ✅ | ❌ | Fine-grained throttling period — rarely changed |
| `nodeConfig.metadata` | — | ✅ | ✅ | ❌ | GCP-injected metadata (`disable-legacy-endpoints`) — not user drift |
| `nodeConfig.resourceLabels` | — | ✅ | ✅ | ❌ | GCP-managed provisioning model labels — not user drift |
| `nodeLocations` | ✅ | ✅ | ✅ | ✅ | Zone removal reduces availability |
| `maxPodsPerNode` | ✅ | ✅ | ✅ | ❌ | Cannot change post-creation |
| `upgradeSettings.maxSurge` | ✅ | ✅ | ✅ | ✅ | Reducing surge capacity slows safe rollouts |
| `upgradeSettings.maxUnavailable` | — | ✅ | ✅ | ✅ | Increasing disrupts running workloads during upgrades |
| `upgradeSettings.strategy` | ✅ | ✅ | ✅ | ✅ | Changing from SURGE affects rollout behaviour |
| `networkConfig.enablePrivateNodes` | — | ✅ | ✅ | ✅ | Disabling private nodes is a network exposure change |
| `networkConfig.createPodRange` | — | ✅ | ✅ | ❌ | Set at creation |
| `id` | — | ✅ | — | ❌ | GCP-assigned full resource path — not in CRD |
| `instanceGroupUrls` | — | ✅ | — | ❌ | GCP-assigned MIG URLs per zone — not in CRD |
| `managedInstanceGroupUrls` | — | ✅ | — | ❌ | GCP-assigned instance group URLs — not in CRD |

---

## 5. GCS Bucket
**GVR:** `storage.gcp.upbound.io/v1beta2/buckets`
**Composition:** `crossplane/composition/objectstorage/gcs/xobjectstorage-gcs.yaml`

| Field | forProvider | atProvider | Writable? | Compare? | Notes |
|-------|:-----------:|:----------:|:---------:|----------|-------|
| `location` | ✅ | ✅ | ✅ | ✅ | Cannot change post-creation |
| `project` | ✅ | ✅ | ✅ | ❌ | Metadata |
| `labels` | — | — | ✅ | ❌ | GCP resource labels — not set on current POC bucket |
| `storageClass` | ✅ | ✅ | ✅ | ✅ | Class change affects cost and retrieval latency |
| `uniformBucketLevelAccess` | ✅ | ✅ | ✅ | ✅ | Disabling weakens IAM consistency |
| `publicAccessPrevention` | ✅ | ✅ | ✅ | ✅ | `enforced` → `inherited` is a security regression |
| `versioning.enabled` | ✅ | ✅ | ✅ | ✅ | Disabling on a versioned bucket can cause data loss risk |
| `rpo` | ✅ | ✅ | ✅ | ✅ | Replication policy — `ASYNC_TURBO` → `DEFAULT` is a durability regression |
| `softDeletePolicy.retentionDurationSeconds` | ✅ | ✅ | ✅ | ✅ | Reducing soft-delete window shortens recovery time |
| `defaultEventBasedHold` | — | ✅ | ✅ | ✅ | Enabling default hold affects lifecycle of all new objects |
| `enableObjectRetention` | — | ✅ | ✅ | ✅ | Enabling affects compliance-level data retention |
| `requesterPays` | — | ✅ | ✅ | ✅ | Changing shifts billing responsibility to the requester |
| `forceDestroy` | — | ✅ | ✅ | ❌ | Crossplane operational flag — not a GCP native attribute |
| `softDeletePolicy.effectiveTime` | — | ✅ | — | ❌ | GCP-assigned timestamp when policy took effect — not in CRD |
| `id` | — | ✅ | — | ❌ | GCP-assigned bucket name — not in CRD |
| `projectNumber` | — | ✅ | — | ❌ | GCP-assigned numeric project ID — not in CRD |
| `selfLink` | — | ✅ | — | ❌ | GCP resource URL — not in CRD |
| `url` | — | ✅ | — | ❌ | `gs://bucket-name` URL — not in CRD |

---

## 6. OmniaDatabase
**GVR:** `omnia.roc.rakuten.com/v1alpha1/omniadatabases`
**Composition:** `crossplane/composition/dbaas/omnia/xdatabase-omnia.yaml`

> atProvider fields depend on what provider-roc exposes. The `omniadatabases` resource type
> was not registered on the current cluster — fields are based on provider implementation
> and marked ⬜. Verify against actual resource data when available:
> `kubectl get omniadatabases -o json | jq '.status.atProvider'`

| Field | forProvider | atProvider | Writable? | Compare? | Notes |
|-------|:-----------:|:----------:|:---------:|----------|-------|
| `serviceName` | ✅ | ⬜ | ✅ | ❌ | Omnia service identifier — static metadata |
| `deploymentName` | ✅ | ⬜ | ✅ | ❌ | Identifier |
| `middleware` | ✅ | ⬜ | ✅ | ✅ | Database type — changing this is significant |
| `middlewareVersion` | ✅ | ⬜ | ✅ | ✅ | Version drift is a key detection target |
| `tenantId` | ✅ | ⬜ | ✅ | ❌ | Identity metadata |
| `tenantName` | ✅ | ⬜ | ✅ | ❌ | Identity metadata |
| `authToken` | ✅ | — | ✅ | ❌ | Credential — changes on every refresh, never stable |
| `backupTime` | ✅ | ⬜ | ✅ | ✅ | Backup window change is an operational concern |
| `fintech` | ✅ | ⬜ | ✅ | ✅ | Compliance flag — toggling has audit implications |
| `clusterEnvironment` | ✅ | ⬜ | ✅ | ✅ | Environment tier change is significant |
| `topology.offeringName` | ✅ | ⬜ | ✅ | ✅ | Cluster topology type |
| `topology.nodes[*].role` | ✅ | ⬜ | ✅ | ⬜ | TBD — verify atProvider exposes this |
| `topology.nodes[*].instanceType` | ✅ | ⬜ | ✅ | ✅ | Node sizing |
| `topology.nodes[*].cpuCores` | ✅ | ⬜ | ✅ | ✅ | Sizing |
| `topology.nodes[*].memoryGib` | ✅ | ⬜ | ✅ | ✅ | Sizing |
| `topology.nodes[*].storageGib` | ✅ | ⬜ | ✅ | ✅ | Disk sizing |
| `topology.nodes[*].storageType` | ✅ | ⬜ | ✅ | ✅ | Storage tier |
| `topology.nodes[*].dc` | ✅ | ⬜ | ✅ | ✅ | Placement |
| `topology.nodes[*].region` | ✅ | ⬜ | ✅ | ✅ | Placement |
| `topology.nodes[*].vlanId` | ✅ | ⬜ | ✅ | ✅ | Network placement |
| `configuration` | ✅ | — | ✅ | ❌ | Complex templated config — too dynamic for field-level drift |
| `deploymentId` | — | ⬜ | ⬜ | ❌ | Omnia-assigned internal ID |
| `state` | — | ⬜ | ⬜ | ❌ | Deployment state — operational status, not drift |
| `endpoints` | — | ⬜ | ⬜ | ⬜ | Actual connection endpoints — useful to verify connectivity |

---

## Known Normalization Gaps

These fields will always show as drift even when nothing was changed:

| Resource | Field | Desired (short) | Actual (normalized by GCP) |
|----------|-------|-----------------|---------------------------|
| GCE Instance | `bootDisk.initializeParams.image` | `debian-cloud/debian-12` | `https://.../debian-12-bookworm-v20260413` |

**Resolution options:**
1. Pin a full image URL in the composition — eliminates the gap but requires manual updates on image refresh
2. Mark `bootDisk.initializeParams.image` as ❌ — image family drift is rarely user-caused
3. Accept it — only alert on `machineType`, security config, network, etc.

Fields where normalization is already handled by the URL suffix check in the watcher:

| Resource | Field | Example |
|----------|-------|---------|
| GCE Instance | `networkInterface[0].network` | `default` → `https://.../networks/default` |
| GKE Cluster | `network` | `default` → `https://.../networks/default` |
| GKE Cluster | `subnetwork` | `default` → `https://.../subnetworks/default` |
