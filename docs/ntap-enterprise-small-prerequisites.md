# `ntap` MDE AIDE Enterprise Small — prerequisite implementation record

**Date:** 2026-09-15  
**Scope:** `ntap` RKE2 cluster, three dedicated worker nodes  
**Target:** MDE AIDE Enterprise Small deployment  
**Status:** Partially complete — NFS client prerequisite remains blocked

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

## Remaining blocker: NFS client support

An NFS validation DaemonSet was scheduled on all three dedicated workers and
attempted to mount the designated MDE configuration export. Each pod failed
before reaching the NFS server because the Ubuntu 24.04 host OS did not have
the `mount.nfs` helper installed.

| Item | Result |
| --- | --- |
| Export accessibility | Not yet validated; mount could not start locally. |
| Cause | Missing `mount.nfs` helper on every worker. |
| Required package | `nfs-common` (Ubuntu 24.04). |
| Affected nodes | All three dedicated MDE workers. |
| Unaffected prerequisites | Namespace, labels, sysctls, `sunrpc`, and host directories were applied successfully. |

## Next approved remediation and verification

1. Install `nfs-common` on each of the three dedicated MDE worker nodes.
2. Re-run a temporary NFS validation workload.
3. Confirm the export mounts read/write on every worker and that a process
   running as UID:GID `4321:4321` can create and remove a test file.
4. Remove the validation workload.
5. Record the final validation result and any network/export-permission gap.

The NFS server address and export path are intentionally redacted from this
public repository. They should be supplied through the protected deployment
configuration or internal runbook.
