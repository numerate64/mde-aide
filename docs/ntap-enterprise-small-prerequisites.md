# `ntap` MDE AIDE Enterprise Small — prerequisite implementation record

**Date:** 2026-09-15  
**Scope:** `ntap` RKE2 cluster, three dedicated worker nodes  
**Target:** MDE AIDE Enterprise Small deployment  
**Status:** Prerequisites validated — NFS read/write test passed on all three workers

## Scope and safety

This is an implementation record, not a deployment manifest. It excludes
kubeconfig material, tokens, certificates, and internal endpoint addresses.
Temporary privileged DaemonSets were used only to apply and validate approved
node-level prerequisites, then removed.

## Initial preflight results

| Check | Result | Effect |
| --- | --- | --- |
| Kubernetes distribution/version | RKE2 `v1.36.4+rke2r1` | Meets the documented Kubernetes/RKE2 compatibility target. |
| Node health | Five of five nodes were `Ready`; three dedicated, untainted workers were available | Supports the three-node HA placement baseline. |
| Worker capacity | Three workers, each 32 vCPU / approximately 130.6 GiB allocatable memory | Meets the Enterprise Small compute baseline; does **not** meet Medium or Large sizing. |
| Existing requested workload capacity | Negligible worker requests; no pending pods | No observed scheduling contention during preflight. |
| StorageClass | `vsphere-csi-sc`, vSphere CSI, backed by `vsphere-asa-flash`, expansion enabled | Block-backed flash StorageClass is suitable in type. Storage performance was confirmed by the infrastructure owner to meet the Small profile requirement (8,000 IOPS / 1,000 Mb/s). |
| RBAC | Namespace, PVC, Secret, Deployment, PV creation and node patch permissions available | Authorized the scoped prerequisite work. |

## Changes applied

### 1. Created the deployment namespace

Created the `aide` namespace.

**Effect:** establishes the intended namespace boundary for the MDE AIDE
deployment and its supporting resources.

### 2. Applied MDE placement labels to the dedicated workers

The following labels were applied to each of the three dedicated `ntap` worker
nodes:

```text
3rdparty-aide-es=true
3rdparty-aide-filebeat=true
3rdparty-aide-k8s-event-exporter=true
3rdparty-aide-kafka=true
3rdparty-aide-logstash=true
3rdparty-aide-nginx=true
3rdparty-aide-postgres=true
3rdparty-aide-rabbitmq=true
3rdparty-aide-redis=true
3rdparty-aide-telegraf=true
microservices-aide=true
```

**Effect:** MDE component scheduling can target all three dedicated workers,
allowing the required HA placement pattern for Elasticsearch, Kafka,
PostgreSQL, RabbitMQ, Redis, and the remaining MDE services.

### 3. Configured node prerequisites

Using a temporary privileged DaemonSet scheduled only to the three dedicated
workers, the following documented prerequisites were applied and persisted:

- Required kernel/sysctl settings, including `vm.max_map_count=262144`
- `sunrpc` module loading and persistence
- Required AIDE host data and log directories
- Ownership of those directories set to UID:GID `4321:4321`
- Directory mode set to `0775`

**Effect:** the workers have the kernel, RPC, filesystem path, and service
account ownership prerequisites required by MDE workloads. Persistence was
configured so these settings survive node restart.

### 4. Removed temporary elevated workloads

Both the host-configuration and NFS-validation DaemonSets were removed after
their work and validation attempts completed.

**Effect:** no temporary privileged DaemonSets remain running in the cluster.

## Validation evidence

Post-change node inspection confirmed all three dedicated workers remained
`Ready` and retained every label listed above. The `aide` namespace exists.
The temporary host-configuration workload completed on all three target
workers before cleanup.

## NFS client remediation

The missing NFS client dependency was remediated through a temporary privileged
DaemonSet restricted to the three MDE workers. It installed Ubuntu 24.04
package `nfs-common` on every worker and confirmed that the host `mount.nfs`
helper and NFS kernel filesystem support were available. The temporary
installer DaemonSet was removed after validation.

**Effect:** the worker-side NFS client prerequisite is complete.

## NFS final validation

After the NFS export was corrected, the validation DaemonSet was retried on
all three dedicated MDE workers. Every pod mounted the designated export and,
while running as UID:GID `4321:4321`, created, inspected, and removed a unique
test file. Each test file was owned by `4321:4321` with mode `0660`.

| Item | Result |
| --- | --- |
| Export accessibility | Mounted successfully from every dedicated MDE worker. |
| Worker client state | `nfs-common`, `mount.nfs`, and NFS kernel support are present on all three workers. |
| UID:GID validation | Passed as `4321:4321` on all three workers. |
| File validation | Create, inspect, and removal passed; test files had mode `0660`. |
| Cleanup | The temporary validation DaemonSet was deleted after evidence collection. |

## Next approved remediation and verification

No NFS remediation remains. Preserve the current export ownership and access
controls for the MDE service account, and re-run the validation after any NFS
server, export, or worker image change.

The NFS server address and export path are intentionally redacted from this
public repository. They should be supplied through the protected deployment
configuration or internal runbook.
