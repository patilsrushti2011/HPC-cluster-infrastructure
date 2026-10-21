# HPC Mini-Cluster Configuration Reference

## Overview

This document describes the main configuration examples maintained in this repository for the HPC mini-cluster running Rocky Linux.

## 1. SLURM

**File:** `configs/slurm/slurm.conf.example`

SLURM manages compute resources and schedules jobs across the cluster.

The documented configuration includes:
- Cluster controller settings
- Compute node definitions
- CPU and memory resources
- Partitions: `low`, `high`, and `hmnode`
- Scheduling and accounting settings

See [`configs/slurm/README.md`](../configs/slurm/README.md).

## 2. NFS

**File:** `configs/nfs/exports.example`

NFS provides shared directories across configured systems.

The documented exports include:
- `/home`
- `/apps`
- `/var/spool/slurmctld`
- `/tftpboot`
- `/install`

Review client access rules and permissions before using these settings in another environment.

See [`configs/nfs/README.md`](../configs/nfs/README.md).

## 3. LDAP

**Files:**
- `configs/ldap/nslcd.conf.example`
- `configs/ldap/nsswitch.conf.example`

These files document LDAP client settings and the system's user, group, and host lookup order.

The documented LDAP search base is `dc=hpc,dc=local`.

See [`configs/ldap/README.md`](../configs/ldap/README.md).

## 4. xCAT

**File:** `configs/xcat/node-definitions.example.txt`

This reference documents selected xCAT node attributes, operating system provisioning settings, and post-provisioning scripts.

See [`configs/xcat/README.md`](../configs/xcat/README.md).

## Important Notes

- These files are documentation examples, not a complete deployable cluster configuration.
- Verify hostnames, IP addresses, node resources, and service settings before applying changes.
- Review NFS export permissions and access restrictions.
- Do not publish passwords, private keys, or other secrets.
- Grafana is part of the project environment, but its configuration is not included in this reference because a Grafana configuration file was not supplied.
