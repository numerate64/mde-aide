# MDE AIDE prerequisite implementation records

This repository contains review records for Kubernetes prerequisites prepared for
NetApp MDE AIDE deployments. It intentionally excludes kubeconfigs, access
tokens, certificates, internal IP addresses, and other credentials.

## Records

- [ntap — Enterprise Small prerequisite implementation](docs/ntap-enterprise-small-prerequisites.md)

## Current status

The Kubernetes and node prerequisites documented in the record were applied to
the three dedicated `ntap` worker nodes. The remaining deployment blocker is
the missing Ubuntu `nfs-common` package on those workers; without it,
Kubernetes cannot mount the MDE NFS configuration export.
