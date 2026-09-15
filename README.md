# MDE AIDE prerequisite implementation records

This repository contains review records for Kubernetes prerequisites prepared for
NetApp MDE AIDE deployments. It intentionally excludes kubeconfigs, access
tokens, certificates, internal IP addresses, and other credentials.

## Records

- [ntap — Enterprise Small prerequisite implementation](docs/ntap-enterprise-small-prerequisites.md)

## Current status

The Kubernetes and node prerequisites documented in the record were applied to
the three dedicated `ntap` worker nodes, including the Ubuntu NFS client. The
designated NFS export now mounts and passes a per-worker write test as UID:GID
`4321:4321`.
