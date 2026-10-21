# SLURM Configuration

## Overview

SLURM (Simple Linux Utility for Resource Management) is the workload manager used in this HPC mini-cluster to allocate compute resources and schedule jobs across compute nodes.

## Configuration File

- `slurm.conf.example` — example SLURM configuration containing cluster settings, node definitions, partitions, scheduling, and accounting settings.

## Compute Nodes

| Node | CPUs | RealMemory (MB) |
|---|---:|---:|
| cn1 | 2 | 1707 |
| cn2 | 2 | 1707 |
| node01 | 2 | 3914 |
| node02 | 2 | 2971 |
| hmcn1 | 4 | 3913 |

## Partitions

| Partition | Nodes | Default |
|---|---|---|
| low | cn1, cn2 | No |
| high | node01, node02 | Yes |
| hmnode | hmcn1 | No |

## Important Settings

- `ClusterName` — identifies the SLURM cluster.
- `SlurmctldHost` — specifies the controller host configuration.
- `AuthType` — configures authentication using Munge.
- `SchedulerType` — selects the backfill scheduling plugin.
- `SelectType` — configures resource selection.
- `AccountingStorageType` — configures SLURM database accounting.

## Notes

This is a documentation example based on the project configuration. Verify controller hostnames, resource values, accounting services, and other settings before deploying it on another system.
