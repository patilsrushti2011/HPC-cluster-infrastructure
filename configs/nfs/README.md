# NFS Configuration

## Overview

Network File System (NFS) allows systems to access directories shared over a network.

## Configuration File

- `exports.example` — documents the directories exported by the HPC cluster server.

## Shared Directories

- `/home` — shared user home directories.
- `/apps` — shared application directory.
- `/var/spool/slurmctld` — SLURM controller state directory.
- `/tftpboot` — xCAT-related provisioning files.
- `/install` — installation and provisioning resources.

## Important Notes

The export rules are examples based on the project's configuration. Verify client networks, directory permissions, and access restrictions before applying them. Avoid broad client access and review `no_root_squash` carefully.
